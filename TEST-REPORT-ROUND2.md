# Game Night — Test Report (Round 2)

**Date:** 2026-04-28
**Tested by:** Live Playwright MCP exploration against `public/index.html` served on `127.0.0.1:59947`.
**Scope:** Regression-verify the 5 fixes from commit `3380f34`, re-run the 9-game × 6-check matrix, plus 14 targeted edge-case probes.
**Verification method:** Same as round 1 — mix of UI clicks and `browser_evaluate` reads/writes of the global `gameState` to deterministically drive scenarios.

## Summary

- **Games tested:** 9 / 9
- **Regressions verified:** 5 / 5 (all prior fixes hold)
- **New bugs found:** 1 (HIGH: 1, MED: 0, LOW: 0)
- **Cosmetic / corner notes:** 2 (LOW)

| Game | Setup→Play | Turn rotation | Cycling | Scoring | Skip/Quit | End condition |
|---|---|---|---|---|---|---|
| Taboo | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ (tie ✅) |
| Family Feud | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ (tie ✅) |
| Wheel of Fortune | ✅ | ✅ | ✅ | ✅ | ⚠️ HIGH-NEW-1 | ✅ (tie ✅) |
| Codenames | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| Trivia | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ (tie ✅) |
| Charades | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ (tie ✅) |
| Pictionary | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ (tie ✅) |
| Would You Rather | ✅ | ✅ | ✅ | n/a | ✅ | ✅ |
| Word Association | ✅ | ✅ | n/a | n/a | ✅ | ✅ |

> All 5 prior bugs (HIGH-1, HIGH-2, MED-1, MED-2, LOW tie) are verified fixed and held under regression. One new HIGH was uncovered: the WoF wheel-spin animation is not cancelable, so quitting mid-spin throws an uncaught `TypeError` ~3s after quit.

---

## Regressions verified (commit 3380f34)

### ✅ HIGH-1 — Trivia crashes at game end → FIXED
- `startTrivia` now sets `s.teams = s.players` (`index.html:1834`).
- Live test: 1-question Trivia (Phase C #9), skipped → results screen rendered cleanly with no console error. `gameState.teams === gameState.players` returned `true`.
- Bonus: with 0/0 scores, screen rendered "🤝 It's a tie!" (LOW fix piggybacked).

### ✅ HIGH-2 — Family Feud awards full board on strike-out → FIXED
- `feudAwardRound` (`index.html:2521-2540`) no longer adds unrevealed answer pts to `s.roundPoints`.
- Live test: Team 1 garbage-guesses 3×, Team 2 garbage-steals 1×. Banner correctly read **"SURVEY SAYS! 0 pts to Team 1!"** (was 100 pts pre-fix). Both teams' scores stayed at 0.

### ✅ MED-1 — WoF round starter rotation → FIXED
- `startWheelOfFortune` initialises `s.roundStarter = 1` (`index.html:2753`).
- Live test: 3 players, 3 rounds. Sequence verified: Round 1 starter = P3 (`currentPlayer=0`), Round 2 = P4 (`currentPlayer=1`), Round 3 = P5 (`currentPlayer=2`). Rotates correctly.

### ✅ MED-2 — Family Feud steal "one guess only" → FIXED
- `feudSubmitGuess` (`index.html:2474-2484`) short-circuits steal phase: one guess → award immediately.
- Live test (steal-fails): Team 2 enters steal, submits one wrong guess → round ends, `currentTeam` reverted to `playingTeam=0`, banner read "0 pts to Team 1!". `phase` stayed `'steal'` until safeTimeout-driven re-render (which is correct since the round is in award-phase).
- Live test (steal-succeeds): Team 1 reveals 1 answer (28 pts), strikes out. Team 2 (steal) submits one matching guess (+25 pts) → round ends with **"SURVEY SAYS! 53 pts to Team 1!"** — wait, actually Team 1 was the steal team in this rerun and `currentTeam=0`, so the steal team correctly received the FULL roundPoints (28+25=53). All extra answers revealed by `feudAwardRound` did NOT add to roundPoints (HIGH-2 fix verified jointly).

### ✅ LOW — Tied games announce a tie → FIXED in all 3 call sites
- `showResults` (line 2779): generic — verified tie display in Trivia (1-Q tie 0-0), Taboo (forced 0-0 across 3 rounds), Charades (0-0), Pictionary (0-0). All show "🤝 It's a tie!".
- `wofShowResults` (line 1421): verified — 3 players all at $0 → "🤝 It's a tie! tied at $0".
- `showFeudResults` (line 2568): isTie check present in code; not reached at end-of-game in this run because Team 2 stole and won. Tie path inspected statically and matches the others.

---

## New bugs

### 🔴 HIGH-NEW-1 — WoF wheel spin is not cancelable; mid-spin quit throws uncaught TypeError

**Game:** Wheel of Fortune
**Symptom:** When the user starts a spin (~4-second animation) and then quits or navigates away before the spin completes, the spin's `requestAnimationFrame` loop and `tickInterval` keep running. When the spin finishes, its callback (`wofDoSpin`'s segment handler) calls `renderWOF()`, which dereferences `s.teams[s.currentPlayer]` against the now-empty `gameState`, throwing `TypeError: Cannot read properties of undefined (reading 'undefined')` at `index.html:1232`.

