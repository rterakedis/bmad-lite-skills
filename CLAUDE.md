# CLAUDE.md — bmad-lite-skills

This repo is a customized fork of [BMAD-LITE](https://github.com/bmad-method/BMAD-LITE). It ships as a Claude Code plugin (`bmad-lite`) via `.claude-plugin/plugin.json`. Skills live in `.claude/skills/` and are registered as `SKILL.md` (uppercase) — the upstream uses `skill.md` (lowercase).

## Purpose

Maintain a curated, customized set of BMAD-LITE skills for use in Claude Code projects. The goal is to selectively pull upstream improvements from BMAD-LITE while preserving intentional local enhancements.

---

## Upstream Sync Workflow

Upstream repo is at `/Users/rterakedis/Git-Repos/BMAD-LITE/` (local clone).

To evaluate and pull upstream changes for a skill:

```bash
# See what changed upstream for a specific skill
diff .claude/skills/<name>/SKILL.md ../BMAD-LITE/skills/<name>/skill.md

# See all skills with upstream divergence
for skill in .claude/skills/*/; do
  name=$(basename "$skill")
  result=$(diff "$skill/SKILL.md" "../BMAD-LITE/skills/$name/skill.md" 2>/dev/null)
  [ -n "$result" ] && echo "DIFFERS: $name"
done
```

When pulling upstream changes, copy content into `SKILL.md` (preserving uppercase filename). Do not blindly overwrite — check against the local customizations documented below.

---

## Local Customizations by Skill

These skills have intentional divergence from upstream. Preserve these when syncing.

### `setup`
Scaffold question 3 changed from a yes/no Apple platform question to a multi-select: "Which Apple platform(s)? (iOS / iPadOS / macOS / none)". Step 3b copies the 6 shared Swift stubs unconditionally when any Apple platform is selected, then adds `ipados-specific.md` if iPadOS is selected and `macos-specific.md` if macOS is selected. Step 3a appends the guardrails block to CLAUDE.md (checks for `## Swift/SwiftUI Guardrails`, not the old `## Modern SwiftUI Patterns` heading).

### `check-readiness`
Added **Check 7** (cross-epic runtime dependency analysis) and **Check 8** (testing targets derived from architecture, with codification into `CLAUDE.md`). These checks are not in upstream.

### `code-review`
Two additions:
1. Clean-review shortcut line: if no findings, write `Clean review — no patches or deferred items.` instead of an empty checklist.
2. **Epic context update pass** after review: scan for discoveries (constraints, schema details, invariants, "learned the hard way" items) and append them as `## Story {id} Learnings` in `docs/epics/epic-<n>-context.md`.

### `create-story`
Added mandatory **cross-epic runtime dependency check** before writing a story: requires explicitly asking whether the story depends on runtime artifacts (tables, migrations, seed data, endpoints) from a different epic. If yes, note under `### Prerequisites` in Dev Notes and flag sequencing risk to the user.

### `deferred`
- `d_id` is a required parameter in `LOG-AND-SCHEDULE` (upstream treats it as optional).
- SCHEDULE return message includes `slotted into Story {epic}.{N}` phrasing; upstream uses slightly different wording.

### `dev-story`
Richer task-tracking instructions:
- Explicit direction to check off `[ ]` → `[x]` on Tasks/Subtasks AND Acceptance Criteria during implementation (not after).
- Explicit direction to check off Architecture Compliance Checklist items (if present in Dev Notes) before marking done.
- `Don't modify` list expanded to include Dev Notes prose and References (upstream is more terse).

### `epics`
Added **cross-epic runtime dependency scan** step after drafting all epics but before writing. Checks whether any story in an earlier epic requires a runtime artifact (migration, seed data, table, endpoint) from a later epic, and annotates both epics with dependency notes or flags potential reordering.

### `github-tracking`
Added **SYNC** operation: reconciles GitHub issue labels and open/closed state against `status:` frontmatter in every story file. Idempotent; prints a diff table before applying. Invocable as `/github-tracking sync`. Upstream only has `setup` and `backfill`.

### `retrospective`
Expanded from five questions to **seven questions**, with two additions:
- Q2: "What went well?" (reinforcement, distinct from pattern codification)
- Q5: "Did last sprint's conventions hold?" (audit stories against prior retro conventions)

Also added a two-pass deferred item audit:
- **Pass 1**: Scan all story files for `[Defer]` entries not yet logged in `docs/deferred-items.md`; call `LOG-AND-SCHEDULE` for any unlogged items.
- **Pass 2**: Verify all logged deferred items are scheduled into open work in `docs/epics.md`.

### `story-flywheel`
Epic discovery sorts by **epic number embedded in milestone title** (e.g., `"Epic 3 — ..."` → 3), not by GitHub milestone ID. This is intentional: GitHub assigns milestone IDs in creation order, which doesn't reflect intended epic sequence. Upstream sorts by milestone ID.

Also uses `gh api` with `--jq` for more precise issue filtering within a milestone (rather than `gh issue list` piped to `first`).

---

### `refresh-swiftui`
New skill with no upstream equivalent. Researches current Swift/SwiftUI best practices from gold-standard sources (Hacking with Swift, Swift with Majid, SwiftLee, Apple WWDC docs, Point-Free) and updates both the project's `docs/setup/swift/` sectioned reference docs and the skills repo stubs. Triggered via `/refresh-swiftui`. Scope is iOS 18 through current stable release — hard-excludes pre-release APIs.

---

## Skills Identical to Upstream

No local changes — safe to overwrite from upstream on sync:

`architecture`, `correct-course`, `discover`, `investigate`, `prd`, `quick-dev`, `security-review`, `setup`, `status`, `ux`

---

## Repo Structure

```
.claude/
  skills/
    <skill-name>/
      SKILL.md        # skill prompt (uppercase — plugin convention)
.claude-plugin/
  plugin.json         # plugin manifest (name, description, author)
```

## Conventions

- Skill files are always named `SKILL.md` (uppercase). Upstream uses `skill.md`.
- No `settings.json` in `.claude/` — this repo is a plugin, not a project config.
- Do not add project-level docs (`docs/`, story files, etc.) — this repo ships skills only.
