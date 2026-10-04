# The Backlog: Project Handoff

Everything about the Backlog app so far, written for Claude Code (or any future chat) to pick up cold. Drop this file in the repo root as `CLAUDE.md` and Claude Code will read it automatically.

---

## 1. What it is

A personal media backlog tracker built for Aaron. It tracks shows, anime, manga, games, movies, cartoons, documentaries, Patreon reaction watches and misc learning, plus mood-based "vibe pairings" (the cheese and wine system). It replaced a pile of scattered iCloud Notes and Discord dumps, and an earlier markdown checklist that couldn't actually be ticked.

- **Live URL:** https://backlog-nu.vercel.app/
- **Hosting:** GitHub repo (`backlog`) connected to Vercel. Committing to GitHub auto-deploys in about 30 seconds.
- **Installed as:** a PWA on Aaron's iPhone home screen (added via Safari share sheet).
- **Main device:** phone. PC browser has its own separate data (see section 5).
- **Portfolio value:** this is a real product-thinking story for Aaron's move into PM in tech. Iterative builds, user-driven QA, non-destructive data migrations.

---

## 2. Tech stack and architecture

- **One self-contained `index.html`** (about 1,530 lines). Inline CSS and vanilla JS, no framework, no build step, no backend.
- **Storage:** `localStorage`, key `backlog-tracker-v4`. **Never change this key.** Changing it orphans all saved data. Versioning is handled inside the data instead (see section 3).
- **PWA:** inline data-URI manifest, apple-touch-icon (gold "B" on near-black), and a blob-registered service worker for offline caching.
- **Rendering:** a single `render()` that rebuilds tabs, stats and content from `state.data`.

### Data shape
```js
{
  version: 12,
  pairings: [{ id, name, desc, items: [string] }],
  categories: {
    shows:    { label, items: [{ id, title, type, done, active?, onhold? }] },
    cartoons, anime, patreon, manga, games, movies, docs, misc
  },
  completed: [title strings]
}
```
- `type`: `'priority'` puts an item in the ★ Priority block at the top of its category. Anything else is normal.
- `active` / `onhold`: flags (mutually exclusive). Marking done clears both.
- `completed`: list of titles shown on the ✅ tab.

### Tabs
🍷 Pairings · 🔴 Currently Active · ⏸️ On Hold · 📺 Shows · 🎭 Cartoons · ⛩️ Anime · 🎬 Patreon · 📚 Manga · 🎮 Games · 🎞️ Movies · 🎥 Docs · 🎵 Misc · ✅ Done

**Currently Active and On Hold are virtual views**, not real categories. They gather every item with the matching flag from across all categories and group them under their home category label (📺 Shows, 📚 Manga etc.) so an anime and a manga of the same name can't be confused. This was the v5 refactor.

---

## 3. The migration system (most important thing to understand)

`localStorage` data always wins over `INITIAL_DATA`, so new defaults never reach an existing user on their own. The fix is a versioned migration array:

- `INITIAL_DATA.version` is the version a brand new user starts at.
- `MIGRATIONS[n]` upgrades data from version n+1 to n+2. `runMigrations()` runs every pending one in order on load, then saves.
- Data with no `version` field is treated as v1.

