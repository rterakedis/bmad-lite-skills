---
name: github-tracking
description: Composable GitHub issue/milestone tracking operations. Called by other skills — not invoked directly by the user.
---

# GitHub Tracking (Composable)

Reference library of tracking operations called by other skills at marked points.

Directly invocable: `/github-tracking setup` (one-time) | `/github-tracking backfill` (retroactive) | `/github-tracking sync` (reconcile labels/state).

**Deterministic ops live in `scripts/gh-track.sh`.** The label mechanics — moving an issue between mutually-exclusive status labels (and *stripping the stale one*), closing, and reconciling — are byte-identical every run and must not be reconstructed in-model (that's where label drift like "two status labels on one issue" comes from). This skill is the **policy layer**: it decides *when* to transition and owns the interactive SETUP/BACKFILL/SYNC confirmation gates; the *how* is one shell call. If `scripts/gh-track.sh` is absent (older project not yet upgraded), fall back to the raw `gh` commands documented per-op below.

## Prerequisites Check

Before composable ops (ENSURE-MILESTONE, CREATE-ISSUE, TRANSITION, CLOSE-ISSUE):
- `gh auth status` + `gh repo view --json nameWithOwner`
- Both pass: proceed. Either fails in composable call: skip silently, warn once.
- Either fails in SETUP/BACKFILL: handle auth inline.

---

## SETUP (one-time)

1. Check auth: `gh auth status`. If no, run `gh auth login` (interactive).
2. Verify auth succeeded: `gh auth status`.
3. Verify repo: `gh repo view --json nameWithOwner`. Fail if no remote.
4. Create labels: `backlog`, `ready-for-dev`, `in-progress`, `review`, `done` with `--force`.
5. Report: auth user, repo. Next: `/backfill` or `/create-story`.

---

## BACKFILL (retroactive)

Retroactively create milestones/issues for stories with `github_issue: 0` (safe to re-run).

1. Verify auth (gh auth status, repo view).
2. Scan docs/epics/: `grep -rl "^github_issue: 0"` + files with no `github_issue:` line.
3. Preview list + confirm: "Create milestones + issues + write numbers back?"
4. For each story: read epic/story/title/Status, add `github_issue: 0` if missing, ENSURE-MILESTONE, label by Status, CREATE-ISSUE, CLOSE-ISSUE if done. Report per-story.
5. Summary: {N} issues, {M} milestones, {K} closed. Next: `/status`.

---

## SYNC (reconcile labels and state)

Reconcile GitHub issue labels and open/closed state against the current `status:` field in every story file. Safe to re-run — idempotent.

### Status → label/state mapping

| story `status:` | Expected GitHub label | Issue state |
|---|---|---|
| `draft` / `ready` | `ready-for-dev` | open |
| `in-progress` | `in-progress` | open |
| `review` | `review` | open |
| `done` / `complete` | `done` | closed |
| `cancelled` | `done` | closed |

### Steps

**Preferred (script does the deterministic reconcile):**
1. Verify auth (`gh auth status`, `gh repo view --json nameWithOwner`). Fail fast if either fails.
2. Dry-run diff: `bash scripts/gh-track.sh sync "<story-glob>"` (e.g. `docs/epics/*.md` or `docs/stories/*.md` — whichever holds the story files, with non-zero `github_issue:` frontmatter). It prints a table (story | issue# | current | expected | action) and a summary; it changes nothing.
3. If the summary shows `0 to-change`: report "All issues in sync." and stop.
4. Confirm with the user: "Apply {N} changes?"
5. On yes: `bash scripts/gh-track.sh sync "<story-glob>" --apply`. The script transitions labels (stripping stale ones), closes done stories, and reopens any wrongly-closed ones. Relay its summary.

**Fallback (no script):** do it inline — for each story file with a non-zero `github_issue:`, read `status:`, map via the table above, `gh issue view {N} --json state,labels`, and if it differs apply TRANSITION (+ close/reopen). Confirm before applying.

> **Legacy format tolerance:** `gh-track.sh sync` reads `status:`/`github_issue:` from YAML frontmatter but falls back to the older `**Status:**` / `**GitHub Issue:** #N` body lines (normalizing `✅ Done` → `done`), so it reconciles pre-frontmatter story files too. New stories use the pinned frontmatter format (`create-story/template.md`); the fallback exists only to recover historical files.

---

## ENSURE-MILESTONE

Find or create milestone for epic. Input: `epic_num`, `epic_title` from epics.md.
- Check if exists: `gh api repos/.../milestones --jq "... select(.title | startswith(...))"`
- If exists: use title. If not: create with POST, description from epics.md.
- Return: milestone_title.

---

## CREATE-ISSUE

Create issue for story. Input: `story_file` (optional), `epic_num`, `story_num`, `story_title`, `milestone_title`, AC summary, `initial_label` (default: `ready-for-dev`).
- Build body from story (statement + ACs + repo path).
- `gh issue create --title "Story {epic}.{num}: {title}" --body --milestone --label "{initial_label}"`
- Extract issue number from URL.
- If `story_file` provided: write back `sed -i '' "s/^github_issue: 0$/github_issue: {N}/" "{file}"`
- If no `story_file` (source is epics.md): skip write-back; issue number will be recorded when `/create-story` generates the story file.
- Report: "Created #N → URL"

---

## TRANSITION

Move issue to a new status label. Input: `story_file` (or issue number directly), `new_label` (ready-for-dev/in-progress/review/done).
- Resolve issue number: from the caller, or `grep "^github_issue:" {story_file} | awk`. If `0`/empty: skip + warn.
- **Preferred:** `bash scripts/gh-track.sh transition {N} {new_label}` — adds the label, **strips every other status label**, and verifies. Prints `#N → {new_label} ✓`.
- **Fallback (no script):** `gh issue edit {N} --add-label {new} --remove-label {each other status label currently present}`. Always remove *all* stale status labels (`backlog ready-for-dev in-progress review done` minus the target), not just one — that prevents the double-label drift.

## CLOSE-ISSUE

Close issue when done. Input: `story_file` (or issue number).
- Resolve issue number (same as TRANSITION).
- **Preferred:** `bash scripts/gh-track.sh close {N} "Story complete..."` — applies `done` then closes (idempotent; closed issues retain `done` for milestone progress).
- **Fallback (no script):** run TRANSITION to `done`, then `gh issue close {N} --comment "Story complete..."`.

---

## Status Label Flow

```
epics.md written → [backlog]
                      ↓ create-story completes
                   [ready-for-dev]
                      ↓ dev-story starts
                   [in-progress]
                      ↓ dev-story DoD passes
                   [review]
                      ↓ code-review passes
                   [done] + issue closed
```

Milestone progress in GitHub UI automatically shows X/Y closed issues.
