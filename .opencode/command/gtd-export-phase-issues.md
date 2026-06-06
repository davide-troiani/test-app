---
description: Export phase plans to deterministic GitHub issue hierarchies
argument-hint: "<phase-number-or-path> [--dry-run] [--repo owner/name]"
requires: [plan-phase]
tools:
  read: true
  bash: true
  glob: true
  grep: true
---

<objective>
Export one phase directory into a deterministic GitHub issue hierarchy.

Write mode (default) applies those mutations through GitHub Issues and
updates the local export manifest.
Dry-run mode parses source plans and renders the labels, parent issues, child
task issues, sub-issue links, blocked-by dependencies, and manifest entries
without writing. 
</objective>

<execution_context>
@/Users/davide/repos/get-tasks-done-demo-app/.opencode/get-tasks-done/workflows/export-phase-issues.md
</execution_context>

<context>
Arguments provided: "$ARGUMENTS"

Required:
- A phase number or phase directory path.

Optional:
- `--dry-run` to preview without writing.
- `--repo owner/name` to override repository detection. Without it, write mode
  uses the existing manifest repo or the current git GitHub remote.
</context>

<process>
Execute the exporter end-to-end. Only if `--dry-run` is not present, require an authenticated 
GitHub CLI and write labels, issues, relationships, dependencies, and the manifest.
Otherwise, report the preview only.
</process>
