# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository overview

Two self-contained, zero-dependency browser games — each is a **single HTML file** with all CSS and JavaScript inline. No build step, no package manager, no framework.

| File | Game | Live URL |
|---|---|---|
| `tictactoe.html` | Tic Tac Toe | `https://nolanle-yahoo.github.io/tictactoe/` |
| `game.html` | DEADZONE (isometric shooter) | `https://nolanle-yahoo.github.io/tictactoe/game.html` |

## Running locally

Open either file directly in a browser — no server required:

```
start msedge tictactoe.html
start msedge game.html
```

## Deployment

Every change must be committed and pushed immediately after editing:

```
git add <file>
git commit -m "description"
git push
```

GitHub Pages auto-deploys from `master` (root `/`). Changes go live at the URLs above within ~1 minute of push.

## tictactoe.html — architecture

Single `<script>` block. Key sections:

- **Theme** — CSS custom properties on `:root` / `body.light`; toggled by adding `.light` to `<body>`
- **Emoji picker** — `EMOJIS[]` constant; `buildPicker()` renders buttons; `pickEmoji()` enforces uniqueness between P1/P2
- **Game logic** — `board[]` array (9 cells), `checkWinner(b)` tests `WINS[]` patterns, `placeMove()` updates both state and DOM
- **AI (vs Computer mode)** — `minimax()` with `minimaxMove()` / `mediumMove()` / `easyMove()`; difficulty is wired in `getBestMove()`
- **Audio** — `AudioContext` created on first click (`ensureAudio()`); `playMove(player)`, `playWin()`, `playDraw()` use oscillators
- **Confetti** — `<canvas id="confetti">` overlay; `launchConfetti()` spawns 120 particles, `animateConfetti()` runs the RAF loop

## game.html (DEADZONE) — architecture

960×640 canvas, isometric top-down shooter. Single `<script>` block, sections in order:

### Coordinate system
- **World space**: float coords in `[0, GRID]` (GRID=20). All entity positions (`wx`, `wy`) live here.
- **Screen space**: pixels. Converted via `worldToScreen(wx,wy)` using `TW=64, TH=32` isometric projection: `screenX = W/2 + (wx-wy)*TW/2 - cam.x`
- **Mouse→world**: `screenToWorld(sx,sy)` inverts the above — used every frame to update `mouse.wx/wy` for aiming
- **Camera**: `cam.{x,y}` tracks `isoX/isoY` of player with 10% lerp per frame

### Game state machine
`gameState` string switches the loop between: `MENU · HELP · HISCORE · PLAYING · PAUSED · LEVEL_COMPLETE · GAME_OVER · VICTORY`

### Entity data flow
- `player` — singleton object; `resetPlayer()` reinitialises per level
- `enemies[]`, `bullets[]`, `eBullets[]`, `pickups[]`, `parts[]` — plain arrays, items spliced out when dead/expired
- Depth sorting: all drawables pushed into a temp array, sorted by `wx+wy`, then drawn — gives correct isometric overlap

### Combat
- **Gun**: `fireBullet()` pushes to `bullets[]`; `updateBulletArr()` moves, wall-tests, and hit-tests each frame
- **Sword**: arc check in world space — `Math.hypot` distance + `normalAngle()` within `±Math.PI*0.35` of `player.angle`
- **Enemy AI types**: grunt (straight pursuit), runner (zigzag via sine offset), shooter (range-stop + fire), tank (slow straight), boss (circular drift + spread shot; boss2 gains phase-2 at 50% HP)

### Level system
`LEVELS[]` array of 10 configs. Each has `waves[]` (type + count), `speedMult`, `walls` (bool), `infiniteAmmo`, and optional `boss`. `spawnWave()` staggers spawns with `setTimeout` (280 ms between each). Next wave triggers when `enemies.length < 4` or timer expires. `totalKills >= goal && enemies.length === 0` triggers level complete.

### Audio
`AudioContext` created on first click. All sound is procedural:
- `tone(type, f0, f1, dur, vol, when)` — oscillator with freq ramp
- `noise(dur, flo, fhi, vol)` — white-noise buffer through bandpass filter
- `SFX` object exposes named effects; `startAmbient()` / `stopAmbient()` manage looping drone oscs

### Walls
`getWalls(bool)` returns hardcoded `[tx,ty]` pairs. `wallAt(wx,wy)` floors coords and checks the array. `slideMove()` tries x and y independently for wall-sliding.
