# GlyphRecall

# Glyph Recall — Product Requirements Document

*Working title. A web app for memory testing, memory training, and light cognitive-rehab support.*

**Version:** 2.0 (draft)
**Status:** For review
**Owner:** (you)

---

## 1. Summary

Glyph Recall is a single-page web app that measures and exercises short-term visual memory. A person memorizes a small set of "glyphs" (a word paired with a symbol), then identifies them from a grid across escalating rounds. Between recall checks, the app inserts arithmetic problems as cognitive interference, increasing the load each round. The session continues as long as the person recalls more than 80% of their glyphs; when they drop below that, the test ends and reports how many levels they cleared — except in Progressive modes, which can also be *won* outright (§5.1).

The app serves three audiences with one mechanic: people testing their memory, people training it over repeated sessions, and people in recovery from injury affecting short-term memory who want a low-stress, repeatable exercise. All intake and result data is stored anonymously in Firebase.

---

## 2. Goals and non-goals

**Goals**

- Provide a fast, low-friction memory test that anyone can start in under 30 seconds.
- Make difficulty meaningful and selectable, with Progressive modes as the primary, default experience.
- Use arithmetic interference to make recall progressively harder, the way real working-memory tasks do.
- Be calm, encouraging, and accessible — appropriate for users with cognitive or visual impairment.
- Store anonymized intake and outcome data in Firebase for the operator's own analysis.
- Make results shareable and the session easy to repeat.

**Non-goals**

- Not a medical device, diagnostic, or clinical instrument. The app must say so plainly.
- No accounts, logins, passwords, or collection of identifying information.
- No leaderboards or social graph in v1 (sharing is a copy/share-sheet action, not a network; personal bests are local to the browser, §8.1).
- No multi-device sync of personal history in v1.

---

## 3. Users

| Persona | Why they're here | What they need |
|---|---|---|
| **The curious tester** | "How good is my memory?" | A quick result, a difficulty they can brag about |
| **The trainer** | Wants to improve over weeks | Repeatability, a clear score to beat |
| **The recovering user** | Rehab after injury affecting short-term memory | Low stress, no time pressure, large targets, plain language, dignity |

The recovering user is the most demanding persona and sets the bar for tone, pacing, and accessibility. Design to them and the other two are served automatically.

---

## 4. Key terms

- **Glyph** — a memorable unit shown as a **symbol + word** (e.g., 🦓 *Zebra*, 🥫 *Can*). The word/symbol pairing aids recognition and accessibility (dual coding).
- **Glyph pool** — the fixed set of **25** glyphs that fills the recall grid.
- **Target glyphs** — the subset the user must remember this session. Fixed-count modes use 3/5/7/9; Progressive modes grow the target set over time (§5.1).
- **Memory check / Recall** — one round where the user picks their target glyphs out of the 5×5 grid.
- **Level** — one completed memory check. "You passed N levels" = N recall checks cleared.
- **Interference** — arithmetic questions shown one at a time between recall checks.
- **Progressive mode** — a difficulty that starts with a small target set and adds more glyphs as levels increase, until a win condition is met (§5.1).

---

## 5. End-to-end flow

```
Intake (name, age, reason, difficulty, disclaimer notice)
        │
        ▼
Memorize target glyphs (user-paced)
        │
        ▼
┌──────────────────── LEVEL LOOP ────────────────────┐
│  (L−1) arithmetic interference questions, one at a │
│  time   →   Memory check (5×5 grid)                │
│                                                    │
│  pass (>80%), win condition met  → Win screen      │
│  pass (>80%), win condition not met → next level   │
│  fail (≤80%) → exit loop, Game Over                │
└────────────────────────────────────────────────────┘
        │
        ▼
Result: "Game Over — you passed N levels on {difficulty}"
   or:   "You Won! — {target} glyphs mastered on {difficulty}"
        │
        ├─ Play again
        └─ Share result
```

### 5.1 Difficulty modes

Two Progressive modes ship as the selectable difficulties; Progressive is pre-selected by default. Fixed-count modes (Easy/Medium/Hard/Extra Hard) remain fully implemented but are commented out of the intake form for now — kept for internal testing and easy re-enabling, not deleted.

| Mode | Status | Glyphs at start | Glyphs added | Win condition |
|---|---|---|---|---|
| **Progressive** | Live, default | 3 | +1 every 3 levels | 24 glyphs mastered |
| **More Progressive** | Live | 6 | +2 every 3 levels | 24 glyphs mastered |
| Easy | Commented out | 3 (fixed) | — | none (Game Over only) |
| Medium | Commented out | 5 (fixed) | — | none (Game Over only) |
| Hard | Commented out | 7 (fixed) | — | none (Game Over only) |
| Extra Hard | Commented out | 9 (fixed) | — | none (Game Over only) |

