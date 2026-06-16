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
| Turn engine | `idle → rolling (repeat while 6s) → dispatch (one die at a time) → animating → next turn`; all timers go through `later()`, which no-ops if `game.gen` changed (so New Game can't be corrupted by stale callbacks) |
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

Single-pass heuristic over `legalActions` (no search): capture, finish (+42), enter home
column (+26), deploy from base (+26), free a prisoner (+30), flee threatened squares,
avoid landing within dice-reach of opponents (scaled by progress at risk), form walls and
avoid breaking them when the Wall rule is on. Random jitter breaks ties.

**Difficulty** (`settings.difficulty`, default `medium`) only tunes how *well* the AI
plays — never the dice, which are a fair `1 + floor(random*6)` for every seat. `AI_PROFILES`
scales the capture/flee/avoid/wall weights and the jitter; `easy` also has a `blunder`
chance to pick a random *legal* move outright. `aiProfile()` reads the active profile each
decision. AI pacing is snappy: ~450 ms before rolling, ~250 ms before each move.

## Online multiplayer (Firebase)

Online play reuses the **same deterministic engine** on every client and syncs *inputs*,
not state. The only randomness is the dice, so the seat that rolls broadcasts the value and
everyone else replays it — given the same rolls + actions, every client reaches the same
board.

- **Backend:** reuses the Checkers Firebase project (its web config is public client config).
  Ludo rooms live under `rooms/ludo-<code>` — a separate keyspace from Checkers' own
  `rooms/<code>`, so the existing `rooms/$c` rule (`auth!=null` read/write) already covers
  them and nothing collides. Anonymous auth, no sign-up.
- **Drive model:** `online`, `isHost`, `mySeat` globals. `drivenByMe(seat)` = my own seat,
  plus AI seats if I'm the host. `aiDrivenByMe(seat)` gates auto-roll/auto-move;
  `myInteractiveTurn()` gates human input. Offline, all helpers fall through to the original
  behaviour (every seat driven locally).
- **Lobby:** host picks table size (2–4 active corners via `ACTIVE_SEATS`) and how many of
  those are AI; the rest are human slots. Room doc holds `seats`, `ctl` (human/ai per seat),
  `players` (seat→uid), shared `rules`, `status`, `game.id`. Host claims the first seat;
  joiners claim the next empty human seat; host hits Start when all human slots are filled.
- **Sync:** `netSend()` pushes `{k:'roll',seat,v}` / `{k:'act',seat,type,t,die}` to
  `events/g<id>`; the `child_added` listener ignores events for seats *I* drive (already
  applied locally) and queues the rest. `pumpRemote()` applies the next queued event only
  when the local engine is at the matching phase (idle for a roll, dispatch for an act), so
  animations stay ordered. `applyRemoteAct()` reconstructs the move's path via
  `legalActionsFor` (deterministic — state is in lockstep). The host drives AI seats and
  broadcasts their rolls/moves exactly like a human seat. Turn advancement needs no event —
  it's deterministic from the inputs. **Undo is offline-only** (can't unsend a move).
- **Rematch:** each human flags `rematch/<seat>`; once all human seats have flagged it the
  host bumps `game/id` and clears the flags; every client restarts on the id change.
- **Disconnect:** each client sets `onDisconnect().update({status:'abandoned'})`; any drop
  flips the room and the others show "a player left — the game ended".
- **Chat:** `messages` is a persistent push feed (survives rematches). `attachChat()` shows
  the chat card and subscribes once per room; `sendChatMsg()` pushes `{seat,text,ts}`;
  `appendChatMsg()` renders each line labelled by the sender's color ("You" for `mySeat`),
  with the body added as a text node (no HTML injection).
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
