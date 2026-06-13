# 🎮 Game Night — Roadmap

This document tracks planned improvements for the Family Game Night app. The app is a single self-contained file (`public/index.html`) deployed on Vercel. Content is family-friendly and tuned for kids ages 8–11 playing alongside the whole family.

Phases are ordered by priority. Check items off as they ship.

---

## Phase 1 — Content ✅ DONE

Massively expanded the content library across every game. All pools now comfortably exceed their targets, and a meaningful slice of each pool is tagged `age:'kids'` so the **Kids age-tier is now fully playable** (previously it had no eligible content and fell below minimum pool sizes everywhere).

| Game | Before | Now | Target | Kids-tagged |
|------|-------:|----:|-------:|------------:|
| Taboo | 191 | 238 | 150 | 22 |
| Trivia | 99 | 231 | 200 | 70 |
| Charades | 92 | 161 | 150 | 35 |
| Pictionary | 88 | 153 | 150 | 30 |
| Would You Rather | 50 | 101 | 100 | 27 |
| Word Association | 39 | 165 | 150 | 25 |
| Family Feud | 25 | 154 | 150 | 90 |
| Codenames | 101 | 202 | 200 | 40 |
| Wheel of Fortune | 72 | 116 | 100 | 16 |
| Dinner Conversations | 235 | 265 | 100 | n/a |

- [x] Expanded content for all games (above targets)
- [x] Mixed difficulty in Trivia / Pictionary (easy for ~8yo through hard)
- [x] Tagged easy content `age:'kids'` so Kids mode clears all `MIN_POOL_SIZE` thresholds
- [x] Deduplicated new content against existing entries
- [x] Validated JS syntax + verified live in browser (Playwright)

---

## Phase 2 — PWA & Offline

Make the app installable and playable with no connection (great for road trips / spotty WiFi).

- [ ] Web app manifest (`manifest.webmanifest`) — name, icons, theme color, `display: standalone`
- [ ] App icons (192px, 512px, maskable)
- [ ] Service worker that caches the single HTML file + any assets (cache-first)
- [ ] Offline fallback + "update available" prompt when a new version deploys
- [ ] Install-to-homescreen affordance (iOS + Android)
- [ ] Verify offline play with Lighthouse PWA audit

---

## Phase 3 — Player Profiles & Stats

Turn one-off game sessions into an ongoing family rivalry.

- [ ] Player name entry / lightweight profiles (avatar emoji + color)
- [ ] Per-game history (date, players, scores, winner)
- [ ] Win tracking & family leaderboard
- [ ] Persistent stats via `localStorage` (with export/clear)
- [ ] "Player of the week / month" highlight
- [ ] Reuse last-used team/player setup for faster starts

---

## Phase 4 — New Games

- [ ] Name That Tune (audio clips + guess; needs asset/offline strategy)
- [ ] Scattergories (letter + categories + timer + scoring)
- [ ] Two Truths and a Lie (per-player entry + voting)
- [ ] Drawing Telephone (Gartic Phone style — draw ↔ guess relay)
- [ ] Family Jeopardy (category board, point values, buzz-in)

---

## Phase 5 — UX Polish

- [ ] Quick-play mode (random game, skip setup)
- [ ] Game favorites / recently played row on home
- [ ] Customizable timer lengths per game
- [ ] Better mobile landscape support
- [ ] Accessibility: screen-reader labels, keyboard nav, focus management, reduced-motion
- [ ] Sound on/off + volume control; haptics on mobile
- [ ] Dark/light theme toggle (currently dark-only)

---

## Phase 6 — Technical Cleanup

- [ ] Split monolithic `index.html` into modules (per-game JS, shared engine, content files)
- [ ] Move game content into separate JSON/JS data files (easier to expand & review)
- [ ] Add analytics (privacy-friendly) — which games are played most, session length
- [ ] Performance: lazy-load per-game code, trim render work
- [ ] Unit tests for game logic (scoring, turn-taking, pool filtering, end-game)
- [ ] CI check that validates content arrays (counts, dedup, schema) on every PR

---

## Notes

- **Single-file constraint (current):** the whole app lives in `public/index.html`. Until Phase 6, new content is appended to the existing data arrays near the top of the `<script>` block (`TABOO_CARDS`, `TRIVIA_QS`, `CHARADES_PROMPTS`, `PICTIONARY_WORDS`, `WYR_QS`, `WA_STARTERS`, `FEUD_SURVEYS`, `CN_WORDS`, `WOF_PUZZLES`, `DINNER_PROMPTS`).
- **Age tiers:** `kids ⊂ family ⊂ adult`. Items without an `age` field default to `family` (shown in Family/Adult, hidden from Kids). Tag genuinely simple, kid-appropriate items `age:'kids'` to surface them in Kids mode.
- **Family-friendly only:** all content must be appropriate for an 8-year-old.
</content>
</invoke>
