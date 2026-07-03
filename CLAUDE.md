# Ludo — Night Carnival

Classic 4-color Ludo as a single self-contained `index.html` (HTML + CSS + JS, no build
step, no dependencies). Deployed via GitHub Pages. Same architecture philosophy as
`rjjeutong/checkers`: everything in one file, canvas-rendered board, Web Audio sounds,
CSS-variable theming, settings modal.

## Running / developing

Open `index.html` in a browser. That's it — no install, no server required.
Debug hook: `window.__ludo` exposes `game`, `settings`, `newGame()`, `legalActions()`,
`performAction()`, `doRoll()` for console-driven testing.

## Architecture (sections inside index.html, in order)

| Section | What it does |
|---|---|
| CSS | `:root` variables (palette, fonts), layout, dice, modals, roster, toasts |
| Markup | header/banner, canvas, side panel (die + roster + rule pills), settings & win modals |
| Constants & geometry | `TRACK` (52 cells, built from segments), `START` offsets, `SAFE`/`STARS` sets, `homeCell()`, base slot coords |
| Settings | persisted to `localStorage` (`ludo-settings`) |
| Game state | `game{order, turn, tokens, dice, selDie, dieChosen, undoStack, phase, allowed, mustCapture, gen, winner}`; token = `{id, seat, slot, state, trackIdx, homeIdx, heldBy}` |
| Rules engine | `pathFor()`, `legalActionsFor()`, `isWallCell()`, `finalCaptures()`, `applySim()` — explicit-state (first arg is a tokens array) so they run on clones for simulation |
| Dispatch search | `bestOutcome()` DFS over dice-dispatch sequences on cloned state, returning lexicographic `(dicePlayed, finalCaptureFlag)`; `computeAllowed()` keeps only `{die, act}` first moves that still reach that best. Real captures are applied at end of turn by `resolveTurnCaptures()` |
| Turn engine | `idle → rolling (repeat while 6s) → dispatch (one die at a time) → animating → next turn`; all timers go through `later()`, which queues into `timers[]` — drained by the rAF loop while visible and by a Web Worker tick (250 ms) while the tab is hidden, because background tabs stop rAF and throttle `setTimeout` (a backgrounded host must never stall the online table's AI). `animate()`/`pumpTimers()` complete slides instantly while hidden. Queued timers still no-op if `game.gen` changed |
| AI | heuristic scorer over allowed pairs (`aiScore`/`aiPick`), tuned per `settings.difficulty` via `AI_PROFILES` (easy/medium/hard) |
| Sound | Web Audio oscillator synth (`tone()` + `sfx` presets), no audio files |
| Rendering | `draw()` on a continuous rAF loop; token slide animation steps cell-by-cell |
| Online | Firebase Realtime DB room sync — `drivenByMe()`/`pumpRemote()`/`netSend()` replay rolls+moves; see the Online multiplayer section |
| Input | canvas pointer hit-testing against actionable tokens |
| UI | banner, roster, dice DOM, toasts, modals |

### Board geometry

15×15 grid. `TRACK[0]` = Crimson's entry at cell (6,13); indices increase **clockwise**.
Entry offsets per seat: `START = [0, 13, 26, 39]`. A token's relative position is
`rel = (trackIdx - START[seat] + 52) % 52`, range 0–50; `rel+roll ≥ 51` turns into the
home column. Cells `homeIdx 0–3` are waypoints; the 5th colored cell (h=4) is the
**finish square** — a piece must land on it by **exact count** (overshooting dice are
illegal for that piece) and is then out of play. Safe cells (absolute): ONLY the four stars
`{8,21,34,47}` — entry squares are normal cells, so deploying from base can capture an
enemy camped on your entry.

Seats: 0 Crimson bottom-left · 1 Emerald, top-left · 2 Amber, top-right ·
3 Cobalt, bottom-right. The human picks their seat in settings (`settings.humanSeat`).
Turn order is clockwise (ascending seat index), rotated so the human rolls first.
Active seats: 1 AI → human + opposite corner, 2 AI → human + next two clockwise,
3 AI → all four.

**Per-player view rotation**: the whole board is rendered rotated in 90° steps so the
local player's corner sits bottom-left — `viewSeat()` (online → `mySeat`, offline →
`settings.humanSeat`) and `viewRot()` = `-viewSeat()*90°`. `draw()` wraps everything in a
single `translate/rotate/translate` about the board center; pointer input is mapped back
with `unrotatePoint()` (the inverse rotation). The square felt is rotation-symmetric so
90° steps never clip; the only counter-rotated element is the stack-count badge text (kept
upright). Seat 0 ⇒ no rotation, so the common case is unchanged.

