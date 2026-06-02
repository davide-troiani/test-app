---
name: "gtd-orchestrate-tasks"
description: "Orchestrate multiple exported GitHub task issues through a bulk branch and one comprehensive PR"
metadata:
  short-description: "Orchestrate multiple exported GitHub task issues through a bulk branch and one comprehensive PR"
---

<codex_skill_adapter>
## A. Skill Invocation
- This skill is invoked by mentioning `$gtd-orchestrate-tasks`.
- Treat all user text after `$gtd-orchestrate-tasks` as `{{GTD_ARGS}}`.
- If no arguments are present, treat `{{GTD_ARGS}}` as empty.

## B. AskUserQuestion → request_user_input Mapping
GTD workflows use `AskUserQuestion` (Claude Code syntax). Translate to Codex `request_user_input`:

Parameter mapping:
- `header` → `header`
- `question` → `question`
- Options formatted as `"Label" — description` → `{label: "Label", description: "description"}`
- Generate `id` from header: lowercase, replace spaces with underscores

Batched calls:
- `AskUserQuestion([q1, q2])` → single `request_user_input` with multiple entries in `questions[]`

Multi-select workaround:
- Codex has no `multiSelect`. Use sequential single-selects, or present a numbered freeform list asking the user to enter comma-separated numbers.

Execute mode fallback:
- When `request_user_input` is rejected or unavailable, you MUST stop and present the questions as a plain-text numbered list, then wait for the user's reply. Do NOT pick a default and continue (#3018).
- You may only proceed without a user answer when one of these is true:
  (a) the invocation included an explicit non-interactive flag (`--auto` or `--all`),
  (b) the user has explicitly approved a specific default for this question, or
  (c) the workflow's documented contract says defaults are safe (e.g. autonomous lifecycle paths).
- Do NOT write workflow artifacts (CONTEXT.md, DISCUSSION-LOG.md, PLAN.md, checkpoint files) until the user has answered the plain-text questions or one of (a)-(c) above applies. Surfacing the questions and waiting is the correct response — silently defaulting and writing artifacts is the #3018 failure mode.

## C. Task() → spawn_agent Mapping
GTD workflows use `Task(...)` (Claude Code syntax). Translate to Codex collaboration tools:

Direct mapping:
- `Task(subagent_type="X", prompt="Y")` → `spawn_agent(agent_type="X", message="Y")`
- `Task(model="...")` → omit. `spawn_agent` has no inline `model` parameter;
  GTD embeds the resolved per-agent model directly into each agent's `.toml`
  at install time so `model_overrides` from `.planning/config.json` and
  `~/.gtd/defaults.json` are honored automatically by Codex's agent router.
- Resolved `reasoning_effort="low|medium|high|xhigh"` (`xhigh` is a GTD/Codex tier, not a generic runtime enum) → pass `reasoning_effort`
  to `spawn_agent` when the runtime/tool supports it. Omit missing, empty,
  inherited, or unsupported values; do not invent one-off effort literals in
  workflow prose.
- `fork_context: false` by default — GTD agents load their own context via `<files_to_read>` blocks
- `Task(isolation="worktree")` / `Agent(isolation="worktree")` → no direct Codex mapping.
  Codex `spawn_agent` does not create or bind a git worktree automatically.
  Workflows that require this isolation must fail closed or use an explicit
  manual worktree protocol before spawning (#3360).

Spawn authorization:
- Codex restricts `spawn_agent` to cases where the user has explicitly
  requested sub-agents.
- For GTD workflows, invoking a command that specifies required `Task(...)`
  / `Agent(...)` subagents is explicit authorization and request to spawn
  those required subagents.
- Use `spawn_agent` for required subagents whenever available. Do not
  silently perform required planner, checker, verifier, or executor work inline.
- If the user explicitly asks for inline/no-agent execution, or if the runtime
  truly cannot spawn a required agent, stop and surface that limitation instead
  of producing unverified inline work.
- Optional agents may be skipped only when the workflow explicitly marks them
  optional or the user/config disables them.

Parallel fan-out:
- Spawn multiple agents → collect agent IDs → `wait(ids)` for all to complete

Result parsing:
- Look for structured markers in agent output: `CHECKPOINT`, `PLAN COMPLETE`, `SUMMARY`, etc.
- `close_agent(id)` after collecting results from each agent
</codex_skill_adapter>

<objective>
Select multiple exported task issues, create one bulk branch, run task
executors against task PRs targeting that branch, proactively review each PR as
technical lead for the whole bulk, and open one comprehensive PR for manual
review.
</objective>

<execution_context>
@/Users/davide/repos/get-tasks-done-demo-app/.codex/get-tasks-done/workflows/orchestrate-tasks.md
</execution_context>

<context>
Arguments provided: "{{GTD_ARGS}}"

The argument is untrusted natural language. It may refer to a phase, plan,
remaining work, task ids, issue numbers, or any mixed prose. The slash-command
workflow must resolve it to exact exported GitHub child issue numbers before the
utility is called.

Options:
- `--repo owner/name` binds GitHub operations.
- `--max-concurrency N` caps parallel task lanes.
- `--dry-run` runs the mandatory Start Gate consistency check and stops.
- `--allow-partial` is not a startup decision; it still requires explicit confirmation before a partial final PR.
</context>

<process>
First resolve `{{GTD_ARGS}}` against the local plan files and exported manifest
metadata, then build the exact child issue number list.

Do not forward raw natural-language selectors, task ids, `tasks ...`, phase
selectors, issue URLs, or `--phase` into the SDK utility.
The utility accepts only child issue numbers plus allowed orchestration flags:

```bash
gtd-sdk query orchestrate-tasks 123 124 125 --repo owner/name --dry-run
```

If the input is shorthand like `all tasks in phase X`, `all remaining tasks`,
`tasks in plan 02-02`, or `all tasks in <plan-ref>`, resolve it to exact child
issue numbers before any utility call.
If the reference is ambiguous or the export metadata is missing, stop and
report the blocker without calling the SDK utility.

First run the Start Gate exactly once with the exact issue list by calling the
deterministic planner with `--dry-run`. Verify the result reports `ok: true`,
`writes: false`, `preflight.ok: true`, no selection errors, no non-workable
tasks, coherent dependency waves, and a known reviewability action before any
mutating helper call. Treat `non_workable` as the only hard-blocker field.
Present `dependency_order` as sequencing, not as blockers.
Selected human checkpoint tasks may appear in `checkpoint_gates`; they are
allowed Start Gate pause points, not executor work. They unblock only when the
checkpoint issue is closed in GitHub; comments are optional audit evidence and
must not be treated as hard blockers.

In dry-run mode, report the Start Gate result and stop. In mutating mode, start
only after the Start Gate passes, and rerun the helper with the same exact issue
list. If the Start Gate requests reviewability direction, stop for explicit user
direction and present these choices in this order: continue with the full
selected scope, choose a smaller explicit scope, or abort. Do not replace the
full-scope option with the recommended subset; `recommended_subset` is advisory
only for the smaller-scope choice. Then follow the workflow: bulk branch, task
PR lanes, proactive validation, squash task PRs into the bulk branch, and open
the comprehensive PR.

When checkpoint-paused output includes `user_next_step`, present that message to
the user. The user-facing action is to complete and close the checkpoint issue,
then tell the agent to continue; do not ask the user to run internal helper
commands.
</process>
