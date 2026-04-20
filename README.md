# glog-skills

A collection of reusable **skills** used in this workspace. Each skill is defined by a `SKILL.md` file and is intended to be invoked by name during an interactive session.

**What this repo is for**
- Store reusable, task-specific skill instructions.
- Keep skills versioned alongside project work.
- Provide a simple layout that tools and agents can discover and apply.

**Repository layout**
- `glog/SKILL.md` — Glog.AI scan + SARIF validation + focused remediation workflow. Produces a remediation report and preserves scan artifacts.
- `glog-diff/SKILL.md` — Glog.AI scan + SARIF validation + focused remediation for changed files. Preserves SARIF and the remediation report.
- `glog-plan/SKILL.md` — Glog.AI scan + SARIF triage + remediation planning only. Does not modify source files.

**Install skills**
Before installing, ensure any required environment variables for the skill are already set. See **Skill requirements** below for the current Glog skill environment variables.

For Codex:
- Use `$skill-installer` to install from this repository.
- Restart Codex after installation so the new skills are picked up.

```bash
$skill-installer install https://github.com/glogai/glog-skills/tree/main/glog
$skill-installer install https://github.com/glogai/glog-skills/tree/main/glog-diff
$skill-installer install https://github.com/glogai/glog-skills/tree/main/glog-plan
```

For Claude Code:
- Claude Code loads personal skills from `~/.claude/skills/<skill-name>/SKILL.md`.
- Claude Code loads project-only skills from `.claude/skills/<skill-name>/SKILL.md`.
- From a checkout of this repository, install the skills for your user with:

```bash
mkdir -p ~/.claude/skills
cp -R ./glog ~/.claude/skills/
cp -R ./glog-diff ~/.claude/skills/
cp -R ./glog-plan ~/.claude/skills/
```

- To install them only for one project, copy the same directories into that project's `.claude/skills/` folder instead of `~/.claude/skills/`.

**Using a skill**
For Codex:
1. In your session, **mention the skill by name** (for example: `glog`) or invoke it directly (for example: `$glog`).
2. Codex will load the corresponding `SKILL.md` and follow its workflow.

For Claude Code:
1. Ask Claude to use the skill by name or invoke it directly with `/glog`, `/glog-diff`, or `/glog-plan`.
2. Claude Code will load the corresponding `SKILL.md` and follow its workflow.

In both tools, follow any prompts required by the skill (for example, language selection for a scan).

Example prompts:
```text
Use the glog skill to scan this repo.
```

**Skill requirements**
Some skills rely on environment variables or external tools. Check each `SKILL.md` for the source of truth.

- Common Glog skill environment variables: `GLOG_TOKEN`, `GITHUB_TOKEN`, and `GITHUB_USER`.
- Scan runtime variables for `glog`, `glog-diff`, and `glog-plan`: `GLOG_CLIENT` and `GLOG_ENV`.
- Additional skill-specific requirements may apply depending on the workflow.

**Adding a new skill**
1. Create a new folder at the repo root (for example, `my-skill/`).
2. Add `my-skill/SKILL.md` with front matter that includes at least `name` and `description`.
3. Keep workflows focused and minimal; document any required inputs or environment variables.

If you update or add skills, keep this README in sync with the available skills list.
