---
name: glog
description: Glog.AI scan + SARIF validation + focused remediation (no SARIF edits, no unrelated refactors). Writes report to .glog and cleans artifacts (preserving SARIF + report).
tools:
  - shell
  - filesystem
---

# Setup: obtain glog-action into cache

Obtain glog-action from GitHub and store it in a cache directory.

Hardcoded repository:
- Repo URL: `https://github.com/glogai/glog-action`
- Branch/ref: `main`

For all Glog skills and actions use environment variables available in the user's shell environment.

These variables may be defined in shell configuration files such as ~/.profile, ~/.zprofile, ~/.bash_profile, or configured via system environment variables (for example on Windows).

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

# Interactive question ordering (STRICT)

The agent must ask configuration questions sequentially, one at a time.

Rules:
- NEVER ask multiple configuration questions in the same message.
- Ask the first required question and WAIT for the user response.
- Do not continue the workflow until the user has answered that question.
- After receiving a valid answer, store the value and only then ask the next question.
- Do not repeat a question if a valid answer has already been provided.
- Do not ask the second question before the first one is answered.

Strict order of questions:

1️⃣ Remediation strategy  
2️⃣ Scan language

Execution protocol:

Step A — Ask remediation strategy question only.

Wait for the user response.

If the response is valid (`local` or `pr-per-finding`):
- store `<REMEDIATION_MODE>`
- proceed to Step B

If invalid:
- ask the remediation strategy question again
- do NOT continue

Step B — Ask scan language question only.

Wait for the user response.

If the response is valid:
- store `<SCAN_LANGUAGE>`
- proceed to scanning workflow

If invalid:
- ask the scan language question again
- do NOT restart Step A

The agent must never ask both questions in the same message.
The agent must never re-ask the language question if it was already answered.

# Remediation strategy configuration (REQUIRED)

Before doing ANY scan work, ask the user:

"How should remediations be applied?

Reply with one of:

local → Apply all fixes locally in the current branch (no pull requests).

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

This question must be asked first according to the Interactive question ordering rules.
Do not ask any other configuration question before this one is answered.

# Scan configuration

This question must be asked only AFTER the remediation strategy question has been answered.
Follow the Interactive question ordering rules strictly.

Before doing ANY scan work, ask the user:

"Which language should be used for scanning? (This controls the optional --lang flag.)  
Reply with a language value like: python, csharp, java, javascript, etc.  
If you want to skip, reply with: skip"

Rules:
- The user response is required (the agent must ask and wait).
- If the user replies with "skip" (or an empty/whitespace-only response), DO NOT pass `--lang` at all.
- Otherwise pass `--lang <user_value>` exactly as provided (trim whitespace).
- Do not invent or default a language.
- Do not hardcode client or env values in this skill.
- `--client` must be sourced from the environment variable `GLOG_CLIENT`.
- `--env` must be sourced from the environment variable `GLOG_ENV`.
- Both `GLOG_CLIENT` and `GLOG_ENV` must already be set before the skill starts execution.
- `--sarif-format-type STANDARD` remains hardcoded.

# Purpose

Use glog-action CLI instructions to run a scan for the CURRENT project, then analyze findings from `.glog/glog-scan.sarif`, validate each finding, and apply focused remediations where genuine.

# Required environment

Use global environment variables (do not prompt for them unless missing):
- GLOG_TOKEN
- GITHUB_TOKEN
- GITHUB_USER
- GLOG_CLIENT
- GLOG_ENV

If any is missing, stop and tell the user exactly which one(s) are missing and how to export them.

If `<REMEDIATION_MODE>` = `pr-per-finding`, one of the following must be available:

- GitHub CLI (`gh`), OR
- GitHub REST API access via `curl` using `GITHUB_TOKEN`

If neither is available, stop with a clear error.

Docker authentication recovery rule:
- If a Docker pull, run, or other GHCR operation fails with an authentication or authorization error for `ghcr.io`, first verify that both `GITHUB_TOKEN` and `GITHUB_USER` are available in the current shell environment.
- On Windows PowerShell, recover by logging in to GitHub Container Registry with the equivalent of:
  `$env:GITHUB_TOKEN | docker login ghcr.io -u $env:GITHUB_USER --password-stdin`
- Retry the failed GHCR Docker operation only after a successful Docker login.
- Do not print the token.
- Do not write the token to disk.
- If Docker login fails, stop and report that GHCR authentication failed.
- Perform GHCR Docker login only when an actual `ghcr.io` authentication/authorization error occurs, not preemptively.

# Critical constraints (must follow)

- Before analysis starts, clean `.glog` directory.
- Run scan with the required flags:
  - `--client <value from GLOG_CLIENT>`
  - `--env <value from GLOG_ENV>`
  - `--sarif-format-type STANDARD`
  - `--lang <user_value>` ONLY if the user provided a language (not skip/empty). Otherwise do not include `--lang` at all.
- `GLOG_CLIENT` and `GLOG_ENV` must be validated before any scan execution begins.
- If either `GLOG_CLIENT` or `GLOG_ENV` is missing, stop immediately and do not run the engine.
- Do not substitute hardcoded fallback values for client or env.
- After scan is finished:
  - DO NOT modify `.glog/glog-scan.sarif` (read-only after creation).
