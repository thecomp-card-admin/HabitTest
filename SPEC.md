# Habit & Task Tracker — Spec

Status: **r3 — v1.1 skills** (2026-10-01). Release `1.1.0`, data schema v2. r2 (v1.0) was signed off 2026-09-23; r3 adds skills per the 2026-10-01 review (§9).

## 1. Scope

A Reminders-style PWA for one user on an iPhone Home Screen. Core loop: add items, check them off, earn XP, level up **skills** and an overall level, unlock achievements. **Lists** organize items (by day, place, task type); **skills** (Strength, Clarity, Learning…) are what items boost, and every item gets one picked automatically. AI capture (in-app photo, or JSON pasted from any Claude chat) is a secondary input path. No accounts, no server, no sync; all data on-device with JSON backup.

Out of scope: notifications, due times, drag-to-reorder, multi-device data sync, Claude pushing items in without a paste step, Siri/Shortcuts, widgets, bonus XP from achievements, items boosting more than one skill.

## 2. Files and architecture

| File | Role |
|---|---|
| `index.html` | All UI, storage, and AI code inline (CSS + JS). Single-page, hash-routed. |
| `game.js` | Pure rules: XP, levels, day keys, streaks, completion planning, skill inference, achievements, quick entry, AI request/response contracts, migration, backup validation. No DOM, no storage, no `Date.now()`. Shared by `index.html`, `tests.html`, and the future native port. |
| `sw.js` | Service worker: precache, network-first for `index.html`/`game.js`, cache-first for the rest, `CACHE_VERSION` bumped per release. |
| `manifest.webmanifest`, `icons/` | Home Screen install. |
| `tests.html` | Unit tests for `game.js` (86). |
| `README.md` | Deploy, install, API key + spend limit, backup/restore, release checklist. |

Layers in `index.html`: `DB` (IndexedDB) → actions (create/complete/reassign, call `Game.*`, persist) → views (render per route) → `Boost` (completion celebration) → `SkillAI` (background Claude skill check) → capture (photo/paste). State is loaded once at launch; every mutation writes through to IndexedDB.

Routes: `#/` Lists tab · `#/skills` Skills tab · `#/skill/<id>` · `#/list/today|scheduled|all|completed` · `#/list/<listId>` · `#/achievements` · `#/settings`. Sheets are overlays; Back closes them.

## 3. Data model (schema v2)

```ts
List   { id, name, color, icon, sortOrder, createdAt }          // IDB store + JSON key "categories" (v1 name, kept)
Skill  { id, name, icon, color, hint, sortOrder, createdAt }    // hint = "what counts", read by the keyword guess and Claude

Item {
  id, text, notes, type: 'task'|'habit', categoryId /* list */, tier: 1..5, parentId|null,
  skillId: string|null,
  skillSource: 'user'|'ai'|'auto'|null,   // your pick | Claude's pick | keyword guess | none yet
  aiCheckedAt: string|null,               // set once Claude has checked this item's skill
  due, repeat, timesPerDay, sortOrder, createdAt
}

Completion { id, itemId, categoryId, skillId|null, kind: 'task'|'habit'|'subtask', dayKey, at, tier, xp, text }

Achievement { id, seedKey?, name, icon, kind: 'count'|'streak'|'level'|'manual', metric?,
  scope: {type:'all'} | {type:'skill',id} | {type:'category',id} | {type:'item',id} | {type:'anySkill'} | {type:'everySkill'},
  threshold, autoTier, seriesId, tier, unlockedAt, createdAt }

Settings { id:'settings', dayStartHour: 4, defaultCategoryId, model, aiSkillCheck: true,
           lastBackupAt, firstLaunchAt, onboarded, schemaVersion: 2 }
Secret   { id:'anthropicKey', value }     // separate store; never exported
```

IndexedDB `habit-tracker` v2: stores `categories`, `skills`, `items`, `completions`, `achievements`, `settings`, `secrets`. Derived, never stored: XP totals (overall, per skill), levels, streaks, done-today.

