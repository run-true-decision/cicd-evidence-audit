---
name: "cicd-evidence-audit"
description: "Audit CI/CD merge evidence and retrofit five controls. Use for 'checks passed - safe to merge?', 'audit our CI gates', or 'retrofit CI evidence controls'."
user-invocable: true
---

# CI/CD evidence audit

Use this when an AI agent (or anyone) reports "checks passed" and the question is whether that evidence actually supports merging *this* code. Two modes:

- **Audit**: map a repository's merge path, find where wrong, stale, missing or advisory evidence could still count, and report gaps. Read-only for the audited repository.
- **Retrofit**: add or repair the five control families below in an existing project, as small reviewable changes. Prefer one control per PR. Each change ships with its own challenge tests (valid case, named fault, missing input).

If the user does not say which mode, run Audit first and offer Retrofit for the gaps found.

## Ground rules

- Audit may run local commands and fault injection only in a disposable copy: clone from the repository's local path into a separate directory outside it, remove the clone's remote, explicitly check out the audited SHA and verify `HEAD` equals it before any probe, then work offline. Do not use a `git worktree` (which writes into the audited repository's `.git`). A network deny is an optional extra. Read each repository script before running it. Run with no credentials or push access in the environment, including credential helpers and SSH agents. Audit never pushes. Propose challenges that need platform CI; execute them only as a separately approved push.
- Local runs: if the CI's interpreter or tool version is not installed locally, use the nearest one and label every local result with the version used. Cap every local run's time. macOS has no `timeout`; a portable cap is `perl -e 'alarm 600; exec @ARGV' -- COMMAND ARGS` (change 600 to the seconds allowed; the command is killed when time runs out).
- Inspect before proposing. Read the CI config, test runner config, scripts and docs before suggesting any change.
- A control's verdict must be deterministic: same inputs, same answer. No language-model judgement inside a verdict. AI review can be *evidence* a control checks; it cannot be the control.
- Every control returns one of: `PASS`, `FAIL`, `UNEVAL` (could not establish the answer), `N/A` (known not applicable, with a reason). Missing information is `UNEVAL`, never `PASS` and never silently dropped.
- A documented platform weakness that applies to the configuration you read, but cannot be challenged read-only, is `FAIL`, labelled "configured only, not challenged". Use `UNEVAL` instead only when the configuration itself cannot be read.
- A guard (a check enforcing the pattern) that meets only part of the pattern is `FAIL (partial guard)`; name every missing part (for example "no owner", "no expiry").
- For each requirement, grade each finding in exactly one control: the one whose question it answers. Other controls use `N/A` (see Control N), rather than counting it again.
- An absent control is `FAIL` ("absent") where the failure can happen; it is `N/A` with a reason only where the failure cannot happen.
- Never weaken a gate to get green: do not delete required checks, add `continue-on-error`, `|| true`, `--exit-zero`, skip markers or exceptions without saying so explicitly and getting approval.
- Do not change branch protection, rulesets, secrets, or push/merge without the user's explicit approval for that specific action.
- Do not claim a live merge barrier from repository files alone. Repo files show configured behaviour; branch protection or rulesets show what the platform actually requires. Say which one you checked.
- Clean-room only: write controls from these patterns and the project's own code. Never copy code, documents or know-how from another employer or client.

## Step 0 — Inventory (read-only)

Collect, and cite file paths for, everything below:

- CI definitions: `.github/workflows/*`, `.gitlab-ci.yml`, `Jenkinsfile`, `azure-pipelines.yml`, `.circleci/`, etc.
- Test runner config and the exact test commands each CI job runs (paths, markers, `-k`, shards).
- Local scripts: preflight, pre-push or pre-commit hooks, `Makefile`/`justfile` targets, agent instructions (`AGENTS.md`, `CLAUDE.md`).
- Saved or generated artifacts that later steps reuse: coverage reports, generated docs or inventories, snapshots, lockfiles, SBOMs.
- Any policy file that lists required checks, and the script that evaluates it.
- Platform enforcement, if the user permits API access: read both classic branch protection (`gh api repos/OWNER/REPO/branches/BRANCH/protection`) and rulesets (`gh api repos/OWNER/REPO/rules/branches/BRANCH`). These are separate; the repo-level branch rules call includes organization rules that apply to that branch. If not permitted, record `UNEVAL` and an open question.
- Who can bypass the gate: the `bypass_actors` list of each active ruleset (`gh api repos/OWNER/REPO/rulesets`, then `gh api repos/OWNER/REPO/rulesets/RULESET_ID`). GitHub returns `bypass_actors` only to a caller with write access to the ruleset, so a successful read that omits the field is `UNEVAL`, not an empty list; do not ask for wider permissions in Audit mode. An organization ruleset returning `bypass_actors: []` may only mean the caller cannot see the list; also read `current_user_can_bypass`, which describes only the caller: a returned value other than `never` proves a bypass exists, but `never` does not prove the list is empty, so record `UNEVAL`.
- Access counts: count the accounts with push access and, separately, with admin access, across all result pages, counting only accounts whose `permissions` show it (for example `gh api --paginate --slurp repos/OWNER/REPO/collaborators | jq '[.[][] | select(.permissions.push == true)] | length'`, and `.permissions.admin` for admins; `--jq` runs once per page and prints page-local counts). Record counts, not names. A count alone does not prove bypass authority: relate it to the active rules and their bypass lists. A refused read is `UNEVAL`.
- Read the response message: a protection HTTP 404 explicitly saying the branch is not protected (observed API behaviour, not documented message text) is `FAIL` evidence for missing classic protection, not proof that rulesets are absent. HTTP 403 or any other HTTP 404 is `UNEVAL` (could not check), not evidence of an absent control.

## Step 1 — Draw the evidence path

Write the path for one merge decision:

`Changed input (files, commit) -> Selected work (job, test command) -> Result (record for a version and run) -> Judge (policy + evaluator) -> Decision (required status + merge authority)`

Note which rules judge the decision and where they are loaded from. Note what happens after merge (main CI, deployment) as separate paths; do not let pre-merge evidence stand in for them.

## Step 2 — Pick merge requirements

Choose the requirements that matter most. Start with one; add a second only when a control would otherwise have nothing to test (for example a generated artifact, so Control 1 has something to check). A second requirement chosen only to exercise Control 1 is a probe of Control 1, not a real merge gate; label it so. Then stop. For each, state what must be true, e.g. "the reused coverage report describes the current source set" or "every applicable suite passed for the PR head commit".

Keep one record per merge requirement. Map each control to each requirement and record its verdict. A control that cannot apply to a requirement (for example Control 1 when no saved artifact is reused) is `N/A` with a reason; do not force all five. Roll up applicable verdicts with the worst winning: `FAIL > UNEVAL > PASS`; if all five are `N/A`, the requirement is `N/A` with a reason.

## Step 3 — Check the five control families

For each control: ask the question, probe the repo, record its verdict for each requirement using Step 2's mapping and roll-up, and (Retrofit) add the minimal pattern.

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
Probe: is there any local flow (a script, a skill, a bot, a documented habit) that produces a "checks passed" claim? If yes and there is no preflight, grade `FAIL` (absent); if none, grade `N/A` with that reason. Is there a local script that runs fast checks? Does it keep a required-check list independent of returned results? Does it compare `git rev-parse HEAD` with the PR head (`gh pr view --json headRefOid -q .headRefOid`)? What does its exit code mean, and does the caller stop on non-zero?
Pattern:
- A required-check list; each check emits one verdict line; the summary adds `UNEVAL` for any required check with no verdict.
- Exit codes: `0` all pass or justified N/A; `1` a completed FAIL in strict mode; `3` any UNEVAL (takes priority within the script's own exit-code contract, separate from the audit roll-up). Default mode may be advisory, but say so in the output.
- Head comparison: equal means continue; both valid and different means N/A for remote evidence (push, then re-obtain evidence and review); missing or malformed IDs mean UNEVAL.
- Any automation that calls preflight (push scripts, agent loops) must use strict mode and stop on non-zero.
Challenge: delete one required check's output (expect UNEVAL, exit 3); create an unpushed commit (expect N/A for remote evidence); blank the PR head ID (expect UNEVAL). A refusal on missing input that stops the caller with a non-zero exit meets the challenge (fails closed); refusing is not a defect.

### 3. Test collection

Question: does a triggered CI job actually select each in-scope test?
Fault: a test passes when run directly but sits outside every CI job's selection.
Probe:
- PR audit: list added or changed test files (`git diff --name-only --diff-filter=AMR BASE...HEAD`). Compare them with the union of what the registered PR jobs collect.
- Branch audit: enumerate every tracked test file (`git ls-files`, using the project's test-file conventions). Compare with the union of what each applicable job collects, not just the PR diff.
- Run each job's exact command in the shell the job uses (for example the workflow's `shell:` setting), not your interactive shell, using the runner's list mode (`pytest --collect-only -q ...`, `jest --listTests ...`, `go test -list . ...`). A plausible count does not prove the flags applied; cross-check with one path the job should exclude, or record why none exists.
- A framework with no list mode (for example Python `unittest`): count by calling the same discovery the CI uses (same loader, start folder and file pattern) and print the count. Compare it with the count in a CI log.
Pattern: a CI check that fails when an in-scope test file is collected by no applicable job. A collection or import error is UNEVAL, not "no route". Exceptions live in a reviewed file with path, reason, owner and expiry.
Challenge: add a test under an unselected directory (expect FAIL); add one under a selected directory (expect PASS); add one with a broken import (expect UNEVAL); make the selection empty (expect not PASS: a runner that prints "0 tests ... OK" and exits 0 must not count as a pass).
Limit: collection is not execution. The audited PR or branch must also trigger each counted job, and the job must run.

### 4. Required CI results

Question: did every applicable suite pass for the intended commit, judged by rules the PR cannot edit?
Faults: a success from another commit or older attempt counts; a missing suite is treated as fine; the PR edits the policy or evaluator to drop its own failing requirement.
**Probe:**
- Is there one aggregate gate? Where does its policy come from? Does it match results by head SHA, event, current run attempt and current Ready cycle? A gate reading `needs.*.result` of its own run is bound to its head SHA and event by construction, and to its attempt except for jobs kept from an earlier attempt by "Re-run failed jobs"; check those retained jobs by comparing each needed job's `run_attempt` with the gate's own attempt (use the run-level job listing with `filter=all` below). Check the Ready cycle too. Is it required by branch protection or rulesets? Which required jobs can skip or conclude neutral?
- A Ready cycle starts when the PR is opened non-draft, or at its latest `ready_for_review` event if it was ever a draft; it ends when the PR returns to draft (`convert_to_draft`) or closes. A non-draft PR with a complete history and no `ready_for_review` event has been ready since opening, not `UNEVAL`; a `reopened` event starts a new Ready cycle, excluding time while closed. A PR reopened while still a draft starts no cycle until it is marked ready (`ready_for_review`).
- Read `gh api --paginate repos/OWNER/REPO/issues/NUMBER/events` and PR metadata (`gh api repos/OWNER/REPO/pulls/NUMBER`) for opening time and draft state; compare cycle boundaries with run creation/start times. Missing metadata or incomplete history is `UNEVAL`.

**Platform facts:**
- Under `pull_request`, GitHub runs the workflow definition from the PR itself, so a PR can change its own gate. Under `push`, it runs the definition from the pushed commit, so a push can change its own gate too; a gate that runs only on `push` is not safe for that reason.
- A commit whose message holds a CI-skip marker (`[skip ci]`, `[ci skip]`, `[no ci]`, `[skip actions]`, `[actions skip]`) starts no `push` or `pull_request` run: for `push` the marker counts in the pushed commit's message, for `pull_request` only in the PR's head commit message. GitHub documents this for those two events only; do not assume it stops other events such as `pull_request_target`. Without a required status, those skipped runs leave no barrier to merging that commit (other restrictions may still apply). Check whether the project's own scripts add such markers.
- On GitHub, a job skipped by a job-level `if:` reports success for branch protection without running. A workflow not triggered because of path or branch filtering leaves required checks pending and blocks merging. GitHub also accepts skipped and neutral check conclusions; platform acceptance alone is not proof a required suite executed.
- An aggregate gate job that `needs:` suite jobs can itself be skipped after a suite fails. Use `if: always()` so the gate runs and fails unless every needed result is `success`, or have an independent gate read results through the API.

**How to grade:**
- For an applicable platform weakness above, apply the Ground rules grade: `FAIL`, "configured only, not challenged".
- No gate and no required status at all: Control 4 is `FAIL` (absent). List the Ready-cycle and run-attempt checks under "Not checked": with nothing that consumes them, they cannot change the verdict.
- A gate that exists but does not depend on the Ready cycle is `FAIL (partial guard)` for that check, noting the missing Ready-cycle check. From sampled history, report whether any merge actually relied on draft-stage evidence; that fact goes in the note and does not change the grade.

**Pattern:**
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

Question: does a failing command make its step, its job and the check the job reports fail?
Scope: this control judges the chain command exit -> step -> job conclusion -> reported check. Whether that check is required, and so changes the merge decision, is Control 4's finding. Mark that finding `N/A (see Control 4)`; that does not make this control `N/A`, which still answers its own question.
Fault: a command fails, but the step, job or pipeline is configured to continue, so the job ends green.
Probe: search CI and scripts for `continue-on-error`, `allow_failure` (GitLab), `catchError` / `unstable` (Jenkins), `|| true`, `set +e`, `--exit-zero`, `if: always()` on gating steps, and pipes without `pipefail` (`cmd | tee log`). For each hit, trace the chain above. Record whether the advisory effect is intentional and documented.
Pattern: make the intended effect explicit per check and event. If a finding must block, remove the continuation for that event or route the finding into the aggregate gate. Never flip an intentionally advisory check to blocking (or the reverse) without approval.
Challenge: propose injecting a failure into the command on a disposable branch and checking that the job and its reported check both turn red on events that should block. Running this through platform CI needs explicit approval for the push; Audit does not push.

### Observed history (read-only)

With permitted API read access, run these reads from the auditor's normal environment, not the offline copy. This is observed evidence for the five controls, mainly Control 4, not a sixth control; place findings in Step 5's evidence path, challenge and verdict fields.

Sample merged PRs and their check rollups (`gh pr list --state merged --json number,mergedAt,headRefOid,statusCheckRollup`) and recent default-branch push runs (`gh run list --branch BRANCH --event push`). Record the sample window and commit/run IDs. Compare run completion times with merge times; a rollup read today does not prove what was visible at merge.

Read each relevant ruleset's `created_at` and `updated_at`. Count sampled merges before and after its creation; merges before the rule existed are not evidence that it protects merges. Record updates as limits on what the sample proves about the current configuration. Classic branch protection has no `created_at`; record its age as `UNEVAL`.

Also list the default-branch commits that did not come from a PR (compare `git log --first-parent BRANCH` with the merged-PR list, or ask `gh api repos/OWNER/REPO/commits/SHA/pulls`) and the commits with a CI-skip marker in the message. Record what each touched (`git diff-tree --no-commit-id --name-only -r SHA`). Group repeated automated commits by subject and touched paths instead of listing each one.

For red runs, read the run-level job listing (`gh api --paginate 'repos/OWNER/REPO/actions/runs/RUN_ID/jobs?filter=all'`; `filter=all` includes earlier attempts) and the suite step's conclusion there, not only the single-job endpoint, the job conclusion or a step count. A failed or cancelled job with no steps is `UNEVAL`. Use logs to distinguish test failures from infrastructure failures; cancelled/skipped tests or failed setup do not prove a test failed. Expired logs (HTTP 410, observed API behaviour, not documented for log downloads) are `UNEVAL`, with that reason. Leave unclear causes `UNEVAL`.

## Step 4 — Challenge every control

For each control, run three cases within the approved scope: valid input, one named fault, missing input. Use a disposable copy for local fault injection; never leave the injected fault in the working tree. Propose platform-CI cases rather than pushing in Audit; without an approved run, grade documented platform weaknesses under Ground rules as `FAIL`, "configured only, not challenged", not observed enforcement. A control that also passes the faulty case is not a control. Record commands and outcomes. If you could not run a case, say so; do not infer the result.

In Audit, when a platform challenge needs a push, default to the next-best local probe: run the repository's own judge or test with fake inputs, or its own control with a deliberate fault, in the disposable copy. Label this local evidence, not observed platform enforcement; if no safe local probe is possible, record that limit.

For example, extract the step's `run:` block (needs PyYAML): `python3 -c 'import sys,yaml; print(yaml.safe_load(sys.stdin)["jobs"][sys.argv[1]]["steps"][int(sys.argv[2])]["run"], end="")' JOB_ID STEP_INDEX < WORKFLOW > step-script` (substitute the job ID, zero-based step index and workflow path). Stop if the command exits non-zero or `step-script` is empty. If you also read the triggers with PyYAML, the key is `True`, not `on`. Replace every `${{ }}` expression with its recorded value. Export workflow, job and step `env:` values (most specific wins), and use the effective `working-directory` and `defaults.run` with step overrides. Run the script in the disposable copy with the job's exact shell flags and the time cap above. The shell is usually `bash`: `bash -e` for default Linux/macOS Bash; `bash --noprofile --norc -eo pipefail` for explicit `shell: bash`. For `pwsh`, `python` or another shell, use GitHub's documented command line for that shell. Plain `bash script` can turn a failing step into a pass.

## Step 5 — Record the audit

Write the worksheet outside the audited repository unless the user asks for it in a PR. Start with:

- **Overall verdict:** PASS / FAIL / UNEVAL / N/A; the same roll-up rule applies across requirements: `FAIL > UNEVAL > PASS` (all N/A needs a reason).
- **Not checked:** each omitted read or challenge, why, and who can resolve it.

Include one row per merge requirement, with each control's verdict (`N/A` with a reason where a control cannot apply) and Step 2's roll-up:

| Field | Content |
|---|---|
| Decision / requirement | What must be true for this merge |
| Evidence path | Input -> work -> result -> judge -> decision, with file paths |
| Failure / effect | How the evidence could look acceptable while the claim is false |
| Control | What checks it, how it is triggered, and what consumes its answer |
| Challenge | Valid / fault / missing: expected vs observed, with commands |
| Verdict | Five control verdicts and rolled-up requirement verdict; proof kind for each (source read, local run, CI run, platform API) |
| Open question and owner | What remains unproved and who resolves it |
| Revisit trigger | Code, tests, selection, policy or trigger changes that require a re-audit |

End with a ranked gap list (blocking first: highest risk that a failure can reach the default branch unnoticed) and, for Retrofit, one proposed PR per gap: files to change, the challenge tests it adds, and the risk of the change.

## Reporting language

- Say "deterministic, rule-based check" for the controls, and name what they read. Do not say the controls prove the code is correct, or that flaky tests become reliable.
- Distinguish configured behaviour (read from files) from observed behaviour (a run you executed or a platform API you queried).
- No compliance, certification or effectiveness claims. The method adapts process failure-mode analysis; it is not a formal assessment.
