---
name: glog-plan
description: Glog.AI scan + SARIF triage + remediation planning only. Produces a detailed security remediation plan and false-positive analysis without modifying source files, creating commits, or opening PRs.
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

Git authentication validation rules:
- Before cloning or fetching `glogai/glog-action`, verify repository access using the same Basic authentication method.
- If the access check fails, stop immediately and report GitHub authentication/access failure.
- If clone or fetch fails, do not continue to `CLI.md` validation.
- If a failed clone creates an incomplete cache directory, remove only that incomplete cache directory before retrying.
- Validate `CLI.md` only after clone or fetch succeeds.

Rules:
- Ensure `GITHUB_TOKEN` is set, otherwise stop with a clear error.
- If the cache folder does not exist: clone the repo into it.
- If it exists: perform a `git fetch` and `git reset --hard origin/main` to ensure it matches the configured ref.
- The cached repo must contain `CLI.md`. If it does not, stop with a clear error.
- Do not prompt the user for repo URL, ref, or path.
- Do not write any secrets to disk.
- Do not echo tokens.

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

1) Scan language
2) Upload final SARIF result
3) Windows staging mode, only when running a large scan on Windows

Execution protocol:

Step A — Ask scan language question only.

Wait for the user response.

If the response is valid:
- store `<SCAN_LANGUAGE>`
- proceed to Step B

Step B — Ask final SARIF upload question only.

Wait for the user response.

If the response is valid (`yes` or `no`):
- store `<UPLOAD_FINAL_SARIF>`
- proceed to Step C

If invalid:
- ask the final SARIF upload question again
- do NOT restart Step A

Step C — Conditional Windows staging mode.

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

# Scan configuration

Before doing ANY scan work, ask the user:

"Which language should be used for scanning? (This controls the optional --lang flag.)
Reply with a language value like: python, csharp, java, javascript, etc.
If you want to skip, reply with: skip"

Rules:
- The user response is required.
- If the user replies with `skip` (or an empty/whitespace-only response), DO NOT pass `--lang` at all.
- Otherwise pass `--lang <user_value>` exactly as provided after trimming whitespace.
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

"Should the final SARIF result be uploaded by the engine?

Reply with one of:

yes → Upload the final SARIF result.

no → Do not upload the final SARIF result.

Type exactly: yes or no"

Rules:
- The user response is required (the agent must ask and wait).
- If the response is anything other than `yes` or `no`, ask again.
- Do not assume a default.
- Store the value as `<UPLOAD_FINAL_SARIF>`.
- If `<UPLOAD_FINAL_SARIF>` = `yes`.
- If `<UPLOAD_FINAL_SARIF>` = `no`.

# Purpose

Use glog-action CLI instructions to run a scan for the CURRENT project root. Always scan the complete current project. Then analyze findings from `.glog/glog-scan.sarif`, validate each finding, identify possible false positives, and produce a detailed remediation plan only.

This skill is planning-only:
- it must NOT modify application source files
- it must NOT apply remediations
- it must NOT create commits
- it must NOT push
- it must NOT create pull requests
- it must NOT edit SARIF
- it must only produce analysis and a remediation plan report

# Required environment

Use global environment variables (do not prompt for them unless missing):
- GLOG_TOKEN
- GITHUB_TOKEN
- GITHUB_USER
- GLOG_CLIENT
- GLOG_ENV

If any is missing, stop and tell the user exactly which one(s) are missing and how to export them.

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
- Always scan the full current project root, except when the Windows large-application module scan override applies.
- Run scan with the required flags:
  - `--client <value from GLOG_CLIENT>`
  - `--env <value from GLOG_ENV>`
  - `--sarif-format-type STANDARD`
  - `--lang <user_value>` ONLY if the user provided a language. Otherwise do not include `--lang`.
  - `-u` ONLY if the `<UPLOAD_FINAL_SARIF>` is `yes` to uploading the final SARIF result. Otherwise do not include `-u`.
- `GLOG_CLIENT` and `GLOG_ENV` must be validated before any scan execution begins.
- If either `GLOG_CLIENT` or `GLOG_ENV` is missing, stop immediately and do not run the engine.
- Do not substitute hardcoded fallback values for client or env.
- After scan is finished:
  - DO NOT modify `.glog/glog-scan.sarif` (read-only after creation).
