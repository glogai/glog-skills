---
name: glog
description: Glog.AI scan + SARIF validation + focused remediation (no SARIF edits, no unrelated refactors). Writes report to .glog and cleans temporary artifacts while preserving SARIF and the remediation report.
tools:
  - shell
  - filesystem
---

# Setup: obtain glog-action into cache

Obtain glog-action from GitHub and store it in a cache directory.

Hardcoded repository:
- Repo URL: `https://github.com/glogai/glog-action`
- Branch/ref: `main`

Cache location (prefer in this order):
1) If `$XDG_CACHE_HOME` is set: `$XDG_CACHE_HOME/glog/glog-action`
2) Else: `~/.cache/glog/glog-action`
3) If home is not available: `/tmp/glog/glog-action`

Auth rules (private repo):
- Use `GITHUB_TOKEN` for git HTTPS authentication.
- Do NOT put the token in the URL.
- Do NOT echo or log the token.
- Use git with an Authorization header (`http.extraHeader`).

Rules:
- Ensure `GITHUB_TOKEN` is set, otherwise stop with a clear error.
- If the cache folder does not exist: clone the repo into it.
- If it exists: perform a `git fetch` and `git reset --hard origin/main` to ensure it matches the configured ref.
- The cached repo must contain `CLI.md`. If it does not, stop with a clear error.
- Do not prompt the user for repo URL, ref, or path.
- Do not write any secrets to disk. Do not echo tokens.

After preparing the cache repo, define:

`<GLOG_ACTION_PATH> = <cache_path>`

and proceed with the workflow using `<GLOG_ACTION_PATH>/CLI.md`.

# Remediation strategy configuration (REQUIRED)

Before doing ANY scan work, ask the user:

"How should remediations be applied?

Reply with one of:

local → Apply all fixes locally in the current branch (no commits, no pull requests).

pr-per-finding → Create a separate branch and Pull Request for EACH genuine vulnerability finding.

Type exactly: local or pr-per-finding"

Rules:
- The user response is REQUIRED (the agent must ask and wait).
- If the response is anything other than `local` or `pr-per-finding`, ask again.
- Do not assume a default.
- Store the value as `<REMEDIATION_MODE>`.

Behavior:

If `<REMEDIATION_MODE>` = `local`
→ Apply all remediations directly in the current working branch.
→ Leave changes uncommitted in the working tree.

If `<REMEDIATION_MODE>` = `pr-per-finding`
→ For EACH genuine finding:
  - Create a new branch:
    `glog/remediation/<sanitized-ruleId>-<short-hash>`
  - Apply only the fix related to that finding.
  - Commit with message:
    `fix(security): remediate <ruleId> at <file>`
  - Push branch using `GITHUB_TOKEN`.
  - Open a Pull Request targeting the original branch.
  - Continue to next finding.

# Scan configuration

Before doing ANY scan work, ask the user:

"Which language should be used for scanning? (This controls the optional --lang flag.)
Reply with a language value like: python, csharp, java, javascript, etc.
If you want to skip, reply with: skip"

Rules:
- The user response is required (the agent must ask and wait).
- If the user replies with `skip` (or an empty/whitespace-only response), DO NOT pass `--lang` at all.
- Otherwise pass `--lang <user_value>` exactly as provided (trim whitespace).
- Do not invent or default a language.
- Keep other flags hardcoded:
  - `--client test`
  - `--env dev`
  - `--sarif-format-type STANDARD`

# Purpose

Use glog-action CLI instructions to run a scan for the CURRENT project. When possible, scan only files changed in the current branch relative to the best available base reference by preparing a temporary filtered workspace. Then analyze findings from `.glog/glog-scan.sarif`, validate each finding, and apply focused remediations where genuine.

# Required environment

Use global environment variables (do not prompt for them unless missing):
- GLOG_TOKEN
- GITHUB_TOKEN
- GITHUB_USER

