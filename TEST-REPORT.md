# Game Night — Test Report

**Date:** 2026-04-28
**Tested by:** Live Playwright MCP exploration against `public/index.html` served on `127.0.0.1:55835`.
**Scope:** All 9 games × 6-check matrix (setup → play, turn rotation, question cycling, scoring rules, skip/quit cleanup, end condition).
**Verification method:** Mix of UI clicks and `browser_evaluate` reads of the global `gameState` to confirm internal transitions deterministically.

## Summary

- **Games tested:** 9 / 9 (Taboo, Family Feud, Wheel of Fortune, Codenames, Trivia, Charades, Pictionary, Would You Rather, Word Association)
- **Bugs found:** 5 (HIGH: 2, MED: 2, LOW: 1)
- **Pass rate per check (54 total checks):** 50 ✅ / 4 ❌

| Game | Setup→Play | Turn rotation | Cycling | Scoring | Skip/Quit | End condition |
|---|---|---|---|---|---|---|
| Taboo | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| Family Feud | ✅ | ✅ | ✅ | ❌ HIGH | ✅ | ✅ |
| Wheel of Fortune | ✅ | ❌ MED | ✅ | ✅ | ✅ | ✅ |
| Codenames | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| Trivia | ✅ | ✅ | ✅ | ✅ | ✅ | ❌ HIGH |
| Charades | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| Pictionary | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| Would You Rather | ✅ | ✅ | ✅ | n/a | ✅ | ✅ |
| Word Association | ✅ | ✅ | n/a | n/a | ✅ | ✅ |

> A LOW-severity tie-break finding affects Taboo / Charades / Pictionary / Family Feud / Wheel of Fortune — not counted in the table since each game's scoring still works; only the announced winner is wrong on a tie.

---

## Bugs

### 🔴 HIGH-1 — Trivia crashes at game end

**Game:** Trivia
**Symptom:** When the last question is answered/skipped, `triviaAdvance` calls `renderTriviaQuestion` which calls `showResults`, which throws `TypeError: gameState.teams is not iterable`. The user never sees a results screen.

**Repro:**
1. Start Trivia with any number of players.
2. Skip / answer all 10 questions.
3. Last question's `triviaAdvance` triggers crash.

**Cause:** `index.html:2764` — `showResults` reads `gameState.teams`, but `startTrivia` (`index.html:1827`) stores players in `gameState.players` and never sets `s.teams`. All other games happen to set `s.teams`.

**Proposed fix:** In `startTrivia` add `s.teams = s.players;` so both references point to the same array. (Or in `showResults`, fall back: `const arr = gameState.teams || gameState.players;`.) Since `triviaAnswer` already mutates `s.players[currentPlayer].score`, sharing the reference is enough.

---

### 🔴 HIGH-2 — Family Feud awards full board on strike-out

**Game:** Family Feud
**Symptom:** When a team strikes out 3 times (and the steal team also strikes out — or even when no answers were ever revealed), the round still awards the **entire** survey's point pool to the original team. A team that misses every guess and never reveals an answer can still earn 100 pts.

**Repro:**
1. Start Family Feud (defaults: 5 rounds, Team 1 vs Team 2).
2. Play any round; submit 3 garbage strings (no reveals) → phase = 'steal', currentTeam = Team 2.
3. Submit 3 more garbage strings (steal fails) → banner reads `SURVEY SAYS! 100 pts to Team 1!` and Team 1's score is +100 despite revealing zero answers.
4. Verified end-to-end across 3 consecutive zero-reveal rounds: 100 pts awarded each round.

**Cause:** `index.html:2508-2530` (`feudAwardRound`) reveals every unrevealed answer and adds `ans.pts` to `s.roundPoints` before awarding. So the awarded amount = full survey total, not the points the playing team actually earned.

**Proposed fix:** In `feudAwardRound`, do not add unrevealed answers to `s.roundPoints`; only reveal them visually (skip the `s.roundPoints+=ans.pts;` line, line ~2522). Then `s.teams[s.currentTeam].score += s.roundPoints` awards only what was actually revealed during play (or 0 if nothing was). This matches real Family Feud rules.

---

