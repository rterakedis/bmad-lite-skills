---
name: harvest-findings
description: Composable operation that harvests inline manual-test findings from an epic's test plan, captures them to the backlog, and turns in-scope ones into a remediation story. Called at the manual-test-pass boundary by /epic-flywheel and /retrospective. Directly invocable as /harvest-findings {N}.
---

# Harvest Findings (Composable)

After an epic's **manual test pass**, the tester records findings as inline bulleted sub-lists directly under the scenarios/steps in `docs/epics/epic-{N}-test-plan.md` (the file `/epic-flywheel` generates at its boundary). This skill harvests those inline findings, captures them durably in the backlog, triages them, and turns the **in-scope** ones into a new remediation story that flows through the normal flywheel.

**Iron rule — done stories are immutable.** This skill **never** reopens or edits a `status: done` story. Fixes for findings always land in a *new* story `{N}.{last+1}`, never by amending completed work.

**Token posture (Pro plan):** Steps 1–2 and 4 are cheap file parses/edits. The only model work is triage (Step 2) and the create-story call (Step 3, delegated). Idempotent — re-running for the same test-pass date does not duplicate anything.

Directly invocable: `/harvest-findings {N}` — harvest Epic {N}'s test plan. Also called as a composable step by `/epic-flywheel` (after the manual test pass) and `/retrospective` (before the deferred sweep).

