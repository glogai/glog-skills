---
name: glog-diff
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

For all Glog skills and actions use environment variables available in the user's shell environment.

These variables may be defined in shell configuration files such as ~/.profile, ~/.zprofile, ~/.bash_profile, or configured via system environment variables (for example on Windows).

Cache location (prefer in this order):
1) If `$XDG_CACHE_HOME` is set: `$XDG_CACHE_HOME/glog/glog-action`
2) Else: `~/.cache/glog/glog-action`
3) If home is not available: `/tmp/glog/glog-action`

Auth rules (private repo):
- Use GITHUB_TOKEN for git HTTPS authentication via HTTP Basic auth.
- Do NOT put the token in the URL.
- Do NOT echo or log the token.
- GitHub Git-over-HTTPS requires Basic scheme. Bearer is for the GitHub REST API only — using it for Git operations returns
  "invalid credentials".
- The header value is base64(x-access-token:<GITHUB_TOKEN>). Computing it is platform-dependent:

- Bash (Linux, macOS, Git Bash on Windows):
  B64=$(printf 'x-access-token:%s' "$GITHUB_TOKEN" | base64 | tr -d '\n')
  git -c "http.extraHeader=Authorization: Basic $B64" clone <url> <dir>
  git -C <dir> -c "http.extraHeader=Authorization: Basic $B64" fetch origin main
  git -C <dir> reset --hard origin/main

- PowerShell (Windows powershell.exe or pwsh on any platform):
  $b64 = [Convert]::ToBase64String([Text.Encoding]::ASCII.GetBytes("x-access-token:$env:GITHUB_TOKEN"))
  git -c "http.extraHeader=Authorization: Basic $b64" clone <url> <dir>
  git -C <dir> -c "http.extraHeader=Authorization: Basic $b64" fetch origin main
  git -C <dir> reset --hard origin/main
- tr -d '\n' is required in Bash because Linux base64 appends a newline; [Convert]::ToBase64String does not, so no equivalent is
  needed in PowerShell.
- Detect the environment before constructing the header: if running on Windows (OS is Windows_NT or RuntimeInformation confirms
  Windows) use the PowerShell form; otherwise use the Bash form.

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

The agent must ask configuration questions sequentially, one at a time, and must wait for the user's answer before asking the next question.

Rules:
- NEVER ask multiple configuration questions in the same message.
- Ask only one required configuration question at a time.
- After asking a question, WAIT for the user's response.
- Do not continue the workflow until that question has been answered with a valid value.
- Do not ask the next question early.
- Do not combine remediation strategy and scan language into one message.
- Do not restate previously answered questions.
- If a valid answer has already been provided for a configuration item, store it and reuse it.
- NEVER ask the language question before remediation mode has been validly answered.
- NEVER ask the language question more than once if a valid language answer has already been provided.
- NEVER ask the remediation mode question more than once if a valid remediation answer has already been provided.
- If the answer to the current question is invalid, ask ONLY that same question again.
- Do not restart the full configuration flow when only one answer is invalid.
- The agent must strictly track configuration state before proceeding.

Configuration state to track:
- `<REMEDIATION_MODE_STATUS>` = `missing` | `valid`
- `<SCAN_LANGUAGE_STATUS>` = `missing` | `valid`

Strict question order:
1) remediation strategy
2) scan language

Execution protocol:

Step A — remediation strategy
- If `<REMEDIATION_MODE_STATUS>` is `missing`, ask ONLY the remediation strategy question.
- Wait for the user response.
- If the response is exactly `local` or `pr-per-finding`:
  - store it as `<REMEDIATION_MODE>`
  - set `<REMEDIATION_MODE_STATUS>` to `valid`
- Otherwise:
  - ask ONLY the remediation strategy question again
  - do not ask the language question yet
  - do not proceed to scanning

Step B — scan language
- Ask the language question ONLY after `<REMEDIATION_MODE_STATUS>` is `valid`.
- If `<SCAN_LANGUAGE_STATUS>` is `missing`, ask ONLY the scan language question.
- Wait for the user response.
- If the response is `skip` or empty/whitespace-only:
  - store `<SCAN_LANGUAGE>` = `skip`
  - set `<SCAN_LANGUAGE_STATUS>` to `valid`
