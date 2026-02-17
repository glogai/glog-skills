---
name: glog-scan-remediate
description: Interactive Glog.AI scan + SARIF validation + focused remediation (no SARIF edits, no unrelated refactors).
tools:
  - shell
  - filesystem
---

# Interactive setup (ask user first)

Before doing ANY work, ask the user these questions (one by one) and wait for answers:

1) "What is the absolute path to your glog-action repo (the directory that contains CLI.md and/or glog.sh)?"
2) "Confirm --lang value (default: csharp)."
3) "Confirm --client value (default: test)."
4) "Confirm --env value (default: dev)."
5) "Confirm --sarif-format-type value (default: STANDARD)."

Rules:
- If the user provides empty answer for a value, use the default shown above.
- If the provided glog-action path does not exist or is missing CLI.md, ask again with a short hint.
- Do not proceed until all values are confirmed.

After collecting inputs, restate the final configuration in one short block and proceed.

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
- Run scan with the user-confirmed flags:
  - --lang
  - --client
  - --env
  - --sarif-format-type
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
- Apply flags: `--lang <...> --client <...> --env <...> --sarif-format-type <...>`
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

## Step 4 — Validate + Remediate
For each finding:
- Decide if genuine in this project context.
- If not genuine:
  - Explain why clearly.
  - Point to the relevant code (paths and what you observed).
- If genuine:
  - Implement remediation consistent with SARIF suggestion.
  - Keep changes minimal and focused.
  - Avoid code duplication by reusing existing utilities.
  - Add tests only if necessary and consistent with existing test stack.

## Step 5 — Limited optimization
- Only optimize code you touched.
- Only deduplicate what was introduced/affected by fixes.

## Step 6 — Final report
Provide:
- Summary of findings from `.glog/glog-scan.sarif`
- Which were genuine vs not genuine (and rationale)
- Actions performed (files changed + what changed)
- Any follow-ups

# Implementation notes (how to operate in shell)

- You may `cd` into `<GLOG_ACTION_PATH>` to run the glog-action entrypoint defined by CLI.md.
- You may `cd` back to the CURRENT project root as needed.
- Use safe shell practices (`set -euo pipefail` if writing scripts inline).
- Do not write to `.glog/glog-scan.sarif` after scan completion.
