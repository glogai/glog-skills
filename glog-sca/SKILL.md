---
name: glog-sca
description: Glog.AI software composition analysis (SCA) scan + dependency finding triage + remediation planning only. Produces a detailed dependency remediation plan and false-positive analysis without modifying source files, creating commits, or opening PRs.
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
- Do not write any secrets to disk.
- Do not echo tokens.

After preparing the cache repo, define:

`<GLOG_ACTION_PATH> = <cache_path>`

and proceed with the workflow using `<GLOG_ACTION_PATH>/CLI.md`.

# Scan configuration

This skill does NOT ask the user for scan language.

The scan engine must always be invoked with:

`--lang oss`

Rules:
- Do NOT prompt the user for a language.
- Do NOT allow `skip`.
- Do NOT replace `oss` with any other value.
- Keep other flags hardcoded:
  - `--client test`
  - `--env dev`
  - `--sarif-format-type STANDARD`

# Purpose

Use glog-action CLI instructions to run a software composition analysis scan for the CURRENT project, then analyze dependency findings from `.glog/glog-scan.sarif`, validate each finding conservatively, identify likely false positives or non-actionable findings, and produce a detailed remediation plan only.

This skill is planning-only:
- it must NOT modify application source files
- it must NOT modify dependency manifests
- it must NOT update package versions automatically
- it must NOT edit lockfiles
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

If any is missing, stop and tell the user exactly which one(s) are missing and how to export them.

# Critical constraints (must follow)

- Before analysis starts, clean `.glog` directory.
- Run scan with the hardcoded flags:
  - `--client test`
  - `--env dev`
  - `--sarif-format-type STANDARD`
  - `--lang oss`
- Always scan the CURRENT project root.
- Do NOT try to limit SCA by changed source files unless CLI.md explicitly requires or supports that behavior for OSS scanning.
- After scan is finished:
  - DO NOT modify `.glog/glog-scan.sarif` (read-only after creation).
- This skill is strictly analysis-and-plan only.
- DO NOT modify any project source file.
- DO NOT modify dependency manifest files such as:
  - `pom.xml`
  - `build.gradle`
  - `build.gradle.kts`
  - `package.json`
  - `package-lock.json`
  - `yarn.lock`
  - `pnpm-lock.yaml`
  - `requirements.txt`
  - `Pipfile`
  - `Pipfile.lock`
  - `poetry.lock`
  - `Gemfile`
  - `Gemfile.lock`
  - `go.mod`
  - `go.sum`
  - `Cargo.toml`
  - `Cargo.lock`
  - any equivalent dependency or lock file
- DO NOT modify configuration files, tests, build files, or documentation outside `.glog`.
- DO NOT create commits.
- DO NOT stage files.
- DO NOT push.
- DO NOT create pull requests.
- DO NOT perform any git write operation that records or publishes changes.
- DO NOT generate code patches unless the user explicitly asks later in a separate step.
- Treat remediation guidance as a focused dependency-security planning task.
- Be conservative:
  - if evidence is insufficient, classify as `needs manual review`
  - do not overstate certainty
- Generate a remediation planning report file inside `.glog` instead of only printing results in chat.
- The report must be saved as `.glog/glog-sca-remediation-plan.md`.

# Execution workflow

## Step 1: Read glog-action CLI.md (source of truth)

- Open and read: `<GLOG_ACTION_PATH>/CLI.md`
- Identify the exact command(s) to run `glog-action` for OSS / SCA scanning.
- Follow CLI.md instructions strictly.
- Confirm how the scan target path should be supplied.
- Confirm how the SARIF output is expected to land in the current project's `.glog/` directory.

## Step 2: Clean .glog and run scan

From the CURRENT project root (the repo you want to scan), do:

1) Clean `.glog`:
- If `.glog` exists: remove it completely
- Recreate `.glog/` directory

2) Execute scan using the invocation defined in CLI.md.
- Always use the current project root as the scan path.
- Apply flags exactly:
  - `--client test`
  - `--env dev`
  - `--sarif-format-type STANDARD`
  - `--lang oss`