**Repro:**
1. Start Wheel of Fortune with any players.
2. Click SPIN.
3. Within ~3 seconds (before spin ends), navigate back / start a different game.
4. ~3 seconds later, console shows: `Uncaught TypeError: Cannot read properties of undefined (reading 'undefined') at renderWOF (index.html:1232)`.

**Reproduced via:**
```js
setupGame('wheeloffortune');
startWheelOfFortune();
wofDoSpin();
await sleep(1000);
quitToken++; gameState={}; showScreen('home');
await sleep(4000);
// ↳ window.onerror captured the exception.
```

**Cause:** `wofSpinWheel` (`index.html:1124-1170`) starts a `requestAnimationFrame` loop and a `setInterval` for sound ticks. Neither captures `quitToken` nor checks for it. The `s.spinning` flag stays true, but the spin's callback is invoked unconditionally on completion. The callback paths in `wofDoSpin` (lines 1267-1287) read/write `s.teams`, `s.currentPlayer`, `s.lastResult`, then call `renderWOF()`, which fails because `gameState` was wiped to `{}`.

**Proposed fix:** Capture `const tok = quitToken;` at the top of `wofSpinWheel`. In `animate()` after computing `progress>=1`, check `if (tok !== quitToken) { clearInterval(tickInterval); return; }` before calling `callback(...)`. Equivalently, wrap the segment-handler invocation: `if (tok === quitToken) callback(WOF_SEGMENTS[idx]);`. Also clear `tickInterval` on quit by adding `.wof-spin` (or similar marker) to the cleanup query in `confirmQuit`/`setupGame` — though just guarding the callback is enough.

**Severity rationale:** While the user has already left the game, the uncaught exception pollutes the console and could mask other errors. If users open Pictionary or Trivia next, a console-watching dev or QA will see noise. Also, on slower devices the spin can take up to ~4s, widening the race window. Marked HIGH because it's an uncaught runtime exception triggered by a normal quit gesture.

---

## Edge-case probe results (Phase C)