Fresh install seeds one list (**Reminders**) and ten skills with fixed IDs: 💪 Strength · 🤸 Dexterity · 🏃 Endurance · ❤️ Vitality · 🧘 Clarity · 📚 Learning · 💼 Career · 💵 Wealth · 🗣️ Social · 🏠 Upkeep. All editable.

## 4. Game rules (`game.js`)

### 4.1 Config

```js
XP_BY_TIER = {1:10, 2:25, 3:50, 4:100, 5:200}; DEFAULT_TIER = 2
LEVEL_BASE = 100, LEVEL_EXP = 1.3          // xpForLevel(L) = round(100 · L^1.3), no cap
TIER_LADDER = [10, 25, 50, 100, 250, 500, 1000]  // then doubling
DAY_START_HOUR = 4
```

### 4.2 XP and levels

Every completion adds its XP to the **overall** level and to its **skill** (`completion.skillId`). Lists don't level. `xpTotals()` → `{ overall, byCategory, bySkill }`; levels come from totals (`levelFromXp`), so undo can lower a level. Same curve for overall and every skill.

### 4.3 Completing

Unchanged from v1.0 (tasks once; habits up to `timesPerDay` per logical day; last open sub-task completes the parent; completing a parent completes its open sub-tasks; unscheduled-day habit check-ins give XP but don't touch streaks). Additions:

- Each completion is stamped with the item's skill. Sub-tasks carry **their own** skill, so a "Morning routine" habit can feed Clarity, Dexterity and Strength through its sub-tasks.
- If the item has no skill at check-off, `planComplete` decides one (keyword guess, else the parent's skill) and returns it in `inferred[]`; the app saves it on the item. If nothing matches, the completion counts toward overall only until a skill is found (§4.5).

### 4.4 Picking a skill (instant, offline)

`inferSkill(text, skills)` scores each skill against its built-in keyword list (seed skills, by ID, so renames keep them), its name, and its editable hint. Single words weigh 1, multi-word phrases 2, generic verbs/places 0.5 (read, call, text, gym, plan…); the last word of a keyword also matches common inflections (lift → lifted, lifting). Highest score wins; ties go to the match later in the text ("run errands" → Upkeep, "read emails" → Career). No match → `null`.

New items: explicit pick (`+skill` token, review sheet) > keyword guess (`auto`) > parent's skill (`auto`) > none. Editing an item's text re-guesses (unless you or Claude picked it) and queues it for Claude.

### 4.5 Claude skill check (background)

When a key is set and **Claude checks skills** is on: items with `skillSource ≠ 'user'` and no `aiCheckedAt` are sent in batches of ≤40, ~2.5 s after the last change (also on launch, foreground, reconnect). Request: `assign_skills` tool forced; payload = skills `{id,name,hint}` + items `{id,text,type,parent?}` (titles only, never notes). Response pairs are validated (known item IDs, skill by ID or name), then each item's skill is set (`ai`) and `aiCheckedAt` stamped; an item renamed or picked by you mid-request is skipped. Failures are silent and retried later. If Claude's decision lands on an item checked off in the last 10 minutes with no skill, that XP is credited and celebrated then.

**History follows the skill**: changing an item's skill (your pick, Claude, deletion) rewrites the `skillId` on all of that item's completions, so its XP moves with it. Your pick shows a "Moved N XP to …" toast.

Deleting a skill re-guesses its items (then Claude re-checks); completions of deleted items in that skill keep overall XP only. Adding a skill or editing a hint re-queues all non-`user` items so Claude can re-sort into the new set.

### 4.6 Logical days and streaks

Unchanged: `dayKey` subtracts the day-start hour; streaks walk scheduled days, multi-target habits need the full count, today-not-done doesn't break a streak.

### 4.7 Achievements

Kinds unchanged (count, streak, level, manual; count/streak auto-tier on the ladder). Scopes: everything, a skill, a list, an item (count/streak); overall, a skill, **highest skill** (`anySkill`), **every skill** (`everySkill`, 0 with no skills) for level. Seeds: Getting Started, Detail Oriented, Habit Builder, Task Slayer, On a Roll, Level 5/10 overall, **Specialist · a skill at Level 5**, **Well-Rounded · every skill at Level 3**. Never revoked; checked after every completion and reassignment.

### 4.8 Quick-entry tokens

`!1…!5` tier · `#list` · **`+skill`** (name prefix, e.g. `+clar`) · `@today/@tomorrow/@fri/@10-3/@2026-10-03` due · `^daily/^weekly/^mon,wed/^weekdays/^weekends` habit · `x8` times a day. Unrecognized tokens stay in the text. The hint line under New Item shows the skill it will boost as you type.

### 4.9 Migration v1 → v2

On first launch of 1.1 (or when importing a v1 backup): seed the ten skills; give each item a skill by keyword, else its parent's skill, else its old seed list's mapping (Fitness → Strength, Learning → Learning, Career → Career, Finance → Wealth, Home → Upkeep, Social → Social); credit every past completion with its item's skill (deleted items: keyword on the stored text, else old list); retarget list-level achievements to the mapped skill (else overall); add Specialist and Well-Rounded once; keep all lists, settings and the default list. Items are left `auto` so Claude re-checks them. Overall XP is unchanged. One toast announces the Skills tab.

## 5. Screens

1. **Install gate / onboarding** — as v1.0; onboarding now explains skills and the Skills tab.
2. **Tab bar** on the two root screens: **Lists** | **Skills**.
3. **Lists tab** — Reminders-style: camera + settings buttons, smart tiles (Today, Scheduled, All, Completed), My Lists (icon, name, count), Add List. No levels here.
4. **List view** — rows: round checkbox, text, meta (due, 🔥 streak, `3/8` today, chip `🧘 25 XP` = skill icon + XP, list dot on smart lists). Sub-tasks indented. Swipe-delete with undo. New Item row with tokens, live hint (skill + parsed tokens), camera.
5. **Skills tab** — Overall card (level, bar, XP to next, total XP), Achievements row (unlocked count), 2-column grid of skill tiles (icon, `Lv N`, name, bar, `into / needed XP`), "+" to add a skill, a note when items aren't linked yet.
6. **Skill view** — big icon + name, level and total XP, progress bar, "Counts: …" hint, **Boosted by**: every active item with this skill across lists (flat, list name and parent shown), checkable in place. "…" opens the skill editor.
7. **Skill editor** — name, what-counts hint, color, emoji, delete (items re-sorted).
8. **Item detail** — as v1.0 plus **Boosts** (skill picker; source shown: Your pick / Picked by Claude / Auto-picked / Not linked yet).
9. **Completion celebration** — checkbox pop with a ring, "+25 XP" floating up from the checkbox, and a bottom card: praise word, total XP counting up, one row per skill boosted (icon, name, `Lv N`, bar filling from old to new XP, `+25`). A skill level-up fills the bar, glows, shows "Level N!", and refills; the overall level-up and achievement unlocks still use the top banner. Quick successive check-offs coalesce into the same card (max 3 skill rows). Reduced Motion shows the end state without animation.
10. **Achievements** — as v1.0; editor scope pickers include skills, highest skill, every skill.
11. **Settings** — General, Backup, **AI** (API key, model, **Claude checks skills** toggle, **Re-check all items** with status, Copy Claude prompt), About, Erase.
12. **Capture review** — per row: checkbox, text, Task/Habit, **skill**, list, tier, times/day; batch due chips; "Add N items".

## 6. Storage, backup, offline, install

As v1.0, with: IDB v2 (adds `skills`; `onversionchange` closes the connection; a blocked upgrade shows a message). Backups are schema 2 and include `skills`; importing schema 1 runs the §4.9 migration. Release `1.1.0` / `habits-v1.1.0`.

## 7. AI contracts

All calls: `POST https://api.anthropic.com/v1/messages`, headers `x-api-key`, `anthropic-version: 2023-06-01`, `anthropic-dangerous-direct-browser-access: true`; model from Settings (`claude-sonnet-5` default, `claude-haiku-4-5-20251001`, or custom).

### 7.1 Photo capture — `record_items`

Context text block: `{ lists:[{id,name}], skills:[{id,name,hint}], active_items:[{id,text,list_id,type}] (≤300), tier_rubric, opened_from_list_id }`. Tool schema items: `text, type, list_id, skill_id, tier, times_per_day?, duplicate_of_id` (required: all but times_per_day), plus `unreadable[{best_guess}]`. The normalizer also accepts v1.0 keys (`category_id`, `category`). Unknown skill → keyword guess (`auto`) → none.

### 7.2 Paste from Claude

"Copy Claude prompt" lists your lists and skills (with hints) and asks for one JSON block: `{"items":[{"text","type","list","skill","tier","times_per_day"}],"unreadable":[{"best_guess"}]}` — names, not IDs. Same review sheet as the photo path.

### 7.3 Skill check — `assign_skills`

```json
{ "model": "…", "max_tokens": 2048, "system": "<sort each to-do into the one skill it trains most …>",
  "tools": [{ "name": "assign_skills", "input_schema": { "type": "object", "required": ["assignments"],
     "properties": { "assignments": { "type": "array", "items": { "type": "object", "required": ["id", "skill_id"],
       "properties": { "id": {"type": "string"}, "skill_id": {"type": "string"} } } } } } }],
  "tool_choice": { "type": "tool", "name": "assign_skills" },
  "messages": [{ "role": "user", "content": "{\"skills\":[{\"id\",\"name\",\"hint\"}],\"items\":[{\"id\",\"text\",\"type\",\"parent?\"}]}" }] }
```

## 8. Assumptions (r3 additions; v1.0's still hold unless replaced)

1. One skill per item (no splits). Sub-tasks have their own skill; the parent's skill is the fallback.
2. Tasks and habits both boost skills.
3. An item's XP history always follows its current skill (reassignment rewrites completions). Deleted items keep their last skill.
4. Lists are organization only: no list levels; list-scoped count/streak achievements still work.
5. The keyword guess runs offline on every new or renamed item; Claude refines it when a key is set and the toggle is on. Your own picks (including "No skill") are never overwritten.
6. Claude receives item titles, item type, parent title for sub-tasks, and the skill list — never notes, dates or history.
7. Fresh installs start with one "Reminders" list; existing installs keep their lists.
8. Skill level-ups celebrate inside the completion card; only the overall level-up and achievements use the top banner.
9. Reassignments (your pick, Claude, deletion) move XP silently except for a toast on your own pick; they can unlock achievements but never trigger level-up banners.
10. One toast at a time; the completion card replaces a non-action toast.

## 9. Decisions from review

v1.0 (2026-09-23): Completed tile = total completions; multi-check habits via `timesPerDay`; quick-entry tokens; captured tasks due today with a batch date control; multi-device capture via Paste from Claude.

v1.1 (2026-10-01): skills are standalone categories boosted by completing items, separate from lists; levels live on their own tab; the skill is decided automatically (instant keyword guess, then Claude double-checks in the background); checking an item off shows which skill it boosted with an animation of the XP going up; tasks and habits both count; a fresh install starts with one Reminders list plus the prebuilt skills.

## 10. Verification

`tests.html`: 86 unit tests green. Headless iPhone-viewport walkthroughs (light + dark): lists/items/sub-tasks/backup 40/40, habits/streaks/achievements 27/27, offline + update 11/11, capture 26/26, skills 52/52, v1.0 → v1.1 migration on real v1 data 20/20.
