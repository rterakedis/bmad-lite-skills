[← Back to README](../README.md)

## Installation

There are two ways to use these skills: **plugin install** (recommended for new setups, see the [README quickstart](../README.md#quickstart)) or **workspace directory** (the original approach, still fully supported).

### Option B — Workspace directory (original approach)

These skills can also be **added to any Claude Code session as a workspace directory** — clone once and reference across all your projects.

#### 1 — Clone once

```bash
git clone https://github.com/rterakedis/bmad-lite-skills ~/repos/bmad-lite-skills
```

Put it wherever you keep shared tools. The path doesn't matter as long as it's consistent.

#### 2 — Add to your Claude Code session

At the start of any session, run:

```
/add-dir ~/repos/bmad-lite-skills
```

Claude can now read all skill files from that directory alongside your project.

#### 3 — Wire up auto-loading (recommended)

To avoid running `/add-dir` manually every session, add it as a startup hook in your project's `.claude/settings.json`:

```json
{
  "hooks": {
    "startup": [
      "add-dir ~/repos/bmad-lite-skills"
    ]
  }
}
```

The `/setup` skill writes this hook for you when initializing a new project — just tell it where you cloned this repo.

## Keeping skills up to date

```bash
cd ~/repos/bmad-lite-skills
git pull
```

All projects that reference this directory pick up the update immediately — nothing to copy or sync per project.

If you installed via the plugin marketplace instead, run `/plugin marketplace update` to pick up new commits.
