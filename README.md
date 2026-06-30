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
`description`). Antigravity 2.0 reads this format directly — no conversion needed. Drop
them into a project's `.claude/commands/` or `.agents/workflows/` directory (whichever
your agent of choice scans) the same way you'd install any other skill.

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
