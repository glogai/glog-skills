---
name: "skill-interactive-demo"
description: "Interactive demo: asks a few questions, then generates a repo report file."
---

# Interactive demo skill

## Behavior
When invoked, ask the user these questions ONE BY ONE and wait for answers:
1) Output filename? (default: SKILL_TEST_REPORT.md)
2) Include git status? (yes/no, default: yes)
3) Scan for TODO/FIXME? (yes/no, default: yes)

After collecting answers, execute:
- Confirm repo root with `git rev-parse --show-toplevel`
- If include git status: run `git status --porcelain`
- If scan enabled: `rg -n "TODO|FIXME" .` (fallback to grep)

Create/overwrite the chosen report file in repo root with the selected sections.

Finally, print confirmation with the full path.