- The scan is considered successful only if the engine produces the final SARIF file at:
  - `.glog/glog-scan.sarif`
- All post-scan analysis, summarization, validation, triage, and remediation planning must continue strictly from:
  - `.glog/glog-scan.sarif`
- Do NOT continue from any other SARIF file, temporary SARIF file, intermediate SARIF file, copied SARIF file, or engine-internal artifact created during execution.
- If multiple SARIF files are created while the engine runs, ignore all of them unless the final expected file is exactly:
  - `.glog/glog-scan.sarif`
- If `.glog/glog-scan.sarif` is missing, treat the scan as failed for the purposes of this workflow, stop, and report the command output plus the best diagnosis.
- This skill is strictly analysis-and-plan only.
- DO NOT modify any project source file.
- DO NOT modify configuration files, tests, build files, or documentation outside `.glog`.
- DO NOT create commits.
- DO NOT stage files.
- DO NOT push.
- DO NOT create pull requests.
- DO NOT perform any git write operation that records or publishes changes.
- DO NOT generate code patches unless the user explicitly asks later in a separate step.
- Treat remediation guidance as a focused security planning task.
- Validate whether each finding is genuine:
  - If likely NOT genuine: explain clearly why it may be a false positive or needs manual review.
  - If likely genuine: explain what should be changed, where, and why.
- Do not invent framework behavior or mitigations that are not visible in the code.
- Be conservative:
  - if evidence is insufficient, classify as `needs manual review`
  - do not overstate certainty
- Generate a remediation planning report file inside `.glog` instead of only printing results in chat.
- The report must be saved as `.glog/glog-remediation-plan.md`.

# Execution workflow

## Step 0: Windows / Git Bash compatibility

If running on Windows with Git Bash / MSYS2, apply these rules before any scan, Docker, bash, or Node-based file operations:

- Set `MSYS_NO_PATHCONV=1` for all relevant commands that invoke bash scripts, Docker, or Node.
- Determine the current project root as usual, but convert the scan target path from Git Bash style to Windows style before passing it to `glog.sh` or any Docker-backed command.
  - Example: `/c/Glog/Projects/foo` → `C:/Glog/Projects/foo`
- Use the Windows-style path consistently as the scan target path:
  - for `--path` in `glog.sh`
  - for any Node-based file reads
  - for any explicit SARIF/JSON path references passed to commands
- Do not rely on raw Git Bash paths like `/c/...` for Docker-backed scan execution.
- Prefer the filesystem Read tool for reading and parsing `.glog/glog-scan.sarif` and other generated JSON files on Windows, to avoid shell path conversion issues.
- Do not use `python3` for SARIF or JSON parsing on Windows, because it may not be installed or may not be reliably available.
- If the scan appears to run successfully but `.glog/glog-scan.sarif` is missing, treat this first as a likely Windows path-conversion / Docker mount issue and report that diagnosis explicitly before any other conclusion.

## Step 1: Read glog-action CLI.md (source of truth)

- Open and read: `<GLOG_ACTION_PATH>/CLI.md`
- Identify the exact command(s) to run `glog-action` (entrypoint, required args, docker usage, path handling, output behavior, etc.)
- Follow CLI.md instructions strictly.
- Confirm how the scan target path should be supplied.
- Confirm how the SARIF output is expected to land in the current project's `.glog/` directory.

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
- If fewer than two module directories can be identified, do not use this override; continue with the normal full-project scan workflow.

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
- Use the current project root as the scan path.
- Always scan the full current project unless the Windows Step 1.5 module scan override applies.
- On Windows, if `wsl2-staging` or `docker-volume-staging` was selected, execute the scan from the staged source path and copy final artifacts back to the original checkout after the scan.
- On Windows, if the Step 1.5 module scan override applies, execute one scan per module directory instead of one full-project scan, then merge module SARIFs into `.glog/glog-scan.sarif`.
- Apply flags exactly:
  - `--client <value from GLOG_CLIENT>`
  - `--env <value from GLOG_ENV>`
  - `--sarif-format-type STANDARD`
  - Apply `--lang <value>` ONLY if the user provided a language. If skipped, do not pass `--lang`.
  - Apply `-u` ONLY if the `<UPLOAD_FINAL_SARIF>` is `yes` to uploading the final SARIF result. If the user answered `no`, do not pass `-u`.

