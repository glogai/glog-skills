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

The agent must ask configuration questions sequentially, one at a time.

Rules:
- NEVER ask multiple configuration questions in the same message.
- Ask the first required question and WAIT for the user response.
- Do not continue the workflow until the user has answered that question.
- After receiving a valid answer, store the value and only then ask the next question.
- Do not repeat a question if a valid answer has already been provided.
- Do not ask the next question before the current one is answered.

Strict order of questions:

1️⃣ Remediation strategy  
2️⃣ Scan language
3️⃣ Upload final SARIF result
4️⃣ Windows staging mode, only when running a large scan on Windows

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
- proceed to Step C

If invalid:
- ask the scan language question again
- do NOT restart Step A

Step C — Ask final SARIF upload question only.

Wait for the user response.

If the response is valid (`yes` or `no`):
- store `<UPLOAD_FINAL_SARIF>`
- proceed to Step D

If invalid:
- ask the final SARIF upload question again
- do NOT restart Step A or Step B

Step D — Conditional Windows staging mode.

Before scan execution, inspect only enough local environment/project metadata to determine whether all of these are true:
- the skill is running on Windows
- the application is large enough that direct Windows scanning may be unreliable, slow, or path-sensitive
- WSL2 or Docker staging may materially improve scan reliability

If any condition is false:
- store `<WINDOWS_STAGING_MODE>` = `not-applicable`
- proceed to scanning workflow

If all conditions are true:
- ask only the Windows staging mode question
- wait for the user response
- if the response is valid, store `<WINDOWS_STAGING_MODE>` and proceed to scanning workflow
- if invalid, ask only the Windows staging mode question again

The agent must never ask multiple configuration questions in the same message.
The agent must never re-ask the language question if it was already answered.
The agent must never re-ask the final SARIF upload question if it was already answered.
The agent must never ask the Windows staging mode question unless the Windows large-application staging conditions are met.

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

# Final SARIF upload configuration

This question must be asked only AFTER the scan language question has been answered.
Follow the Interactive question ordering rules strictly.

Before doing ANY scan work, ask the user:

"Should the final SARIF result be uploaded?

Reply with one of:

yes → Upload the final SARIF result.

no → Do not upload the final SARIF result.

Type exactly: yes or no"

Rules:
- The user response is required (the agent must ask and wait).
- If the response is anything other than `yes` or `no`, ask again.
- Do not assume a default.
- Store the value as `<UPLOAD_FINAL_SARIF>`.

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
  - `-u` ONLY if the `<UPLOAD_FINAL_SARIF>` is `yes` to uploading the final SARIF result. Otherwise do not include `-u`.
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

## Step 1.5: Windows large-application staging and module scan override

Apply the Windows staging option ONLY when running on Windows and the current application is large enough to make a direct full-project scan unreliable, slow, or likely to fail.
Apply the module scan override only when the project can also be divided into clear modules.

Windows staging configuration for large applications:
- When the Windows staging conditions are met, ask the user:

"Large Windows application detected. Which staging mode should be used?

Reply with one of the available options:

wsl2-staging → Copy the original source code to WSL2 and run the glog CLI there using wsl bash calls. Recommended when WSL2 is installed.

docker-volume-staging → Copy the original source code into a Docker volume and scan from that staged source.

windows-direct → Continue from the original Windows checkout.

Type exactly one option."

- Offer only options whose prerequisites are available, plus `windows-direct`.
- Recommend `wsl2-staging` when a WSL2 distro is available.
- Use `docker-volume-staging` as the fallback staging option when WSL2 is not available but Docker is available.
- Do not assume a staging mode. Store the answer as `<WINDOWS_STAGING_MODE>`.

Command-generation rules:
- Generate the concrete PowerShell, WSL, and Docker commands at execution time for the current machine. Do not blindly reuse hardcoded examples.
- Before generating commands, discover and validate:
  - current Windows project root and path quoting requirements
  - available WSL distros and WSL version (`wsl -l -q`, `wsl -l -v`)
  - Docker availability, when Docker staging is considered
  - required environment variables in the current process and Windows User/System scopes
  - generated/dependency/cache directories present in this repository
- Commands must fail fast on missing tools, failed copy operations, missing staged files, missing env vars, or missing final SARIF.
- Commands must never print `GLOG_TOKEN` or `GITHUB_TOKEN`; diagnostics may show only masked values.
- Examples in this skill are patterns only. Adapt distro name, paths, quoting, copy tooling, and exclude rules to the actual environment.

Original-source-only staging rule:
- Scan only original codebase content.
- When staging to WSL2 or Docker volume, copy only source files needed to scan/build context.
- Strictly exclude generated build directories, dependency folders, local caches, IDE folders, VCS metadata, `.glog`, and dot-directories.
- Exclude at least: `bin`, `obj`, `out`, `build`, `target`, `dist`, `node_modules`, `bower_components`, `__pycache__`, `.venv`, `venv`, `env`, `.tox`, `*.egg-info`, `.cache`, `.pytest_cache`, `.next`, `.nuxt`, `coverage`, `.vs`, `.idea`, `.gradle`, `.git`, `.glog`, and all directories whose name starts with `.`.
- If any other generated directory is visible in the project, exclude it too.

