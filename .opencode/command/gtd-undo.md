---
description: "Safe git revert. Roll back phase or plan commits using the phase manifest with dependency checks."
argument-hint: "--last N | --phase NN | --plan NN-MM"
requires: [phase]
tools:
  read: true
  bash: true
  glob: true
  grep: true
  question: true
---

<objective>
Safe git revert — roll back GTD phase or plan commits using the phase manifest, with dependency checks and a confirmation gate before execution.

Three modes:
- **--last N**: Show recent GTD commits for interactive selection
- **--phase NN**: Revert all commits for a phase (manifest + git log fallback)
- **--plan NN-MM**: Revert all commits for a specific plan
</objective>

<execution_context>
@/Users/davide/repos/get-tasks-done-demo-app/.opencode/get-tasks-done/workflows/undo.md
@/Users/davide/repos/get-tasks-done-demo-app/.opencode/get-tasks-done/references/ui-brand.md
@/Users/davide/repos/get-tasks-done-demo-app/.opencode/get-tasks-done/references/gate-prompts.md
</execution_context>

<context>
$ARGUMENTS
</context>

<process>
Execute end-to-end.
</process>
