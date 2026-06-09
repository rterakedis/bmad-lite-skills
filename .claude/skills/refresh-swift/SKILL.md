---
name: refresh-swift
description: Research current Swift/SwiftUI/platform best practices from gold-standard sources and update the docs/setup/swift/ reference files and the modern-swiftui.md guardrails stub. Use when the user says "refresh swift", "update swift guidance", "refresh swift best practices", or "refresh swiftui".
---

# Refresh Swift Best Practices Skill

**Goal:** Research the current state of Swift language, SwiftUI, concurrency, testing, and platform-specific patterns from primary sources and update the sectioned reference docs in `docs/setup/swift/` plus the `modern-swiftui.md` guardrails stub in the skills repo. These files are baked into every new Apple platform project — keeping them current is critical for guiding correct architectural choices.

**Scope:** iOS 18 through the current stable release only. **Hard exclude** any pre-release, beta, or unannounced OS API. When in doubt, omit.

---

## Step 1 — Locate Files

Find the files to update:

```bash
# Project's working copies (these are what get updated in use)
ls docs/setup/swift/

# Skills repo stub originals (also update these so new projects get current content)
find . -path "*/setup/stubs/swift/*.md" | sort
find . -path "*/setup/stubs/modern-swiftui.md"
```

Read every file completely before proceeding — treat the current content as the baseline.

If `docs/setup/swift/` does not exist, ask whether the user wants to run `/setup` first or update just the stubs in the skills repo.

---

## Step 2 — Research Current Best Practices

Use WebSearch and WebFetch to pull current content from the gold-standard sources below. For each finding, note the iOS/Swift version where the pattern was introduced or changed.

### Gold-Standard Sources

- **Hacking with Swift** — hackingwithswift.com (Paul Hudson)
- **Swift with Majid** — swiftwithmajid.com (Majid Jabrayilov)
- **SwiftLee** — swiftlee.com (Antoine van der Lee)
- **Apple Developer Documentation** — developer.apple.com/documentation
- **Apple WWDC sample apps** — developer.apple.com/documentation/SampleCode
- **Apple Swift updates** — developer.apple.com/documentation/updates/swift
- **Point-Free** — pointfree.co (modern state management)

### Research by Target File

Research each section in turn. For each: compare findings against the existing file content, note what is still accurate, what is outdated, and what is missing.

**`state-management.md`**
- `@Observable` macro — any changes in iOS 18+?
- `@State`, `@Binding`, `@Bindable`, `@Environment` — any new wrappers or deprecations?
- Unidirectional data flow patterns

**`concurrency.md`**
- Swift 6 strict concurrency — what changed for SwiftUI apps?
- `.task` / `.task(id:)` — any new overloads or behaviors?
- `@MainActor`, `actor` — any new guidance?
- `TaskGroup` patterns

**`architecture.md`**
- Feature-based project structure — any WWDC sample app changes?
- Service injection via `@Environment` — any new patterns?
- `NavigationStack` — any new value-based navigation APIs?

**`ui-composition.md`**
- New SwiftUI layout containers introduced in iOS 18+?
- `List` vs `ScrollView` — any new lazy layout options?
- View modifier changes

**`testing.md`**
- Swift Testing framework updates (new annotations, `@Suite` changes)?
- Any new Xcode test runner integration?
- Core Data test patterns

**`anti-patterns.md`**
- New patterns AI tools commonly generate that should be added to the rejection list?
- Any anti-patterns that are now acceptable (rare — document the reason)?

**`ipados-specific.md`** (if present in `docs/setup/swift/` or stubs)
- `NavigationSplitView` — any new column options or behaviors in iPadOS 18+?
- Multi-window and scene APIs — any new `openWindow` or `WindowGroup` options?
- Drag-and-drop — any new `Transferable` representations or drop modifiers?
- Pointer/hover — any new `hoverEffect` styles?
- Stage Manager implications — any new multi-window guidance from WWDC?

**`macos-specific.md`** (if present in `docs/setup/swift/` or stubs)
- `MenuBarExtra` — any new style or content options?
- `Table` — any new column types, row actions, or sorting APIs?
- `Settings` scene — any new navigation or tab patterns?
- Window management — any new `openWindow`, `WindowGroup`, or `defaultSize` options?
- File operations — any changes to `fileImporter`/`fileExporter` modifiers?
- macOS 15+ (Sequoia) — any new scene types or system integration APIs?

---

## Step 3 — Write Updated Files

For each file with changes:

1. Update `docs/setup/swift/{file}.md` with current content.
2. Update the corresponding stub at `{skills_path}/.claude/skills/setup/stubs/swift/{file}.md` so new projects get the same content.
3. Add or update a `> **Updated:** {today's date} — iOS/iPadOS/macOS {N}+` line at the top of each changed file, using the correct platform label for that file.

### Formatting Rules (preserve these exactly)

- `#` title, then `> iOS {N}+ | one-line scope note`
- `##` section headers
- Fenced code blocks with `// ✅` correct and `// ❌` rejected patterns side-by-side
- Tables for comparison content (property wrappers, List vs ScrollView decisions)
- No conversational filler — declarative, RFC-style tone

### Guardrails Block (`modern-swiftui.md`)

After updating the sectioned files, review the guardrails stub (`modern-swiftui.md`). Update it only if:
- A wrapper was added to the "hard rejection" table
- A wrapper was removed from the rejection table (document why)
- The property wrapper quick reference table has a new row
- A checklist item changed

The guardrails file must stay under ~50 lines — it lives in CLAUDE.md and is loaded every turn.

---

## Step 4 — Report to User

Summarize:

1. **What changed:** Bullet list per file — new patterns added, outdated patterns removed, iOS/Swift version bumps.
2. **What stayed the same:** Brief confirmation that still-valid content was preserved.
3. **Sources consulted:** Which gold-standard sources had relevant current content.
4. **Reminder:** Existing projects that already have the sectioned files in `docs/setup/swift/` will have their copies updated by this run. Projects that haven't run `/setup` yet will get the updated stubs when they do.

---

## Step 5 — Offer Audit Handoff

After reporting, ask:

> "Guidance updated. Run `/swift-audit` now to check the current codebase against the new patterns? (y/n)"

If yes: invoke `/swift-audit` immediately. If no: stop.