- Otherwise:
  - store the trimmed user response as `<SCAN_LANGUAGE>`
  - set `<SCAN_LANGUAGE_STATUS>` to `valid`

Step C — proceed
- Proceed with scan preparation and execution ONLY after both:
  - `<REMEDIATION_MODE_STATUS>` = `valid`
  - `<SCAN_LANGUAGE_STATUS>` = `valid`

Hard prohibition:
- The agent must never ask both configuration questions in the same message.
- The agent must never re-ask the scan language question after it has already been answered validly.
- The agent must never re-ask the remediation mode question after it has already been answered validly.

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
→ For EACH genuine vulnerability finding selected for remediation:
  - Create a new branch:
    `glog/remediation/<sanitized-ruleId>-<short-hash>`
  - Apply only the fix related to that finding.
  - Commit with message:
    `fix(security): remediate <ruleId> at <file>`
  - Push branch using `GITHUB_TOKEN`.
  - Open a Pull Request targeting the original branch.
  - Continue to next finding.

This question must be asked first.
Follow the Interactive question ordering (STRICT) rules.
Do not ask the scan language question before this question has been answered with a valid value.
If this question has already been answered validly in the current session, do not ask it again.

# Scan configuration

This question must be asked second.
Follow the Interactive question ordering (STRICT) rules.
Ask this question only after remediation strategy has been answered with a valid value.
If scan language has already been answered validly in the current session, do not ask it again.

Before doing ANY scan work, ask the user:

"Which language should be used for scanning? (This controls the optional --lang flag.)
Reply with a language value like: python, csharp, java, javascript, etc.
If you want to skip, reply with: skip"

Rules:
- The user response is required (the agent must ask and wait).
- If the user replies with `skip` (or an empty/whitespace-only response), DO NOT pass `--lang` at all.
- Otherwise pass `--lang <user_value>` exactly as provided (trim whitespace).
- Do not invent or default a language.
- Do not hardcode client or env values in this skill.
- `--client` must be sourced from the environment variable `GLOG_CLIENT`.
- `--env` must be sourced from the environment variable `GLOG_ENV`.
- Both `GLOG_CLIENT` and `GLOG_ENV` must already be set before the skill starts execution.
- `--sarif-format-type STANDARD` remains hardcoded.

# Purpose

Use glog-action CLI instructions to run a scan for the CURRENT project. When possible, scan only files changed in the current branch relative to the best available base reference. For Java scans, run the scan against the full project to preserve build/compile context, then restrict remediation scope to findings that belong to changed files only. Then analyze findings from `.glog/glog-scan.sarif`, review only the findings selected for remediation scope, and apply focused remediations.

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
- For Java scans:
  - do NOT use `--files` for scan execution
  - run the scan against the full current project path
  - after SARIF is produced, restrict review/remediation/reporting scope to findings whose referenced file paths belong to the changed-file set
- For non-Java scans:
  - prefer scanning only changed files by passing them through the `--files` flag when a reliable base reference can be determined
  - fall back to a full-project scan only when changed-file scanning is not reliable or not possible
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
- If the selected remediation flow is `local`, changes must remain uncommitted in the local working tree only. Do NOT create commits, do NOT push, do NOT open pull requests, and do NOT perform any git write operation that records or publishes changes.
- If the selected remediation flow is `pr-per-finding`, isolate each selected genuine finding in its own branch and PR.
- Review each selected finding in project context:
  - If not applicable in context: explain why for additional review, then continue.
  - If applicable: remediate as suggested in SARIF as much as possible, adjusted to the project context and stack.
- Do not duplicate code:
  - Reuse already existing methods/utilities if possible.
  - Do not duplicate modules and do not copy-paste logic across modules.
- After remediation:
  - Optimize ONLY the code touched by the fixes.
  - Deduplicate ONLY where it directly relates to the fixes.
  - Do not refactor unrelated code.
