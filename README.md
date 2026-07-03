# GlyphRecall

# Glyph Recall — Product Requirements Document

*Working title. A web app for memory testing, memory training, and light cognitive-rehab support.*

**Version:** 1.0 (draft)
**Status:** For review
**Owner:** (you)

---

## 1. Summary

Glyph Recall is a single-page web app that measures and exercises short-term visual memory. A person memorizes a small set of "glyphs" (a word paired with a symbol), then identifies them from a grid across escalating rounds. Between recall checks, the app inserts arithmetic problems as cognitive interference, increasing the load each round. The session continues as long as the person recalls more than 80% of their glyphs; when they drop below that, the test ends and reports how many levels they cleared.

The app serves three audiences with one mechanic: people testing their memory, people training it over repeated sessions, and people in recovery from injury affecting short-term memory who want a low-stress, repeatable exercise. All intake and result data is stored anonymously in Firebase.

---

## 2. Goals and non-goals

**Goals**

- Provide a fast, low-friction memory test that anyone can start in under 30 seconds.
- Make difficulty meaningful and selectable (Easy → Extra Hard).
- Use arithmetic interference to make recall progressively harder, the way real working-memory tasks do.
- Be calm, encouraging, and accessible — appropriate for users with cognitive or visual impairment.
- Store anonymized intake and outcome data in Firebase for the operator's own analysis.
- Make results shareable and the session easy to repeat.

**Non-goals**

- Not a medical device, diagnostic, or clinical instrument. The app must say so plainly.
- No accounts, logins, passwords, or collection of identifying information.
- No leaderboards or social graph in v1 (sharing is a copy/share-sheet action, not a network).
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
- **Target glyphs** — the subset the user must remember this session (3/5/7/9 depending on difficulty).
- **Memory check / Recall** — one round where the user picks their target glyphs out of the 5×5 grid.
- **Level** — one completed memory check. "You passed N levels" = N recall checks cleared.
- **Interference** — arithmetic questions shown one at a time between recall checks.

---

## 5. End-to-end flow

```
Intake (name, age, reason, difficulty, disclaimer)
        │
        ▼
Memorize target glyphs (user-paced)
        │
        ▼
┌──────────────────── LEVEL LOOP ────────────────────┐
│  (L−1) arithmetic interference questions, one at a │
│  time   →   Memory check (5×5 grid)                │
│                                                    │
│  pass (>80%) → next level (more interference)      │
│  fail (≤80%) → exit loop                           │
└────────────────────────────────────────────────────┘
        │
        ▼
Result: "Game Over — you passed N levels on {difficulty}"
        │
        ├─ Play again
        └─ Share result
```

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