Both Progressive modes currently win at the same glyph count (24) by design choice, not by a shared/hardcoded constant — each mode carries its own win-condition value, so future modes are free to use a different target or a different kind of win condition entirely (e.g. a level count) without disturbing existing modes.

Since the glyph pool is fixed at 25 (§10), a 24-glyph win leaves exactly one non-target glyph in the recall grid at the winning check. Any future increase to the win target requires growing the glyph pool first.

### Interference schedule

Each level adds one more arithmetic question before its memory check. Multiplication and division are introduced at Level 6.

| Level | Interference questions before recall | Question types |
|---|---|---|
| 1 | 0 | — |
| 2 | 1 | + − |
| 3 | 2 | + − |
| 4 | 3 | + − |
| 5 | 4 | + − |
| 6 | 5 | + − × ÷ |
| 7 | 6 | + − × ÷ |
| N | N − 1 | + − × ÷ (from L6) |

Interference grows without a hard cap in v1; the rising count is what eventually makes recall fail on fixed-count and (pre-win) Progressive modes alike. (See open question 11.2 if you'd prefer a ceiling.)

---

## 6. Functional requirements

### 6.1 Intake screen

Collected before the test begins:

- **First name** — free text, required (max 10 characters). Used only for a friendly greeting; not an identifier.
- **Age** — integer, required, range 10–99.
- **Why are you training?** — required, single-select tile group (same visual pattern as Difficulty, not a dropdown), one of:
  - Improving my memory
  - Rehab & Recovery
  - Just for fun
- **Difficulty** — required, single-select tile group: Progressive (default, pre-selected) and More Progressive are shown; Easy/Medium/Hard/Extra Hard exist in code but are commented out of the form (§5.1).
- **Disclaimer** — a static, non-interactive notice (not a checkbox to acknowledge): *"Glyph Recall is for general wellness and entertainment, and is not medical advice, diagnosis, or treatment."* It does not gate the Start button.
- **Start** — disabled until name, age, reason, and difficulty are all valid. (No disclaimer checkbox to satisfy, since it's no longer interactive.)

On Start: the memorize phase begins immediately. The Firebase write happens once, at the end of the session (§8), not on Start.

### 6.2 Memorize phase

- Show the target glyphs as large, clearly separated cards (symbol + word).
- **User-paced** — no countdown timer in v1. The recovering persona must not feel rushed. A "Start the test" button advances when the user is ready. (Optional study timer noted in §11.)
- In Progressive modes, newly-added glyphs are re-shown (highlighted as new) each time the target set grows, via an interstitial "new glyph" screen before the next memory check.
- The target glyphs are **not shown again** outside these moments for the rest of the session.

### 6.3 Memory check (recall)

- Render all **25 glyphs in a 5×5 grid**.
- **Grid order is reshuffled every check** so users can't memorize positions instead of glyphs.
- The user selects tiles. Selection is **capped at N** (their current target count) to prevent guaranteed-pass by selecting everything. A visible counter shows "Selected X of N." Tiles can be deselected.
- **Submit** is enabled once N tiles are selected.
- **Scoring:** `accuracy = correctlySelected / N`.
- **Pass rule:** advance if `accuracy > 0.80` (strictly greater — see §7).
- **Win check:** on a pass, if the mode's win condition is met (currently: target glyph count has reached the mode's threshold), the session ends in a win instead of advancing to the next level.

### 6.4 Arithmetic interference

- Presented one question at a time, with a numeric input and a submit/continue action.
- **Correctness does not gate progression** — interference exists to occupy working memory, not to block the user. The answer is recorded for analytics; right or wrong, the user proceeds.
- Difficulty is intentionally modest:
  - **Addition / subtraction:** operands roughly 1–20; subtraction never goes negative.
  - **Multiplication:** single-digit times tables (2–9).
  - **Division:** generated from a clean product so the answer is a whole number.
- Type pool by level per §5.

### 6.5 Result screen

Two shapes, depending on outcome:

**Game Over (fail):**
```
Game Over
You passed {N} levels on {difficulty} difficulty
Bookmark this page to try again
Share your result
```

**You Won (Progressive win condition met):**
```
You Won!
{target} glyphs mastered on {difficulty} difficulty
[note thanking the player, with a mailto link to the team]
Bookmark this page to try again
Share your result
```

- **Bookmark** — instructional (the browser performs the bookmark); the result page is the same URL, so bookmarking returns the user to a fresh start.
- **Share** — uses the Web Share API where available (`navigator.share`), with copy-to-clipboard fallback.
  - Fail: *"I passed {N} levels on {difficulty} in Glyph Recall."*
  - Win: *"I won Glyph Recall — mastered all {target} glyphs in {difficulty} mode!"*
  - Both include the app URL.
- **Play again** — restarts at intake (new target glyphs each session; a fresh session ID is generated).
- **Personal bests** — best runs per difficulty are kept in browser `localStorage` (not Firebase) and shown on the result screen; this is local history, not a synced or shared leaderboard.
- Tone is dignified and encouraging, not punitive. "Game Over" is the headline on a loss; surrounding copy frames the score as an achievement either way.

---

## 7. The 80% pass rule — important design note

The rule "advance while the user gets **more than** 80% correct" interacts with the glyph counts. Because the target sets are small, strict ">80%" means:

| Difficulty | Glyphs | Result needed to pass (>80%) | Misses allowed |
|---|---|---|---|
| Easy | 3 | 3 / 3 (100%) | 0 |
| Medium | 5 | 5 / 5 (100%) | 0 |
| Hard | 7 | 6 / 7 (≈86%) | 1 |
| Extra Hard | 9 | 8 / 9 (≈89%) | 1 |
| Progressive / More Progressive | grows over time (3→24 / 6→24) | scales with current target count | scales with current target count |

So on Easy and Medium, strict ">80%" effectively requires **perfect** recall, while Hard and Extra Hard allow one miss. On Progressive modes, how many misses are tolerated grows as the target set grows. This is implemented exactly as specified, with the threshold isolated in a single constant so it's trivial to change.

---

## 8. Data model and privacy (Firebase)

**Principle:** fully anonymous. No auth identity beyond an anonymous session, no email, no device fingerprinting. First name and age are stored as self-described, non-identifying attributes.

**Resilience:** if Firebase is not configured or unreachable, the app still runs end-to-end and logs locally — data collection is best-effort and never blocks the experience.

### 8.1 Local personal bests

Independent of Firebase: the browser keeps a small `localStorage` record of the best (highest `levelsPassed`) runs per difficulty, capped at a handful of entries per difficulty. This powers the "Your best" list on the result screen. It's per-browser, not synced, and not part of the anonymized Firebase dataset — clearing browser storage clears it.

---

## 9. Non-functional requirements

- **Accessibility (priority):** WCAG AA contrast; full keyboard operation; visible focus rings; tap targets ≥44px; screen-reader labels on glyph tiles and selection state; `prefers-reduced-motion` respected.
- **Calm by design:** no harsh reds, no alarms, no surprise timers; gentle motion; supportive copy.
- **Responsive:** works from ~320px phones up to desktop; the 5×5 grid stays usable on small screens.
- **Performance:** single page, no heavy assets (symbols are Unicode/emoji), instant transitions.
- **Resilience:** graceful degradation if fonts or Firebase fail to load.
- **Privacy:** no third-party trackers; only Firebase, only anonymized.

---

## 10. Glyph pool (25)

A fixed, visually distinct set. Each is symbol + word:

Zebra, Can, Apple, Anchor, Bell, Cactus, Crown, Diamond, Drum, Feather, Guitar, Hammer, Hut, Key, Leaf, Moon, Mushroom, Owl, Pencil, Rocket, Star, Sun, Tree, Umbrella, Whale.

Selected for low confusability (no near-duplicates) and clear single-word labels.

This is also the hard ceiling on any win condition expressed as a glyph count — see §5.1.

---

## 11. Open questions and recommendations

1. **Pass threshold** — strict ">80%" (as specified, requires perfection on Easy/Medium) vs "≥80%".
2. **Interference ceiling** — let question count grow forever, or cap it.
3. **Memorize timer** — keep user-paced (chosen, for the rehab persona) or add an optional study countdown for "test mode"?
4. **Math gating** — non-gating in v1 (interference only). Score math to discourage cheaters.
5. **Fixed-count modes** — Easy/Medium/Hard/Extra Hard are implemented and functional but hidden from the intake form. Decide whether they return to the UI, stay as an internal/testing-only path, or get removed outright once the Progressive modes are validated.
6. **Progressive win target above 24** — since the glyph pool is capped at 25, raising the win threshold on any Progressive mode requires growing the glyph pool first (§10).

---

## 12. Future enhancements (out of scope for v1)

- Operator dashboard or leaderboard for anonymized Firestore aggregates.
- Additional Progressive-family modes with their own start/step/win-condition tuning (the mode config is already structured to support this per-mode, without a shared/universal win rule).

---

## 13. Acceptance criteria (v1)

- A user can complete intake, memorize, play the escalating loop, and reach a result (Game Over or Win) without errors.
- Progressive is the pre-selected default difficulty; More Progressive is selectable alongside it.
- Linear modes (Easy/Medium/Hard/Extra Hard) remain functional in code, hidden from users.
- Interference schedule matches §5, with ×/÷ appearing from Level 6.
- Pass rule is `accuracy > 0.80`, isolated in one constant.
- Grid reshuffles each check; selection is capped at the current target count N.
- Both Progressive modes win at 24 glyphs mastered, each via its own per-mode win-condition value (not a shared constant).
- Result screen shows the correct shape for the outcome (Game Over vs Won) and offers play-again and share.
- A session record is written to Firebase on game-over; the app still works when Firebase is unreachable.
- Keyboard-only and reduced-motion users can complete the full flow.
