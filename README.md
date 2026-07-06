# agent-skills

This repo used to hold the two Sandcastle night-shift skills (`sandcastle-setup`,
`sandcastle-issue-runner`) as a backup for the live copy at `~/.claude/skills/`. Both
migrated to
[jeffwlawson/sandcastle-config](https://github.com/jeffwlawson/sandcastle-config/tree/main/skills) —
canonical source, version history, and the ADR explaining the move now live there,
alongside the `main.mts`/`review-pr.mts` orchestration those skills bootstrap and launch.
That repo already owned the orchestration side of the same pipeline, so the skills that
install and drive it live next to it instead of split across two repos.

This repo stays around, empty, as a place for other custom Claude Code / Antigravity
skills that aren't specific to Sandcastle — nothing currently lives here.
