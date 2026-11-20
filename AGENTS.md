# AGENTS.md — Mariam's 27th Birthday Smash (Padel Americano)

This guide gives coding agents full context on the architecture, domain invariants, UI direction, and database extension path for **Mariam's 27th Birthday Smash** (`index.html`).

---

## 1. Project Overview & Deployment Model

- **What it is:** A mobile-first web app for a 12-player, 11-round Padel Americano birthday tournament celebrating **Mariam's 27th Birthday**.
- **Single-File Architecture:** The entire application (HTML, CSS, inline SVG mascot, and vanilla JavaScript) lives in a single self-contained `index.html` file with **zero build step, zero npm dependencies, and zero bundlers**. Keep it single-file unless explicitly instructed otherwise.
- **Target Hosting:** Free static hosting (GitHub Pages or Cloudflare Pages) over HTTPS, accessed primarily on phones (`390px`–`430px` viewports) at the padel courts.
- **Sandbox & Offline Resilience:**
  - Always render the UI synchronously (`renderAll()`) before async storage hydration so the app never loads blank.
  - Never rely on `structuredClone()` without a fallback; use `deepClone()` (`JSON.parse(JSON.stringify(...))`).
  - Wrap all `localStorage` and `sessionStorage` access in `try/catch` blocks with in-memory fallbacks.
  - Keep `window.onerror` hooked to `#runtimeErrorBanner` so runtime issues never fail silently.
  - Keep closed overlays/modals set to `display: none; pointer-events: none;` so invisible layers never block taps.

---

## 2. Visual Direction & UI Guidelines

- **Theme:** Tactile **Neo-Retro 27th Birthday Party Zine / Comic Tournament Pass** — intentionally playful, humorous, and celebratory.
- **Avoid "AI Slop" Styling:** Do **not** switch to dark-mode neon gradients, generic glassmorphism, or sterile corporate SaaS styling.
- **Core Design Tokens (`:root` in `index.html`):**
  - Background: Vanilla Cake Cream (`#FFFDF5`) with subtle radial confetti dots.
  - Cards: Crisp White (`#FFFFFF`) with `2.5px solid #1C1917` Espresso Ink borders and hard offset comic shadows (`2px`/`4px`/`6px`).
  - Accents: Padel Lime (`#C8F542`), Party Pink (`#FF4D8D`), Birthday Gold (`#FFD23F`), Crybaby Slate (`#1E293B` / `#0F172A`).
  - Typography: `Fraunces` (display titles), `Plus Jakarta Sans` (UI controls), `JetBrains Mono` with `font-variant-numeric: tabular-nums` (scores, round pills, standings).
- **Mobile Ergonomics:**
  - Container locked to `max-width: 480px`, centered.
  - All interactive buttons, steppers, pills, and inputs must maintain a minimum `44px` tap target and `touch-action: manipulation`.

---

## 3. Tournament Domain Invariants (Do Not Break)

### A. Exact 12-Player Roster & Starting Head Starts (`buildDefaultPlayers()`)
| `id` | Name | Special Role | Default Starting Bonus |
| :--- | :--- | :--- | :--- |
| `0` | **Mariam** | `isBirthdayQueen: true` | `+15` pts |
| `1` | **Fons (Omar)** | — | `+10` pts |
| `2` | **Arwa** | Girl | `+8` pts |
| `3` | **Youmna** | Girl | `+8` pts |
| `4` | **Halla** | Girl | `+8` pts |
| `5` | **Maram** | Girl | `+8` pts |
| `6`–`11` | **Hedaya, Karim Ehab, Sasa, Osama, Sharawy, Nour** | — | `+0` pts |

- Head starts are toggleable on the Leaderboard (`state.headStartsActive`) and adjustable per player in **Crew & Faces** by Admin.

### B. Balanced 11-Round Schedule (`buildBalancedSchedule()`)
- Uses a cyclic $\mathbb{Z}_{11}$ Whist Americano construction: **11 rounds, 3 courts per round, 33 total matches**.
- Every player partners with each of the other 11 players exactly once across the 11 rounds.

