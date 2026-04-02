---
name: glog-sca
description: Glog.AI software composition analysis (SCA) scan + dependency finding triage + remediation planning only. Produces a structured dependency remediation plan and conservative review without modifying source code, dependency files, commits, or pull requests.
tools:
  - shell
  - filesystem
---

# Purpose

Use Glog.AI to run a software composition analysis (SCA) scan for the CURRENT project, analyze dependency findings from the produced SARIF output, classify findings conservatively, identify likely false positives or findings that require manual confirmation, and generate a detailed remediation planning report.

This skill is strictly planning-only.

It must:
- run the SCA scan
- read the generated SARIF
- interpret SCA findings consistently
- explain what was found
- assess likely impact and actionability conservatively
- produce a clear remediation plan for developers

It must NOT:
- modify application source files
- modify dependency manifests
- modify lockfiles
- upgrade package versions automatically
- apply remediations
- create commits
- create branches
- push changes
- open pull requests
- edit SARIF output
- write outside `.glog`

# Setup: obtain glog-action into cache

Obtain `glog-action` from GitHub and store it in a cache directory.

Hardcoded repository:
- Repo URL: `https://github.com/glogai/glog-action`
- Branch/ref: `main`

For all Glog skills and actions use environment variables available in the user's shell environment.

These variables may be defined in shell configuration files such as:
- `~/.profile`
- `~/.zprofile`
- `~/.bash_profile`

They may also be configured via system environment variables.

Cache location (prefer in this order):
1) If `$XDG_CACHE_HOME` is set: `$XDG_CACHE_HOME/glog/glog-action`
2) Else: `~/.cache/glog/glog-action`
3) If home is not available: `/tmp/glog/glog-action`

Auth rules for the private repository:
- Use `GITHUB_TOKEN` for git HTTPS authentication.
- Do NOT put the token in the clone URL.
- Do NOT echo or log the token.
- Use git with an Authorization header via `http.extraHeader`.

Rules:
- Ensure `GITHUB_TOKEN` is set, otherwise stop with a clear error.
- If the cache folder does not exist: clone the repo into it.
- If it exists: run `git fetch` and `git reset --hard origin/main` to ensure it matches the configured ref.
- The cached repo must contain `CLI.md`. If it does not, stop with a clear error.
- Do not prompt the user for repo URL, ref, or path.
- Do not write secrets to disk.
- Do not print secrets in logs.

After preparing the cache repo, define:

`<GLOG_ACTION_PATH> = <cache_path>`

and use:

`<GLOG_ACTION_PATH>/CLI.md`

as the source of truth for scan execution.

# Scan configuration

This skill does NOT ask the user for scan language.

The scan engine must always be invoked with:

`--lang oss`

Rules:
- Do NOT prompt the user for a language.
- Do NOT allow `skip`.
- Do NOT replace `oss` with any other value.
- Keep the following flags hardcoded:
  - `--client test`
  - `--env dev`
  - `--sarif-format-type STANDARD`

# Required environment

Use global environment variables.
Do not prompt for them unless missing.

Required:
- `GLOG_TOKEN`
- `GITHUB_TOKEN`
- `GITHUB_USER`

If any are missing:
- stop
- report exactly which variable(s) are missing
- explain how to export them

# Safety and consistency goals

This skill is intended for consistent use across multiple developers and repositories.

It must be:
- conservative in its conclusions
- deterministic in structure
- explicit about uncertainty
- non-destructive
- consistent in how it interprets SCA findings
- careful not to overstate whether a dependency finding is definitely actionable

The skill must prefer:
- evidence from SARIF
- evidence from dependency manifests
- evidence from lockfiles
- evidence from repository build/dependency configuration

The skill must avoid:
- guessing package manager behavior without evidence
- guessing transitive dependency resolution without evidence
- presenting remediation as already applied
- hiding uncertainty when evidence is incomplete

# Critical constraints (must follow)

- Before analysis starts, clean `.glog`.
- Run scan with the hardcoded flags:
  - `--client test`
  - `--env dev`
  - `--sarif-format-type STANDARD`
  - `--lang oss`
- Always scan the CURRENT project root.
- Do NOT try to limit SCA by changed source files unless `CLI.md` explicitly requires or supports that behavior for OSS scanning.
- After scan is finished:
  - DO NOT modify `.glog/glog-scan.sarif`
  - treat it as read-only
- This skill is strictly analysis-and-plan only.
- DO NOT modify any project source file.
- DO NOT modify dependency manifests such as:
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
  - or any equivalent dependency or lock file