### 🟡 MED-1 — Wheel of Fortune round starter rotates one round late

**Game:** Wheel of Fortune
**Symptom:** Round 1 and Round 2 are started by the same player. With 3 players the starting-player sequence is **P1, P1, P2, P3, P1, …** instead of **P1, P2, P3, P1, P2, …**. The first rotation is skipped.

**Repro:**
1. Start WoF with 3 players, 3 rounds.
2. Solve round 1 → round 2 begins. Inspect `gameState.currentPlayer` — it is `0` (Player 1) instead of `1`.
3. Solve round 2 → round 3 begins with currentPlayer `1` (Player 2).

**Cause:** `index.html:1412-1413` (`wofStartRound`) does `s.currentPlayer = s.roundStarter % length; s.roundStarter++;`. `startWheelOfFortune` initialises `roundStarter = 0` (`index.html:2741`). On the first call after round 1, `roundStarter` is still 0, so currentPlayer ends up at index 0 again — the same player who started round 1.

**Proposed fix:** Initialise `s.roundStarter = 1` in `startWheelOfFortune` so the next round picks player 1 (mod n). Equivalent: pre-increment in `wofStartRound` (`s.roundStarter++; s.currentPlayer = s.roundStarter % length;`).

---

### 🟡 MED-2 — Family Feud steal phase doesn't enforce "one guess only"

**Game:** Family Feud
**Symptom:** In real Family Feud, the steal team gets exactly **one** guess: if right, they keep all the points; if wrong, the points go back to the original team. This implementation lets the steal team keep guessing until they either reveal **all** answers or strike out 3 times — turning steal into a near-guaranteed full-board win for whoever's better at trivia.

**Repro:**
1. Drive Team 1 to 3 strikes (phase → 'steal', currentTeam → Team 2).
2. Have Team 2 submit a correct guess — round does NOT end; they can continue guessing.
3. Verified: Team 2 can reveal multiple answers without round ending, only ending when all-revealed or 3 steal strikes.

**Cause:** `feudSubmitGuess` (`index.html:2455-2475`) doesn't check phase. The only round-end conditions are (a) all answers revealed (`feudRevealAnswer` line 2469) or (b) 3 strikes in steal phase (`feudStrike` line 2502).

**Proposed fix:** In `feudSubmitGuess`, if `s.phase === 'steal'`, after processing the single guess: if `found` → `feudAwardRound()` (Team 2 keeps all roundPoints since `s.currentTeam` is already them); if not found → set `s.currentTeam = s.playingTeam; feudAwardRound();`. Disable the input/Guess button after one steal attempt.

> Coupled with HIGH-2: if HIGH-2 is fixed AND this rule is enforced, the failing-steal branch awards just what Team 1 earned earlier, matching real Feud.

---

### 🟢 LOW — Tied games don't announce a tie

**Games affected:** Taboo, Charades, Pictionary, Family Feud, Wheel of Fortune (all use the generic `showResults` or sort-by-score-desc).
**Symptom:** When two teams end with the same score, the team that happens to come first in the array is announced as winner. No "It's a tie!" handling.

**Repro:** Drive any of the above games to a 0-0 (or equal-score) end. Result reads `Team 1 Wins! with 0 points` even though Team 2 also has 0.

**Code:** `index.html:2763-2785` (`showResults`) — `teams[0]` is taken as the winner unconditionally.

**Proposed fix:** Before rendering the winner, check `if (teams[0].score === teams[1].score) { /* show "It's a tie!" header */ }`. Apply the same to `showFeudResults` (`index.html:2554`) and `wofShowResults` (`index.html:1417`).

---

## Per-game evidence (passing checks)

### Taboo (`index.html:598-1822`)
- 35 cards drawn in one turn → 35 unique words; deck of 191 has plenty of headroom.
- Scoring: correct +1, buzz -1, buzz at 0 clamps at 0, skip 0.
- 6 turns / 3 rounds rotation: T1 r1 → T2 r1 → T1 r2 → T2 r2 → T1 r3 → T2 r3 → results.
- Quit cleanup: `gameState` wiped, timer cleared, no overlay nodes left in DOM.

