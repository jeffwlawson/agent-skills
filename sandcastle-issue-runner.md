---
name: sandcastle-issue-runner
description: >
  Launches a Sandcastle AFK run against this repo's Sandcastle-labeled issue backlog.
  Use when the user says things like "run sandcastle", "kick off the sandcastle run",
  "start the night shift", "let's ship the backlog", "run the planner now" — or when
  sandcastle-setup hands off here right after its smoke test ("ready to run a smoke-test
  issue through Sandcastle"). Takes no issue number or mode argument: the planner inside
  `npm run ship` reads the whole labeled backlog itself, every round.
---

# sandcastle-issue-runner

## What this does

Launches one detached terminal window running `npm run ship`, then gets out of the way.
Fire-and-forget — no polling, no waiting on it, no progress reporting back to this
conversation. `parallel-planner-with-review`'s own planner decides what to work on every
round by reading whatever's open and labeled `Sandcastle`.

## Precondition

Check that `.sandcastle` exists at the workspace root.
- Missing → say so plainly and stop. Nothing to run. Point at `sandcastle-setup`.

## What NOT to do

- **Don't ask for or accept a specific issue number.** There is no `npm run ship -- <n>`
  mechanism — neither this template's nor `sequential-reviewer`'s generated script reads a
  CLI argument for issue selection. Targeting a specific issue means controlling which
  issues carry the `Sandcastle` label *before* launch, not passing an argument here.
- **Don't launch a second detached window against the same repo while one is already
  running.** `parallel-planner-with-review` fans out internally; two outer OS processes
  racing the same repo sit a layer above Sandcastle's own worktree/branch isolation and
  defeat it.
- **Don't poll, tail, or wait on the launched window.** Confirm it started, then stop.

## Steps

1. Confirm `.sandcastle` exists (see Precondition above).

2. Fetch a live GitHub token for this session only — never write it to `.env`:
   ```powershell
   $env:GH_TOKEN = (gh auth token)
   ```
   This pairs with `sandcastle-setup` leaving `GH_TOKEN=` commented out in
   `.sandcastle/.env`. If either skill changes, keep that pairing intact — a stale empty
   `GH_TOKEN=` in `.env` silently overrides the system's authenticated `gh` keyring.

3. Prepend Git-for-Windows' `usr\bin` to PATH for this window, ahead of everything else.
   Confirmed fix for a real `spawn cp ENOENT` — Sandcastle shells out to `cp` to seed
   `node_modules` into the worktree, and Windows has no native `cp`:
   ```powershell
   $env:PATH = "C:\Tools\Git\usr\bin;$env:PATH"
   ```

4. Launch one detached window, titled for the repo, and return — don't wait on it:
   ```powershell
   Start-Process powershell -ArgumentList "-NoExit", "-Command", `
     "`$Host.UI.RawUI.WindowTitle = 'Sandcastle — <repo-name>'; npm run ship"
   ```

5. Tell the user the window launched and that this skill won't report back further —
   they'll watch progress in the new window, and PRs land per-branch as each branch's
   review passes (once the push+PR step is wired in).

## The one legitimate exception

Right after `sandcastle-setup` finishes, it may hand off here to run a single smoke-test
issue. That still goes through this exact same no-argument `npm run ship` path — it works
because the smoke test runs before any real backlog exists, so the `Sandcastle`-labeled
set is exactly one issue at that moment. It isn't a different code path; it's a different
*moment* to invoke the same one.

## Explicitly retired from the old version

Modes 1–4, manual issue-number selection, per-issue detached windows, and any notion of
"run just issue #42" as a normal mode. Those fit `sequential-reviewer`'s one-issue-per-loop
model; they don't fit a template where one process reads the whole filtered backlog and
fans out N agents itself. If "run just this one urgent thing right now" becomes a real
felt need, it gets solved separately (a non-backlog-driven `sandcastle.run()` call, or
temporarily unlabeling everything else) — not by resurrecting Modes 1–4 here.
