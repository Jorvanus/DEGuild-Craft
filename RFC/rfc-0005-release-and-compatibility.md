# GuildCrafts Protocol — RFC 0005

**Status:** Informational  
**Applies to:** GuildCrafts v1.0.0+

---

## Table of Contents

1. [Version Fields](#1-version-fields)
2. [Version Bump Checklist](#2-version-bump-checklist)
3. [Interface / TOC Versioning](#3-interface--toc-versioning)
4. [Compatibility Layers](#4-compatibility-layers)
   - 4.1 [Tooltip Pipeline (2.5.6 / Classic Era 1.15.9)](#41-tooltip-pipeline-256--classic-era-1159)
   - 4.2 [TradeSkill API (defensive compatibility)](#42-tradeskill-api-defensive-compatibility)
5. [Git Workflow](#5-git-workflow)
6. [Branch Strategy](#6-branch-strategy)
7. [Pull Request & Merge Process](#7-pull-request--merge-process)
8. [Release Process](#8-release-process)
9. [CHANGELOG Conventions](#9-changelog-conventions)
10. [CurseForge Distribution](#10-curseforge-distribution)

---

## 1. Version Fields

GuildCrafts uses four distinct version identifiers:

| Field | Location | Type | Purpose |
|-------|----------|------|---------|
| `DISPLAY_VERSION` | `Core.lua` | string `"X.Y.Z"` | Human-readable version shown in tooltips, `/gc version`, and CurseForge. |
| `## Version` | `GuildCrafts_Forever.toc` | string `"X.Y.Z"` | Loaded by WoW and reported in the addon list. Must match `DISPLAY_VERSION`. |
| `GuildCrafts.VERSION` | `Core.lua` | integer | Wire protocol version. Carried in every sync envelope. Bump only on backward-incompatible wire format changes. Currently `2`. |
| `GuildCrafts.DATA_FORMAT_VERSION` | `Core.lua` | integer | Member entry schema version. Bump when the sync payload structure changes. See RFC-0003 §11. Currently `2`. |

`VERSION` and `DATA_FORMAT_VERSION` are independent. A display-version bump
(e.g. 1.10.0 → 1.11.0) typically leaves both integers unchanged.

---

## 2. Version Bump Checklist

Before committing a display-version bump, update **all three** of these
locations. They must always match:

| File | Field |
|------|-------|
| `GuildCrafts/Core.lua` | `GuildCrafts.DISPLAY_VERSION = "X.Y.Z"` |
| `GuildCrafts/GuildCrafts_Forever.toc` | `## Version: X.Y.Z` |
| `CHANGELOG.md` | `## X.Y.Z — YYYY-MM-DD` (new entry at the top) |

Wire protocol integers (`VERSION`, `DATA_FORMAT_VERSION`) are bumped
separately and only when the corresponding protocol layer changes. They do
not need to match the display version.

---

## 3. Interface / TOC Versioning

The `## Interface` field in `GuildCrafts_Forever.toc` must match the client
build number of WoW Classic Forever:

| Game version | Interface number |
|---|---|
| WoW Classic Forever 1.15.x | `11507` |

The interface number is bumped when Blizzard releases a new client patch that
changes the number. It is **not** bumped for every display-version release.

Blizzard performs a tolerated-mismatch window for minor patch differences, so
an addon with a slightly older `## Interface` value will still load. However,
setting the exact current value suppresses the "outdated addon" warning in the
UI.

---

## 4. Compatibility Layers

### 4.1 Tooltip Pipeline (2.5.6 / Classic Era 1.15.9)

TBC 2.5.6 (released July 21, 2026) aligned with Classic Era 1.15.9 and
introduced the modern `TooltipDataProcessor` API, replacing the
`OnTooltipSetItem` script event.

GuildCrafts detects the API at runtime:

```lua
-- Tooltip.lua, OnEnable()
if TooltipDataProcessor then
    -- 2.5.6 / modern pipeline
    TooltipDataProcessor.AddTooltipPostCall(Enum.TooltipDataType.Item, handler)
else
    -- 2.5.5 / legacy
    self:SecureHookScript(GameTooltip,    "OnTooltipSetItem", handler)
    self:SecureHookScript(ItemRefTooltip, "OnTooltipSetItem", handler)
end
```

This guard must be preserved whenever Tooltip.lua is modified. The handler
body (`tooltip:GetItem()` and crafter lookup) is identical for both paths.

### 4.2 TradeSkill API (defensive compatibility)

GuildCrafts targets WoW Classic Forever only, but the scanner still guards
against API surface differences between client builds. Compatibility wrappers
live in `Modules/Data.lua` on `main`:

| Classic API | Modern replacement | Wrapper |
|---|---|---|
| `GetSpellInfo(id)` → multiple returns | `C_Spell.GetSpellInfo(id).name` | `GetSpellName(spellID)` local helper |
| `GetNumSkillLines()` + `GetSkillLineInfo(i)` loop | `C_SkillLine.GetSkillLines()` table | `IterSkillLines()` local iterator |
| `GetNumTradeSkills()` (may be absent) | — | `if not GetNumTradeSkills then return end` nil guard |
| `GetTradeSkillNumReagents()` (may be absent) | — | `if not GetTradeSkillNumReagents then return nil end` nil guard |

Each wrapper prefers the modern API when the specific function exists and
falls back to the classic API, so the scanner keeps working if a future
Classic Forever client build introduces the modern namespace.

---

## 5. Git Workflow

- **Branch naming:** `feature/patch-N-description`
- **Merges:** squash-merge into `main`, branch deleted after merge.
- **Feature branch force-push:** allowed (branches are never shared before PR).
- **After a rebase:** `git push --force-with-lease` (preferred over `--force`).

**Never** commit, push, create a PR, or merge without explicit instruction
from the maintainer.

---

## 6. Branch Strategy

| Branch | Purpose |
|--------|---------|
| `main` | Production-ready WoW Classic Forever release. |
| `feature/patch-N-*` | Short-lived feature/fix branches, squash-merged into `main`. |

`wotlk-migration` and `feature/multi-expansion-support` were intermediate
development branches and have both been retired; multi-expansion support was
later removed entirely in favor of a single WoW Classic Forever edition (see
CHANGELOG 3.0.0).

---

## 7. Pull Request & Merge Process

```bash
# Push the feature branch
git push -u origin feature/patch-N-description

# Create the PR
gh pr create --title "feat: ..." --base main

# Squash-merge and delete branch
gh pr merge <num> --squash --delete-branch --subject "feat: ..."
```

PR titles follow conventional commits where practical:
- `feat:` — new user-visible feature
- `fix:` — bug fix
- `chore:` — internal/tooling change with no user impact
- `docs:` — documentation only

---

## 8. Release Process

After merging a PR that bumps the display version:

1. Confirm all three version fields are consistent (§2).
2. Confirm `## Interface` matches the current live client build (§3).
3. Zip the release, **excluding `.DS_Store`** (macOS Finder creates them
   automatically and they must not be distributed):

```bash
zip -r GuildCrafts-X.Y.Z.zip GuildCrafts/ -x "*.DS_Store"
```

4. Upload the zip to CurseForge. Update the description if required.

---

## 9. CHANGELOG Conventions

File: `CHANGELOG.md` at the repository root.

**Entry format:**

```markdown
## X.Y.Z — YYYY-MM-DD

### New features
- ...

### Improvements
- ...

### Fixes
- ...
```

Rules:
- Date format: `YYYY-MM-DD` (ISO 8601).
- New entries are inserted **at the top**, below the document header.
- Sections (`### New features`, `### Improvements`, `### Fixes`) are only
  included when there is at least one item for that section.
- Credit contributors by name in the relevant bullet when applicable.
- Release entries are maintained in the root `CHANGELOG.md` on `main`.

---

## 10. CurseForge Distribution

- Project page: GuildCrafts on CurseForge (WoW Classic category).
- Supported client: WoW Classic Forever.
- `CURSEFORGE_DESCRIPTION.md` contains the rendered project description
  (Markdown is rendered by CurseForge).
- Do not include development files (`spec/`, `tools/`, `.DS_Store`,
  `CLAUDE.md`, `CONTRIBUTING.md`) in the release zip — the zip command targets
  the `GuildCrafts/` subdirectory only.