- DO NOT modify tests, configuration files, build files, or documentation outside `.glog`.
- DO NOT create commits.
- DO NOT stage files.
- DO NOT push.
- DO NOT create pull requests.
- DO NOT perform any git write operation that records or publishes changes.
- DO NOT generate code patches unless the user explicitly asks later in a separate step.
- Treat remediation guidance as a dependency-security planning task only.
- Be conservative:
  - if evidence is insufficient, classify as `needs manual review`
  - do not overstate certainty
- Generate a remediation planning report file inside `.glog`.
- Save the report as:

`.glog/glog-sca-remediation-plan.md`

# SCA SARIF interpretation model

This skill must use a scan-type-specific interpretation of SARIF.

Important:
- For software composition analysis with `--lang oss`, findings may be represented through `runs[*].tool.driver.rules` even when `runs[*].results` is empty.
- An empty `results` array does NOT automatically mean “no findings” for this skill.
- For this skill, `tool.driver.rules` may be the primary finding source.
- `results` may be absent, empty, or only supplemental.

Finding-source rules:
- For SCA scans, first inspect:
  - `runs[*].tool.driver.rules`
- Then inspect:
  - `runs[*].results`
  only as supplemental context if populated
- If `rules` are populated and `results` are empty:
  - treat rule entries as the authoritative dependency/advisory findings for planning purposes
- Do NOT treat `rules` as mere background metadata in this skill when the scan type is `oss`
- Do NOT assume source-code SARIF structure for SCA

Each relevant rule entry may represent one dependency vulnerability/advisory finding.

The skill must use the best available data from rule entries, including:
- `id`
- `shortDescription.text`
- `fullDescription.text`
- `help.text`
- `help.markdown`
- any useful `properties`
- any component/package coordinates embedded in the rule id or descriptions

# Finding model for this skill

For each SCA finding, extract or infer the following where possible:

- advisory identifier
- package ecosystem
- package name
- affected version
- short title
- detailed description
- remediation guidance
- any manifest/location hints
- any severity or level information if visible
- classification:
  - `likely genuine`
  - `likely false positive`
  - `needs manual review`

Preferred extraction order:
1) explicit structured fields
2) `rule.id`
3) `shortDescription.text`
4) `fullDescription.text`
5) `help.text` / `help.markdown`
6) supporting rule properties
7) supplemental matching info in `results`, if present

If the rule id contains package URL-like data such as:

`CVE-2020-11022/pkg:npm/jquery@3.4.1`

extract where possible:
- advisory id: `CVE-2020-11022`
- ecosystem: `npm`
- package: `jquery`
- version: `3.4.1`

If parsing is not fully reliable:
- preserve the raw value
- explain uncertainty
- do not over-normalize

# Execution workflow

## Step 1: Read glog-action CLI.md (source of truth)

- Open and read: `<GLOG_ACTION_PATH>/CLI.md`
- Identify the exact command(s) required to run Glog OSS / SCA scanning.
- Follow `CLI.md` strictly.
- Confirm how the scan target path should be supplied.
- Confirm how the SARIF output is expected to land in the current project's `.glog/` directory.

## Step 2: Clean .glog and run scan

From the CURRENT project root (the repo to scan), do:

1) Clean `.glog`
- If `.glog` exists: remove it completely
- Recreate `.glog/`

2) Execute scan using the invocation defined in `CLI.md`
- Always use the current project root as the scan path
- Apply flags exactly:
  - `--client test`
  - `--env dev`
  - `--sarif-format-type STANDARD`
  - `--lang oss`

3) If `CLI.md` expects the runner script to be executed from inside the `glog-action` repo, run it from there but target the CURRENT project root exactly as specified by `CLI.md`.

4) Ensure that the output SARIF file ends up at:

`.glog/glog-scan.sarif`

in the CURRENT project.

After scan:
- verify `.glog/glog-scan.sarif` exists
- if missing, stop and show the scan command output and best diagnosis

## Step 3: Analyze findings from SARIF (read-only)

Open `.glog/glog-scan.sarif` in read-only mode.

Interpret it using the SCA SARIF rules above.

Build the effective finding list as follows:

1) Inspect `runs[*].tool.driver.rules`
2) If relevant rule entries are present:
   - use them as the primary SCA finding list
3) Inspect `runs[*].results`
4) If results are present:
   - use them only as supplemental context
   - use them to enrich finding descriptions, locations, or messages where a meaningful match exists