3) If CLI.md expects the runner script to be executed from inside glog-action repo, run it from there but target the current project root exactly as specified by CLI.md.

4) Ensure that the output SARIF file ends up at:
- `.glog/glog-scan.sarif` in the CURRENT project.

After scan:
- Verify `.glog/glog-scan.sarif` exists.
- If missing, stop and show the scan command output and your best diagnosis.

## Step 3: Analyze findings from SARIF (read-only)

- Open `.glog/glog-scan.sarif` read-only.
- Iterate through the `results` list.
- Summarize findings:
  - ruleId / title (if present)
  - severity / level (if present)
  - message
  - locations (file + line/region) if present
  - package / dependency coordinates if visible
  - version information if visible
  - suggested remediation text (if present)

If the SARIF contains no results:
- Still create `.glog/glog-sca-remediation-plan.md`
- State clearly that no findings were present in the SARIF
- Include scan metadata

## Step 3.1: Determine remediation instructions source (SARIF)

Important: The SARIF file may not populate `result.fixes`.

Instead, remediation guidance may be located in:
- `result.message.markdown` (primary)
- `rule.help.markdown` or `rule.help.text` (secondary fallback)
- rule metadata / properties, if present
- dependency coordinates or advisory references visible in the SARIF payload

Rules:
- Prefer remediation steps explicitly described in `result.message.markdown`.
- If markdown contains multiple suggestions, pick the one that best matches:
  - the reported package
  - the affected version
  - the current ecosystem / package manager
  - the vulnerability type described by the ruleId and message
- If there is no clear remediation text, classify as `needs manual review` and explain why.

## Step 4: Validate + Explain + Plan (no code changes)

For each finding:

### 4.0 Planning-only guardrail

- This skill must not modify source code.
- This skill must not modify dependency declarations.
- This skill must not update package versions.
- This skill must not create branches, commits, pushes, or pull requests.
- This skill must not write outside `.glog`.
- The only new file that may be created is:
  - `.glog/glog-sca-remediation-plan.md`

### 4.1 Validate whether the finding is genuine

- Inspect the dependency context visible in the repository and SARIF.
- Consider whether the dependency appears to be:
  - direct
  - transitive
  - development-only
  - test-only
  - runtime-relevant
  - build-time only
  - unused but still present
- Use only evidence visible in the codebase and SARIF context.
- If evidence strongly indicates the finding is not actionable in the current context, classify it as one of:
  - `likely false positive`
  - `needs manual review`
- If evidence supports that the vulnerable dependency is genuinely present and relevant, classify it as:
  - `likely genuine`

### 4.2 Explain the reasoning

For each finding, explain:
- what the dependency/security issue means in practical terms
- which package or component appears to be affected
- whether the affected dependency appears direct or transitive, if visible
- why the dependency may be risky in this project context
- whether the issue appears runtime-relevant, test-only, or unclear
- why the finding is likely genuine, likely false positive, or needs manual review

### 4.3 Extract remediation guidance

- Parse `result.message.markdown` and identify actionable remediation steps.
- If remediation is vague, infer a concrete remediation plan that stays consistent with the intent of the advisory guidance.
- Prefer minimal dependency changes.
- Do not invent unrelated refactors.
- Do not produce patch text or apply edits.

### 4.4 Produce a remediation plan only

For each finding, produce a concrete plan that includes:
- the affected package / component
- the current version if visible
- the likely safe target version or remediation direction if visible
- whether the fix likely belongs in a direct dependency manifest or requires transitive dependency control
- the file(s) that should be reviewed, such as dependency manifests or lockfiles
- the likely root cause
- the minimal safe change that should be made in a future remediation step
- whether the dependency should be upgraded, replaced, excluded, overridden, or manually reviewed
- what must be avoided during the fix
- whether compatibility testing is likely needed
- what kind of verification should be performed after the future fix

