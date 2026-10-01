# Habit & Task Tracker (v1.1)

Reminders-style tasks and habits with a light game layer: every item boosts a **skill** (Strength, Clarity, Learning…), skills and your overall level climb with no cap, and achievements unlock along the way. Lists stay purely for organizing. Plain HTML/CSS/JS, no build step, no server: everything lives in IndexedDB on the phone. See `SPEC.md` for the rules and data model.

```
index.html             app (UI, storage, AI) — all inline
game.js                pure rules + AI contracts (shared with tests and the future native app)
sw.js                  service worker: offline cache, network-first index.html/game.js
manifest.webmanifest   Home Screen install metadata
icons/                 180 / 192 / 512 / 512-maskable PNGs
tests.html             unit tests for game.js (open in a browser)
SPEC.md                spec
```

## 1. Deploy to GitHub Pages

1. Create a repo (public is fine — it never contains your key or data) and push these files to `main` at the repo root.
2. Repo → **Settings → Pages** → Source: *Deploy from a branch* → Branch: `main`, folder `/ (root)` → Save.
3. After a minute, open `https://<you>.github.io/<repo>/` in Safari on the iPhone. All paths are relative, so a sub-path works.

Local preview: `python3 -m http.server 8000` in the folder, then `http://localhost:8000/`. Service workers need HTTPS or localhost.

## 2. Install on the iPhone

1. Open the Pages URL in **Safari**.
2. **Share** → **Add to Home Screen** → **Add**. (The app shows these steps itself while it runs in a tab.)
3. Open **Habits** from the Home Screen → **Get started**.

The Home Screen app has its own storage, separate from the Safari tab, so use only the installed app. Removing the icon deletes that storage — export a backup first.

**Updating from 1.0:** export a backup, deploy, then open the app once with a connection. Your data upgrades automatically: lists stay as they are, ten skills are added, and your past check-offs are credited to skills (a toast confirms it).

## 3. How skills work

- The **Lists** tab is for organizing (by day, place, type — start with "Reminders" and add your own). The **Skills** tab holds your overall level, each skill's level, and achievements.
- Every item boosts one skill, picked for you: a keyword guess the moment you add it (journaling → Clarity, lifting → Strength, stretching → Dexterity, reading → Learning), then Claude double-checks in the background if your API key is set. The chip on each row (`🧘 25 XP`) shows the pick.
- Disagree? Open the item → **Boosts** → pick another skill. Its past XP moves with it. Your picks are never overwritten.
- Checking something off shows which skill it boosted, with its bar filling up; level-ups glow in that card.
- Skills are editable (Skills tab → tap a skill → "…", or "+" for a new one). The **What counts** text steers both the keyword guess and Claude — e.g. a "Creativity" skill with "guitar, drawing".

## 4. API key and spend limit (photo capture + skill checks)

1. [console.anthropic.com](https://console.anthropic.com) → **API Keys** → create a key for this app only.
2. Console → **Settings → Limits** (or Billing → Limits): set a **monthly spend limit**, e.g. $5. A card photo is a few thousand input tokens; a skill check is a few hundred tokens per batch of new items. Check current pricing in the Console; the limit is the backstop.
3. In the app: **Settings → AI → API key → Save**. Stored on-device only, sent only to `api.anthropic.com`, never exported; **Clear key** removes it.
4. **Model**: `claude-sonnet-5` (default) or `claude-haiku-4-5-20251001` (cheaper, plenty for skill checks). *Custom…* accepts any model ID.
5. **Claude checks skills** (on by default) sends only item titles and your skill list — never notes. Turn it off to keep typed items fully offline; the keyword guess still works. **Re-check all items** re-sorts everything you haven't picked yourself.

Photo capture: camera button (Lists tab or a list's New Item row) → **Take Photo** → review (text, type, skill, list, tier, batch due date; likely duplicates start unchecked) → **Add N items**.

### Capture from another device or account (Paste from Claude)

Settings → AI → **Copy Claude prompt** (it includes your lists and skills), paste it into a Claude Project's instructions or a chat with the card photo, copy the JSON block Claude returns, then in the app tap camera → **Paste from Claude**. Universal Clipboard carries it from a Mac/iPad. Re-copy the prompt after adding or renaming lists or skills.

## 5. Back up and restore

- **Settings → Backup → Export backup…** opens the share sheet → **Save to Files** → iCloud Drive (`habit-tracker-backup-YYYY-MM-DD.json`, schema 2, key excluded). Desktop browsers download it.
- **Import backup…** replaces everything after a confirm. 1.0 backups import too and are upgraded to skills on the way in.
- Home shows a nudge when the last backup is older than 7 days. **Erase all data** wipes the device copy, key included.

## 6. Quick entry

| token | meaning |
|---|---|
| `!1` … `!5` | difficulty tier (10 / 25 / 50 / 100 / 200 XP) |
| `#reminders` | list (name prefix) |
| `+clarity` | skill (name prefix, e.g. `+clar`) — otherwise it's picked for you |
| `@today` `@tomorrow` `@fri` `@10/3` `@2026-10-03` | due date |
| `^daily` `^weekly` `^mon,wed,fri` `^weekdays` `^weekends` | habit with that schedule |
| `x8` | habit target of 8 check-ins a day (implies `^daily`) |

The line under New Item previews what will happen, including the skill (`🧘 Clarity · Tier 3 · 50 XP`).

## 7. Releasing an update

1. Bump `APP_VERSION` in `index.html` **and** `CACHE_VERSION` in `sw.js` together (currently `1.1.0` / `habits-v1.1.0`).
2. Run `tests.html` in a browser — all green.
3. Push to `main`. On the phone, opening the app with a connection loads the new code (network-first); an open app shows **Update ready — Reload**.

## 8. Troubleshooting

- **An item has no skill icon**: nothing matched the keyword guess yet. With a key set, Claude sorts it within seconds; otherwise pick one under **Boosts**. The Skills tab lists how many are waiting.
- **Skill checks never run**: Settings → AI shows the status (needs API key / off / last check failed). "Key rejected" = wrong or revoked key.
- **Photo failed with a network error**: AI needs a connection; retry when online.
- **"The app is open in another tab"** after an update: close the other copy (e.g. a Safari tab) and reopen.
- **Items vanished**: you're probably in the Safari tab instead of the Home Screen app (separate storage). Import the latest backup.
