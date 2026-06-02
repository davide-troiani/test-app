---
description: Ingest external plans with conflict detection against project decisions before writing anything.
argument-hint: "--from <filepath>"
tools:
  read: true
  write: true
  edit: true
  bash: true
  glob: true
  grep: true
  question: true
  agent: true
---

<objective>
Import external plan files into the GTD planning system with conflict detection against PROJECT.md decisions.

- **--from**: Import an external plan file, detect conflicts, write as GTD PLAN.md, validate via gtd-plan-checker.
</objective>

<execution_context>
@/Users/davide/repos/get-tasks-done-demo-app/.opencode/get-tasks-done/workflows/import.md
@/Users/davide/repos/get-tasks-done-demo-app/.opencode/get-tasks-done/references/ui-brand.md
@/Users/davide/repos/get-tasks-done-demo-app/.opencode/get-tasks-done/references/gate-prompts.md
@/Users/davide/repos/get-tasks-done-demo-app/.opencode/get-tasks-done/references/doc-conflict-engine.md
</execution_context>

<context>
$ARGUMENTS
</context>

<process>
Execute the import workflow end-to-end.
</process>