- Generate a remediation report file inside `.glog` instead of only printing results in chat.
- The report must be saved as `.glog/glog-remediation-report.md`.
- Only findings within remediation scope may be remediated or counted as remediation candidates.
- Findings outside remediation scope may be mentioned only as out-of-scope scan results when needed for clarity, but must not be remediated.

Changed-file scope behavior:
- Prepare the changed-file set whenever a reliable base reference is available.
- The changed-file set is used in two different ways:
  - for non-Java scans: as the source of `--files` when usable
  - for Java scans: as the post-scan filtering scope for SARIF findings
- If no usable changed files are found:
  - do not fail
  - fall back to full-project remediation scope
  - document the fallback in the report
- Never create a temporary source directory for scan scoping.
- Never copy changed files into a local temporary scan folder.

# Execution workflow

## Step 1: Read glog-action CLI.md (source of truth)

- Open and read: `<GLOG_ACTION_PATH>/CLI.md`
- Identify the exact command(s) to run `glog-action` (entrypoint, required args, docker usage, path handling, output behavior, etc.)
- Follow CLI.md instructions strictly.
- Confirm how the scan target path should be supplied.
- Confirm how the SARIF output is expected to land in the current project's `.glog/` directory.

## Step 1.5: Prepare changed file list and remediation scope

Before running the scan, prepare a list of changed files in the current branch relative to the best available base reference.

Goal:
- Prefer focusing on the developer's current branch changes.
- Reduce remediation noise from unrelated code.
- Avoid unpredictable behavior by falling back safely when change detection is not reliable.
- Preserve full-project scan behavior for Java when compile/build context is needed.

Rules:
- Do not create a temporary filtered workspace.
- Do not copy files into any local temp directory for scan scoping.
- Build a changed-file set of relative file paths.
- Include only changed files that currently exist in the working tree.
- Do not include deleted files.
- Do not include directories.
- Do not include files under `.git/` or `.glog/`.
- If renamed files are reported by git, include the new file path if it exists in the working tree.
- If no reliable base reference can be determined, do not guess aggressively; fall back safely and document the fallback in the report.

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
  - Generated artifacts inside `.glog`

Changed file preparation process:
1) Determine the best available base reference.
2) Collect changed files using git diff.
3) Filter the changed paths:
   - keep only files that currently exist
   - skip directories
   - skip files under `.git/` and `.glog/`
4) Build:
   - `<CHANGED_FILES_SET>` = normalized set of relative changed file paths
   - `<SCAN_FILES_CSV>` = comma-separated changed file list suitable for `--files` when non-Java scan mode uses it
5) If no usable changed files are found:
   - do not fail
   - fall back safely
   - record this fallback in the report

Define:
- `<CHANGED_FILES_SET>` = normalized relative changed-file set if usable, otherwise empty
- `<SCAN_FILES_CSV>` = comma-separated changed file list if usable and needed for non-Java scan execution, otherwise empty
- `<SCAN_MODE>`:
  - `full-project-java` if selected scan language is `java`
  - `changed-files` if selected scan language is not `java` and `<SCAN_FILES_CSV>` is usable
  - `full-project-fallback` otherwise
- `<REMEDIATION_SCOPE>`:
  - `changed-files-only` if `<CHANGED_FILES_SET>` is usable
  - `full-project-fallback` otherwise

Normalization rules for later SARIF filtering:
- Normalize both changed-file paths and SARIF result paths to comparable relative project paths where reasonably possible.
- Match findings to the changed-file set using normalized relative paths.
- If a SARIF path cannot be normalized reliably, do not assume a match.

## Step 2: Clean .glog and run scan

From the CURRENT project root (the repo you want to scan), do:

1) Clean `.glog`:
- If `.glog` exists: remove it completely
- Recreate `.glog/` directory