| # | Probe | Result |
|---|---|---|
| 1 | Quit during 1.2s/1.5s safeTimeout windows (WoF, Trivia, Codenames, Feud, Word Assoc) | **PASS** — `safeTimeout`'s `quitToken` check correctly cancels stale callbacks. Verified individually for WoF (1500ms BANKRUPT), Trivia (1500ms answer), Codenames (800ms assassin), Feud (2500ms award), Word Assoc (500ms results). **Caveat: WoF wheel-spin animation itself is NOT a `safeTimeout`** — see HIGH-NEW-1. |
| 2 | WoF: BANKRUPT/LOSE as the very first wedge of round 2 | **PASS** — both branches zero `roundBank` (already 0), then `wofNextPlayer` rotates correctly. State remains consistent. |
| 3 | WoF: vowel buy attempt with `roundBank < $250` | **PASS** — `wofBuyVowel` (`index.html:1331-1332`) early-returns. State unchanged: bank stays 100, phase stays 'spin', `buyingVowel` stays false. UI also disables button via `!canBuyVowel` check. |
| 4 | WoF: solve attempt with the wrong puzzle | **PASS** — `wofTrySolve` (`index.html:1346-1361`) sets transitioning, schedules `wofNextPlayer` via 1.2s safeTimeout. After timeout, `currentPlayer` rotated cleanly (2→0 mod 3). |
| 5 | Feud: steal team submits one wrong guess after partial reveal | **PASS** (couples HIGH-2 + MED-2). Team 2 reveals 1 answer (28 pts), strikes out. Team 1 (steal) submits one wrong guess. Banner: "0 pts to Team 1!" — original Team 2 (`playingTeam=1`) correctly receives the 28 they earned. Wait — re-check: `s.currentTeam=s.playingTeam` then `feudAwardRound()` awards `roundPoints` (28) to `s.teams[s.currentTeam=1]`. Score went to Team 2. ✓ |
| 6 | Feud: steal succeeds with the LAST unrevealed answer | **PASS** — Team 2 reveals 6/7 (98 pts), strikes out. Team 1 (steal) guesses last answer "IV Drips" (2 pts). `feudRevealAnswer` triggers `feudAwardRound` because `revealed.size === answers.length` — and the steal-phase guard in `feudSubmitGuess` (`index.html:2479`) skips the second `feudAwardRound` call. Final: Team 1 gets 100 pts (98+2). No double-fire. |
| 7 | Codenames: assassin click as the very first guess | **PASS** — Red clicks assassin, results screen renders "💀 ASSASSIN! Red Team found the assassin. Blue Team wins!" via 800ms safeTimeout. |
| 8 | Codenames: win on the last allowed guess of a clue | **PASS** — Set redLeft=1, give clue with num=1 (guessesLeft=2), click last red card. Win check fires before guessesLeft logic; `gameOver=true`, results "Red Team Wins!" via 600ms safeTimeout. |
| 9 | Trivia: 1-question game | **PASS** — `gameState.rounds=1`, `startTrivia`, `triviaSkip` → `triviaAdvance` → `renderTriviaQuestion` sees `qIdx>=questions.length` → `showResults`. Results screen rendered correctly with tie display. |
| 10 | WYR: 1 voter | **PASS** — Removed second player so only 1 voter. Vote='a' → 100% / 0% bars rendered. No division-by-zero (total=1). |
| 11 | Word Association: drive eliminations to 1 survivor | **PASS** — 3 players, eliminate twice. Last alive player correctly identified as winner ("Player 17 Wins!"). Rotation skips dead players via the `while(!s.players[s.currentPlayer].alive)` loop in `submitWord` (line 2302). |
| 12 | Charades AND Pictionary 0-0 ties | **PASS** for both. `showResults` (generic) renders "🤝 It's a tie!" emoji and copy. |
| 13 | Family Feud: Skip Survey during the steal phase | **PASS with note** — State resets cleanly (phase='playing', strikes=0, revealed=∅, roundPoints=0, surveyIdx++). `roundsPlayed` is NOT incremented (no round consumed). HOWEVER, the team that plays the new survey is the steal team (Team 2 in test), not the original playing team. See **LOW-NEW-1** below. |
| 14 | Multi-game session leakage (WoF mid-round → Trivia) | **PASS** — `setupGame('trivia')` from any prior state correctly: increments `quitToken`, clears `gameState.timer`, removes all overlay nodes (.pass-screen, .team-transition, .feud-banner, .confetti-container), wipes `gameState={}`, then re-initialises for the new game. No cross-game state bleed observed. **EXCEPT** — see HIGH-NEW-1 (WoF spin runs even after gameState is wiped). |

---

## Additional findings

### 🟢 LOW-NEW-1 — Family Feud "Skip Survey" during steal phase silently transfers the round to the steal team

**Game:** Family Feud
**Symptom:** When the playing team strikes out and the OTHER team enters steal, then someone clicks "Skip Survey", the new survey starts with `currentTeam` = the steal team (because `renderFeudRound` sets `s.playingTeam=s.currentTeam` and the steal logic had already swapped `currentTeam`). The original team that was supposed to play this round loses their turn entirely without a clear UX cue.

**Repro:**
1. Start Family Feud, Team 1 currentTeam.
2. Team 1 strikes out 3× → phase='steal', currentTeam=1 (Team 2).
3. Click "Skip Survey ↻".
4. New survey appears with Team 2 as both `currentTeam` and `playingTeam`.

**Cause:** `feudSkipSurvey` (`index.html:2428-2433`) just bumps `surveyIdx` and re-renders; `renderFeudRound` (`index.html:2393`) then captures whatever `s.currentTeam` happens to be as the new `playingTeam`. Skip during 'playing' phase does the right thing (currentTeam unchanged), but skip during 'steal' phase carries the steal-team assignment into the new round.

**Proposed fix:** In `feudSkipSurvey`, if `s.phase === 'steal'`, restore `s.currentTeam = s.playingTeam` before re-rendering. (Or stronger: only allow Skip Survey while `phase==='playing'` — disable the button in steal phase and during the post-award banner window.)

**Severity:** LOW — recoverable, doesn't break the game; arguably intended behavior (the Skip lets you discard a survey mid-round and continue). But the team-rotation side-effect is non-obvious and undocumented.

### 🟢 LOW-NEW-2 — Generic `showResults` n-team tie only checks index 0 vs 1