3) If CLI.md expects the runner script to be executed from inside glog-action repo, run it from there but target the current project root exactly as specified by CLI.md.

4) Ensure that the output SARIF file ends up at:
- `.glog/glog-scan.sarif` in the CURRENT project.

Important:
- The workflow must treat `.glog/glog-scan.sarif` as the only valid final SARIF output for subsequent steps.
- Do NOT select any other `.sarif` file produced during engine execution.
- Do NOT analyze helper, temporary, partial, intermediate, fallback, copied, or engine-generated SARIF artifacts from other paths.
- Even if other SARIF files exist, continue only if the final engine output exists exactly at:
  - `.glog/glog-scan.sarif`

After scan:
- Verify `.glog/glog-scan.sarif` exists.
- If missing, stop and show the scan command output and your best diagnosis.
- Record that the scan target was the full current project root.

## Step 3: Analyze findings from SARIF (read-only)

- Open `.glog/glog-scan.sarif` read-only.
- Iterate through the `results` list.
- Summarize findings:
  - ruleId / title (if present)
  - severity / level (if present)
  - message
  - locations (file + line/region)
  - suggested remediation text (if present)

If the SARIF contains no results:
- Still create `.glog/glog-remediation-plan.md`
- State clearly that no findings were present in the SARIF
- Include scan metadata and scan target details

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
- If there is no clear remediation text, classify as `needs manual review` and explain why.

## Step 4: Validate + Explain + Plan (no code changes)

For each finding:

### 4.0 Planning-only guardrail

- This skill must not modify source code.
- This skill must not apply fixes.
- This skill must not create branches, commits, pushes, or pull requests.
- This skill must not write outside `.glog`.
- The only new file that may be created is:
  - `.glog/glog-remediation-plan.md`

### 4.1 Validate whether the finding is genuine

- Inspect the exact location(s) in code referenced by the SARIF.
- Consider runtime behavior, framework defaults, visible existing mitigations, input flow, output flow, authorization checks, encoding, validation, sanitization, and configuration relevant to the finding.
- Use only evidence visible in the codebase and the SARIF context.
- If evidence strongly indicates the finding is not actually exploitable, classify it as one of:
  - `likely false positive`
  - `needs manual review`
- If evidence supports exploitability or unsafe behavior, classify it as:
  - `likely genuine`

### 4.2 Explain the reasoning

For each finding, explain:
- what the security issue means in practical terms
- why the reported code path may be vulnerable
- what data or control flow makes it risky
- what visible code mitigations already exist, if any
- why those mitigations are sufficient or insufficient
- why the finding is likely genuine, likely false positive, or needs manual review

### 4.3 Extract remediation guidance

- Parse `result.message.markdown` and identify actionable remediation steps.
- If remediation is vague, infer a concrete remediation plan that stays consistent with the intent of the markdown guidance.
- Do not invent unrelated refactors.
- Do not produce patch text or apply edits.

### 4.4 Produce a remediation plan only

For each finding, produce a concrete plan that includes:
- the file(s) and code area(s) that should be reviewed
- the likely root cause
- the minimal safe change that should be made
- reusable utilities or existing abstractions that should be preferred
- what must be avoided during the fix
- whether tests should be added
- what kind of verification should be performed after the future fix

The plan must be implementation-oriented, but must not modify code.

### 4.5 Verification guidance for the future fix

