---
name: sandcastle-issue-runner
description: >
  Launches a Sandcastle AFK run against this repo's Sandcastle-labeled issue backlog.
  Always use this skill when the user wants to kick off, run, start, or resume unattended
  Sandcastle work — phrases like "run sandcastle", "kick off the sandcastle run", "start
  the night shift", "let's ship the backlog", "run the planner now", "work through the
  AFK issues" — or when sandcastle-setup hands off here right after its smoke test ("ready
  to run a smoke-test issue through Sandcastle"). Takes no issue number or mode argument:
  the planner inside `npm run ship` reads the whole labeled backlog itself, every round.
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

2. Fetch a live GitHub token for this session only — never write it to `.env` — and
   **verify it actually came back non-empty before proceeding.** Confirmed real failure:
   `gh auth token` can fail silently into an empty or error-text assignment if the host
   isn't authenticated (expired session, never logged in), and PowerShell doesn't raise
   anything visible for that — `$env:GH_TOKEN` just ends up set to something falsy but
   *present*. A present-but-bad `GH_TOKEN` in the environment is exactly as damaging as the
   empty `.env` line this same setup already guards against: it silently overrides any
   valid keychain-based `gh` auth instead of being treated as missing. Don't just run the
   assignment and move on — check the result:
   ```powershell
   $env:GH_TOKEN = (gh auth token)
   if ([string]::IsNullOrWhiteSpace($env:GH_TOKEN)) {
     Write-Error "gh auth token returned empty — host is not authenticated. Run 'gh auth login' first."
     return
   }
   ```
   If this fails, stop here and tell the user plainly to run `gh auth login` — don't launch
   the window with a bad token, since the failure won't surface until deep inside the
   container, on a command like `gh issue list`, far from the actual cause.

   This pairs with `sandcastle-setup` leaving `GH_TOKEN=` **blank** (present, but empty) in
   `.sandcastle/.env` — confirmed necessary, not just tidy: Sandcastle's `.env` resolver
   only falls through to `process.env` for keys that exist as a line in `.env` at all, even
   blank ones. A deleted line is never checked against `process.env`, so removing it
   entirely breaks this token fetch too. If either skill changes, keep that pairing intact
   — the danger case is a *populated* `GH_TOKEN=` with a stale or wrong value, which would
   win over a good live-fetched token here, not a blank one.

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
   review passes.

## If you see `could not add label` or `gh: please run gh auth login` inside the container

That symptom surfaces deep in a `!`gh issue list`` ``-style prompt expansion or label call
— far from its actual cause. It does not mean `gh auth login` needs to run *inside* the
sandbox; Sandcastle has no mechanism for that and isn't supposed to need one. It means
step 2 above let a bad/empty `GH_TOKEN` through on the host before launch. Re-run step 2's
verification manually (`gh auth token` should print a real token starting with `gh[ospu]_`,
not blank or an error) before relaunching.

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