### Rules for any future change
1. **Never edit or reorder existing migrations.** Only append new ones to the end.
2. Every content change needs **two edits**: update `INITIAL_DATA` (for new users) AND add a migration (for Aaron's existing data). Bump `INITIAL_DATA.version` by one.
3. Migrations must be **idempotent and non-destructive**: check by title or id before adding, never wipe user ticks, flags or custom items.
4. UI-only changes (CSS, handlers) need no migration and no version bump.
5. After editing, validate: extract the `<script>` contents and run `node -c` on it. For bigger migrations, simulate an older data object through `runMigrations` and assert the result.

### Migration history
| Ver | What it did |
|---|---|
| v2 | Replaced generic "Pokémon ROM Hacks" with 11 individual hacks + PokeMMO (Renegade Platinum as priority) |
| v3 | Replaced Hype Shonen pairing with 🎭 Identity / Becoming |
| v4 | Psychological gained Hannibal + Mr. Robot; Identity lost Homunculus + 20th Century Boys; Mr. Robot added to Shows (priority) |
| v5 | **Big refactor.** Deleted the real `currently`/`onhold` categories, moved those items back to home categories with `active`/`onhold` flags |
| v6 | Added 8 new pairings (p10 to p17) |
| v7 | QA pass: deleted Mastery, Underdog, Genius/Death Game, Comfy, Conspiracy; renamed KH/Adventure to "Adventure + No.1s" (+Magi); On The Pitch to "Sports" (+Safety); Gamaran into Samurai; Get Out removed |
| v8 | Deleted Weight of the World; Steins;Gate + Detroit into Sci-Fi; Pokémon ROMs from Healing to Childhood; +Severance, +Mindhunter, +Andor, +Liar Game, +Cyberpunk, +March Comes In Like a Lion |
| v9 | +Chrono Trigger (Adventure), +The Wire and My Dearest Self (Crime); added missing tickable items Liar Game, March Comes In Like a Lion, Chrono Trigger |
| v10 | +Nippon Sangoku (Anime + Society/Revolution) |
| v11 | +Kagurabachi (manga, priority), +Ys 1-8, +Resident Evil lineup |
| v12 | +Widow's Bay, +House of the Dragon, +Silo S3 (shows), +Daemons of the Shadow Realm (anime); HotD into Society/Revolution, Silo into Sci-Fi, Daemons into Adventure + No.1s |

---

## 4. Interaction design (decisions made from real use)

- **Tap the checkbox** to toggle done. The checkbox has an enlarged tap target.
- **Tapping the title does nothing.** It used to open the action menu, but that felt risky (fear of accidentally hitting delete), so it was removed.
- **Long press (0.5s) anywhere on a row** opens the action sheet: Mark/Remove Currently Active, Mark/Remove On Hold, Mark done, Move to another category, Delete.
- **Scroll protection:** moving more than 8px cancels the long press, so scrolling never triggers anything.
- **Tap-through bug fix:** when the long press opens the sheet, all sheet buttons ignore taps for 400ms. Without this, lifting the finger instantly triggered whichever button landed under it (items were silently deleted or moved).
- **+ button** (top right) opens the add sheet: title, category, normal/priority. Pre-selects the current tab's category unless on a virtual tab.
- Footer reminder: "Max: 1 show · 1 anime · 1 manga · 1 game".

---

## 5. Known limitations

- **No sync between devices.** Phone and PC each have their own `localStorage`. Decision: the phone is the real tracker. Since v12 the ✅ Done tab has **Export / Import** backup buttons (JSON file; uses the iOS share sheet in the PWA). Import replaces all data on that device and runs migrations on older backups. Real sync would need a backend like Supabase (bigger project).
- If stored data ever fails to parse, it is stashed under `backlog-tracker-v4-unreadable-<timestamp>` before defaults load, so it can be recovered.
- PWAs cache hard. After a deploy, fully close and reopen the app (or delete and re-add the home screen icon if it refuses to update).
- Vibe pairings are just strings. They are not linked to items, so a pairing can mention a title that isn't tickable. An audit was run at v9 and fixed the gaps, but this can drift. A future improvement would be linking pairing entries to item ids.
- Live site was confirmed on v11 on 2026-10-05 (tag `v11` in git is the rollback point before the game-layer work).

---

## 6. Current pairings (v12)

1. 🗡️ **Dark Fantasy / Souls**: Elden Ring, Berserk, Dororo
2. ⚔️ **Samurai / Honour**: Ghost of Tsushima, Vagabond, GAMARAN, Blue Eye Samurai, Shogun
3. 🧠 **Psychological**: Persona 3 Reload, Monster, Homunculus, Hannibal, Mindhunter
4. 🎭 **Identity / Becoming**: Disco Elysium, Persona 5 Royal, Mob Psycho 100, Bunny Girl Senpai, The OA, Severance, American Beauty, Fight Club, The Talented Mr Ripley
5. 🌌 **Sci-Fi / Existential**: NieR: Automata, Detroit: Become Human, Fire Punch, All You Need Is Kill, Code Geass, Steins;Gate, Eureka Seven, Silo
6. 🕵️ **Crime / Gritty**: Yakuza 0, Sun-Ken Rock, My Dearest Self, Breaking Bad → BCS, The Wire
7. 🏰 **Adventure + No.1s**: Kingdom Hearts 2 Critical, FMA Brotherhood / manga, Frieren, Magi, Chrono Trigger, Daemons of the Shadow Realm
8. 💔 **Healing / Reset**: Vinland Saga, March Comes In Like a Lion, A Silent Voice, BoJack, Erased, Gurren Lagann
9. 🌍 **Society / Revolution**: Eden of the East, The Running Man, Squid Game, The Hunt (2020), 20th Century Boys, Mr. Robot, Andor, Liar Game, Nippon Sangoku, House of the Dragon, Cyberpunk 2077, society/corruption video essays
10. ⚽ **Sports**: Blue Lock (anime), Blue Lock, Ao Ashi, Be Blues, Diamond no Ace, Safety (Disney+)
11. 🎮 **Childhood Reconnection**: Kingdom Hearts (KH2 + CoM + Re:Coded), Ocarina of Time, Pokémon ROMs, KH music on piano, throwback cartoons

Pairing ids are not contiguous (p10 to p13, p15, p16 were deleted). New pairings should use p18 onward.

### Pairing rules Aaron has set (respect these)
- Pairings are moods that combine media, ideally across a game, a manga and an anime/show/movie.
- **Check that a title genuinely fits before adding it.** Past mistakes: Gamaran (a sword manga) was put in a death-game pairing; Re:Zero and All You Need Is Kill were wrongly called death games (they're time loops).
- Avoid overlap. A title should usually live in one pairing. Sections that almost any anime fits (Underdog, Mastery) were deleted for being too broad.
- Genius / Death Game was deleted on purpose. Revisit only when genuinely good new death-game titles exist.

---

## 7. Item counts (v12 defaults)

Shows 72 · Cartoons 12 · Anime 55 · Patreon 26 · Manga 40 · Games 60 · Movies 60 · Docs 4 · Misc 7 · Completed 22

Notable groups: individual Pokémon ROM hacks (Renegade Platinum priority, then Refined Platinum, Xenoverse, Sors, Saiph, Definitive, Azure, Redux, Empire, Gamma Emerald, PokeMMO); Resident Evil (2002 Remake, RE2/3/4 Remakes, RE7, Village, Requiem; RE5/6/Code Veronica skipped until remakes exist).

---

## 8. Ideas parked for later

- **Design refresh:** a design brief (`Backlog_Design_Brief.md`) was written for Claude Design, aiming for "refined premium" (Linear / Things 3 feel: serif title, rounded rows, status pills, warmer palette) rather than flashy animation. Not built yet. Aaron felt the gain wasn't deep enough to justify the effort for now.
- **"Start ONE thing" system:** a slot at the top for the single thing Aaron is starting today, with its reason. Aimed at the "I know I'll love it but can't start it" problem. Not built.
- Aaron's mum suggested selling it. Assessment: not sellable without accounts, backend, sync and support, but strong as a portfolio piece. Aaron has floated accounts and community/forum features as long-term ideas, with the condition they stay sleek and not overwhelming.

---

## 9. How Aaron deploys (without Claude Code)

1. github.com → `backlog` repo → `index.html` → pencil icon
2. Select all, delete, paste the new file
3. Commit changes
4. Wait ~30 seconds, fully close and reopen the app on the phone

With Claude Code in the repo, the flow becomes: edit `index.html`, validate, commit and push.

**Bigger features go on a branch first.** Vercel builds a preview URL for every branch, so Aaron can try it on his phone before it touches the real app (the preview is a different site, so it has its own empty data). Merge to `main` only once he is happy. Each release is tagged (`v11`, `v12`, ...) so any version can be restored.

**Rollback safety rule:** new features must only *add* new keys to the data object and never change the shape of existing ones. Older code ignores keys it doesn't know and skips migrations when the data version is ahead, so rolling back keeps all ticks.

---

## 10. Related docs from the same chat

- `Aaron_Leisure_Doctrine.md`: the why/when/how behind the backlog (core truth, rules, current stack, intentional experiences like FMA manga + Frieren and Bleach before July 25th, pairings, new-release method). Keep its pairing section in sync with the app.
- `Backlog_Design_Brief.md`: the visual redesign brief.
- Working preferences: Aaron does real QA and will catch wrong fits, so verify titles (search if unsure) instead of guessing. Explain deploy steps plainly. Keep answers direct.