**Board themes**: `THEMES` (carnival / classic / wood / neon) holds every board-drawing
color; the renderer reads `TH()` each frame, and page chrome switches via
`body[data-theme]` CSS variable overrides. Player token colors never change with theme.

## Rules

### Classic core (with house turn structure)
- **Roll-all-first**: a turn starts with a rolling phase. Rolling a 6 means roll again;
  rolling continues until a non-6 lands. All rolled values form a **dice queue**
  (e.g. 6, 6, 3). Only then does the player dispatch them — one move per die, any order,
  any tokens. A 6 die can be spent to deploy from base (or free a prisoner). No
  three-sixes penalty. Unusable dice are forfeited; the turn ends when no die can be spent.
  **Die choice & undo (UI):** if a tapped piece could move by more than one pending die,
  the input layer refuses to guess — the player must tap the die chip first (gated by
  `game.dieChosen`, reset each dispatch step). `undoMove()` pops `game.undoStack` (a
  pre-move clone pushed in `performPair` for human moves) to take a move back while dice
  remain; the stack clears at `startTurn`. Safe because captures aren't committed until
  end of turn.
- **Play every die**: the dispatch must play as many dice as possible. A choice that
  strands a die is illegal if another choice plays more — this obligation outranks the
  capture obligation.
- **Captures are mandatory** (within max-dice routes): among the dispatches that play
  the maximum number of dice, if any leaves one of your pieces **resting** on an enemy,
  you must take such a route; with several capturing routes the player picks freely.
  But capturing never costs you a playable die — if the only way to play all your dice
  carries the piece off the enemy (reordering, overshooting, or carrying on after a
  deploy), you do that and no capture happens.
- **Capture is judged on the final resting board, not on intermediate landings.**
  Passing over (or briefly touching, then carrying on past) an enemy never captures —
  a piece captures only the enemies sitting on the square where it *comes to rest* at
  the end of the dispatch (`finalCaptures()`). This is what makes the deploy case work:
  rolling 6-5 with only an enemy on your entry deploys (6) and carries the same piece on
  5 — it doesn't rest on the entry, so it doesn't capture, and both dice are played.
  Both obligations are enforced by `bestOutcome()` — a DFS that returns the lexicographic
  max `(dicePlayed, finalCaptureFlag)` over all dispatch sequences (capture flag computed
  only at the leaves, on resting positions); `computeAllowed()` keeps only first moves
  that still achieve it. The UI dims forbidden pieces and explains on tap (`blockReasons`:
  'waste'/'capture'). Captures are applied for real once, at end of turn, in
  `resolveTurnCaptures()` (called from `endTurn()`). There is no per-turn "freeze" flag:
  a piece that ends the turn on an enemy is inherently done moving.
- Tokens race clockwise around the 52-square track, then up their colored home column.
- Coming to rest on a square holding opponent tokens captures them **all** (back to their
  bases — or jail, see Prisoner rule) — entry squares included; deploying from base
  captures too (when the deployed piece stays there). There are no safe squares; a piece
  may pass through an enemy square mid-dispatch, but two colors never *rest* together (an
  enemy wall under the Wall rule blocks landing/passing outright instead).
- **Exact roll to finish**: a piece is home when it lands exactly on the last colored
  square of its corridor; a die that would overshoot cannot be used on that piece.
  Finished pieces leave the board (shown in the center triangle). Games can be played
  with **2 or 4 tokens** per player (settings). First player with all tokens home wins.