### Family Feud (`index.html:2347-2570`)
- 3-strike triggers steal phase: `currentTeam` swaps, `playingTeam` preserved, `.team-score-box.active-team` updates, status text reads `${name} can STEAL!`.
- Fuzzy match works: `"earring"` matched answer `"Earrings"` (substring match in `feudMatchGuess`, line 2371).
- Skip Survey resets state correctly via re-render (re-runs `renderFeudRound` which zeroes strikes/revealed/roundPoints/phase).
- Reveal-all path correctly awards points to currentTeam (verified Team 2 revealing all 7 pizza-topping answers → +100 to Team 2).
- Game ends at `roundsPlayed >= rounds`.

### Wheel of Fortune (`index.html:1052-1440, 2732-2757`)
- Consonant scoring: $500 × 3 N's = +$1500 to roundBank in puzzle "MAKING SNOW ANGELS".
- Vowel buy: -$250 from roundBank, no earnings from count.
- Wrong consonant: triggers transition to next player after 1.2s `safeTimeout`.
- BANKRUPT (replicated branch): zeroes the player's roundBank.
- Solve: roundBank → score, advances round, resets all roundBanks to 0.
- Puzzle cycling: 3 different puzzles ("MAKING SNOW ANGELS", "HOT CHOCOLATE", "NORTH POLE") in 3 rounds; deck has 72 puzzles.

### Codenames (`index.html:2575-2727`)
- Board: 25 cards, color counts {red:9, blue:8, neutral:7, assassin:1}.
- Red turn first; spymaster toggle works.
- Clue with num=N gives N+1 guesses.
- Correct color: count--, guesses--, turn stays; wrong opponent color: opponent count--, turn ends; neutral: turn ends, no count change.
- Win on `redLeft===0`, loss on assassin click — both render correct results screen.
- Post-`gameOver` clicks are no-ops.

### Trivia (`index.html:1827-1893`)
- 3-player rotation: P10 → P11 → P12 → P10.
- 10 unique questions in 10-question game.
- Correct +1, wrong/skip 0.
- 1500ms `safeTimeout` advance respects `quitToken`.

### Charades (`index.html:1898-1981`)
- 31 unique cards in one run; deck of 92.
- Score +1 on correct.
- Round/team rotation matches Taboo's harness; results screen renders.

### Pictionary (`index.html:1986-2129`)
- Reveal/hide word toggle works (`drawWord` element class flips between `draw-word-hidden`/`draw-word-revealed`).
- Color tool sets `gameState.drawColor`.
- 27 unique cards drawn; deck of 88.
- Same turn/round/end logic as Charades.

### Would You Rather (`index.html:2134-2228`)
- 3-voter rotation: votingPlayer 0 → 1 → 2 → 3, then results bars rendered.
- Vote tally correct (e.g. 2 'a' / 1 'b' → 67% / 33%).
- `wyrNext` resets `votingPlayer` and `currentVotes` for the next question.
- End screen renders summary ("You went through 10 dilemmas!").

### Word Association (`index.html:2233-2342`)
- 4 players, 1 starter word from `WA_STARTERS`.
- Submit advances chain, increments `roundNum`, rotates `currentPlayer`.
- `waEliminate` flips `alive=false`; rotation skips dead players (loop at line 2300).
- Game ends with `alivePlayers.length <= 1`; winner = last alive; chain shown.

---

## Setup & teardown notes

- Server: `python3 -m http.server 0 --bind 127.0.0.1 --directory public` (initial IPv6-only bind required `--bind 127.0.0.1` to be reachable by the Playwright browser).
- Console showed only one error (favicon 404) — unrelated.
- The ~1-line crash from HIGH-1 is the only console error logged from gameplay.

---

## Recommended fix order

1. **HIGH-1** (Trivia crash) — one-line fix, blocks the entire Trivia results flow.
2. **HIGH-2** (Family Feud strike-out scoring) — single-line fix in `feudAwardRound`; restores fair scoring.
3. **MED-1** (WoF round starter) — one-line init change.
4. **MED-2** (Family Feud steal "one guess") — best paired with HIGH-2 since they share the steal flow.
5. **LOW** (tie handling) — small UX polish, three call sites.