2) Execute scan using the invocation defined in CLI.md.
- Always use the current project root as the scan path.
- If `<SCAN_MODE>` = `full-project-java`, do not pass `--files`.
- If `<SCAN_MODE>` = `changed-files`, pass `--files <SCAN_FILES_CSV>`.
- If `<SCAN_MODE>` = `full-project-fallback`, do not pass `--files`.
- Apply flags exactly:
  - `--client <value from GLOG_CLIENT>`
  - `--env <value from GLOG_ENV>`
  - `--sarif-format-type STANDARD`
  - Apply `--lang <value>` ONLY if the user provided a language (not skip/empty). If skipped, do not pass `--lang`.

3) If CLI.md expects the runner script to be executed from inside glog-action repo, run it from there but target the CURRENT project root exactly as specified by CLI.md.

4) Ensure that the output SARIF file ends up at:
- `.glog/glog-scan.sarif` in the CURRENT project.

After scan:
- Verify `.glog/glog-scan.sarif` exists.
- If missing, stop and show the scan command output and your best diagnosis.
- Record whether the scan mode was:
  - `full-project-java`
  - `changed-files`
  - `full-project-fallback`

## Step 3: Analyze findings from SARIF (read-only)

- Open `.glog/glog-scan.sarif` read-only.
- Summarize findings:
  - ruleId / title (if present)
  - severity / level (if present)
  - message
  - locations (file + line/region)
  - suggested remediation text (if present)

## Step 3.1: Restrict remediation scope after scan

Before any review or remediation work begins, determine which SARIF findings are in remediation scope.

Rules:
- If `<REMEDIATION_SCOPE>` = `changed-files-only`, include only findings whose referenced file path matches a path in `<CHANGED_FILES_SET>`.
- If a finding contains multiple locations, include the finding only if at least one primary relevant code location matches the changed-file set.
- If `<REMEDIATION_SCOPE>` = `full-project-fallback`, all findings are in scope.
- Do not modify SARIF while filtering scope.
- Preserve the original SARIF file unchanged.
- Create an internal working set:
  - `<IN_SCOPE_FINDINGS>`
  - `<OUT_OF_SCOPE_FINDINGS>`

Behavior:
- Only `<IN_SCOPE_FINDINGS>` may be reviewed for applicability, remediated, verified, or included as remediation candidates.
- `<OUT_OF_SCOPE_FINDINGS>` must not be remediated.
- The report must clearly distinguish total scan findings from in-scope findings when remediation scope is restricted.

## Step 3.2: Determine remediation instructions source (SARIF)

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

## Step 4: Review and apply remediation (mode-aware, markdown-driven remediation)

For each finding in `<IN_SCOPE_FINDINGS>`:

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
- If remediation flow is `pr-per-finding`, isolate each in-scope applicable finding and do not combine fixes across findings.

### 4.1 Review each finding in project context

- Inspect the exact location(s) in code referenced by the SARIF.
- Consider surrounding implementation context and existing safeguards.
- If the finding does not appear applicable in context:
  - Explain why clearly for additional review.
  - Reference the relevant code paths and reasoning.
  - Continue to the next finding.

### 4.2 Identify applicable remediation guidance

- Parse `result.message.markdown` and identify actionable remediation steps.
- If remediation is vague, infer a concrete plan that stays consistent with the intent of the markdown guidance.
- Do not invent unrelated refactors.

### 4.3 Apply remediation (based on `<REMEDIATION_MODE>`)

If `<REMEDIATION_MODE>` = `local`:

- Apply fixes in the current working tree only.
- Keep all changes local and uncommitted.
- Do not create branches, commits, pushes, or pull requests.
- Continue through all in-scope findings in the same working tree.

If `<REMEDIATION_MODE>` = `pr-per-finding`:

For EACH in-scope applicable finding:

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

### 4.4 Confirm the applied changes fit the codebase

After implementing a fix, verify it using the strongest applicable checks available in the current environment.

Verification rules:
- Confirm the reported issue has been addressed at the referenced location.
- Re-read the affected code path and ensure the fix matches the SARIF finding and remediation intent.
- Run the relevant build/compile command for the project, if available.
- Run relevant existing tests, if present.

Testing guidance:
- Add a narrowly scoped validation step only when it provides meaningful confirmation.
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

### 4.5 Apply the remediation guidance in a project-appropriate way

