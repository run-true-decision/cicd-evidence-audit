---
name: "cicd-evidence-audit"
description: "Audit CI/CD merge evidence and retrofit five controls. Use for 'checks passed - safe to merge?', 'audit our CI gates', or 'retrofit CI evidence controls'."
user-invocable: true
---

# CI/CD evidence audit

Use this when an AI agent (or anyone) reports "checks passed" and the question is whether that evidence actually supports merging *this* code. Two modes:

- **Audit**: map a repository's merge path, find where wrong, stale, missing or advisory evidence could still count, and report gaps. Read-only for the audited repository.
- **Retrofit**: add or repair the five control families below in an existing project, as small reviewable changes.

If the user does not say which mode, run Audit first and offer Retrofit for the gaps found.

## Ground rules

- Audit may run local commands and fault injection only in a disposable copy: a fresh export or clone outside the audited repository, not a `git worktree` (which writes into the audited repository's `.git`). Read each repository script before running it. Run with no credentials or push access in the environment, including credential helpers and SSH agents. Audit never pushes. Propose challenges that need platform CI; execute them only as a separately approved push. Until observed, label that control "configured only" and use `UNEVAL` for unproved behaviour.
- Inspect before proposing. Read the CI config, test runner config, scripts and docs before suggesting any change.
- A control's verdict must be deterministic: same inputs, same answer. No language-model judgement inside a verdict. AI review can be *evidence* a control checks; it cannot be the control.
- Every control returns one of: `PASS`, `FAIL`, `UNEVAL` (could not establish the answer), `N/A` (known not applicable, with a reason). Missing information is `UNEVAL`, never `PASS` and never silently dropped.
- Never weaken a gate to get green: do not delete required checks, add `continue-on-error`, `|| true`, `--exit-zero`, skip markers or exceptions without saying so explicitly and getting approval.
- Do not change branch protection, rulesets, secrets, or push/merge without the user's explicit approval for that specific action.
- Do not claim a live merge barrier from repository files alone. Repo files show configured behaviour; branch protection or rulesets show what the platform actually requires. Say which one you checked.
- Clean-room only: write controls from these patterns and the project's own code. Never copy code, documents or know-how from another employer or client.
- Prefer one control per PR. Each change ships with its own challenge tests (valid case, named fault, missing input).

## Step 0 — Inventory (read-only)

Collect, and cite file paths for, everything below:

- CI definitions: `.github/workflows/*`, `.gitlab-ci.yml`, `Jenkinsfile`, `azure-pipelines.yml`, `.circleci/`, etc.
- Test runner config and the exact test commands each CI job runs (paths, markers, `-k`, shards).
- Local scripts: preflight, pre-push or pre-commit hooks, `Makefile`/`justfile` targets, agent instructions (`AGENTS.md`, `CLAUDE.md`).
- Saved or generated artifacts that later steps reuse: coverage reports, generated docs or inventories, snapshots, lockfiles, SBOMs.
- Any policy file that lists required checks, and the script that evaluates it.
- Platform enforcement, if the user permits API access: `gh api repos/OWNER/REPO/branches/BRANCH/protection` and `gh api repos/OWNER/REPO/rules/branches/BRANCH`. If not permitted, record it as an open question.

## Step 1 — Draw the evidence path

Write the path for one merge decision:

`Changed input (files, commit) -> Selected work (job, test command) -> Result (record for a version and run) -> Judge (policy + evaluator) -> Decision (required status + merge authority)`

Note which rules judge the decision and where they are loaded from. Note what happens after merge (main CI, deployment) as separate paths; do not let pre-merge evidence stand in for them.

## Step 2 — Pick merge requirements

Choose the requirements that matter most (start with one). For each, state what must be true, e.g. "the reused coverage report describes the current source set" or "every applicable suite passed for the PR head commit".

## Step 3 — Check the five control families

For each control: ask the question, probe the repo, record the verdict, and (Retrofit) add the minimal pattern.

### 1. Source comparison

Question: does a saved or generated artifact still describe the current inputs?
Fault: inputs change after the artifact was produced; the filename and green status still look right.
Probe: find artifacts committed to the repo or reused across jobs. Check whether anything compares them with current inputs.
Pattern (pick the simpler that fits):
- Regenerate and compare in CI: start from a clean disposable checkout, run a deterministic generator, then `git status --porcelain -- <artifact>`. Fail if the output is non-empty, including untracked artifacts. The same inputs must produce byte-identical output; remove or control timestamps.
- Or store a hash per input path plus a combined hash of the defined input set (sources, tests, the tool itself) inside the artifact. Re-enumerate that set from its definition before every comparison, not just from the saved path list. Changed, added or missing input means re-measure.
Challenge: matching inputs (accept); one input edited without regenerating (reject); one input deleted (UNEVAL or reject, never accept); a new file added inside the defined input set (require re-measure); an untracked generated artifact (reject).
Limit: freshness proves which inputs the artifact belongs to. It does not repeat the measurement.

### 2. Local preflight

Question: is the evidence about this commit, and did every required check answer?
Faults: evidence or approval for commit A reused for local commit B; a check crashes before its verdict and disappears from the summary.
Probe: is there a local script that runs fast checks? Does it keep a required-check list independent of returned results? Does it compare `git rev-parse HEAD` with the PR head (`gh pr view --json headRefOid -q .headRefOid`)? What does its exit code mean, and does the caller stop on non-zero?
Pattern:
- A required-check list; each check emits one verdict line; the summary adds `UNEVAL` for any required check with no verdict.
- Exit codes: `0` all pass or justified N/A; `1` a completed FAIL in strict mode; `3` any UNEVAL (takes priority). Default mode may be advisory, but say so in the output.
- Head comparison: equal means continue; both valid and different means N/A for remote evidence (push, then re-obtain evidence and review); missing or malformed IDs mean UNEVAL.
- Any automation that calls preflight (push scripts, agent loops) must use strict mode and stop on non-zero.
Challenge: delete one required check's output (expect UNEVAL, exit 3); create an unpushed commit (expect N/A for remote evidence); blank the PR head ID (expect UNEVAL).

### 3. Test collection

Question: does a CI job that this PR triggers actually select the new or changed test?
Fault: a test passes when run directly but sits outside every CI job's selection.
Probe: list added test files (`git diff --name-only --diff-filter=A BASE...HEAD`). For each registered PR job, run its runner in list mode with the job's exact arguments (`pytest --collect-only -q ...`, `jest --listTests ...`, `go test -list . ...`). Compare.
Pattern: a CI check that fails when a new test file is collected by no PR job. A collection or import error is UNEVAL, not "no route". Exceptions live in a reviewed file with path, reason, owner and expiry.
Challenge: add a test under an unselected directory (expect FAIL); add one under a selected directory (expect PASS); add one with a broken import (expect UNEVAL).
Limit: collection is not execution. The PR must also trigger that job, and the job must run.

### 4. Required CI results

Question: did every applicable suite pass for the intended commit, judged by rules the PR cannot edit?
Faults: a success from another commit or older attempt counts; a missing suite is treated as fine; the PR edits the policy or evaluator to drop its own failing requirement.
Probe: is there one aggregate gate? Where does its policy come from? Under `pull_request`, GitHub runs the workflow definition from the PR itself, so a PR can change its own gate. Does the gate match results by head SHA, event, current run attempt and current Ready cycle (the latest transition from draft to ready)? Is the gate the required status in branch protection? Which required jobs can skip or conclude neutral?
On GitHub, a job skipped by a job-level `if:` reports success for branch protection without running. A workflow not triggered because of path or branch filtering leaves required checks pending and blocks merging. GitHub also accepts skipped and neutral check conclusions; platform acceptance alone is not proof a required suite executed.
An aggregate gate job that `needs:` suite jobs can itself be skipped after a suite fails. Use `if: always()` so the gate runs and fails unless every needed result is `success`, or have an independent gate read results through the API.
Pattern:
- One aggregate gate check is the required status. It maps changed paths to required suites using policy and evaluator code loaded from the base branch, and treats the PR diff as data.
- Run the gate with accepted base-branch code, for example `pull_request_target` with a base checkout. In that privileged context, never check out or execute PR head code.
- Set explicit least-privilege job `permissions:` in that privileged context. Grant only the reads needed and, if publishing a status requires it, the narrow necessary write permission.
- Never interpolate PR-controlled titles, bodies or branch names into `run:` scripts. Pass them as data, not executable shell text, and never evaluate them.
- Treat artifacts downloaded from a PR run as untrusted data. Validate their structure and association with the intended commit and run; never execute or source them.
- Each required suite must execute and appear exactly once for the head SHA, event, current attempt and current Ready cycle, and conclude success. Missing, skipped or neutral evidence means FAIL or UNEVAL, never PASS. A success from the draft stage does not establish ready-stage execution.
- Policy changes are tested by normal jobs on the PR, and take effect only after an authorized merge.
Challenge: required suite fails (expect gate FAIL); required suite absent (expect not PASS); required job skips (expect gate FAIL or UNEVAL, never PASS); old success for a previous SHA, event, attempt or Ready cycle only (expect not PASS); PR deletes the failing suite from its own policy copy (expect gate still FAIL).
Dependency challenge: make one suite job fail and confirm the aggregate gate reports failure, not skipped.

### 5. Workflow effect

Question: does a failing check actually change the merge decision?
Fault: a command fails, but the step, job or pipeline is configured to continue, so the job ends green.
Probe: search CI and scripts for `continue-on-error`, `allow_failure` (GitLab), `catchError` / `unstable` (Jenkins), `|| true`, `set +e`, `--exit-zero`, `if: always()` on gating steps, and pipes without `pipefail` (`cmd | tee log`). For each hit, trace: command exit -> step -> job conclusion -> required status -> branch protection. Record whether the advisory effect is intentional and documented.
Pattern: make the intended effect explicit per check and event. If a finding must block, remove the continuation for that event or route the finding into the aggregate gate. Never flip an intentionally advisory check to blocking (or the reverse) without approval.
Challenge: propose injecting a failure into the command on a disposable branch and checking that the job and required status both turn red on events that should block. Running this through platform CI needs explicit approval for the push; Audit does not push.

## Step 4 — Challenge every control

For each control, run three cases within the approved scope: valid input, one named fault, missing input. Use a disposable copy for local fault injection; never leave the injected fault in the working tree. Propose platform-CI cases rather than pushing in Audit; without an approved run, record "configured only", not observed enforcement. A control that also passes the faulty case is not a control. Record commands and outcomes. If you could not run a case, say so; do not infer the result.

## Step 5 — Record the audit

Write the worksheet outside the audited repository unless the user asks for it in a PR. Include one row per merge requirement:

| Field | Content |
|---|---|
| Decision / requirement | What must be true for this merge |
| Evidence path | Input -> work -> result -> judge -> decision, with file paths |
| Failure / effect | How the evidence could look acceptable while the claim is false |
| Control | What checks it, how it is triggered, and what consumes its answer |
| Challenge | Valid / fault / missing: expected vs observed, with commands |
| Verdict | PASS / FAIL / UNEVAL / N/A, and what kind of proof (source read, local run, CI run, platform API) |
| Open question and owner | What remains unproved and who resolves it |
| Revisit trigger | Code, tests, selection, policy or trigger changes that require a re-audit |

End with a ranked gap list (blocking first) and, for Retrofit, one proposed PR per gap: files to change, the challenge tests it adds, and the risk of the change.

## Reporting language

- Say "deterministic, rule-based check" for the controls, and name what they read. Do not say the controls prove the code is correct, or that flaky tests become reliable.
- Distinguish configured behaviour (read from files) from observed behaviour (a run you executed or a platform API you queried).
- No compliance, certification or effectiveness claims. The method adapts process failure-mode analysis; it is not a formal assessment.