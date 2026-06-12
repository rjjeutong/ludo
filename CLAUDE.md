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
| Game state | `game{order, turn, tokens, dice, selDie, phase, allowed, mustCapture, gen, winner}`; token = `{id, seat, slot, state, trackIdx, homeIdx, heldBy}` |
| Rules engine | `pathFor()`, `legalActionsFor()`, `isWallCell()`, `resolveLanding()` — explicit-state (first arg is a tokens array) so they run on clones for simulation |
| Forced-capture search | `captureReachable()` DFS over dice-dispatch sequences on cloned state; `computeAllowed()` filters `{die, act}` pairs that would forfeit a reachable capture |
| Turn engine | `idle → rolling (repeat while 6s) → dispatch (one die at a time) → animating → next turn`; all timers go through `later()`, which no-ops if `game.gen` changed (so New Game can't be corrupted by stale callbacks) |
| AI | heuristic scorer over allowed pairs (`aiScore`) |
| Sound | Web Audio oscillator synth (`tone()` + `sfx` presets), no audio files |
| Rendering | `draw()` on a continuous rAF loop; token slide animation steps cell-by-cell |
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
- **Play every die**: the dispatch must play as many dice as possible. A choice that
  strands a die is illegal if another choice plays more — this obligation outranks the
  capture obligation.
- **Captures are mandatory** (within max-dice routes): if a dispatch that plays the
  maximum number of dice can reach a capture, the player must take a capturing route;
  with several capture options the player picks freely. Capturing once lifts the
  obligation for the rest of the turn.
- **Capture-freeze**: a piece that captures (including by deploying onto an enemy at
  its entry) is spent — it cannot move again until its owner's next turn. Remaining
  dice must go to other pieces.
  Both obligations are enforced by `bestOutcome()` — a DFS that returns the
  lexicographic max `(dicePlayed, capturedFlag)` over all dispatch sequences;
  `computeAllowed()` keeps only first moves that still achieve it. The UI dims
  forbidden/frozen pieces and explains on tap (`blockReasons`: 'waste'/'capture').
- Tokens race clockwise around the 52-square track, then up their colored home column.
- Landing on a square holding opponent tokens captures them **all** (back to their
  bases — or jail, see Prisoner rule) — entry squares included; deploying from base
  captures too. Only the four **star** squares are safe, so enemy tokens can only ever
  share a square on a star (or behind the Wall rule, which blocks landing outright).
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

### Custom rule 2 — Wall (toggle, default off)
Two or more same-color tokens on one square form a wall. **No token may pass over a
wall, and opponents may not land on it** — but the owner CAN land more of their own
tokens there to grow the stack (walls of any height). Deploying from base onto your own
wall at the entry square is allowed. Walls break when the owner moves stacked tokens
off. Mixed-color squares (coexisting tokens) are not walls.

## AI

Single-pass heuristic over `legalActions` (no search): capture (+40, +12 more with
Prisoner rule), finish (+42), enter home column (+26), deploy from base (+26), free a
prisoner (+30), land on safe (+10), flee threatened squares (+9/threat), avoid landing
within dice-reach of opponents (−11/threat, scaled by progress at risk), form walls (+7)
and avoid breaking them (−3) when the Wall rule is on. Small random jitter breaks ties.

## Modes roadmap

The turn engine is seat-based: `controllers[seat] = 'human' | 'ai'` and every choice is a
**serializable action object** (`{type:'enter'|'move'|'free', t:tokenId}`), so new modes
slot in without rework:

1. **vs Computer** (done) — human seat 0, AI seats from settings.
2. **Pass & play** (planned) — set multiple seats to `'human'`; the input layer already
   routes by current seat.
3. **Online via Firebase** (planned) — add a `'remote'` controller that publishes the
   local player's actions to a Realtime Database room and replays opponents' actions;
   mirror the checkers repo's room-code + anonymous-auth approach.

## Conventions

- Keep everything in `index.html`; no external assets beyond Google Fonts (with system
  fallbacks).
- Rendering reads state; game logic never touches the canvas. Animations commit logical
  state first, then play, so interrupted animations can't corrupt state.
- Sounds are synthesized in `sfx` — add new ones there, never audio files.
- Bump the `build N` stamp in the footer on every push — it's how we confirm which
  version a player is running (GitHub Pages + browser caching can lag).
