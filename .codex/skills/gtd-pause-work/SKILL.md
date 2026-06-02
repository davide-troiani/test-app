---
name: "gtd-pause-work"
description: "Create context handoff when pausing work mid-phase"
metadata:
  short-description: "Create context handoff when pausing work mid-phase"
---

<codex_skill_adapter>
## A. Skill Invocation
- This skill is invoked by mentioning `$gtd-pause-work`.
- Treat all user text after `$gtd-pause-work` as `{{GTD_ARGS}}`.
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
Create `.continue-here.md` handoff file to preserve complete work state across sessions.

Routes to the pause-work workflow which handles:
- Current phase detection from recent files
- Complete state gathering (position, completed work, remaining work, decisions, blockers)
- Handoff file creation with all context sections
- Git commit as WIP
- Resume instructions
</objective>

<execution_context>
@/Users/davide/repos/get-tasks-done-demo-app/.codex/get-tasks-done/workflows/pause-work.md
</execution_context>

<context>
State and phase progress are gathered in-workflow with targeted reads.
</context>

<process>
If `--report` is in {{GTD_ARGS}}:
Read and execute `/Users/davide/repos/get-tasks-done-demo-app/.codex/get-tasks-done/workflows/session-report.md` end-to-end.

**Follow the pause-work workflow**.

The workflow handles all logic including:
1. Phase directory detection
2. State gathering with user clarifications
3. Handoff file writing with timestamp
4. Git commit
5. Confirmation with resume instructions
</process>