### Custom rule 1 — Prisoner (toggle, default off)
A captured token does **not** return to its owner's base — it is held caged beside the
**capturer's** base. To recover it the owner must roll a 6 and spend it freeing the
prisoner (token returns to owner's base); a later 6 brings it onto the track as normal.
On any 6 the player chooses ONE: move a token 6, deploy from base, or free a prisoner.
Toggling the rule off mid-game releases all prisoners to their owners' bases.

### Custom rule 2 — Wall (toggle, default on)
Two or more same-color tokens on one square form a wall. **No token may pass over a
wall, and opponents may not land on it** — but the owner CAN land more of their own
tokens there to grow the stack (walls of any height). Deploying from base onto your own
wall at the entry square is allowed. Walls break when the owner moves stacked tokens
off. Mixed-color squares (coexisting tokens) are not walls.

## AI

Single-pass heuristic over `legalActions` (no search): win-now (a finish that empties the
board returns `1e6` so a winning move is never passed up), capture, finish (+42), enter home
column (+26), deploy from base (+26), free a prisoner (+30), flee threatened squares,
avoid landing within dice-reach of opponents (scaled by progress at risk), form walls and
avoid breaking them when the Wall rule is on. Random jitter breaks ties.

**Gang up on the leader.** Beyond playing its own race, the AI actively tries to *stop
whoever is winning* — the more so as that player nears victory. `seatProgress(seat)` is the
fraction of a player's race completed (done tokens weighted most); `winThreat(seat)` turns
that into a "must be stopped" urgency that spikes when a player is close to home and has few
live tokens left (so the lone survivor of an almost-finished player is the prime target).
Three offensive terms read it:
- `captureValue()` — capturing a piece is worth more the more advanced it is, with a
  near-home kicker, *plus* a bonus scaled by the victim's `winThreat`. The leader gets hunted.
- `threatAt(p, absIdx)` — the offensive mirror of `dangerAt`: resting 1–6 squares **behind**
  a track enemy (so a future roll could capture it) scores points, amplified by the prey's
  `winThreat`. This is the *ambush* — the AI parks within striking range and waits for the
  front-runner to come into reach instead of always racing ahead.
- `blockValue(p, dest)` — value of resting a **wall** (2+ of its tokens) on a square: it
  denies each enemy ahead the die rolls that would land on or cross it (rolls `dist..6`),
  weighted hard toward the leader. A wall one step in front of a leader's lone piece is
  nearly a full stop, and the AI is reluctant to break a wall that's penning the leader in.
  A player walled in with no legal move is auto-skipped by `beginDispatch` (empty `allowed`
  → end turn with a "no moves left" toast) — no special-casing needed.

**Difficulty** (`settings.difficulty`, default `medium`) only tunes how *well* the AI
plays — never the dice, which are a fair `1 + floor(random*6)` for every seat. `AI_PROFILES`
scales the capture/flee/avoid/wall/hunt weights and the jitter; `hunt` (0 easy / 2 medium /
3 hard) gates the offensive threat + wall-blocking terms, so only medium and hard gang up.
`easy` also has a `blunder` chance to pick a random *legal* move outright. `aiProfile()`
reads the active profile each decision. AI pacing is snappy: ~450 ms before rolling, ~250 ms
before each move.

## Online multiplayer (Firebase)

Online play reuses the **same deterministic engine** on every client and syncs *inputs*,
not state. The only randomness is the dice, so the seat that rolls broadcasts the value and
everyone else replays it — given the same rolls + actions, every client reaches the same
board.

- **Backend:** reuses the Checkers Firebase project (its web config is public client config).
  Ludo rooms live under `rooms/ludo-<code>` — a separate keyspace from Checkers' own
  `rooms/<code>`, so nothing collides. Anonymous auth, no sign-up. `database.rules.json`
  (repo root) is the RTDB ruleset to paste into the Firebase console: ludo rooms get
  create-only room writes (host-owned), per-seat claims, membership-gated events/chat with
  shape validation and a rules-pinned `uid` field; non-`ludo-` rooms keep the old
  open-to-auth behaviour so Checkers is unaffected.
- **Drive model:** `online`, `isHost`, `mySeat` globals. `drivenByMe(seat)` = my own seat,
  plus AI seats if I'm the host. `aiDrivenByMe(seat)` gates auto-roll/auto-move;
  `myInteractiveTurn()` gates human input. Offline, all helpers fall through to the original
  behaviour (every seat driven locally).
- **Lobby:** host picks table size (2–4 active corners via `ACTIVE_SEATS`) and how many of
  those are AI; the rest are human slots. Room doc holds `seats`, `ctl` (human/ai per seat),
  `players` (seat→uid), `members` (uid→seat, what the security rules key off), `names`
  (seat→display name typed on the online screen; optional, ≤16 chars, remembered in
  localStorage `ludo-name`), shared `rules`, `status`, `game.id`. `playerName(seat)` feeds
  names into `seatName()` (roster/banner/toasts/win screen), the lobby, and chat labels —
  always HTML-escaped via `esc()` wherever they land in innerHTML. `createRoom` picks the 4-digit code with a transaction
  (retrying on collision — never overwrites a live room); joiners claim the first empty
  human seat with a per-seat transaction (`joinRoom`), so simultaneous joins can't both get
  the same seat. Host hits Start when all human slots are filled. Game-affecting settings
  (seat, AI count, tokens, difficulty, wall, prisoner) are locked while `online`
  (`lockedOnline()`) — changing them locally would desync the lockstep engines.
- **Sync:** `netSend()` pushes `{k:'roll',seat,v,uid,ts}` / `{k:'act',seat,type,t,die,uid,ts}`
  to `events/g<id>`; the `child_added` listener drops anything that fails `validEvent()`
  (shape check + the event's `uid` must be the uid that owns that seat — `roomData.players[seat]`,
  or `roomData.host` for AI seats; the rules pin `uid` to the authenticated writer), then
  ignores events for seats *I* drive (already applied locally) and queues the rest. `pumpRemote()` applies the next queued event only
  when the local engine is at the matching phase (idle for a roll, dispatch for an act), so
  animations stay ordered. `applyRemoteAct()` reconstructs the move's path via
  `legalActionsFor` (deterministic — state is in lockstep). The host drives AI seats and
  broadcasts their rolls/moves exactly like a human seat. Turn advancement needs no event —
  it's deterministic from the inputs. **Undo is offline-only** (can't unsend a move).
