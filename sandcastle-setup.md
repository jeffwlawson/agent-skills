---
name: sandcastle-setup
description: >
  Bootstraps Sandcastle into a repo that doesn't have it configured yet — runs init,
  customizes the Dockerfile based on what the repo actually needs, walks through the
  one-time OAuth token step, and offers a smoke test before declaring it ready. Use when
  the user asks to set up, initialize, install, or bootstrap Sandcastle in this workspace,
  or when handed off to from the issue-planning skill after the user agreed to set it up.
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
5. **Apply any Dockerfile additions identified in step 3.** If research found nothing missing, say so and leave the default Dockerfile as generated.
6. **Add the run script.** Add `"ship": "tsx .sandcastle/main.ts"` to `package.json`'s `scripts` block (creating one if the repo doesn't already have a `package.json` at the root — ask the user first if that's the case, since adding one to a non-Node repo is a real structural choice, not a default to apply silently). If a `package.json` was just created from scratch, also run `npm install --save-dev @ai-hero/sandcastle tsx` via `run_command` — without this, the script has nothing to actually run, and the failure won't surface until the first real launch.
7. **Prepare `.sandcastle/.env`.** `init` only creates `.env.example`, not `.env` itself — copy it over if `.env` doesn't already exist. The generated file includes an empty `GH_TOKEN=` line; find and comment it out (or remove it) immediately, via `replace_file_content` or equivalent. This isn't optional tidiness — an empty value here overrides valid system/keychain `gh` auth for any call made inside the container, a bug already confirmed on this exact project. Do this regardless of whether `sandcastle-issue-runner`'s live token fetch is also in place: which one actually matters depends on whether Sandcastle's env resolution prioritizes `.env` or `process.env` when both define the same key, which isn't confirmed either way — fixing both costs nothing and is correct under either precedence order, so don't treat them as alternatives.
8. **Walk through the OAuth token step explicitly — do not attempt to do this yourself.** `claude setup-token` opens a real browser login; there's no tool call that completes it on your behalf. Tell the user to run it in their own terminal, wait for them to confirm they have, then actually check that `.sandcastle/.env` contains a populated `CLAUDE_CODE_OAUTH_TOKEN` before treating this step as done. Don't assume it succeeded just because the user said so — verify the file.
9. **Offer a smoke test, don't run one unprompted.** Ask whether the user wants to validate the setup before trusting it with anything real, with two explicit options:
   - **An existing issue number**, if they already have one in mind — hand off to `sandcastle-issue-runner` for that number once it exists.
   - **A new throwaway smoke-test issue**, generated right now. Don't route this through `unattended-ready-issue-planner` — that skill's investigation and verification steps exist for real work where wrong scope has real cost, and a smoke test has none. File something fixed and minimal directly via `gh issue create`, labeled `Sandcastle`:
     - **Title:** `Sandcastle smoke test`
     - **Body:** "This is a throwaway smoke test confirming the pipeline connects and reads correctly — it is intentionally a no-op. Read this issue, confirm you understand it, and stop. Do not create, modify, or delete any files. Do not make any commits. There is nothing to implement."
     - **Acceptance criteria:** no commits were made; no files were changed.

     This deliberately tests only connectivity and issue-reading, not the commit/review/push/PR mechanics — narrower than testing the full chain, but targeted at the actual layers that have caused every real failure on this project so far (container launch, file access, gh auth, the planner reading correctly), rather than layers that haven't.

     One thing to set expectations on explicitly: a deliberate zero-commit smoke test is silently dropped before the review and push+PR phases ever run. Confirmed against the actual `main.mts` source: `if (implement.commits.length > 0)` gates the reviewer, and `completedBranches` only includes issues with commits — a zero-commit result exits cleanly without error and without opening a PR. This is correct behavior, not a bug. The smoke test still validates the layers it's intended to validate. Once filed, hand off to `sandcastle-issue-runner` the same as the existing-issue path.
10. **Report what's actually configured** — sandbox, agent, tracker, template, any Dockerfile additions, the cp/Docker environment check results, and whether the OAuth token was verified — so the user has a clear record of what just happened rather than just a "done."

## What this skill deliberately doesn't do

- Doesn't reconfigure or repair an existing `.sandcastle` setup.
- Doesn't guess at Dockerfile contents from the repo's general appearance — always grounds that decision in what `research` actually finds.
- Doesn't declare success on the OAuth step without checking the file.
- Doesn't run a real issue through the pipeline on its own initiative — that's always the user's call, via the other two skills.