Interference grows without a hard cap in v1; the rising count is what eventually makes recall fail. (See open question 11.2 if you'd prefer a ceiling.)

---

## 6. Functional requirements

### 6.1 Intake screen

Collected before the test begins:

- **First name** — free text, required. Used only for a friendly greeting; not an identifier.
- **Age** — integer, required, sane range (e.g., 5–120).
- **Reason for taking the test** — dropdown, required, one of:
  - Memory testing
  - Improving my memory
  - Rehabilitation / recovery support
  - Just for fun
- **Difficulty** — selectable, required: Easy (3), Medium (5), Hard (7), Extra Hard (9).
- **Disclaimer acknowledgment** — required checkbox: *"I understand Glyph Recall is for general wellness and entertainment and is not medical advice, diagnosis, or treatment."*
- **Start** — disabled until all required fields are valid and the disclaimer is acknowledged.

On Start: a record is written to Firebase (§8) and the memorize phase begins.

### 6.2 Memorize phase

- Show the target glyphs as large, clearly separated cards (symbol + word).
- **User-paced** — no countdown timer in v1. The recovering persona must not feel rushed. A "Start the test" button advances when the user is ready. (Optional study timer noted in §11.)
- The target glyphs are **not shown again** for the rest of the session.

### 6.3 Memory check (recall)

- Render all **25 glyphs in a 5×5 grid**.
- **Grid order is reshuffled every check** so users can't memorize positions instead of glyphs.
- The user selects tiles. Selection is **capped at N** (their target count) to prevent guaranteed-pass by selecting everything. A visible counter shows "Selected X of N." Tiles can be deselected.
- **Submit** is enabled once N tiles are selected.
- **Scoring:** `accuracy = correctlySelected / N`.
- **Pass rule:** advance if `accuracy > 0.80` (strictly greater — see §7).

### 6.4 Arithmetic interference

- Presented one question at a time, with a numeric input and a submit/continue action.
- **Correctness does not gate progression** — interference exists to occupy working memory, not to block the user. The answer is recorded for analytics; right or wrong, the user proceeds.
- Difficulty is intentionally modest:
  - **Addition / subtraction:** operands roughly 1–20; subtraction never goes negative.
  - **Multiplication:** single-digit times tables (2–9).
  - **Division:** generated from a clean product so the answer is a whole number.
- Type pool by level per §5.

### 6.5 Result screen

Displays exactly the required shape:

```
Game Over
You passed {N} levels on {difficulty} difficulty
Bookmark this page to try again
Share your result
```

- **Bookmark** — instructional (the browser performs the bookmark); the result page is the same URL, so bookmarking returns the user to a fresh start.
- **Share** — uses the Web Share API where available (`navigator.share`), with copy-to-clipboard fallback. Shared text: *"I passed {N} levels on {difficulty} in Glyph Recall."* plus the app URL.
- **Play again** — restarts at intake (new target glyphs each session).
- Tone is dignified and encouraging, not punitive. "Game Over" is the headline; surrounding copy frames the score as an achievement.

---

## 7. The 80% pass rule — important design note

The rule "advance while the user gets **more than** 80% correct" interacts with the glyph counts. Because the target sets are small, strict ">80%" means:

| Difficulty | Glyphs | Result needed to pass (>80%) | Misses allowed |
|---|---|---|---|
| Easy | 3 | 3 / 3 (100%) | 0 |
| Medium | 5 | 5 / 5 (100%) | 0 |
| Hard | 7 | 6 / 7 (≈86%) | 1 |
| Extra Hard | 9 | 8 / 9 (≈89%) | 1 |

So on Easy and Medium, strict ">80%" effectively requires **perfect** recall, while Hard and Extra Hard allow one miss. This is implemented exactly as specified, with the threshold isolated in a single constant so it's trivial to change.

**Recommendation:** decide deliberately whether you want perfection required on the easier modes. Switching the comparison to "**≥** 80%" would let Medium pass at 4/5 (80%) — a gentler curve that better fits the rehab persona — though Easy would still need 3/3. This is the single most consequential tuning decision in the product; it's called out here so it's a choice, not an accident.

---

## 8. Data model and privacy (Firebase)

**Principle:** fully anonymous. No auth identity beyond an anonymous session, no email, no device fingerprinting. First name and age are stored as self-described, non-identifying attributes.

**Auth:** Firebase Anonymous Authentication. Each session gets a throwaway anonymous UID.

**Firestore collections (suggested):**

`sessions/{sessionId}`
| Field | Type | Notes |
|---|---|---|
| firstName | string | greeting only |
| age | number | |
| reason | string | enum from §6.1 |
| difficulty | string | Easy/Medium/Hard/ExtraHard |
| glyphCount | number | 3/5/7/9 |
| targetGlyphIds | array | which glyphs this session |
| createdAt | timestamp | server time |
| levelsPassed | number | written at game over |
| endedAt | timestamp | written at game over |

`sessions/{sessionId}/checks/{n}` *(optional, for research depth)*
| Field | Type | Notes |
|---|---|---|
| level | number | |
| accuracy | number | 0–1 |
| passed | boolean | |
| selectedIds | array | what they picked |
| interferenceBefore | number | question count this level |

`sessions/{sessionId}/math/{n}` *(optional)*
| Field | Type | Notes |
|---|---|---|
| level | number | |
| prompt | string | e.g. "7 × 6" |
| answer | number | user's answer |
| correct | boolean | recorded, non-gating |

**Security rules (intent):** writes allowed only to a session owned by the current anonymous UID; no public reads of raw session data; aggregate/analytics access happens server-side or in the console. Final rules to be authored at implementation.

**Resilience:** if Firebase is not configured or unreachable, the app still runs end-to-end and logs locally — data collection is best-effort and never blocks the experience.

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

---

## 11. Open questions and recommendations

1. **Pass threshold** — strict ">80%" (as specified, requires perfection on Easy/Medium) vs "≥80%". 
2. **Interference ceiling** — let question count grow forever, or cap it.  
3. **Memorize timer** — keep user-paced (chosen, for the rehab persona) or add an optional study countdown for "test mode"?
4. **Math gating** — non-gating in v1 (interference only). Score math to discourage cheaters. 


---

## 12. Future enhancements (out of scope for v1)

- Operator dashboard or leaderboard for anonymized Firestore aggregates.

---

## 13. Acceptance criteria (v1)

- A user can complete intake, memorize, play the escalating loop, and reach a result without errors.
- Difficulty maps to 3/5/7/9 glyphs.
- Interference schedule matches §5, with ×/÷ appearing from Level 6.
- Pass rule is `accuracy > 0.80`, isolated in one constant.
- Grid reshuffles each check; selection is capped at N.
- Result screen matches the required four-line shape and offers play-again and share.
- A session record is written to Firebase when configured; the app still works when it isn't.
- Keyboard-only and reduced-motion users can complete the full flow.