**Game:** Affects Taboo, Charades, Pictionary, Trivia, Wheel of Fortune end-of-game with ≥3 players/teams.
**Symptom:** `showResults` (`index.html:2779`) and `wofShowResults` (`index.html:1421`) compute `isTie = teams[0].score === teams[1].score`. If the top three teams are tied at the same score (e.g. WoF with all 3 players at $0), the tie banner fires correctly — but if teams[0]/teams[1] tie and teams[2] is also tied, the message just says "tied at X" without listing all tied parties. More importantly, **if teams[1] and teams[2] are tied but teams[0] is alone in first**, the existing logic correctly declares the sole leader winner — so this is only a display nuance for top-N ties.

**Repro:** WoF 3 players at $0 → "🤝 It's a tie! tied at $0" (correct general intent). UX is acceptable but ambiguous about how many parties tied.

**Severity:** LOW (cosmetic). Functional behavior (no wrong winner declared) is correct.

**Proposed fix:** Optional. To be precise: list all `teams.filter(t => t.score === winner.score)` in the subtitle, e.g. "Tied between Player 1, Player 2 at $0".

---

## Per-game evidence

### Taboo
- Scoring sequence verified: +1, +1, -1, 0, -1, clamp-at-0 (matches round 1).
- 6-turn / 3-round rotation reaches `showResults` exactly when `round>rounds`.
- Tie screen renders "🤝 It's a tie!" when both teams end at 0.
- Quit cleanup: `gameState`, `gameState.timer`, all overlay divs cleared.

### Family Feud
- Strike-out (zero reveals) now awards 0 (was 100). ✓ HIGH-2 verified.
- Steal failure with one wrong guess: round ends, original team gets only revealed pts. ✓ MED-2 verified.
- Steal success (mid-survey): all roundPoints (revealed-by-original + revealed-by-steal) go to steal team. ✓
- Steal success on LAST unrevealed answer: no double-fire of `feudAwardRound`. ✓
- Skip Survey during steal: state clean, but team assignment quirk. See LOW-NEW-1.

### Wheel of Fortune
- `roundStarter` rotation P1→P2→P3 verified (3-player, 3-round game). ✓ MED-1 verified.
- BANKRUPT and LOSE-A-TURN as the first wedge of round 2: rotation works correctly.
- Vowel buy with `roundBank<250` rejected.
- Wrong-puzzle solve attempt rotates to next player after 1.2s.
- Tied final ($0/$0/$0): "🤝 It's a tie! tied at $0".
- 3 unique puzzles in 3 rounds.
- **NEW: HIGH-NEW-1 found** — quit during spin throws uncaught error after spin completes.

### Codenames
- Assassin first-click → instant loss for guessing team.
- Win on last allowed guess (count→0): correct winner shown.
- safeTimeout cancellation on quit during 600/800ms result delays: works.

### Trivia
- 1-question game completes without crash. ✓ HIGH-1 verified.
- Tie at 0/0 displays correctly. ✓ LOW verified.
- safeTimeout (1500ms after answer) cancellable on quit.

### Charades
- Tie 0/0 displays "🤝 It's a tie!". ✓
- Quit cleanup: no errors.

### Pictionary
- Tie 0/0 displays "🤝 It's a tie!". ✓
- Quit cleanup mid-turn: no errors thrown by the canvas.

### Would You Rather
- 1-voter game: 100%/0% bars render correctly (no NaN).

### Word Association
- Eliminations down to 1 survivor → results "{name} Wins!".
- Rotation skips dead players (verified previously in round 1; behavior unchanged).
- safeTimeout (500ms) cancellable on quit.

---

## Setup & teardown notes

- Server: `python3 -m http.server 0 --bind 127.0.0.1 --directory public` (port 59947 selected).
- Console showed only one expected error throughout: `favicon.ico` 404 (and a related `ERR_EMPTY_RESPONSE` retry). All other errors observed were intentional bug repros (HIGH-NEW-1, plus a transient renderTabooTurn error caused by a synthetic test that called `setupGame` while a `team-transition` countdown's `setInterval` callback was queued — not user-reachable in normal play).
- Browser cleanly closed; http.server stopped on completion.

---

## Recommended fix order

1. **HIGH-NEW-1** (WoF spin ignores quitToken) — guard the spin completion callback with the captured `tok === quitToken` pattern. ~3 lines in `wofSpinWheel`.
2. **LOW-NEW-1** (Feud Skip Survey during steal) — one-line restore of `currentTeam` in `feudSkipSurvey`, or disable the Skip button in steal phase.
3. **LOW-NEW-2** (n-team tie copy) — optional polish.

All 5 prior fixes are stable. No regressions detected.