WSL2 staging requirements:
- Do not install Claude Code CLI in WSL2.
- Use `wsl bash -c` or `wsl bash -lc` calls to prepare staging and invoke the glog CLI.
- Ensure `GITHUB_TOKEN`, `GLOG_TOKEN`, `GLOG_CLIENT`, `GLOG_ENV`, and `GITHUB_USER` are passed from Windows User/System scope to WSL2 using `WSLENV`.
- Use `WSLENV` entries equivalent to `GITHUB_TOKEN/u:GLOG_TOKEN/u:GLOG_CLIENT/u:GLOG_ENV/u:GITHUB_USER/u`.
- These variables must be available to every process executed inside the WSL2 session.
- Prepare or reuse the glog-action cache inside WSL2 using the same authentication rules.
- Run the scan from the staged WSL2 source path, not from the original Windows checkout.
- After the scan finishes, copy final scan artifacts from WSL2 back to the original host checkout using PowerShell `Copy-Item` from the WSL UNC path, for example `\\wsl$\<distro>\...`, to avoid WSL permission issues.

Docker volume staging requirements:
- Create a unique Docker volume for the staged source.
- Mount the original Windows checkout read-only when copying into the volume.
- Copy only original source content into the volume, applying the same exclude rules.
- Run the scan from the staged volume path when Docker staging is selected.
- Remove temporary containers after use; remove the temporary volume after final artifacts are safely copied back when it is no longer needed.

Module detection:
- Identify module directories from build/project markers such as `.sln`, `.csproj`, `.fsproj`, `.vbproj`, `pom.xml`, `build.gradle`, `settings.gradle`, `package.json`, `go.mod`, `Cargo.toml`, or similar stack-specific module files.
- Exclude dependency, build, generated, and VCS directories such as `.git`, `.glog`, `node_modules`, `target`, `build`, `dist`, `bin`, and `obj`.
- If fewer than two module directories can be identified, do not use this override; continue with the normal scan workflow.

Windows module scan behavior:
- Override the skill's default full-scan behavior for the scan execution step.
- Scan each module separately by running one engine scan per module directory.
- Apply the same required engine flags to every module scan:
  - `--client <value from GLOG_CLIENT>`
  - `--env <value from GLOG_ENV>`
  - `--sarif-format-type STANDARD`
  - `--lang <value>` only when provided
  - `-u` only when `<UPLOAD_FINAL_SARIF>` is `yes`
- If subagents are available and the user's request explicitly permits delegation, a separate agent may handle each module scan only when each agent has an isolated workspace or otherwise cannot race on the shared `.glog/glog-scan.sarif` output. If isolation is not available, run module scans sequentially.

PowerShell storage pattern for module SARIFs:

```powershell
$glogModuleSarifDir = Join-Path $env:TEMP ("glog-module-sarifs-" + [guid]::NewGuid().ToString())
New-Item -ItemType Directory -Force -Path $glogModuleSarifDir | Out-Null

# After each module scan completes and before starting the next scan:
$safeModuleName = ($moduleName -replace '[^A-Za-z0-9_.-]', '_')
Copy-Item -LiteralPath ".glog\glog-scan.sarif" -Destination (Join-Path $glogModuleSarifDir "$safeModuleName.sarif") -Force
```

Module SARIF storage rule:
- After each module scan completes, immediately verify `.glog/glog-scan.sarif` exists.
- Before starting the next module scan, copy `.glog/glog-scan.sarif` to the temporary directory outside `.glog/`.
- Never save intermediate module SARIF files inside `.glog/`; each subsequent scan cleanup may delete them.
- After all module scans complete, merge all temporary module SARIF files using a structured JSON/SARIF merge, not text concatenation.
- Write only the final merged SARIF back to `.glog/glog-scan.sarif`.
- Treat the final merged `.glog/glog-scan.sarif` as read-only after the merge is complete.

## Step 2: Clean .glog and run scan

From the CURRENT project root (the repo you want to scan), do:

1) Clean `.glog`:
- If `.glog` exists: remove it completely
- Recreate `.glog/` directory

2) Execute scan using the invocation defined in CLI.md.
- On Windows, if `wsl2-staging` or `docker-volume-staging` was selected, execute the scan from the staged source path and copy final artifacts back to the original checkout after the scan.
- On Windows, if the Step 1.5 module scan override applies, execute one scan per module directory instead of one full-project scan, then merge module SARIFs into `.glog/glog-scan.sarif`.
- Apply flags exactly:
  - `--client <value from GLOG_CLIENT>`
  - `--env <value from GLOG_ENV>`
  - `--sarif-format-type STANDARD`
  - Apply `--lang <value>` ONLY if the user provided a language (not skip/empty). If skipped, do not pass `--lang`.
  - Apply `-u` ONLY if the `<UPLOAD_FINAL_SARIF>` is `yes` to uploading the final SARIF result. If the `<UPLOAD_FINAL_SARIF>` is `no`, do not pass `-u`.
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
- If Windows staging was considered:
  - selected staging mode: `wsl2-staging`, `docker-volume-staging`, or `windows-direct`
  - original host checkout path
  - staged source path, if staging was used
  - confirmation that final artifacts were copied back to the original checkout, if staging was used
- If Windows module scan was used:
  - modules scanned
  - temporary module SARIF directory
  - confirmation that final merged SARIF was written to `.glog/glog-scan.sarif`
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

- Use http.extraHeader with GITHUB_TOKEN for git operations. Always use Basic auth scheme with base64-encoded x-access-token:<TOKEN> — never Bearer (see Auth rules above).
- Never print tokens.
- Do not modify final SARIF after scan completion. In Windows module-scan mode, merging temporary module SARIFs into `.glog/glog-scan.sarif` is part of scan completion; after that merge, treat the final SARIF as read-only.
- Use safe shell practices.