For each finding, recommend how the future fix should be verified:
- confirm the vulnerable pattern is no longer present at the reported location
- re-read the affected code path and ensure the intended protection is actually enforced
- run the relevant build/compile command for the project, if available
- run relevant existing tests, if present
- add a minimal targeted test only if it meaningfully verifies the remediation
- prefer a unit test when the future fix is isolated to business logic, validation, sanitization, parsing, authorization checks in service logic, or reusable helper methods
- prefer an integration, web-layer, repository, or framework-level test when the issue depends on runtime wiring, HTTP behavior, persistence, templating/rendering, authentication/authorization configuration, or other framework behavior
- verify that malicious, unsafe, or invalid input is now rejected, sanitized, escaped, validated, or safely handled as appropriate
- verify that expected valid behavior still works after the future change
- ensure the future fix does not introduce regressions or break public API behavior

### 4.6 False-positive handling

If a finding appears not to be real, explain clearly:
- what the scanner likely matched
- which visible code behavior reduces or eliminates the risk
- whether the issue is fully disproven or only uncertain
- what a developer should manually confirm before dismissing it

Do not dismiss a finding casually.
If uncertainty remains, prefer `needs manual review`.

## Step 5: Focus and safety rules for the plan

- Stay focused on security findings from SARIF only.
- Do not drift into unrelated cleanup or refactoring advice.
- Prefer minimal, targeted remediation recommendations.
- Reuse existing methods and abstractions where possible.
- Do not recommend broad rewrites unless clearly necessary.
- Avoid overstating certainty.
- If the code context is incomplete, say so explicitly.
- If the remediation guidance from SARIF is incomplete or not aligned with the code, explain the mismatch and propose a safer, project-aligned remediation direction.

## Step 6: Final report

Create:
- `.glog/glog-remediation-plan.md`

# Markdown formatting rules for the report

The remediation report must be formatted using strict Markdown structure to ensure high readability.

Formatting rules:
  - Use headings for major sections.
  - Each finding must be its own subsection.
  - Use tables for metadata fields.
  - Use bullet lists for remediation plans and verification guidance.
  - Use code blocks for file locations.
  - Leave a blank line between sections.
  - Never print large text blocks without structure.
  - Do not collapse multiple fields into one paragraph.
  - Do not print metadata outside the intended metadata tables unless absolutely necessary.
  - Do not omit any subsection. If information is unavailable, write `not available`.
  - If a finding has multiple locations, include all of them in the Location section.
  - Be detailed but concise.
  - Prefer short paragraphs and bullet lists over long prose.
  - Keep each subsection focused on evidence from the code and SARIF only.
  - Do not allow the Findings section to degrade into one long paragraph.

The report must use the following structure exactly in this section order:

# Glog Remediation Plan

## Table of Contents
- Scan Metadata
- Scan Target Details
- SARIF Summary
- Findings
- Overall Recommendations

## Scan Metadata

| Field | Value |
|------|------|
| Date/Time | <timestamp> |
| Selected Language | <language or skipped> |
| Client | <value from GLOG_CLIENT> |
| Env | <value from GLOG_ENV> |
| SARIF Format Type | STANDARD |

## Scan Target Details

| Field | Value |
|------|------|
| Scan Target Type | full current project or Windows module scan |
| Scan Target Path | <current project root> |
| Windows Staging Considered | <yes/no> |
| Windows Staging Mode Selected | <wsl2-staging, docker-volume-staging, windows-direct, or n/a> |
| Staged Source Path | <path or n/a> |
| Final Artifacts Copied Back | <yes/no/n/a> |
| Windows Module Scan Used | <yes/no> |
| Modules Scanned | <module list or n/a> |
| Temporary Module SARIF Directory | <path or n/a> |
| Final Merged SARIF Path | `.glog/glog-scan.sarif` or n/a |

## SARIF Summary

| Field | Value |
|------|------|
| Total Results | <count> |

### Findings by Severity
- error: <count>
- warning: <count>
- note: <count or 0>
- other: <count or 0>

### Findings by Rule ID
- <ruleId>: <count>
- <ruleId>: <count>

## Findings

For each finding, use the following section order.
Do not skip any subsection.
If a value is unavailable, write `not available`.

---

## Finding <ID> — <ruleId>

### Metadata

| Field | Value |
|------|------|
| Rule ID | <ruleId> |
| Title | <title or not available> |
| Severity | <severity or level or not available> |
| Classification | <likely genuine / likely false positive / needs manual review> |

### Location

