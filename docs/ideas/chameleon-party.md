# Chameleon Game — PRD & Design Doc
*(First module on the `mafia_games` platform)*

## Goal
Ship a web app for playing a Chameleon-style social-deduction party game
with friends — one secret word/picture, one or more chameleons who don't
know it, a single device passed around the table — as the first title
under `mafia_games`, the owner's broader home for mafia-style party
games.

## Problem
- **Install friction kills casual game-night adoption.** Apps like
  Wavelength are well-reviewed but their top requested fix is better
  pass-and-play, because asking a whole table to install something before
  a single round is real friction. A no-login, no-install web link
  removes that barrier entirely.
- **Existing digital Chameleon options are single-purpose and native.**
  The board game and its app support only one chameleon and text-only
  words, limiting replay value for bigger or younger groups.
- **Setup and handoff friction, not the core loop, is where these games
  break at the table** — typing names, remembering who goes first,
  avoiding accidental peeking during handoff.

## Users
- **Party/game-night groups** (4-8 friends/family) sharing one phone —
  primary audience.
- **Hosts/organizers** who want to open one link and start playing with
  zero setup asked of guests.
- **Families with kids** — served by picture-based simple mode and large
  touch targets.
- **Larger groups (8+)** — served by configurable multiple chameleons.
- **Repeat players** — want custom names/avatars and a leaderboard that
  persists across rounds on the same device.

## Chosen approach
**Alternative A — "Single Link, Single Device."** A static, backend-free
web app. The device it's opened on is physically passed around for every
private moment (secret reveal); no accounts, no room codes, no
real-time server.

**Decision rationale:** Of the three alternatives explored (A: single
device/no backend, B: hybrid passed-device + phone-join, C: full
room-code platform with everyone on their own device), A was chosen to
ship first because:
- It's the lowest effort and lowest operational risk path — no real-time
  server, no room-lifecycle/reconnection engineering, no ongoing hosting
  cost — letting the team validate whether people actually want to play
  a digital Chameleon at all before investing in shared platform
  infrastructure.
- It's the most faithful to the physical game's tactile identity (one
  object passed hand to hand), which was explicitly called out as part
  of what makes the original fun, rather than a departure from it (as C
  would be, moving everyone to their own screen).
