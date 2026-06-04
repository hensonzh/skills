---
name: momcozy-agent-architecture
description: Use when explaining, reviewing, extending, or refactoring the momcozy-agent intelligent-agent architecture, including Responses API loop design, service skill selection, request context, tool registry, session state, safety boundaries, and Momcozy service workflows.
---

# Momcozy Agent Architecture

This skill captures the architectural design of the `momcozy-agent` project. Use it when the user asks about the agent solution, wants to add a service workflow, changes tool exposure, adjusts context strategy, or reviews whether a code change fits the existing architecture.

## Core Model

The project uses OpenAI Responses API directly. It does not use the OpenAI Agents SDK and does not run an application-side router before the model call.

The central flow is:

```text
web/app.js
  -> POST /api/ag-ui
  -> server.py parses AG-UI payload into RuntimeInputs
  -> ChatSession restores previous_response_id, loaded_skill_ids, ContextState
  -> agents.py run_agent_loop()
  -> build_agent_request()
  -> client.responses.create()
  -> model emits text or function_call
  -> application executes registered tool handler
  -> function_call_output is sent back to Responses API
  -> text deltas and process events stream to frontend over SSE
```

## Key Files

- `src/momcozy_agent/agents.py`: Responses API request construction, tool loop, streaming event emission, safe tool event payloads.
- `src/momcozy_agent/server.py`: local HTTP server, `/api/ag-ui` SSE endpoint, session state, AG-UI payload conversion.
- `src/momcozy_agent/static_context.py`: stable agent instructions, safety policy, service skill manifest injection.
- `src/momcozy_agent/contexts.py`: lightweight per-turn `request_context` and `ContextState`.
- `src/momcozy_agent/skills.py`: service skill registry, manifest parsing, `load_skill`, `read_skill_file`, script allowlist.
- `src/momcozy_agent/tool_registry.py`: exposed tools, deferred namespaces, handler dispatch.
- `src/momcozy_agent/tool_schemas.py`: Responses API function schemas.
- `docs/agent/README.md`: source-of-truth architecture notes.
- `docs/web-agent-communication.md`: web protocol and event contract.

## Design Principles

Keep these design constraints intact unless the user explicitly asks for a redesign:

1. The model selects service workflows from the skill manifest. The app does not pre-route requests to a service skill.
2. Stable behavior lives in top-level Responses `instructions` to preserve prompt cache value.
3. Per-turn mutable information is sent as `request_context` plus `user_message` input parts.
4. The app stores session state by `threadId`: `previous_response_id`, `loaded_skill_ids`, and `ContextState`.
5. Skills are progressive context modules. Do not load every `skills/*/SKILL.md` into the initial prompt.
6. The model never reads arbitrary files or runs shell commands. Skill file access goes through `load_skill`, `search_skill_assets`, `read_skill_file`, and approved script handlers only.
7. Tool schemas are stable and conservative. Immediate tools are always available; specialized business tools sit behind deferred namespaces and `tool_search`.
8. Tool execution is application-side. The model proposes function calls; `tool_registry.py` dispatches to local handlers.
9. Safety is handled by stable instructions plus service skill safety limits, not by a separate risk-evaluation tool.
10. AG-UI events expose sanitized progress and artifacts, not raw user data or full tool arguments.

## Agent Request Shape

`build_agent_request()` produces a Responses request with:

```python
{
    "model": options.get("model", "gpt-5.5"),
    "instructions": STATIC_AGENT_INSTRUCTIONS,
    "input": [
        {
            "role": "user",
            "content": [
                {"type": "input_text", "text": request_context},
                {"type": "input_text", "text": "user_message:\n..."},
                {"type": "input_image", "image_url": "...", "detail": "auto"},
            ],
        }
    ],
    "tools": select_runtime_tools(),
    "tool_choice": "auto",
    "reasoning": {"effort": "medium"},
    "text": {"format": {"type": "text"}, "verbosity": "medium"},
    "store": True,
    "prompt_cache_key": "momcozy-agent-v2",
}
```

`previous_response_id` is added only when present, allowing Responses API to maintain multi-turn state.

## Service Skill Selection

The service skills live under project `skills/` and are exposed through the manifest assembled in `static_context.py`.

Current service skill ids:

- `birth-prep`
- `milk-management`
- `emotion-support`
- `device-guidance`

When adding or changing a service skill:

1. Update the skill directory and `SKILL.md` frontmatter.
2. Update `SkillId` in `src/momcozy_agent/types.py`.
3. Update `SERVICE_SKILL_IDS` in `src/momcozy_agent/skills.py`.
4. Add or adjust tool schemas only if the workflow has a real application-side handler.
5. Keep safety limits in the service skill frontmatter and detailed workflow instructions in the skill body or references.

## Context Strategy

`ContextState` tracks:

- `environment_sent`: whether locale and timezone have already been injected.
- `loaded_references`: references loaded through tool results, later summarized into `loaded_reference_context`.
- `client_events`: recent external page or app events, later summarized into `client_event_context`.

`build_request_context()` sends locale and timezone only once per session. It sends `message_sent_at` every turn because it belongs to the current user action.

Do not move large profile, baby, calendar, feeding, or service records into every prompt by default. Prefer read tools such as `profile_get` or future retrieval tools.

## Tool Exposure

`select_runtime_tools()` exposes:

- `tool_search`
- immediate tools: `profile_get`, skill runtime tools, `ui_form_create`, `ui_card_create`, `ibclc_consult_card_create`
- deferred namespaces: `care_handoffs`, `device_support`

Do not expose placeholder write tools until there is a real handler and confirmation path. If a tool performs side effects, make idempotency, confirmation, and safe frontend feedback explicit.

## Safety And Privacy

Maintain these boundaries:

- Do not diagnose medical, lactation, infant health, or mental health conditions.
- For emergency, red-flag, or uncertain risk, prioritize escalation guidance over service workflow completion.
- Do not stream complete tool arguments, raw records, or sensitive profile data to the browser.
- Use `safe_tool_arguments()` and `safe_tool_result()` patterns when adding tool events.
- Images are for the current user request. They are not diagnostic proof and should not become long-lived hidden state.

## Implementation Checklist

When changing architecture or adding a workflow:

1. Identify whether the need is direct answer, service skill, immediate tool, deferred tool, or frontend artifact.
2. Keep the initial Responses request cache-friendly.
3. Add the smallest tool surface with a real handler.
4. Make session state explicit in `ChatSession` or `ContextState` only when it must survive turns.
5. Add AG-UI events only through sanitized payloads.
6. Update `docs/agent/README.md` and `docs/web-agent-communication.md` if the protocol or architecture changes.
7. Add focused tests around request construction, skill loading, tool dispatch, and SSE event order when behavior changes.
