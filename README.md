# ⚡ Get Tasks Done in under 5 minutes ⚡

This repo is a disposable demo project for trying out [`ai-is-gonna/get-tasks-done`](https://github.com/ai-is-gonna/get-tasks-done).

It comes with pre-seeded planning for a small task management API, so you do not need to start from a blank repo. The idea is simple: clone this repo, push it to your own GitHub repo, export the phase tasks to GitHub Issues, and let your agent orchestrate a couple of them.

## What you are testing

- A demo repo with planning already in place
- Exporting planned work to GitHub Issues
- Running `get-tasks-done` against a disposable repo
- Orchestrating the first 2 tasks with your preferred agent

## Quick start

1. Create a new empty repo on GitHub and copy its URL.
2. Clone this repo:

```bash
git clone https://github.com/ai-is-gonna/get-tasks-done-demo-app
cd get-tasks-done-demo-app
```

3. Point `origin` at your repo:

```bash
git remote set-url origin [YOUR_REPO_URL]
```

4. Push the repo to GitHub:

```bash
git push -u origin main
```

5. Install GTD for your favorite agent:

```bash
npx @ai-is-gonna/get-tasks-done@latest --codex
```

If you install it into this repo and new files are added, commit those files before moving on.

6. Export phase 1 issues to GitHub:

```bash
.codex/get-tasks-done/bin/gtd-tools.cjs export-phase-issues 1
```

7. Commit the export manifest.
8. Open your favorite agent in this repo and type:

```text
$gtd-orchestrate-tasks first 2 tasks
```

You will be involved in one human-in-the-loop issue and for a code review.

## Notes

- The tasks are built for demo purposes and intentionally simple to focus on process rather than complexity.
- This repo is meant to be disposable. Use a fresh test repo, not a production project.
- The planning is already seeded under `.planning/`.
- You will likely need to be authenticated with GitHub locally before exporting issues.

## Problems?

Open an issue here: [ai-is-gonna/get-tasks-done issues](https://github.com/ai-is-gonna/get-tasks-done/issues/new)

`get-tasks-done` is still a fresh project. If you try it and something feels rough, that feedback is useful.