If any is missing, stop and tell the user exactly which one(s) are missing and how to export them.

If `<REMEDIATION_MODE>` = `pr-per-finding`, one of the following must be available:
- GitHub CLI (`gh`), OR
- GitHub REST API access via `curl` using `GITHUB_TOKEN`

If neither is available, stop with a clear error.

# Critical constraints (must follow)

- Before analysis starts, clean `.glog` directory.
- Prefer scanning only changed files by using a temporary filtered workspace when a reliable base reference can be determined.
- Fall back to a full-project scan only when changed-file scanning is not reliable or not possible.
- Run scan with the hardcoded flags:
  - `--client test`
  - `--env dev`
  - `--sarif-format-type STANDARD`
  - `--lang <user_value>` ONLY if the user provided a language (not skip/empty). Otherwise do not include `--lang` at all.
- After scan is finished:
  - DO NOT modify `.glog/glog-scan.sarif` (read-only after creation).
- Treat remediation advice as a focused security remediation task.
- If the selected remediation flow is `local`, changes must remain uncommitted in the local working tree only. Do NOT create commits, do NOT push, do NOT open pull requests, and do NOT perform any git write operation that records or publishes changes.
- If the selected remediation flow is `pr-per-finding`, isolate each genuine finding in its own branch and PR.
- Validate whether each finding is genuine:
  - If NOT genuine: explain why, for additional review, then continue.
  - If genuine: remediate as suggested in SARIF as much as possible, adjusted to the project context and stack.
- Do not duplicate code:
  - Reuse already existing methods/utilities if possible.
  - Do not duplicate modules and do not copy-paste logic across modules.
- After remediation:
  - Optimize ONLY the code touched by the fixes.
  - Deduplicate ONLY where it directly relates to the fixes.
  - Do not refactor unrelated code.
- Generate a remediation report file inside `.glog` instead of only printing results in chat.
- The report must be saved as `.glog/glog-remediation-report.md`.

# Execution workflow

## Step 1: Read glog-action CLI.md (source of truth)

- Open and read: `<GLOG_ACTION_PATH>/CLI.md`
- Identify the exact command(s) to run `glog-action` (entrypoint, required args, docker usage, path handling, output behavior, etc.)
- Follow CLI.md instructions strictly.
- Confirm how the scan target path should be supplied.
- Confirm how the SARIF output is expected to land in the current project's `.glog/` directory.

## Step 1.5: Prepare filtered workspace from changed files

Before running the scan, prepare a temporary filtered workspace that contains only files changed in the current branch relative to the best available base reference.

Goal:
- Scan only the developer's current branch changes when possible.
- Reduce noise from unrelated code.
- Avoid unpredictable behavior by falling back safely when change detection is not reliable.

Temporary workspace:
- Create a temporary directory inside the current project root:
  - `.glog-filtered-src/`

Rules:
- Preserve relative paths when copying files.
- Include only changed files that currently exist in the working tree.
- Do not copy deleted files.
- Do not modify the original project files during workspace preparation.
- Do not use git commit, git push, or PR operations during this step.
- Do not include `.git`, `.glog`, or `.glog-filtered-src` as scan source content.
- If renamed files are reported by git, include the new file path if it exists in the working tree.
- If no reliable base reference can be determined, do not guess aggressively; fall back to scanning the full project and document the fallback in the report.

How to determine the best available base reference:
- First, prefer the upstream tracking branch of the current branch, if it exists and is a suitable comparison base.
- Otherwise prefer one of these remote refs if present:
  - `origin/main`
  - `origin/master`
- If none of the above exists or can be fetched/referenced safely, no reliable base reference is available.

Recommended diff scope:
- Use changed files from the current branch compared to the base reference.
- Include file statuses that represent files currently present and relevant to scanning:
  - Added
  - Copied
  - Modified
  - Renamed
- Exclude:
  - Deleted files
  - Missing paths
  - Generated artifacts inside `.glog` or `.glog-filtered-src`