- Treat remediation advice as a focused security remediation task.
- If the selected remediation flow is local, changes must remain uncommitted in the local working tree only. Do NOT create commits, do NOT push, do NOT open pull requests, and do NOT perform any git write operation that records or publishes changes.
- Validate whether each finding is genuine:
  - If NOT genuine: explain why, for additional review, then continue.
  - If genuine: remediate as suggested in SARIF as much as possible, adjusted to the project context/stack.
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
- Identify the exact command(s) to run `glog-action` (entrypoint, required args, docker usage, etc.)
- Follow CLI.md instructions strictly.

## Step 2: Clean .glog and run scan

From the CURRENT project root (the repo you want to scan), do:

1) Clean `.glog`:
- If `.glog` exists: remove it completely
- Recreate `.glog/` directory

2) Execute scan using the invocation defined in CLI.md.
- Apply flags exactly:
  - `--client <value from GLOG_CLIENT>`
  - `--env <value from GLOG_ENV>`
  - `--sarif-format-type STANDARD`
  - Apply `--lang <value>` ONLY if the user provided a language (not skip/empty). If skipped, do not pass `--lang`.
- If CLI.md expects the runner script to be executed from inside glog-action repo, run it from there but target the current project as specified by CLI.md.
- Ensure that the output SARIF file ends up at: `.glog/glog-scan.sarif` in the CURRENT project.

After scan, verify `.glog/glog-scan.sarif` exists.
If missing, stop and show the scan command output and your best diagnosis.

## Step 3: Analyze findings from SARIF (read-only)

- Open `.glog/glog-scan.sarif` read-only.
- Summarize findings:
  - ruleId / title
  - severity/level
  - message
  - locations
  - suggested remediation text

## Step 3.1: Determine remediation instructions source (SARIF)

Important: The SARIF file does NOT populate `result.fixes`.

Instead, remediation guidance is located in:
- `result.message.markdown` (primary)
- `rule.help.markdown` or `rule.help.text` (secondary fallback)

## Step 4: Validate + Remediate (mode-aware, markdown-driven remediation)

For each finding:

### 4.0 Remediation flow guardrail
- If remediation flow is local, apply fixes only in the local workspace and leave all changes uncommitted. Do not run git commit, git push, git merge, git rebase, git cherry-pick, or create/open PRs.

### 4.1 Validate whether the finding is genuine

- Inspect referenced code.
- If NOT genuine:
  - Explain why.
  - Continue.

### 4.2 Extract remediation steps

- Parse `result.message.markdown`.
- Infer concrete plan if vague.

### 4.3 Apply remediation (based on `<REMEDIATION_MODE>`)

If `<REMEDIATION_MODE>` = `local`:

- Apply fixes in working tree.
- After all findings:
  - Verify build/tests.
  - Commit once:
    `fix(security): glog remediation batch`

If `<REMEDIATION_MODE>` = `pr-per-finding`:

For EACH genuine finding:

1) Ensure working tree is clean.
2) Create branch:
   `glog/remediation/<sanitized-ruleId>-<short-hash>`
3) Apply ONLY that finding's fix.
4) Verify build/tests.
5) Commit.
6) Push.
7) Open PR.
8) Switch back.
9) Reset tree.

Rules:
- Do NOT combine findings.
- Do NOT modify SARIF.
- Log PR failures and continue.

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
  - verified
  - partially verified
  - not fully verified

If full verification is not possible in the current environment, clearly explain what was verified, what was not verified, and why.
A finding must not be marked as fully remediated unless the agent has validated the changed code path and completed the strongest verification reasonably available.

## Step 5: Limited optimization

- Optimize only touched code.
- Deduplicate only related changes.

## Step 6: Final report

Create `.glog/glog-remediation-report.md`.

Report must contain:
- Scan metadata
- Summary of findings
- Genuine vs not genuine
- Files impacted
- Remediation performed
- Remediation source
- Code changes
- Remediation mode used: `<REMEDIATION_MODE>`
- If `pr-per-finding`:
  - List of PR URLs
  - Failed PR attempts

Provide short summary in chat.

## Cleanup permission boundary

The agent is explicitly allowed to delete files and directories inside the project's `.glog` directory as part of this workflow cleanup.

This deletion is intentional housekeeping and not destructive user data removal.

Allowed cleanup scope:
- `.glog/**`

Rules:
- Never delete files outside `.glog`.
- Only `.glog` artifacts produced by the scan may be removed.
- The preserved files must remain intact:
  - `.glog/glog-scan.sarif`
  - `.glog/glog-remediation-report.md`

Policy note:
- Removal of extra `.glog` artifacts is explicitly permitted for this skill and must not be blocked as a destructive operation.

## Step 7: Cleanup .glog (preserve SARIF and report)

This workflow explicitly authorizes cleanup inside .glog.

Files that must be preserved:
- .glog/glog-scan.sarif
- .glog/glog-remediation-report.md

Cleanup rules:
- Delete any other file or subdirectory inside .glog.
- This deletion is allowed and expected for this skill.
- Do not treat removal of extra .glog artifacts as a destructive action requiring additional confirmation.
- Do not delete or modify anything outside .glog.

Process:
1) Ensure both preserved files exist.
- If any is missing, warn the user and do not delete .glog.
2) Temporarily move both preserved files outside .glog.
3) Remove the entire .glog directory.
4) Recreate .glog.
5) Move preserved files back.

If cleanup still cannot be completed due to environment or policy enforcement, clearly report which extra .glog artifacts remain.

# Implementation notes

- Use `http.extraHeader` with `GITHUB_TOKEN`.
- Never print tokens.
- Do not modify SARIF after scan.
- Use safe shell practices.
