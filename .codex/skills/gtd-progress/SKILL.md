---
name: "gtd-progress"
description: "Check progress, advance workflow, or dispatch freeform intent — the unified GTD situational command"
metadata:
  short-description: "Check progress, advance workflow, or dispatch freeform intent — the unified GTD situational command"
---

<codex_skill_adapter>
## A. Skill Invocation
- This skill is invoked by mentioning `$gtd-progress`.
- Treat all user text after `$gtd-progress` as `{{GTD_ARGS}}`.
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
Check project progress, summarize recent work and what's ahead, then intelligently route to the next action.

Three modes:
- **default**: Show progress report + intelligently route to the next action (execute or plan). Provides situational awareness before continuing work.
- **--next**: Automatically advance to the next logical step without manual route selection. Reads STATE.md, ROADMAP.md, and phase directories. Supports `--force` to bypass safety gates.
- **--do "task description"**: Analyze freeform natural language and dispatch to the most appropriate GTD command. Never does the work itself — matches intent, confirms, hands off.
- **--forensic**: Append a 6-check integrity audit after the standard progress report.
</objective>

<flags>
- **--next**: Detect current project state and automatically invoke the next logical GTD workflow step. Scans all prior phases for incomplete work before routing. `--next --force` bypasses safety gates.
- **--do "..."**: Smart dispatcher — match freeform intent to the best GTD command using routing rules, confirm the match, then hand off.
- **--forensic**: Run 6-check integrity audit after the standard progress report.
- **(no flag)**: Standard progress check + intelligent routing (Routes A through F).
</flags>

<execution_context>
@/Users/davide/repos/get-tasks-done-demo-app/.codex/get-tasks-done/workflows/progress.md
@/Users/davide/repos/get-tasks-done-demo-app/.codex/get-tasks-done/workflows/next.md
@/Users/davide/repos/get-tasks-done-demo-app/.codex/get-tasks-done/workflows/do.md
@/Users/davide/repos/get-tasks-done-demo-app/.codex/get-tasks-done/references/ui-brand.md
</execution_context>

<process>
Arguments provided: "{{GTD_ARGS}}"
Parse the first token from the provided arguments:
- If it is `--next`: strip the flag, execute the next workflow (passing remaining args e.g. --force).
- If it is `--do`: strip the flag, pass remainder as freeform intent to the do workflow.
- Otherwise: execute the progress workflow end-to-end (pass --forensic through if present).

Preserve all routing logic from the target workflow.
</process>
