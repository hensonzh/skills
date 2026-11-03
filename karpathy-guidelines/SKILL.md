---
name: karpathy-guidelines
description: Use when writing, reviewing, or refactoring code to keep changes focused, state consequential assumptions, and verify the requested behavior.
license: MIT
---

# Karpathy Guidelines

- Read the relevant code and conventions before editing; state assumptions that affect behavior or scope.
- Ask when ambiguity changes the outcome, authorization, or costly tradeoffs. Resolve routine implementation choices from existing patterns.
- Implement only the requested behavior. Add abstractions or configuration when current requirements justify them.
- Preserve unrelated code, comments, formatting, and local changes. Remove only unused code introduced by this task.
- Define an observable success condition. For a bug fix, reproduce the failure and verify the corrected behavior.
- Run checks appropriate to the change; add regression coverage when it protects meaningful behavior. Report failures and unverified cases accurately.
- For multi-step work, outline only the steps needed to reach and verify the result.

Based on [Andrej Karpathy's observations](https://x.com/karpathy/status/2015883857489522876).
