# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project

Vanilla JS Tetris using HTML5 Canvas. No dependencies, no build step, no `package.json`.

## Running the game

Just open `index.html` in a browser, or serve it statically:

```bash
python3 -m http.server 8000   # or: npx serve .
```

There is no test suite and no linter configured.

## Architecture

All game logic lives in `game.js` (~300 lines, single global scope, no modules). `index.html` only provides the DOM/canvas shell; `style.css` is purely cosmetic.

- **Board model**: a `ROWS × COLS` (20×10) matrix. Each cell is `0` (empty) or a color index `1–7` identifying which piece type occupies it.
- **Pieces**: defined as square matrices in `PIECES`. Rotation is done via `rotateCW` (transpose + row-reverse), not by storing pre-rotated states.
- **Collision detection**: `collide(shape, ox, oy)` is the single choke point used everywhere — movement, rotation, ghost-piece projection, and the spawn check that triggers game over.
- **Wall kicks**: `tryRotate` calls `rotateCW` then retries the rotated shape at x-offsets `[0, -1, 1, -2, 2]` via `collide`, keeping the first that fits.
- **Game loop**: `requestAnimationFrame`-driven `loop(ts)`. Accumulates elapsed time into `dropAccum`; once it exceeds `dropInterval`, the piece drops a row (or locks if blocked).
- **Locking pipeline**: `lockPiece()` → `merge()` (bakes the piece into `board`) → `clearLines()` (bottom-up scan, `splice` + `unshift` an empty row) → `spawn()` (promotes `next` to `current`, generates a new `next`, checks for game-over collision).
- **Scoring/leveling**: `LINE_SCORES = [0, 100, 300, 500, 800]` multiplied by `level`; hard drop adds 2 pts/row, soft drop 1 pt/row. `level = floor(lines / 10) + 1`; drop speed follows `dropInterval = max(100, 1000 - (level - 1) * 90)`.
- **Ghost piece**: `ghostY()` projects the current piece straight down via repeated `collide` calls; drawn with `globalAlpha = 0.2` using the same `drawBlock` helper as everything else.
- **Rendering**: two canvases — `#board` (main play field) and `#next-canvas` (preview) — both drawn through the shared `drawBlock(context, x, y, colorIndex, size, alpha)` helper.
- **State**: a flat set of module-level `let` variables (`board, current, next, score, lines, level, paused, gameOver, dropInterval, ...`), all reset together in `init()`.

## Tunable constants (in `game.js`)

| Constant | Meaning |
| --- | --- |
| `COLS` / `ROWS` | Board dimensions (10 × 20) |
| `BLOCK` | Pixel size of each cell (30) |
| `COLORS` | Per-piece-type color palette |
| `LINE_SCORES` | Points for 1–4 lines cleared |
| `dropInterval` | Initial fall speed in ms (1000) |

If you change `COLS`, `ROWS`, or `BLOCK`, also update the `width`/`height` attributes of `<canvas id="board">` in `index.html` to match (`COLS × BLOCK` by `ROWS × BLOCK`).