Workspace preparation process:
1) Remove any previous `.glog-filtered-src/` directory.
2) Recreate `.glog-filtered-src/`.
3) Determine the best available base reference.
4) Collect changed files using git diff.
5) Filter the changed paths:
   - keep only files that currently exist
   - skip directories
   - skip files under `.git/`, `.glog/`, and `.glog-filtered-src/`
6) For each remaining changed file:
   - create its parent directory inside `.glog-filtered-src/`
   - copy the file into the filtered workspace preserving relative path
7) If no usable changed files are found:
   - do not fail
   - fall back to scanning the full current project
   - record this fallback in the report

Verification of filtered workspace:
- If `.glog-filtered-src/` contains at least one copied file, use it as the scan target.
- Otherwise use the current project root as the scan target.

Define:
- `<SCAN_TARGET_PATH>` = `.glog-filtered-src` if filtered workspace is usable
- otherwise `<SCAN_TARGET_PATH>` = current project root

## Step 2: Clean .glog and run scan

From the CURRENT project root (the repo you want to scan), do:

1) Clean `.glog`:
- If `.glog` exists: remove it completely
- Recreate `.glog/` directory

2) Execute scan using the invocation defined in CLI.md.
- Use `<SCAN_TARGET_PATH>` as the scan path.
- `<SCAN_TARGET_PATH>` is:
  - `.glog-filtered-src` if changed files were successfully prepared
  - otherwise the current project root
- Apply flags exactly:
  - `--client test`
  - `--env dev`
  - `--sarif-format-type STANDARD`
  - Apply `--lang <value>` ONLY if the user provided a language (not skip/empty). If skipped, do not pass `--lang`.

3) If CLI.md expects the runner script to be executed from inside glog-action repo, run it from there but target `<SCAN_TARGET_PATH>` exactly as specified by CLI.md.

4) Ensure that the output SARIF file ends up at:
- `.glog/glog-scan.sarif` in the CURRENT project.

After scan:
- Verify `.glog/glog-scan.sarif` exists.
- If missing, stop and show the scan command output and your best diagnosis.
- Record whether the scan target was:
  - filtered workspace only
  - full project fallback

## Step 3: Analyze findings from SARIF (read-only)

- Open `.glog/glog-scan.sarif` read-only.
- Summarize findings:
  - ruleId / title (if present)
  - severity / level (if present)
  - message
  - locations (file + line/region)
  - suggested remediation text (if present)

## Step 3.1: Determine remediation instructions source (SARIF)

Important: The SARIF file does NOT populate `result.fixes`.

Instead, remediation guidance is located in:
- `result.message.markdown` (primary)
- `rule.help.markdown` or `rule.help.text` (secondary fallback)

Rules:
- Prefer remediation steps explicitly described in `result.message.markdown`.
- If markdown contains multiple suggestions, pick the one that best matches:
  - the reported file path / code location
  - the project's technology stack
  - the vulnerability type described by the ruleId and message
- If there is no clear remediation text, treat the finding as `needs manual review` and explain why.

## Step 4: Validate + Remediate (mode-aware, markdown-driven remediation)

For each finding:

### 4.0 Remediation flow guardrail

- If remediation flow is `local`, apply fixes only in the local workspace and leave all changes uncommitted.
- In `local` mode, do not run:
  - `git commit`
  - `git push`
  - `git merge`
  - `git rebase`
  - `git cherry-pick`
  - PR creation commands
  - any git write operation that records or publishes changes
- If remediation flow is `pr-per-finding`, isolate each genuine finding and do not combine fixes across findings.

### 4.1 Validate whether the finding is genuine

- Inspect the exact location(s) in code referenced by the SARIF.
- Consider runtime behavior, framework defaults, and existing mitigations.
- If NOT genuine:
  - Explain why clearly for additional review.
  - Reference the relevant code paths and reasoning.
  - Continue to the next finding.