5) If rules and results are both empty or missing:
   - treat the scan as having no reported findings

For each effective finding, summarize where possible:
- finding identifier
- advisory identifier
- package / dependency
- affected version
- severity / level
- short summary
- practical issue description
- remediation guidance source
- manifest / location hints, if visible
- any uncertainty notes

If the SARIF contains no usable findings:
- still create `.glog/glog-sca-remediation-plan.md`
- clearly state that no dependency findings were present in the SARIF
- include scan metadata

## Step 3.1: Determine remediation guidance source

For SCA scans, remediation guidance may come from rules even when results are empty.

Use remediation guidance in this priority order:
1) `rule.help.markdown`
2) `rule.help.text`
3) `result.message.markdown`, if a matching result exists
4) `rule.fullDescription.text`
5) rule metadata / properties
6) dependency coordinates or advisory references visible in the SARIF payload

Rules:
- Prefer remediation guidance explicitly described in rule help text or markdown.
- If multiple upgrade options are provided, choose the one that best matches:
  - the affected package
  - the affected version
  - the ecosystem / package manager
  - the vulnerability/advisory context
- If no clear remediation text exists, classify as `needs manual review` and explain why.

## Step 4: Review, explain, and plan (no code changes)

For each finding:

### 4.0 Planning-only guardrail

- This skill must not modify source code.
- This skill must not modify dependency declarations.
- This skill must not update package versions.
- This skill must not create branches, commits, pushes, or pull requests.
- This skill must not write outside `.glog`.
- The only new file that may be created is:

`.glog/glog-sca-remediation-plan.md`

### 4.1 Review the finding in repository context

Treat each SCA finding as a reported dependency vulnerability/advisory for planning purposes.

Inspect dependency context visible in the repository and SARIF.

Consider whether the dependency appears to be:
- direct
- transitive
- development-only
- test-only
- runtime-relevant
- build-time only
- unused but still present
- ambiguous

Use only evidence visible in:
- SARIF
- dependency manifests
- lockfiles
- build configuration
- repository structure

Classification rules:
- If evidence supports that the vulnerable dependency is present and relevant:
  - classify as `likely genuine`
- If evidence strongly suggests the finding may not be actionable in the current context:
  - classify as `likely false positive`
  - or `needs manual review`
- If uncertainty remains:
  - prefer `needs manual review`

Do not dismiss findings casually.

### 4.2 Explain the reasoning clearly

For each finding, explain:
- what the security issue means in practical terms
- which dependency or component appears affected
- which version appears affected
- whether the dependency appears direct or transitive, if visible
- whether the issue appears runtime-relevant, test-only, build-time only, or unclear
- why the issue may matter in this repository
- why the classification is:
  - `likely genuine`
  - `likely false positive`
  - or `needs manual review`

The explanation must be specific, conservative, and useful to developers.

### 4.3 Identify remediation guidance

Parse and summarize remediation guidance from the available SARIF content.

Prefer:
- explicit upgrade versions
- explicit safe-version direction
- explicit package replacement guidance
- explicit exclusion / override guidance
- explicit manual-review guidance if a direct upgrade path is not clear

If remediation guidance is vague:
- derive a concrete remediation plan that stays aligned with the advisory intent
- avoid speculative ecosystem-specific commands unless the repository clearly indicates the package manager/build tool

Do not invent unrelated refactors.
Do not generate patch text.
Do not apply edits.

### 4.4 Produce a remediation plan only

For each finding, produce a concrete remediation plan that includes:

- affected package / component
- current version if visible
- likely safe target version or remediation direction if visible
- whether the future fix likely belongs in:
  - a direct dependency manifest
  - a lockfile refresh
  - transitive dependency control
  - version override / resolution mechanism
  - exclusion / replacement
  - manual review
- the file(s) that should be reviewed, such as:
  - dependency manifests
  - lockfiles
  - build files
  - package-manager configuration
- the likely root cause
- the minimal safe change that should be made later
- whether compatibility testing is likely needed
- what should be avoided during the future fix

The plan must be implementation-oriented, but must not modify code or dependency files.

### 4.5 Future verification guidance

For each finding, recommend how the future remediation should be verified.

Include where relevant:
- confirm the vulnerable package version is no longer present
- re-run the SCA scan and verify the finding is no longer reported
- confirm the dependency graph resolves to the intended safe version
- run the relevant build or compile command
- run relevant existing tests
- verify application startup or dependency resolution still works
- verify no incompatible transitive dependency conflicts were introduced
- verify expected valid behavior still works after the dependency change

