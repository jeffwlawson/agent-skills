# agent-skills

Custom Antigravity / Claude Code skills for the AFK coding pipeline — the night-shift
counterpart to Matt Pocock's [mattpocock/skills](https://github.com/mattpocock/skills),
which covers day-shift planning (`grill-me`, `grill-with-docs`, `to-prd`, `to-issues`).

These skills install and operate [Sandcastle](https://github.com/mattpocock/sandcastle)
for unattended implementation, review, and PR creation. They don't modify Matt's day-shift
skills or his Sandcastle prompt templates — only the bootstrap and launch mechanics around
them.

## Skills in this repo

- **`sandcastle-setup`** — bootstraps a repo from nothing to a working Sandcastle pipeline:
  runs `init`, customizes the Dockerfile for the repo's actual language/build needs, walks
  through the one-time OAuth token step, and offers a smoke test before declaring it ready.
  Pulls the project's `main.mts` from
  [jeffwlawson/sandcastle-config](https://github.com/jeffwlawson/sandcastle-config) rather
  than using Sandcastle's stock scaffolded version.

- **`sandcastle-issue-runner`** — launches a Sandcastle run against the repo's
  `Sandcastle`-labeled backlog. Fire-and-forget: one detached terminal window running
  `npm run ship`, no polling, no argument for targeting a specific issue (the planner
  inside the run decides what to work on every round by reading the label-filtered
  backlog).

## Install / usage

These are plain Claude-skill-format `.md` files with YAML frontmatter (`name` +
`description`). Antigravity 2.0 reads this format directly — no conversion needed.

These run from a **global skills path**, not copied per-project — one live copy that every
Antigravity/Claude Code session reads, rather than N per-project copies to keep in sync.
This repo is the backup and version history for that global copy, not the thing itself:
- Edit a skill in place at the global path, same as before.
- When a change is worth keeping, copy it from the global path into this repo and commit —
  that's the "if this machine dies, here's what to restore" copy.
- New machine or reinstall → copy from this repo back out to the global path, once.

One-directional sync (global path → repo), on purpose, not automatic — no git pull, no
symlink, no submodule. Skills change rarely enough that manual, deliberate propagation
costs less than the complexity of keeping something auto-synced.

Both skills are user-invoked: they trigger on natural-language intent in their
`description` frontmatter (Antigravity has no slash-command support yet), not on a typed
`/` prefix.

## Scope discipline

Before adding a new skill here, ask whether it's closing a confirmed gap — verified
against real source or a real bug hit while running this for real — or solving a problem
that hasn't actually shown up yet. If it's the latter, don't add it. Several skills that
seemed useful on paper (a dual HITL/AFK label system, a dedicated resolve-HITL skill, a
formal router/decompose/ground-and-file chain duplicating `to-issues`) were designed and
then deliberately not added here for exactly this reason.
