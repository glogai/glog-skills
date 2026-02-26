# glog-skills

A small collection of reusable **skills** used in this workspace. Each skill is defined by a `SKILL.md` file and is intended to be invoked by name during an interactive session.

**What this repo is for**
- Store reusable, task-specific skill instructions.
- Keep skills versioned alongside project work.
- Provide a simple layout that tools and agents can discover and apply.

**Repository layout**
- `glog/SKILL.md` — Glog.AI scan + SARIF validation + focused remediation workflow. Produces a remediation report and preserves scan artifacts.

**Install a skill**
Before installing, ensure any required environment variables for the skill are already set. For example, the `glog` skill expects `GLOG_TOKEN`, `GITHUB_TOKEN`, and `GITHUB_USER`.

Install `glog` with:
```bash
$skill-installer install https://github.com/glogai/glog-skills/tree/main/glog
```

**Using a skill**
1. In your session, **mention the skill by name** (for example: `glog`) or invoke it directly (for example: `$glog`).
2. The agent or tool will load the corresponding `SKILL.md` and follow its workflow.
3. Follow any prompts required by the skill (for example, language selection for a scan).

Example prompts:
```text
Use the glog skill to scan this repo.
```

**Skill requirements**
Some skills rely on environment variables or external tools. For example, the `glog` skill expects tokens such as `GLOG_TOKEN`, `GITHUB_TOKEN`, and `GITHUB_USER` to already be set in the environment. Check each `SKILL.md` for exact requirements.

**Adding a new skill**
1. Create a new folder at the repo root (for example, `my-skill/`).
2. Add `my-skill/SKILL.md` with front matter that includes at least `name` and `description`.
3. Keep workflows focused and minimal; document any required inputs or environment variables.

If you update or add skills, keep this README in sync with the available skills list.