- **Rematch:** each human flags `rematch/<seat>`; once all human seats have flagged it the
  host bumps `game/id` and clears the flags; every client restarts on the id change.
- **Disconnect (presence):** a `.info/connected` watcher (`armPresence`/`refreshPresence`)
  arms stage-appropriate `onDisconnect` ops. **Lobby** (`status:'waiting'`): a drop only
  frees the seat (`players/<seat>` + `members/<uid>` removed) — a locked phone must never
  brick the room — and on reconnect the client re-claims its seat by transaction.
  **Playing:** a drop still flips `status:'abandoned'` and the others see "a player left".
  Leaving deliberately mirrors this: `netLeave()` frees just the seat for a lobby joiner,
  abandons the room otherwise; `newGame()` calls it too, so a local New Game can no longer
  strand the table silently. A lobby joiner whose room is closed under them gets
  "The room was closed." (`listenRoom`'s abandoned branch).
- **Chat:** `messages` is a persistent push feed (survives rematches). `attachChat()` shows
  the chat card and subscribes once per room via `limitToLast(100)`; `sendChatMsg()` pushes
  `{seat,text,ts,uid}`; incoming messages are dropped unless `seat` is a valid seat index and
  `text` is a string (re-truncated to 200 chars). `appendChatMsg()` renders each line
  labelled by the sender's color ("You" for `mySeat`), with the body added as a text node
  (no HTML injection).
- Firebase compat SDK (app/database/auth) is loaded from CDN in `<head>`. To use a dedicated
  Ludo project instead, swap `FIREBASE_CONFIG`.

## Modes roadmap

The turn engine is seat-based: `controllers[seat] = 'human' | 'ai'` and every choice is a
**serializable action object** (`{type:'enter'|'move'|'free', t:tokenId}`):

1. **vs Computer** (done) — human seat 0, AI seats from settings.
2. **Online via Firebase** (done) — 2–4 seats, any mix of humans (one client each) and
   host-driven AI; see the Online section above.
3. **Chat in online rooms** (done) — per-room `messages` feed, color-labelled by seat.
4. **Pass & play** (planned) — set multiple seats to `'human'` on one device; the input
   layer already routes by current seat.

## Conventions

- Keep everything in `index.html`; no external assets beyond Google Fonts (with system
  fallbacks).
- Rendering reads state; game logic never touches the canvas. Animations commit logical
  state first, then play, so interrupted animations can't corrupt state.
- Sounds are synthesized in `sfx` — add new ones there, never audio files.
- Bump the `build N` stamp in the footer on every push — it's how we confirm which
  version a player is running (GitHub Pages + browser caching can lag).