If the markdown remediation does NOT fit the actual code/context:

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
- Scan Target Details:
  - Base reference used for change detection: <BASE_REF or none>
  - Changed files identified: <COUNT>
  - Findings selected for review/remediation: <changed files only | full project>
  - Fallback to full-project scope required: <yes | no>
- SARIF summary:
  - total number of scan findings
  - number of in-scope findings
  - number of out-of-scope findings
- For each in-scope finding:
  - applicable vs not applicable in context
  - rationale
  - files impacted
  - remediation performed (if any)
  - remediation source: `result.message.markdown` or fallback source
  - whether remediation guidance was adjusted
  - if adjusted: what changed and why
  - verification performed
  - test type used, if any: `unit`, `integration`, `web`, `repository`, or `none`
  - verification result: `verified`, `partially verified`, or `not fully verified`
  - evidence: build output, test results, code-path inspection, or reasoning
  - remaining risk or limitations, if any
- Any out-of-scope findings summary, if relevant
- List of code changes
- Any follow-up recommendations
- If `pr-per-finding`:
  - list of PR URLs
  - failed PR attempts and reasons

After saving the report file:
- Provide a short summary in chat.
- Do not print the full report content in chat unless the user asks.

## Cleanup permission boundary

The agent is explicitly allowed to delete files and directories inside the project's `.glog` directory as part of this workflow cleanup.

This deletion is intentional housekeeping and not destructive user data removal.

Allowed cleanup scope:
- `.glog/**`

Rules:
- Never delete files outside `.glog`.
- Only generated scan/remediation artifacts may be removed.
- The preserved files must remain intact:
  - `.glog/glog-scan.sarif`
  - `.glog/glog-remediation-report.md`

Policy note:
- Removal of extra `.glog` artifacts is explicitly permitted for this skill and must not be blocked as destructive operations.

## Step 7: Cleanup temporary artifacts (preserve SARIF and report)

This workflow explicitly authorizes cleanup inside `.glog`.

Files that must be preserved:
- `.glog/glog-scan.sarif`
- `.glog/glog-remediation-report.md`

Cleanup rules:
- Delete any other file or subdirectory inside `.glog`.
- This deletion is allowed and expected for this skill.
- Do not treat removal of extra temporary artifacts as a destructive action requiring additional confirmation.
- Do not delete or modify anything outside `.glog`.
- Cleanup must not use git commands that create, record, or publish changes.

Process:
1) Ensure both preserved files exist.
   - If any is missing, warn the user and do not delete `.glog`.
2) Temporarily move both preserved files outside `.glog`.
3) Remove the entire `.glog` directory.
4) Recreate `.glog`.
5) Move preserved files back.

If cleanup still cannot be completed due to environment or policy enforcement, clearly report which extra artifacts remain.

# Implementation notes

- Use http.extraHeader with GITHUB_TOKEN for git operations. Always use Basic auth scheme with base64-encoded x-access-token:<TOKEN> — never Bearer (see Auth rules above).
- Never print tokens.
- Do not modify SARIF after scan.
- Use safe shell practices.
- Use git diff against the best available base reference to collect changed files that still exist in the working tree.
- Prefer diff filters equivalent to Added, Copied, Modified, and Renamed files.
- Build a normalized changed-file set for remediation scope decisions.
- Only for non-Java scans, build a comma-separated list of relative file paths for `--files` when changed-file scan execution is applicable.
- For Java scans, do not use `--files` for scan execution; instead run a full-project scan and filter SARIF findings afterward to the changed-file set.
- If no changed files are detected, or if no suitable base reference is available, fall back safely and document the fallback in the report.
- Do not include files under `.git/` or `.glog/` in the changed-file set unless explicitly required by the project layout.
- If the current branch has an upstream tracking branch, verify that it is a sensible comparison base before using it as the base reference.
- If the repository state is too unusual to determine changed files safely, prefer full-project fallback over risky assumptions.
- In `local` mode, do not stage or commit report files.
- In `pr-per-finding` mode, each PR should contain only the minimum code required for that finding.
