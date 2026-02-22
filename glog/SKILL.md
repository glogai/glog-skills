---
name: glog
description: Glog.AI scan + SARIF validation + focused remediation (no SARIF edits, no unrelated refactors). Writes report to .glog and cleans artifacts (preserving SARIF + report).
tools:
  - shell
  - filesystem
---

# Setup

Use this hardcoded glog-action path:

`/home/matija/glog/glog-action`

Do not prompt the user for it.

Rules:
- If hardcoded path does not exist or is missing CLI.md, stop with a clear error..

# Scan configuration

Before doing ANY scan work, ask the user:

"Which language should be used for scanning? (This controls the optional --lang flag.)  
Reply with a language value like: python, csharp, java, javascript, etc.  
If you want to skip, reply with: skip"

Rules:
- The user response is required (the agent must ask and wait).
- If the user replies with "skip" (or an empty/whitespace-only response), DO NOT pass `--lang` at all.
- Otherwise pass `--lang <user_value>` exactly as provided (trim whitespace).
- Do not invent or default a language.
- Keep other flags hardcoded:
  - `--client test`
  - `--env dev`
  - `--sarif-format-type STANDARD`

# Purpose

Use glog-action CLI instructions to run a scan for the CURRENT project, then analyze findings from `.glog/glog-scan.sarif`, validate each finding, and apply focused remediations where genuine.

# Required environment

Use global environment variables (do not prompt for them unless missing):
- GLOG_TOKEN
- GITHUB_TOKEN
- GITHUB_USER

If any is missing, stop and tell the user exactly which one(s) are missing and how to export them.

# Critical constraints (must follow)

- Before analysis starts, clean `.glog` directory.
- Run scan with the hardcoded flags:
  - --client test
  - --env dev
  - --sarif-format-type STANDARD
  - `--lang <user_value>` ONLY if the user provided a language (not skip/empty). Otherwise do not include `--lang` at all.
- After scan is finished:
  - DO NOT modify `.glog/glog-scan.sarif` (read-only after creation).
- Treat remediation advice as a focused security remediation task.
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

## Step 1 — Read glog-action CLI.md (source of truth)
- Open and read: `<GLOG_ACTION_PATH>/CLI.md`
- Identify the exact command(s) to run `glog-action` (entrypoint, required args, docker usage, etc.)
- Follow CLI.md instructions strictly.

## Step 2 — Clean .glog and run scan
From the CURRENT project root (the repo you want to scan), do:

1) Clean `.glog`:
- If `.glog` exists: remove it completely
- Recreate `.glog/` directory

2) Execute scan using the invocation defined in CLI.md.
- Apply flags exactly:
  - `--client test`
  - `--env dev`
  - `--sarif-format-type STANDARD`
  - Apply `--lang <value>` ONLY if the user provided a language (not skip/empty). If skipped, do not pass `--lang`.
- If CLI.md expects the runner script to be executed from inside glog-action repo, run it from there but target the current project as specified by CLI.md (e.g., via mount/path arguments).
- Ensure that the output SARIF file ends up at: `.glog/glog-scan.sarif` in the CURRENT project.

After scan, verify `.glog/glog-scan.sarif` exists.
If missing, stop and show the scan command output and your best diagnosis.

## Step 3 — Analyze findings from SARIF (read-only)
- Open `.glog/glog-scan.sarif` read-only.
- Summarize findings:
  - ruleId / title (if present)
  - severity/level (if present)
  - message
  - locations (file + line/region)
  - suggested remediation text (if present)

## Step 3.1 — Determine remediation instructions source (SARIF)

Important: The SARIF file does NOT populate `result.fixes`.  
Instead, remediation guidance is located in:

- `result.message.markdown` (primary)
- `rule.help.markdown` or `rule.help.text` (secondary fallback)

Rules:
- Prefer remediation steps explicitly described in `result.message.markdown`.
- If markdown contains multiple suggestions, pick the one that matches:
  - the reported file path / code location
  - the project’s technology stack
  - the vulnerability type described by ruleId/message
- If there is no clear remediation text, treat the finding as "needs manual review" and explain why.


## Step 4 — Validate + Remediate (markdown-driven remediation)

For each finding:

### 4.1 Validate whether the finding is genuine
- Inspect the exact location(s) in code referenced by the SARIF.
- Consider runtime behavior, framework defaults, and existing mitigations.
- If NOT genuine:
  - Explain why clearly (for additional review).
  - Reference relevant code paths and reasoning.
  - Continue to the next finding.

### 4.2 Extract remediation steps (from result.message.markdown)
- Parse `result.message.markdown` and identify actionable remediation steps.
- If remediation is vague, infer a concrete plan that stays consistent with the intent of the markdown guidance.
- Do not invent unrelated refactors.

### 4.3 Apply remediation (minimal changes, no duplication)
- Implement the fix as suggested in the markdown guidance as closely as possible.
- Adapt to the current project’s architecture and tech stack.
- Avoid code duplication:
  - reuse existing helper methods/utilities
  - avoid copy-pasting logic across modules
- Do not refactor unrelated code.

### 4.4 Verify remediation is correct for this codebase
After implementing a fix, verify it is correct by doing ALL applicable checks:

- Build/compile succeeds (or equivalent for the stack).
- Tests pass (if tests exist); add minimal targeted tests only if necessary.
- The vulnerable pattern is no longer present at the reported location.
- The fix does not introduce regressions or break API behavior.

### 4.5 If remediation guidance is incorrect or incomplete, adapt it
If the markdown remediation does NOT fit the actual code/context (e.g. wrong framework assumption, wrong API usage, breaking change, not resolving the issue):

- Modify the remediation to a correct variant for this project, while keeping the original intent.
- Clearly document in the report:
  - what the markdown suggested
  - why it didn’t fit
  - what you changed instead and why

Stop short of broad refactors; change only what is needed to properly remediate the finding.


## Step 5 — Limited optimization
- Only optimize code you touched.
- Only deduplicate what was introduced/affected by fixes.

## Step 6 — Final report
Create a remediation report file at:
`.glog/glog-remediation-report.md`

The report must contain:
- Scan metadata (date, lang, client, env)
- Summary of findings from `.glog/glog-scan.sarif`
- For each finding:
  - Genuine vs not genuine
  - Rationale
  - Files impacted
  - Remediation performed (if any)
  - Remediation source: result.message.markdown (and whether adapted)
  - If adapted: what was changed and why
- List of code changes
- Any follow-ups / recommendations

After saving the report file:
- Provide a short summary in chat
- Do not print the full report content in chat unless the user asks

## Step 7 — Cleanup .glog (preserve SARIF and report)
After all analysis and remediation work is finished:

Files that must be preserved:
- `.glog/glog-scan.sarif`
- `.glog/glog-remediation-report.md`

Process:
1) Ensure both files exist.
   - If any is missing, warn the user and do not delete `.glog`.
2) Temporarily move both files outside `.glog`.
3) Remove the entire `.glog` directory.
4) Recreate `.glog`.
5) Move preserved files back.

Do not modify file contents during cleanup.

# Implementation notes (how to operate in shell)

- You may `cd` into `<GLOG_ACTION_PATH>` to run the glog-action entrypoint defined by CLI.md.
- You may `cd` back to the CURRENT project root as needed.
- Use safe shell practices (`set -euo pipefail` if writing scripts inline).
- Do not write to `.glog/glog-scan.sarif` after scan completion.