### C. Strict 16-Point Match Rule (`MATCH_TOTAL_POINTS = 16`)
- Every match strictly satisfies `scoreA + scoreB === 16`.
- Unplayed matches default to `8` vs `8`.
- Score `−`/`+` steppers are auto-linked (`scoreB = 16 - scoreA`), clamped to `[0, 16]`, with boundary buttons disabled at `0` and `16`.
- Quick score presets: `12–4`, `10–6`, `9–7`, `7–9`, `6–10`, `4–12`.
- Always pass loaded, imported, remote-synced, or confirmed matches through `sanitizeMatch16(match)`.

### D. Standings & "Mariam Always #1" Grand Finale
- **Math Ranking (`computeStandings(false)`):**
  1. Total Points (`matchPoints + appliedBonus`) descending
  2. Match Point Differential (`matchPoints - pointsAgainst`) descending
  3. Wins descending
- **Birthday Law (`state.birthdayLawActive`):** Places Mariam (`isBirthdayQueen: true`) at `#1` when enabled.
- **Grand Finale Takeover (`runGrandFinaleReveal()`):**
  - Triggered from the Leaderboard or automatically offered after confirming the final unconfirmed match (Match #33).
  - Reveals mathematical standings from `#12` up to `#1`.
  - If Mariam is not already `#1`, slams the `"🚨 ARTICLE 27 INVOKED: BIRTHDAY QUEEN EXECUTIVE OVERRIDE! 🚨"` stamp and uses the Web Animations API (FLIP technique) to glide Mariam's row to `#1` while pushing everyone else down and firing confetti.

### E. 50/50 Split-Screen Match Reveal (`openSplitReveal(match)`)
- **Top Half (Winners' VIP #27 Lounge):** Padel Lime background, bouncing `celebratingPhoto` portraits, HTML5 Canvas confetti, and dynamic headlines/hype pulled from `COMEDY_POOLS` via `pickFreshLine()` (session anti-repeat).
  - Dedicated pools exist for **Mariam wins**, **Mariam loses**, **blowouts** (margin $\ge 7$ pts), **nail-biters** (margin $\le 2$ pts), and **standard wins**.
- **Bottom Half (Losers' Crybaby Corner):** Deep Slate background, wobbling grayscale `sadPhoto` portraits, HTML5 Canvas falling rain, and anti-repeat roasts.

---

## 4. Access Control & Permissions

- **Hardcoded Admin Password:** `ADMIN_PASSWORD = "MARIAM27"` (case-insensitive check).
- **Session Persistence:** Stored in `sessionStorage` under `mariam_padel_admin_unlocked` with an in-memory fallback (`adminUnlockedMemory`).
- **Role Matrix:**

| Action | View-Only (Guests) | Admin (`MARIAM27`) |
| :--- | :---: | :---: |
| Browse rounds & view read-only scoreboards | Allowed | Allowed |
| Replay confirmed match reveals (`🎬 Replay Reveal`) | Allowed | Allowed |
| View leaderboard & play Grand Finale Reveal | Allowed | Allowed |
| Export JSON backup (`💾 Export Backup`) | Allowed | Allowed |
| Upload selfie to an **empty** photo slot | Allowed | Allowed |
| Overwrite an **existing** photo slot | Blocked (prompts unlock) | Allowed |
| Edit scores, presets, lineups, confirm matches, add bonus match | Hidden / Blocked | Allowed |
| Rename players, edit starting bonuses, toggle Leaderboard rules | Hidden / Blocked | Allowed |
| Import JSON backup, Demo 32/33 seed, Reset scores | Hidden / Blocked | Allowed |

---

## 5. Data Model & Storage Abstraction (`TournamentStorageProvider`)

### A. State Shape
```json
{
  "activeRound": 1,
  "headStartsActive": true,
  "birthdayLawActive": true,
  "decreeIndex": 0,
  "players": [
    {
      "id": 0,
      "name": "Mariam",
      "isBirthdayQueen": true,
      "bonusPoints": 15,
      "celebratingPhoto": "data:image/jpeg;base64,...",
      "sadPhoto": "data:image/jpeg;base64,..."
    }
  ],
  "matches": [
    {
      "id": 1,
      "round": 1,
      "court": 1,
      "teamA": [11, 0],
      "teamB": [1, 2],
      "scoreA": 8,
      "scoreB": 8,
      "confirmed": false
    }
  ]
}
```

### B. Storage Provider Interface
All persistence is isolated behind `TournamentStorageProvider`:
- `init(defaultState)` — Loads initial state (falls back to `defaultState`).
- `saveState(state)` — Persists tournament metadata or batch updates.
- `updateMatch(matchId, matchData, fullState)` — Called **only** when **Confirm Result** is tapped.
- `updatePlayer(playerId, playerData, fullState)` — Called on name edits, bonus adjustments, or photo uploads.
- `resetAll(defaultState)` — Resets match schedule.
- `subscribe(onRemoteChange)` — Hook for real-time listeners (returns unsubscribe function).

Currently active:
```js
const storageProvider = new LocalStorageProvider("mariam_bday27_padel_16pt_v2");
```

### C. Write Behavior & Image Compression
- **Confirm-Only Match Writes:** Tapping `−`/`+` steppers, score presets, or swapping lineups mutates local memory only and adds `match.id` to `dirtyMatches` (displaying `● UNSAVED`). `storageProvider.updateMatch()` is only called when **Confirm Result** is clicked.
- **Photo Compression (`compressImageFile()`):** Center-crops uploaded camera/gallery images to `200px` by `200px` JPEG at quality `0.76` (~15–35 KB per Base64 string, ~0.5–1.0 MB total for all 24 images).

---

## 6. Cloud Database Integration Blueprint (Next Task)

When wiring a free, no-credit-card backend (recommended: **Firebase Cloud Firestore on the Spark plan** via ESM CDN imports in `index.html`, or **Supabase Free Tier**):

1. **Swap One Line:** Implement a `TournamentStorageProvider` subclass (replacing `CloudStorageProviderTemplate`) and assign it to `const storageProvider = ...`.
2. **Use Granular Documents (Not One Giant Blob):**
   - `players/{playerId}` (12 docs): Stores `name`, `bonusPoints`, `celebratingPhoto`, `sadPhoto`. Keeping photos split across 12 player docs avoids Firestore's `1 MiB` per-document limit and avoids requiring paid Firebase Cloud Storage buckets.
   - `matches/{matchId}` (33 docs): Stores `round`, `court`, `teamA`, `teamB`, `scoreA`, `scoreB`, `confirmed`. Prevents simultaneous court submissions from overwriting each other.
   - `meta/tournament` (1 doc): Stores `headStartsActive`, `birthdayLawActive`, `decreeIndex`.
3. **Protect Local Edits During Real-Time Sync (`subscribe`):**
   - When remote snapshots arrive, do **not** overwrite any match whose `id` is currently in `dirtyMatches`.
   - Skip re-rendering the Crew tab if `document.activeElement` is a `[data-player-name]` input or `[data-lineup-match]` select.
   - Sanitize every incoming remote match through `sanitizeMatch16()`.
4. **Keep `localStorage` as a Read/Write Cache:** Render cached local data immediately on boot while cloud listeners hydrate in the background (padel courts often have spotty mobile signal).
5. **Enforce Security Rules Server-Side:**
   - Attach an `adminKey` field to Admin writes and validate `request.resource.data.adminKey == "MARIAM27"` in Firestore Security Rules.
   - For guest selfie uploads on `players/{playerId}`, allow unauthenticated updates **only** when the existing `celebratingPhoto` or `sadPhoto` field is empty (`""`) and no other fields (`name`, `bonusPoints`) change.

---

## 7. Verification Checklist for Agents

Before finishing any edit to `index.html`, verify:
- [ ] App renders all 11 round pills and Round 1 matches immediately on first load (no blank screen, no console errors).
- [ ] Mobile viewport (`390px` by `844px`) has no horizontal overflow and all touch targets are at least `44px`.
- [ ] View-Only mode blocks score edits, lineup swaps, bonus edits, and overwriting existing photos, while allowing round browsing, replay reveals, leaderboard viewing, Finale Reveal, JSON export, and uploading into empty photo slots.
- [ ] Unlocking with `MARIAM27` (case-insensitive) unlocks all Admin controls; wrong passwords shake the modal and display an error.
- [ ] Every score stepper tap and preset enforces `scoreA + scoreB === 16`, disables boundary buttons at `0` and `16`, and marks the match `● UNSAVED` until **Confirm Result** is clicked.
- [ ] Confirming a match triggers the 50/50 split-screen reveal with confetti (top) and rain (bottom), and uses Mariam-specific pools when Mariam plays.
- [ ] Clicking **⚡ Demo 32/33** (Admin) and confirming Round 11 Match #33 reveals the **🏆 Unveil Final Podium!** button and plays the Article 27 FLIP animation gliding Mariam to `#1`.