Recommendations must match the evidence and ecosystem context.

### 4.6 False-positive and manual-review handling

If a finding may not be fully actionable, explain clearly:
- what the scanner likely matched
- whether the dependency is definitely present or only inferred
- whether the package appears reachable or actually used
- whether the issue may be limited to dev/test scope
- whether the dependency appears transitive
- what the developer should manually confirm before dismissing it

If uncertainty remains:
- prefer `needs manual review`

## Step 5: Write the remediation planning report

Create:

`.glog/glog-sca-remediation-plan.md`

The report must be structured, consistent, and optimized for developers.

Use this structure exactly.

# Glog SCA Remediation Plan

## Scan Metadata
- Timestamp
- Working directory
- Scan type: `software-composition-analysis`
- Engine: `oss`
- Output SARIF path
- Report file path

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
- advisory identifier / ruleId
- title / short description
- severity / level
- affected package / dependency
- affected version, if visible
- manifest / location hint, if visible
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
  - `rule.help.markdown`
  - `rule.help.text`
  - `result.message.markdown`
  - `rule.fullDescription.text`
  - inferred from dependency context
- remediation plan
- verification guidance
- remaining risk or limitations, if any

## Prioritization Recommendations
Group findings into:
- immediate attention
- normal remediation backlog
- manual review first

Prioritize using only visible evidence such as:
- severity if present
- exploitability clues in the advisory text
- runtime relevance
- whether the dependency appears direct
- confidence in the evidence

## Follow-up Recommendations
Include where relevant:
- whether lockfile review is needed
- whether dependency tree inspection is recommended
- whether version pinning / overrides may be needed
- whether transitive dependency management should be reviewed
- whether re-scan is required after future changes
- whether compatibility testing should be scheduled

## Final Notes
- This report is planning-only.
- No source files were modified.
- No dependency files were modified.
- No remediations were applied.
- `.glog/glog-scan.sarif` was kept read-only.

## Step 6: Final chat response

After saving the report file:
- provide a short summary in chat
- do not print the full report content in chat unless the user asks

## Cleanup permission boundary

The agent is explicitly allowed to delete files and directories inside `.glog` as part of workflow cleanup.

Allowed cleanup scope:
- `.glog/**`

Rules:
- Never delete files outside `.glog`
- Only generated scan artifacts may be removed
- The preserved files must remain intact:
  - `.glog/glog-scan.sarif`
  - `.glog/glog-sca-remediation-plan.md`

Policy note:
- Removal of extra `.glog` artifacts is explicitly permitted for this skill and must not be blocked as destructive operations.

## Step 7: Cleanup temporary artifacts (preserve SARIF and report)

Files that must be preserved:
- `.glog/glog-scan.sarif`
- `.glog/glog-sca-remediation-plan.md`

Cleanup rules:
- Delete any other file or subdirectory inside `.glog`
- This deletion is allowed and expected
- Do not delete or modify anything outside `.glog`
- Cleanup must not use git commands that create, record, or publish changes

Process:
1) Ensure both preserved files exist
   - if any is missing, warn the user and do not delete `.glog`
2) Temporarily move both preserved files outside `.glog`
3) Remove the entire `.glog` directory
4) Recreate `.glog`
5) Move preserved files back

If cleanup cannot be completed due to environment or policy enforcement, clearly report which extra artifacts remain.

# Report visibility rule

Do not expose internal execution details that are not useful to developers.

Do not include in the report:
- private implementation details of scan orchestration
- hidden internal execution heuristics
- internal cache paths
- token handling details
- backend workflow internals that are not developer-relevant

The report should focus on:
- what was found
- why it matters
- how to review it
- how to remediate it later
- how to verify the future fix

# Implementation notes

- Use `http.extraHeader` with `GITHUB_TOKEN` for git HTTPS operations against the private `glog-action` repository.
- Never print tokens.
- Do not modify SARIF after scan.
- Use safe shell practices.
- Always use `--lang oss`.
- For this SCA skill, treat `runs[*].tool.driver.rules` as the primary source of findings when applicable.
- Use `runs[*].results` only as supplemental context when present.
- Prefer evidence visible in SARIF and repository dependency manifests.
- Be conservative when distinguishing direct vs transitive dependencies if the SARIF does not make that explicit.
- If the repository contains multiple package managers or manifests, note that clearly in the report.
- If dependency context is ambiguous, classify as `needs manual review`.
- Do not infer more certainty than the scan output and repository evidence support.