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
home column (`homeIdx 0–4`), and the 6th home step finishes the token (**overshoot is
allowed** — no exact-roll requirement). Safe cells (absolute): ONLY the four stars
`{8,21,34,47}` — entry squares are normal cells, so deploying from base can capture an
enemy camped on your entry.

Seats (fixed): 0 Crimson = human, bottom-left · 1 Emerald, top-left · 2 Amber, top-right ·
3 Cobalt, bottom-right. Turn order is seat order (clockwise). Active seats by AI count:
1 → `[0,2]`, 2 → `[0,1,2]`, 3 → all four.

## Rules

### Classic core (with house turn structure)
- **Roll-all-first**: a turn starts with a rolling phase. Rolling a 6 means roll again;
  rolling continues until a non-6 lands. All rolled values form a **dice queue**
  (e.g. 6, 6, 3). Only then does the player dispatch them — one move per die, any order,
  any tokens. A 6 die can be spent to deploy from base (or free a prisoner). No
  three-sixes penalty. Unusable dice are forfeited; the turn ends when no die can be spent.
- **Captures are mandatory**: if ANY way of dispatching the queue leads to a capture —
  including chaining several dice onto one token — the player must take a capturing
  route. Dispatches that would forfeit every reachable capture are illegal (the UI dims
  them and explains). Capturing **once** lifts the obligation for the rest of the turn;
  with several capture options the player picks freely.
- Tokens race clockwise around the 52-square track, then up their colored home column.
- Landing on a **single** opponent token captures it (back to its base — or jail, see
  Prisoner rule) — entry squares included; deploying from base captures too. Only the
  four **star** squares are safe. Landing on a square with 2+ opponent tokens captures
  nothing (tokens coexist).
- No exact roll needed to finish. Games can be played with **2 or 4 tokens** per player
  (settings). First player with all tokens home wins; game ends.

### Custom rule 1 — Prisoner (toggle, default off)
A captured token does **not** return to its owner's base — it is held caged beside the
**capturer's** base. To recover it the owner must roll a 6 and spend it freeing the
prisoner (token returns to owner's base); a later 6 brings it onto the track as normal.
On any 6 the player chooses ONE: move a token 6, deploy from base, or free a prisoner.
Toggling the rule off mid-game releases all prisoners to their owners' bases.

### Custom rule 2 — Wall (toggle, default off)
Two or more same-color tokens on one square form a wall. **No token — opponents' or your
own — may pass over or land on a wall square** (so a 3-stack can't be formed while the
rule is on). Walls break only when the owner moves a stacked token off. Mixed-color
squares (coexisting tokens) are not walls.

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