- It still solves the *specific* install-friction problem this doc set
  out to fix (it's a link, not an app-store install) without requiring
  the specific *multi-device* friction problem (Alternative C's full
  fix) to be solved on day one.
- The explicit tradeoff being accepted: this alternative does **not**
  build the shared room/session/leaderboard infrastructure that `B` or
  `C` would have produced for `mafia_games`'s future titles. That work is
  deliberately deferred until a second game is greenlit and justifies
  the investment (see Non-goals).

## Design

### Screens & flow
1. **Home screen.** Single "New Game" call to action. If a prior
   player roster + leaderboard exist in local storage, offer "Continue
   with [N] players" to skip re-entering names.
2. **Game setup screen.** Steppers for player count (3-12) and
   chameleon count (1 up to `max(1, floor(players / 3))`, enforced live
   as the player count changes), a category picker (e.g. Animals, Food,
   Movies, Places, Random), and a "Simple mode (pictures)" toggle.
3. **Player setup screen.** One row per player slot: text input for
   name (required, 1-20 chars) and a tap-to-open avatar picker (preset
   icon/color gallery — no photo upload in v1). Order here is not the
   turn order; turn order and starting player are randomized later.
4. **Reveal loop (one screen per player, sequential).**
   - Interstitial: "Pass the device to **[Avatar] [Name]**" with a
     single "I'm ready" button that *that* player must tap themselves —
     this small deliberate step forces a real handoff moment rather than
     trusting the previous player to look away.
   - Reveal card: tap-and-hold to show the secret word (or picture, in
     simple mode) or a "You're the Chameleon" card; releasing hides it
     immediately. A "Got it — pass to next" button (disabled until the
     card has been revealed at least once) advances to the next player.
   - No back navigation is possible once a player has advanced — the
     previous player's card can never be re-shown on that device.
5. **Starting player announcement.** After the last player's reveal,
   one screen announces the randomly chosen starting player
   ("**[Avatar] [Name]** goes first!") and a short reminder: "Go around
   the table, everyone gives one clue, then discuss and decide out loud
   who you think the chameleon(s) are." A "Ready to reveal" button
   proceeds only when the table taps it (no timer forced).
6. **Reveal & outcome screen.** Shows each chameleon's name/avatar.
   Below that, for each chameleon, two host-operated toggles the table
   fills in by consensus after their own discussion: **Caught /
   Not caught**, and if caught, **Guessed the secret correctly? Y/N**.
   This is the only "data entry" step in the whole app — everything else
   is either random generation or tap-to-reveal.
7. **Round scoring.** Computed automatically from step 6 (rules below)
   and applied to the on-device leaderboard.
8. **Leaderboard screen.** All current players, cumulative score,
   sorted descending. Persisted in local storage on this device/browser
   only; a "Reset scores" action is available and requires a confirm
   tap.
9. **Play again.** Re-randomizes secret, chameleon assignment, and
   starting player; keeps the same player roster and cumulative scores;
   returns to the reveal loop (step 4).

### Default scoring rules (v1)
- If **all** chameleons in the round are marked "Caught": every
  non-chameleon player gets **+1**.
- Each individual chameleon marked "Not caught" gets **+1** for that
  chameleon.
- A chameleon marked "Caught" who is also marked "Guessed the secret
  correctly" gets **+1** (their catch is offset — this mirrors the
  physical game's comeback mechanic).
- These are defaults, not hardcoded assumptions the owner has already
  validated — see Open questions.

### Edge cases
- **Player/chameleon count validation:** chameleon count is clamped
  live to `1 ≤ chameleons ≤ max(1, floor(players/3))`; reducing player
  count below what the current chameleon count allows auto-reduces
  chameleon count with a brief inline notice.
- **Repeat words:** the random word/picture picker excludes the
  previous 1-2 rounds' picks within the same category to avoid
  back-to-back repeats in a session.
- **Accidental re-reveal:** addressed structurally by disallowing back
  navigation in the reveal loop (see step 4) rather than relying on a
  warning dialog.
- **Player drops out mid-setup:** removing a player is only possible
  from the player setup screen (step 3), before the reveal loop starts;
  once reveal has begun, the round must be completed or abandoned
  (returning to Home discards the in-progress round but not the
  leaderboard).
- **Small screens / simple mode legibility:** pictures render at a
  minimum size that fills most of the reveal card on a 360px-wide
  viewport; no horizontal scrolling or clipped controls at that width.
- **Offline use:** the app is installable as a PWA and fully functional
  offline after first load, since there is no server-side call in this
  architecture (see Non-goals).
- **Local storage cleared/new device:** leaderboard and roster are
  expected to be lost if the browser's storage is cleared or the app is
  opened on a different device — this is an accepted, by-design
  limitation of the single-device architecture, not a bug.

## Requirements
- **FR1:** Host can configure player count from 3 to 12 before starting
  a round.
- **FR2:** Host can configure chameleon count, live-constrained to
  `1 ≤ chameleons ≤ max(1, floor(players/3))`.
- **FR3:** Each player has a required, non-empty custom name (max 20
  chars) and a required avatar chosen from a preset gallery.
- **FR4:** A "Simple mode" toggle switches the secret's content type
  from text word to picture/icon for the whole round.
- **FR5:** On round start, the system randomly selects (a) the secret
  word/picture from the chosen category, excluding the last 1-2 rounds'
  picks in that category, (b) which player(s) are chameleon(s), uniform
  random without replacement, and (c) the starting player, uniform
  random among all players.
- **FR6:** The reveal screen shows secret content for exactly one player
  at a time, requires an explicit "ready" tap from the incoming player
  before content is shown, hides content on release, and cannot be
  navigated backward to a previous player's card.
- **FR7:** After all players complete reveal, the app displays the
  randomly chosen starting player before free play/discussion begins.
- **FR8:** The host manually triggers a "Reveal chameleon(s)" screen
  after table discussion; the app does not auto-reveal on a timer.
- **FR9:** For each chameleon, the app records a "Caught/Not caught"
  outcome and, if caught, a "guessed correctly" outcome, entered by the
  table after step FR8; the app computes round scores per the Design
  section's default rules and updates the on-device leaderboard.
- **FR10:** The leaderboard persists across app reloads on the same
  browser/device via local storage, and offers a confirm-gated reset.
- **FR11:** "Play again" re-randomizes the round (FR5) while preserving
  the existing player roster and cumulative leaderboard scores.
- **FR12:** The app requires no login, no account, and makes no
  server-side network calls at runtime; it is installable as a PWA and
  fully usable offline after first load.
- **FR13:** The app is usable without horizontal scrolling or clipped
  controls at viewport widths of 360px and up.

## Success metrics
- **Session completion rate:** % of rounds that reach a recorded outcome
  (step 6) vs. abandoned — target ≥90%.
- **Rounds per session:** average number of "Play again" loops per
  session — target ≥4, as a proxy for replayability/fun.
- **Return usage:** % of sessions on a device with an existing local
  leaderboard (i.e., a repeat open) within 30 days — target directional
  growth; exact benchmark TBD after a baseline period.
- **Simple mode usage:** % of sessions with picture mode enabled —
  informs whether the v1 emoji/icon asset set is sufficient or needs a
  richer picture pack investment later.
- **Qualitative signal:** an optional single post-round question ("would
  you play this instead of the physical game?") — majority-positive is
  the bar for treating this as validated enough to invest in Alternative
  B/C's platform infrastructure next.
- **Caveat:** FR12 rules out server-side calls, so any of the above
  requires either (a) accepting no usage data for v1, or (b) adding a
  minimal, privacy-respecting client analytics snippet as an explicit,
  separate decision — this doc does not resolve that tension (see Open
  questions).

## Non-goals
- No native mobile app in this phase — web only.
- No user accounts or login.
- No cross-device sync of players/leaderboard — single device only, by
  design of Alternative A.
- No room codes or real-time multiplayer (Alternative B/C) — explicitly
  deferred, not built incrementally toward here.
- No licensed or custom-illustrated picture packs — v1 simple mode uses
  an emoji/icon-based asset set.
- No photo-upload avatars — preset avatar gallery only.
- No in-app digital voting or clue-text capture — clue-giving and
  accusation remain spoken/table-driven; the app only records the final
  outcome for scoring.
- No shared `mafia_games` platform infrastructure (rooms, cross-game
  identity, cross-game leaderboard) build-out in this phase.

## Open questions
- Are the default scoring rules in the Design section the intended house
  rules, or should scoring be configurable/toggleable per group?
- Analytics: accept no usage data for v1 (consistent with FR12's
  no-backend constraint), or add a minimal client-side analytics
  snippet to measure the Success metrics above? These are in tension and
  need an explicit decision.
- Simple-mode picture/category asset list is a content task, not decided
  in this doc — needs the actual category and asset list finalized
  before build.
- Exact module layout inside `mafia_games` (e.g. `games/chameleon/` vs.
  another structure) is an implementation detail for whoever scaffolds
  the repo, not decided here.
- This architecture is intentionally not a stepping stone to Alternative
  B or C — if/when a second `mafia_games` title or room-code play is
  greenlit, expect a meaningful rework rather than an incremental
  extension of this codebase.