The plan must be implementation-oriented, but must not modify code or dependency files.

### 4.5 Verification guidance for the future fix

For each finding, recommend how the future fix should be verified:
- confirm the vulnerable package version is no longer present
- re-run the SCA scan and verify the finding is no longer reported
- confirm the dependency graph resolves to the intended safe version
- run the relevant build/compile command for the project, if available
- run relevant existing tests, if present
- verify application startup or package resolution still works
- verify no incompatible transitive dependency conflicts were introduced
- verify expected valid behavior still works after the dependency change

### 4.6 False-positive and manual-review handling

If a finding appears not to be fully actionable, explain clearly:
- what the scanner likely matched
- whether the dependency is definitely present or only inferred
- whether the package appears reachable or actually used
- whether the issue may be limited to test/dev scope
- whether the dependency is transitive and needs ecosystem-specific confirmation
- what a developer should manually confirm before dismissing it

Do not dismiss a finding casually.
If uncertainty remains, prefer `needs manual review`.

## Step 5: Write the remediation planning report

Create:

`.glog/glog-sca-remediation-plan.md`

The report must be clear, structured, and optimized for developers.

Use this structure:

# Glog SCA Remediation Plan

## Scan Metadata
- Timestamp
- Working directory
- Scan type: `software-composition-analysis`
- Engine: `oss`
- Output SARIF path
- Base report file path

## Summary
- Total findings
- Count by severity/level, if available
- Count by classification:
  - `likely genuine`
  - `likely false positive`
  - `needs manual review`

## Dependency Findings Overview
For each finding, include:
- Finding number
- ruleId / title
- severity / level
- affected package / dependency
- affected version, if visible
- manifest / location, if visible
- advisory / message summary
- classification

## Detailed Analysis
For each finding, include:
- finding identifier
- package / version details
- evidence from SARIF
- practical risk explanation
- direct vs transitive assessment, if visible
- runtime vs test/build-only assessment, if visible
- classification
- reasoning
- remediation guidance source:
  - `result.message.markdown`
  - `rule.help.markdown`
  - `rule.help.text`
  - inferred from dependency context
- remediation plan
- verification guidance
- remaining risk or limitations, if any

## Prioritization Recommendations
Group findings into:
- immediate attention
- normal remediation backlog
- manual review first

Prioritize based on:
- severity if present
- exploitability clues visible in the finding
- runtime relevance
- whether the dependency appears direct
- confidence in the evidence

## Follow-up Recommendations
Include:
- whether lockfile review is needed
- whether dependency tree inspection is recommended
- whether version pinning / overrides may be needed
- whether transitive dependency management should be reviewed
- whether re-scan is required after future changes

## Final Notes
- This report is planning-only.
- No source files were modified.
- No dependency files were modified.
- No remediations were applied.
- `.glog/glog-scan.sarif` was kept read-only.

## Step 6: Final chat response

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
- Only generated scan artifacts may be removed.
- The preserved files must remain intact:
  - `.glog/glog-scan.sarif`
  - `.glog/glog-sca-remediation-plan.md`

Policy note:
- Removal of extra `.glog` artifacts is explicitly permitted for this skill and must not be blocked as destructive operations.

## Step 7: Cleanup temporary artifacts (preserve SARIF and report)

This workflow explicitly authorizes cleanup inside `.glog`.

Files that must be preserved:
- `.glog/glog-scan.sarif`
- `.glog/glog-sca-remediation-plan.md`

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

- Use `http.extraHeader` with `GITHUB_TOKEN` for git HTTPS operations against the private glog-action repository.
- Never print tokens.
- Do not modify SARIF after scan.
- Use safe shell practices.
- Always use `--lang oss`.
- Prefer evidence visible in SARIF and repository dependency manifests.
- Be conservative when distinguishing direct vs transitive dependencies if the SARIF does not make that explicit.
- If the repository contains multiple package managers or manifests, note that clearly in the report.
- If dependency context is ambiguous, classify as `needs manual review`.