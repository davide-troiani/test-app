---
name: "gtd-workstreams"
description: "Manage parallel workstreams — list, create, switch, status, progress, complete, and resume"
metadata:
  short-description: "Manage parallel workstreams — list, create, switch, status, progress, complete, and resume"
---

<codex_skill_adapter>
## A. Skill Invocation
- This skill is invoked by mentioning `$gtd-workstreams`.
- Treat all user text after `$gtd-workstreams` as `{{GTD_ARGS}}`.
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

# $gtd-workstreams

Manage parallel workstreams for concurrent milestone work.

## Usage

`$gtd-workstreams [subcommand] [args]`

### Subcommands

| Command | Description |
|---------|-------------|
| `list` | List all workstreams with status |
| `create <name>` | Create a new workstream |
| `status <name>` | Detailed status for one workstream |
| `switch <name>` | Set active workstream |
| `progress` | Progress summary across all workstreams |
| `complete <name>` | Archive a completed workstream |
| `resume <name>` | Resume work in a workstream |

## Step 1: Parse Subcommand

Parse the user's input to determine which workstream operation to perform.
If no subcommand given, default to `list`.

## Step 2: Execute Operation

### list
Run: `gtd-sdk query workstream.list --raw --cwd "$CWD"`
Display the workstreams in a table format showing name, status, current phase, and progress.

### create
Run: `gtd-sdk query workstream.create <name> --raw --cwd "$CWD"`
After creation, display the new workstream path and suggest next steps:
- `$gtd-new-milestone --ws <name>` to set up the milestone

### status
Run: `gtd-sdk query workstream.status <name> --raw --cwd "$CWD"`
Display detailed phase breakdown and state information.

### switch
Run: `gtd-sdk query workstream.set <name> --raw --cwd "$CWD"`
Also set `GTD_WORKSTREAM` for the current session when the runtime supports it.
If the runtime exposes a session identifier, GTD also stores the active workstream
session-locally so concurrent sessions do not overwrite each other.

### progress
Run: `gtd-sdk query workstream.progress --raw --cwd "$CWD"`
Display a progress overview across all workstreams.

### complete
Run: `gtd-sdk query workstream.complete <name> --raw --cwd "$CWD"`
Archive the workstream to milestones/.

### resume
Set the workstream as active and suggest `$gtd-resume-work --ws <name>`.

## Step 3: Display Results

Format the JSON output from gtd-sdk query into a human-readable display.
Include the `${GTD_WS}` flag in any routing suggestions.
