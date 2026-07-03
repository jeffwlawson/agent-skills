---
name: sandcastle-setup
description: >
  Bootstraps Sandcastle into a repo that doesn't have it configured yet — runs init,
  customizes the Dockerfile based on what the repo actually needs, walks through the
  one-time OAuth token step, and offers a smoke test before declaring it ready. Always use
  this skill when the user asks to set up, initialize, install, configure, or bootstrap
  Sandcastle in a workspace, mentions wanting AFK/unattended coding agents for a repo that
  doesn't have `.sandcastle/` yet, or when handed off to from a day-shift planning skill
  after the user agreed to set it up — even if they phrase it casually ("get sandcastle
  going here", "let's wire up the AFK pipeline for this repo").
---

# Sandcastle Setup

This skill bootstraps a repo from nothing to a working Sandcastle pipeline. It doesn't plan issues or run them — that's the other two skills. Its only job is getting `.sandcastle/` into a state where those two skills have something to work with.

## When to use this skill

- The user directly asks to set up, initialize, or bootstrap Sandcastle in this workspace.
- A day-shift planning skill hands off here after finding no `.sandcastle` folder and the user agreed to set one up.

## Precondition

Check whether `.sandcastle` already exists at the workspace root. If it does, say so plainly and stop — this skill bootstraps from nothing, it doesn't audit or repair an existing setup. That's a different, riskier task for later if it's ever actually needed.

## Process

1. **Check the real current CLI surface before using it.** Run `npx @ai-hero/sandcastle init --help` via `run_command` and read the actual flags it reports, rather than assuming names from memory. Sandcastle's CLI can change between versions same as anything else — confirm before relying on it.
2. **Verify the environment a run will actually depend on — don't just assume it.** Two specific, real failure modes to check explicitly via `run_command`, both confirmed to bite silently otherwise:
   - **Docker reachability:** run `docker info`. If it fails, stop here and tell the user plainly — Docker Desktop isn't installed or isn't running — rather than letting `init` or `docker build-image` fail with a confusing partial error later.
   - **`cp` availability:** run `Get-Command cp -ErrorAction SilentlyContinue`. Sandcastle copies `node_modules` into the git worktree using `cp`, which doesn't exist natively on Windows — this fails with `CopyToWorktreeError: spawn cp ENOENT` the first time an agent actually runs, not during `init`, so it's easy to miss if you don't check for it now. If not found, search common Git-for-Windows locations (`C:\Tools\Git\usr\bin`, `C:\Program Files\Git\usr\bin`) for `cp.exe` and report which one exists — this doesn't fix anything itself, the `sandcastle-issue-runner` skill's launch command already accounts for it, but confirm here that one of its candidate paths actually matches reality on this machine rather than assuming. If `cp.exe` isn't found in any candidate location at all, that's a real blocker: tell the user they need Git for Windows installed before any run will succeed.
3. **Delegate to the `research` subagent to determine real Dockerfile needs.** Brief: check `package.json`, any `.csproj`/`.sln` files, `requirements.txt`/`pyproject.toml`, and existing CI workflow files, and report back what language runtimes and build tools this repo's tests and build actually depend on. The default Sandcastle image ships Node 22, git/curl/jq, `gh`, and the Claude Code CLI — the question for research to answer is specifically what's missing for *this* repo, not a generic audit.
4. **Run `npx @ai-hero/sandcastle init` via `run_command`, with every flag passed explicitly.** `run_command` gives the process no TTY, and every one of `--agent`, `--sandbox`, `--template`, `--issue-tracker`, `--create-label`, and `--build-image` defaults to an interactive prompt when omitted — any one left out causes `init` to fail fast rather than wedge, but it still fails. State these plainly as you go rather than asking permission for each one individually, since they're all easy to change later by hand-editing the generated files:
   - `--sandbox docker`
   - `--agent claude-code`
   - `--issue-tracker github-issues --create-label true`
   - `--template parallel-planner-with-review`
   - `--build-image false` — always pass this explicitly regardless of whether you want the image built now. Building the image is step 5's job, run separately, so a failure there never leaves `init` itself in a half-finished state.

   Note: `parallel-planner-with-review` generates a `main.mts` that imports `zod`. Sandcastle's `init` detects this and offers to install it automatically — accept the prompt. If it doesn't offer (older version), run `npm install --save-dev zod` manually. Missing `zod` produces an `ERR_MODULE_NOT_FOUND` crash on the first `npm run ship` and nothing else, so check for it now.

   **Verify the `Sandcastle` label actually exists — don't trust `--create-label true` silently.** Confirmed on a real run: `--create-label true` did not create the label, and the failure only surfaced downstream as `could not add label: 'Sandcastle' not found` when an issue was later being labeled. Check explicitly via `run_command`:
   ```
   gh label list --search Sandcastle
   ```
   If it's not there, create it directly rather than re-running `init` or retrying the flag:
   ```
   gh label create Sandcastle -c 000000 -d "Issues for Sandcastle"
   ```
   Do this regardless of whether `init` reported success — its reported success and the label's actual presence on GitHub aren't confirmed to be the same thing.
5. **Overwrite the scaffolded `main.mts` with the canonical version from `jeffwlawson/sandcastle-config`.** `init` generates a working but unoptimized `main.mts` — stock model assignments (one model across every phase) and a local-merge Phase 3 with no push or PR. The canonical version in `sandcastle-config` fixes both: Opus for planning only, Sonnet for implement/review, and a per-branch `git push` + `gh pr create` in place of the local merger. Fetch and overwrite via `run_command`:
   ```powershell
   $token = gh auth token
   Invoke-WebRequest -Uri "https://raw.githubusercontent.com/jeffwlawson/sandcastle-config/main/main.mts" `
     -Headers @{ Authorization = "token $token" } -OutFile .sandcastle/main.mts
   ```
   This requires `p-limit` as a dependency (the canonical file imports it for concurrency capping) — run `npm install --save-dev p-limit` if it isn't already present. Confirm the overwrite actually landed by checking that `.sandcastle/main.mts` contains `p-limit` and `gh pr create`, not the stock `Promise.allSettled(issues.map(...))`-with-no-cap pattern — don't just trust the fetch command exited 0.
6. **Apply any Dockerfile additions identified in step 3.** If research found nothing missing, say so and leave the default Dockerfile as generated.
7. **Add the run script.** Add `"ship": "tsx .sandcastle/main.ts"` to `package.json`'s `scripts` block (creating one if the repo doesn't already have a `package.json` at the root — ask the user first if that's the case, since adding one to a non-Node repo is a real structural choice, not a default to apply silently). If a `package.json` was just created from scratch, also run `npm install --save-dev @ai-hero/sandcastle tsx` via `run_command` — without this, the script has nothing to actually run, and the failure won't surface until the first real launch.
8. **Prepare `.sandcastle/.env`.** `init` only creates `.env.example`, not `.env` itself — copy it over if `.env` doesn't already exist. The generated file includes an empty `GH_TOKEN=` line — **leave it blank, don't comment it out or delete it.**

   Confirmed against the installed package's own source (`dist/index.js`, the `.env` resolver): for every key present in `.sandcastle/.env`, Sandcastle resolves its value as `sandcastleEnv[key] || process.env[key]` — `.env` wins if non-empty, otherwise it falls through to the host process's environment. Critically, that fallback only fires for keys that exist as a line in `.env` *at all*, even blank ones — the resolver iterates `Object.keys(sandcastleEnv)`, so a deleted line is never checked against `process.env` and a removed `GH_TOKEN` line silently means no GitHub auth reaches the container, full stop. The blank line is required scaffolding for `sandcastle-issue-runner`'s live-token-fetch design to work, not a danger to clean up.

   The actual thing worth checking here is the opposite of the old guidance: confirm the line is genuinely blank (`GH_TOKEN=` with nothing after it), not populated with a stale or wrong value — a real-but-wrong token there *would* win the `||` and silently shadow a good `process.env.GH_TOKEN`, since `||` only falls through on falsy values, not just "wrong" ones.
9. **Walk through the OAuth token step explicitly — do not attempt to do this yourself.** `claude setup-token` opens a real browser login; there's no tool call that completes it on your behalf. Tell the user to run it in their own terminal, wait for them to confirm they have, then actually check that `.sandcastle/.env` contains a populated `CLAUDE_CODE_OAUTH_TOKEN` before treating this step as done. Don't assume it succeeded just because the user said so — verify the file.
10. **Offer a smoke test, don't run one unprompted.** Ask whether the user wants to validate the setup before trusting it with anything real, with two explicit options:
    - **An existing issue number**, if they already have one in mind — hand off to `sandcastle-issue-runner` for that number once it exists.
    - **A new throwaway smoke-test issue**, generated right now. Don't route this through `to-issues` — that skill's decomposition and acceptance-criteria framing exist for real work where wrong scope has real cost, and a smoke test has none. File something fixed and minimal directly via `gh issue create`, labeled `Sandcastle`:
      - **Title:** `Sandcastle smoke test`
      - **Body:** "This is a throwaway smoke test confirming the pipeline connects and reads correctly — it is intentionally a no-op. Read this issue, confirm you understand it, and stop. Do not create, modify, or delete any files. Do not make any commits. There is nothing to implement."
      - **Acceptance criteria:** no commits were made; no files were changed.

      This deliberately tests only connectivity and issue-reading, not the commit/review/push/PR mechanics — narrower than testing the full chain, but targeted at the actual layers that have caused every real failure on this project so far (container launch, file access, gh auth, the planner reading correctly), rather than layers that haven't.

      One thing to set expectations on explicitly: a deliberate zero-commit smoke test is silently dropped before the review and push+PR phases ever run. Confirmed against the actual `main.mts` source: `if (implement.commits.length > 0)` gates the reviewer, and `completedBranches` only includes issues with commits — a zero-commit result exits cleanly without error and without opening a PR. This is correct behavior, not a bug. The smoke test still validates the layers it's intended to validate. Once filed, hand off to `sandcastle-issue-runner` the same as the existing-issue path.
11. **Report what's actually configured** — sandbox, agent, tracker, template, the `main.mts` swap from step 5, any Dockerfile additions, the cp/Docker environment check results, and whether the OAuth token was verified — so the user has a clear record of what just happened rather than just a "done."

## What this skill deliberately doesn't do

- Doesn't reconfigure or repair an existing `.sandcastle` setup.
- Doesn't guess at Dockerfile contents from the repo's general appearance — always grounds that decision in what `research` actually finds.
- Doesn't declare success on the OAuth step without checking the file.
- Doesn't run a real issue through the pipeline on its own initiative — that's always the user's call, via the other two skills.