**Compose, don't reimplement.** This skill orchestrates existing skills:
- `skills/deferred/SKILL.md` — **LOG-AND-SCHEDULE** for `[defer]` findings.
- `skills/create-story/SKILL.md` — authoring the remediation story file.
- `skills/github-tracking/SKILL.md` — opening the tracking issue.
- `skills/docs-sync/SKILL.md` — **OPERATIONAL** reconcile of the human guides (referenced in the story's DoD).

---

## Activation

1. **Resolve the epic.** Accept `/harvest-findings {N}`; if no number given, infer the most recently completed epic (latest `docs/epics/epic-{N}-test-plan.md`). Require the test plan to exist:
   ```bash
   test -f docs/epics/epic-{N}-test-plan.md || echo "no test plan"
   ```
   If absent, report "No test plan for Epic {N} — run /epic-flywheel to its boundary first" and stop.

---

## Step 1 — Harvest (read-only parse)

Parse `docs/epics/epic-{N}-test-plan.md`. It has scenario/flow headings (`### Flow: {name}`, `### Edge cases`) with `- [ ] {step} → {expected}` checklist lines. A **finding** is any bullet the tester *added underneath* a scenario/step — an indented sub-list bullet that is not one of the generated step/expected lines (typically prose like "❌ crashes when…", "shows wrong total", "button misaligned", or a nested `- ` under a step).

Collect a normalized list, one entry per finding:
`{scenario-id, scenario-title, finding-text}` — where `scenario-id` is a stable anchor (the flow/edge-case heading text, plus the step it hangs under if any).

**If no inline findings are found:** report `No test findings to harvest.` and stop. (This is the common, green case — zero cost beyond the parse.)

---

## Step 2 — Capture to `docs/epics.md` FIRST (durability before authoring)

**Before creating any story**, write the harvested + triaged findings back into `docs/epics.md` under the Epic {N} section. This guarantees the findings are captured in the canonical backlog even if a later step fails, and gives Step 3 a stable source to read from.

**Triage each finding** as one of:
- **[in-scope]** — a defect or gap in *this epic's* delivered behavior; fix now, this epic.
- **[defer]** — genuinely future work, out of this epic's scope, or lower-priority polish.

Append a checklisted block under the Epic {N} heading (idempotent — see below):

```markdown
### Epic {N} — Post-Test Findings (harvested {date})
- [ ] {finding-text} — _{scenario-title}_ — **[in-scope]**
- [ ] {finding-text} — _{scenario-title}_ — **[defer]**
```

**Idempotency (required):** the block is keyed by `(Epic {N}, harvested {date})`. Before writing, check for an existing `### Epic {N} — Post-Test Findings (harvested {date})` heading for the same date. If present, **merge** — add only findings whose `finding-text` is not already listed; never duplicate the block or an existing row. A re-run on a different date creates a new dated block (a distinct test pass).

Use today's date (`date +%Y-%m-%d`).

---

## Step 3 — Author the remediation story (in-scope findings only)

Only if Step 2 produced **≥1 [in-scope]** finding.

1. **Determine the story number.** Read `docs/epics.md`; the new story is `{N}.{last+1}` where `last` is the highest existing story number in Epic {N}.

2. **Author the story via create-story.** Invoke `skills/create-story/SKILL.md` to generate `docs/epics/{N}-{last+1}-post-test-findings.md` with:
   - **One acceptance criterion per in-scope finding**, each in Given/When/Then form and each **linking back to its source scenario** (e.g. "_(source: Flow: Checkout → step 'apply promo code')_").
   - Title: `Post-test findings — Epic {N}`; marked `*(remediation)*`.
   - Dev Notes citing the test plan and the harvested block in `docs/epics.md` as the origin.
   - `status: ready-for-dev` in frontmatter (the pinned format).
   - **Definition of Done additions** (write these explicitly into the story so dev-story enforces them at close):
     - [ ] Reconcile `docs/architecture.md`, `docs/prd.md`, and `docs/ux/*` for any behavior these fixes change (via `docs-sync` **OPERATIONAL** — spawn `bmad-docs-sync`, Haiku).
     - [ ] Reset `docs/epics/epic-{N}-test-plan.md` (Step 4 below) so the plan is clean for the next re-test pass.

3. **Append the story row to `docs/epics.md`** under Epic {N}'s stories and bump the epic's story count (mirror the `deferred` skill's new-story entry format). Do this **before** the tracking step so the milestone/issue is filed against a story that exists in the backlog.

4. **Create the tracking issue under Epic {N}'s milestone — required.** The new story must be tracked exactly like any other, so its status flows `ready-for-dev → in-progress → review → done` as `/dev-story` and `/code-review` run on it. `create-story`'s own **GitHub Tracking** step (invoked in item 2) already does this: **ENSURE-MILESTONE** for Epic {N}, then — finding no pre-existing issue for `{N}.{last+1}` (this story was never written to `backlog` by `/epics`) — **CREATE-ISSUE** at `ready-for-dev` and write `github_issue: {number}` back to the frontmatter.

   **Do not double-create.** If create-story's tracking already ran, only **verify** here: `grep '^github_issue:' docs/epics/{N}-{last+1}-post-test-findings.md` is non-zero and the issue sits under Epic {N}'s milestone (`gh issue view {N} --json milestone,labels`). If create-story skipped tracking (older variant) or `github_issue:` is still `0`, run **ENSURE-MILESTONE** + **CREATE-ISSUE** from `skills/github-tracking/SKILL.md` yourself now. Skip silently only if `gh` is unavailable — then note it in the output so the user knows the story is untracked until `/github-tracking backfill`.

   > A remediation story goes straight to `ready-for-dev` (not `backlog`) because it's actionable immediately — consistent with how the `deferred` skill schedules new remediation stories. From here the status lifecycle is automatic: the flywheels and `gh-track.sh` drive the label transitions off the `github_issue:` frontmatter, and `/code-review` closes the issue when the story is done.

5. **Route every [defer] finding through `deferred`.** For each `[defer]` row, call **LOG-AND-SCHEDULE** from `skills/deferred/SKILL.md` (`title` = short finding summary, `detail` = finding-text, `source` = `docs/epics/epic-{N}-test-plan.md` scenario). This slots it into an existing backlog story or a remediation story — **never** into the new post-test-findings story (that one holds in-scope items only). Capture the returned D-IDs.

If there are **zero [in-scope]** findings, skip story authoring entirely — every finding was a `[defer]`, already routed in this step's item 5 — and still proceed to Step 4.

---

## Step 4 — Reset the test plan

Remove the **harvested inline finding bullets** from `docs/epics/epic-{N}-test-plan.md` — leave the scenarios, flows, steps, and the physical-device section fully intact. The plan must be clean and re-runnable for the next test pass; only the tester's finding annotations come out (they now live durably in `docs/epics.md` and the story).

This reset is also listed in the remediation story's DoD (Step 3.2) — running it here keeps the plan clean immediately; the DoD item is the backstop ensuring it happened before the story closes.

---

## Output

Report one line:

```
Epic {N}: {n} findings harvested → {x} in-scope → story {N}.{last+1} (issue #{issue}), {y} deferred ({D-ids}). Test plan reset.
```

Adjust for the zero cases: `No test findings to harvest.` (Step 1 empty), or `{n} findings harvested → 0 in-scope, {y} deferred ({D-ids}). No remediation story needed. Test plan reset.` (all deferred).

---

## Callers

- **`/epic-flywheel`** — at the manual-test-pass boundary: after the user works through `docs/epics/epic-{N}-test-plan.md`, run `/harvest-findings {N}` to capture what they found before the retrospective / next epic.
- **`/retrospective`** — before the deferred sweep, offer `/harvest-findings {N}` so test findings are captured and scheduled alongside the retro's other bookkeeping.