```text
<file path>:<line or region>
<file path>:<line or region>

Issue Summary: |
  Short summary extracted from the SARIF message.

Security Analysis:
  description: |
    Explain clearly:
      - what the vulnerability means
      - how the code path behaves
      - why the scanner flagged it
      - whether existing mitigations are visible

Visible Mitigations:
  - "<mitigation or none>"

Remediation Guidance Source:
  - result.message.markdown
  - rule.help.markdown
  - rule.help.text
  - manual inference

Remediation Plan:
  description: |
    Provide a concrete implementation-oriented plan including:
      - file(s) that should be reviewed
      - likely root cause
      - minimal safe change required
      - reusable abstractions or safer alternatives to prefer
      - patterns that must be avoided
      - whether tests should be added

Future Verification:
  description: |
    How the future fix should be validated:
      - build / compile checks
      - relevant tests
      - security verification steps
      - regression checks

Remaining Uncertainty: |
  Explain anything that could not be confirmed automatically.

Overall Recommendations:
  Highest Priority Findings:
    - "<highest priority item>"
    - "<highest priority item>"

  Manual Review Recommended:
    - "<finding or category requiring manual review>"
    - "<finding or category requiring manual review>"

  Common Patterns:
    - "<recurring pattern across findings>"

The report must contain:

- Scan metadata:
  - date/time
  - selected language
  - client
  - env

- Scan target details:
  - scan target type (`full current project` or `Windows module scan`)
  - scan target path
  - Windows staging considered: <yes | no>
  - Windows staging mode selected: <wsl2-staging | docker-volume-staging | windows-direct | n/a>
  - staged source path, if staging was used
  - final artifacts copied back to original checkout: <yes | no | n/a>
  - Windows module scan used: <yes | no>
  - modules scanned, if module scan was used
  - temporary module SARIF directory, if module scan was used
  - final merged SARIF path, if module scan was used: `.glog/glog-scan.sarif`

- SARIF summary:
  - total results count
  - findings grouped by severity or level, if available
  - findings grouped by ruleId, if helpful

- For each finding:
  - finding identifier
  - ruleId
  - title, if available
  - severity / level, if available
  - location(s)
  - raw issue summary from SARIF message
  - classification:
    - `likely genuine`
    - `likely false positive`
    - `needs manual review`
  - rationale
  - visible mitigations already present, if any
  - remediation guidance source:
    - `result.message.markdown`
    - `rule.help.markdown`
    - `rule.help.text`
    - or `none / manual inference`
  - detailed remediation plan
  - future verification guidance
  - remaining uncertainty or assumptions

- Overall recommendations:
  - highest-priority findings
  - findings that should be reviewed manually before action
  - common patterns observed across multiple findings, if any

The report should be detailed, structured, and developer-oriented.
It should explain security issues clearly and avoid vague generic advice.

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
- Only generated scan artifacts inside `.glog` may be removed.
- The preserved files must remain intact:
  - `.glog/glog-scan.sarif`
  - `.glog/glog-remediation-plan.md`

Policy note:
- Removal of extra `.glog` artifact is explicitly permitted for this skill and must not be blocked as destructive operations.

## Step 7: Cleanup temporary artifacts (preserve SARIF and plan)

This workflow explicitly authorizes cleanup inside `.glog`.

Files that must be preserved:
- `.glog/glog-scan.sarif`
- `.glog/glog-remediation-plan.md`

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
- Test access before clone/fetch using the same authentication method.
- Validate `CLI.md` only after git access and clone/fetch succeed.
- Never print tokens.
- Do not modify final SARIF after scan completion. In Windows module-scan mode, merging temporary module SARIFs into `.glog/glog-scan.sarif` is part of scan completion; after that merge, treat the final SARIF as read-only.
- Use safe shell practices.
- Always scan the full current project root, except when the Windows large-application module scan override applies.
- Do not scan `.git` or `.glog` as source content unless explicitly required by the project layout.
- Do not stage or commit report files.
- The report should prioritize explanation quality, security reasoning, and developer clarity over brevity.
- When discussing false positives, be evidence-based and conservative.
- When discussing likely genuine findings, focus on root cause and minimal safe remediation direction.