### 4.2 Extract remediation steps

- Parse `result.message.markdown` and identify actionable remediation steps.
- If remediation is vague, infer a concrete plan that stays consistent with the intent of the markdown guidance.
- Do not invent unrelated refactors.

### 4.3 Apply remediation (based on `<REMEDIATION_MODE>`)

If `<REMEDIATION_MODE>` = `local`:

- Apply fixes in the current working tree only.
- Keep all changes local and uncommitted.
- Do not create branches, commits, pushes, or pull requests.
- Continue through all findings in the same working tree.

If `<REMEDIATION_MODE>` = `pr-per-finding`:

For EACH genuine finding:

1) Ensure the working tree is clean before starting that finding's isolated branch flow.
2) Create branch:
   `glog/remediation/<sanitized-ruleId>-<short-hash>`
3) Apply ONLY that finding's fix.
4) Verify build/tests.
5) Commit with a finding-specific message.
6) Push the branch.
7) Open a Pull Request targeting the original branch.
8) Switch back to the original branch.
9) Reset the tree so the next finding starts clean.

Rules:
- Do NOT combine findings.
- Do NOT modify SARIF.
- Log PR failures and continue to the next finding.
- If a finding cannot be safely isolated for PR creation, document the reason clearly.

### 4.4 Verify remediation is correct for this codebase

After implementing a fix, verify it using the strongest applicable checks available in the current environment.

Verification rules:
- Confirm the vulnerable pattern is no longer present at the reported location.
- Re-read the affected code path and ensure the fix matches the SARIF finding and remediation intent.
- Run the relevant build/compile command for the project, if available.
- Run relevant existing tests, if present.

Testing guidance:
- Add a minimal targeted test only if it meaningfully verifies the remediation.
- Prefer a unit test when the fix is isolated to business logic, validation, sanitization, parsing, authorization checks in service logic, or reusable helper methods.
- Prefer an integration, web-layer, repository, or framework-level test when the issue depends on runtime wiring, HTTP behavior, persistence, templating/rendering, authentication/authorization configuration, or other framework behavior.
- Do not create a unit test by default if it cannot realistically verify the security behavior that was fixed.
- Choose the narrowest test type that can genuinely validate the remediation.
- Do not add broad or unrelated test coverage.

Behavioral verification:
- Verify that malicious, unsafe, or invalid input is now rejected, sanitized, escaped, validated, or safely handled as appropriate.
- Verify that expected valid behavior still works after the change.
- Ensure the fix does not introduce regressions or break public API behavior.

Verification result:
- For each finding, record verification as one of:
  - `verified`
  - `partially verified`
  - `not fully verified`

If full verification is not possible in the current environment, clearly explain what was verified, what was not verified, and why.
A finding must not be marked as fully remediated unless the agent has validated the changed code path and completed the strongest verification reasonably available.

### 4.5 If remediation guidance is incorrect or incomplete, adapt it

If the markdown remediation does NOT fit the actual code/context (for example wrong framework assumption, wrong API usage, breaking change, or not actually resolving the issue):

- Modify the remediation to a correct variant for this project, while keeping the original intent.
- Clearly document in the report:
  - what the markdown suggested
  - why it did not fit
  - what was changed instead and why

Stop short of broad refactors; change only what is needed to properly remediate the finding.

## Step 5: Limited optimization

- Optimize only code touched by the fixes.
- Deduplicate only what was introduced or directly affected by the fixes.
- Do not perform unrelated cleanup or refactoring.

## Step 6: Final report

Create:
- `.glog/glog-remediation-report.md`

The report must contain:
- Scan metadata:
  - date/time
  - selected language
  - client
  - env
  - remediation mode used: `<REMEDIATION_MODE>`
- Scan target details:
  - scan target type (`filtered workspace` or `full project fallback`)
  - base reference used for change detection, if available
  - number of changed files copied into the filtered workspace, if available
  - whether fallback to full-project scan was required
