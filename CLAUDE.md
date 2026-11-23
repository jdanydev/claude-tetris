# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project

A vanilla JavaScript falling-block puzzle game rendered on HTML5 Canvas. There are no dependencies, no `package.json`, no build step, no linter and no test suite — just `index.html`, `style.css`, and `game.js`. User-facing text (README, UI labels, overlay messages) is in Spanish; keep new UI strings consistent with that.

## Running

Serve the directory statically and open `http://localhost:8000`:

```bash
python3 -m http.server 8000
```

Opening `index.html` directly in a browser also works. Verification is manual: play the game in the browser.

## Architecture

All game logic lives in `game.js`, a single non-module script loaded at the end of `<body>` that runs `init()` immediately. State is held in top-level `let` globals (`board`, `current`, `next`, `score`, `lines`, `level`, `paused`, `gameOver`, `dropInterval`, `animId`, …) that every function reads and mutates directly.

Key conventions that span multiple functions:

- **Cell values double as piece type and color index.** `board` is a `ROWS × COLS` matrix of `0` (empty) or `1–7`. Each shape in `PIECES[type]` is filled with its own type number, so `merge()` copies those values straight onto the board and `drawBlock()` looks them up in `COLORS[index]`. `PIECES`, `COLORS`, and `randomPiece()` (which picks `1..7`) must stay aligned.
- **Pieces** are `{ type, shape, x, y }` where `shape` is a square matrix (copied in `randomPiece()` so rotation never mutates `PIECES`). Rotation is `rotateCW` (transpose + reverse); `tryRotate` applies simple horizontal wall kicks `[0, -1, 1, -2, 2]` — this is not SRS.
- **`collide(shape, ox, oy)`** is the single source of truth for legality (walls, floor, locked cells; cells above row 0 are allowed). Movement, rotation, ghost projection (`ghostY`), drops, and game-over detection all go through it.
- **Piece lifecycle:** `lockPiece()` → `merge()` → `clearLines()` (updates lines/score/level/speed) → `spawn()` (promotes `next` to `current`; if it immediately collides → `endGame()`).
- **Game loop:** `loop(ts)` via `requestAnimationFrame` accumulates `dropAccum` and applies gravity once it exceeds `dropInterval`, then redraws everything each frame. Pause and game over stop the loop with `cancelAnimationFrame(animId)`; resuming resets `lastTime` and calls `loop` again.
- **Scoring/speed:** `LINE_SCORES[cleared] * level`; soft drop +1 per row, hard drop +2 per row; `level = floor(lines / 10) + 1`; `dropInterval = max(100, 1000 - (level - 1) * 90)`.

## Coupled values

- `COLS × BLOCK` and `ROWS × BLOCK` must match the `<canvas id="board">` `width`/`height` in `index.html` (currently 300 × 600).
- `drawNext()` assumes a 4×4 grid at 30px, matching `<canvas id="next-canvas">` at 120 × 120.
- `game.js` grabs DOM elements by id (`board`, `next-canvas`, `score`, `lines`, `level`, `overlay`, `overlay-title`, `overlay-score`, `restart-btn`); renaming any id in `index.html` requires updating `game.js`.
- The overlay is shown/hidden by toggling the `hidden` class defined in `style.css`.