- Summary of findings from `.glog/glog-scan.sarif`
- For each finding:
  - genuine vs not genuine
  - rationale
  - files impacted
  - remediation performed (if any)
  - remediation source: `result.message.markdown` or fallback source
  - whether remediation guidance was adapted
  - if adapted: what changed and why
  - verification performed
  - test type used, if any: `unit`, `integration`, `web`, `repository`, or `none`
  - verification result: `verified`, `partially verified`, or `not fully verified`
  - evidence: build output, test results, code-path inspection, or reasoning
  - remaining risk or limitations, if any
- List of code changes
- Any follow-up recommendations
- If `pr-per-finding`:
  - list of PR URLs
  - failed PR attempts and reasons

After saving the report file:
- Provide a short summary in chat.
- Do not print the full report content in chat unless the user asks.

## Cleanup permission boundary

The agent is explicitly allowed to delete files and directories inside the project's `.glog` directory and the temporary `.glog-filtered-src` directory as part of this workflow cleanup.

This deletion is intentional housekeeping and not destructive user data removal.

Allowed cleanup scope:
- `.glog/**`
- `.glog-filtered-src/**`

Rules:
- Never delete files outside `.glog` or `.glog-filtered-src`.
- Only generated scan/remediation artifacts and temporary filtered workspace contents may be removed.
- The preserved files must remain intact:
  - `.glog/glog-scan.sarif`
  - `.glog/glog-remediation-report.md`

Policy note:
- Removal of extra `.glog` artifacts and removal of `.glog-filtered-src` are explicitly permitted for this skill and must not be blocked as destructive operations.

## Step 7: Cleanup temporary artifacts (preserve SARIF and report)

This workflow explicitly authorizes cleanup inside `.glog` and `.glog-filtered-src`.

Files that must be preserved:
- `.glog/glog-scan.sarif`
- `.glog/glog-remediation-report.md`

Cleanup rules:
- Delete any other file or subdirectory inside `.glog`.
- `.glog-filtered-src` is temporary and may be removed completely.
- This deletion is allowed and expected for this skill.
- Do not treat removal of extra temporary artifacts as a destructive action requiring additional confirmation.
- Do not delete or modify anything outside `.glog` or `.glog-filtered-src`.
- Cleanup must not use git commands that create, record, or publish changes.

Process:
1) Ensure both preserved files exist.
   - If any is missing, warn the user and do not delete `.glog`.
2) Temporarily move both preserved files outside `.glog`.
3) Remove the entire `.glog` directory.
4) Recreate `.glog`.
5) Move preserved files back.
6) Remove `.glog-filtered-src` completely if it exists.

If cleanup still cannot be completed due to environment or policy enforcement, clearly report which extra artifacts remain.

# Implementation notes

- Use `http.extraHeader` with `GITHUB_TOKEN` for git HTTPS operations against the private glog-action repository.
- Never print tokens.
- Do not modify SARIF after scan.
- Use safe shell practices.
- To prepare the filtered workspace, use git diff against the best available base reference and copy only changed files that still exist in the working tree.
- Prefer diff filters equivalent to Added, Copied, Modified, and Renamed files.
- Preserve directory structure when copying into `.glog-filtered-src`.
- If no changed files are detected, or if no suitable base reference is available, fall back to scanning the full project and document the fallback in the report.
- Do not scan `.git`, `.glog`, or `.glog-filtered-src` as source content unless explicitly required by the project layout.
- After the workflow completes, `.glog-filtered-src` may be removed as a temporary artifact.
- If the current branch has an upstream tracking branch, verify that it is a sensible comparison base before using it as the base reference.
- If the repository state is too unusual to determine changed files safely, prefer full-project fallback over risky assumptions.
- In `local` mode, do not stage or commit report files.
- In `pr-per-finding` mode, each PR should contain only the minimum code required for that finding.