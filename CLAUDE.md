# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project overview

Classic Tetris implemented in vanilla JavaScript with HTML5 Canvas and CSS. No dependencies, no build tooling, no bundler/transpiler — no `package.json` exists. The entire project is three files:

- `index.html` — DOM structure: the main `#board` canvas (300×600, 10×20 grid of 30px blocks), the `#next-canvas` preview (120×120), HUD elements (score/lines/level), and the pause/game-over overlay.
- `style.css` — dark/retro arcade visual theme (flexbox layout, `backdrop-filter` for overlays).
- `game.js` — all game logic (~300 lines), plain script tag, no modules.

## Running the game

No install or build step. Either open `index.html` directly in a browser, or serve it with any static server, e.g.:

```bash
python3 -m http.server 8000
# or
npx serve .
```

There is no test suite, linter, or CI config in this repo.

## Architecture (`game.js`)

Everything lives in module-level (`let`/`const`) globals — there is no state container object, so functions read/write shared variables like `board`, `current`, `next`, `score`, `level`, `dropInterval` directly.

- **Board model**: `board` is a `ROWS × COLS` matrix; each cell is `0` (empty) or a piece-color index `1–7`.
- **Pieces**: `PIECES` are square matrices (index 0 is unused/null so indices line up with `COLORS`). Rotation is done with `rotateCW`, a transpose-and-reverse — there is no precomputed rotation-state table like SRS.
- **Collision** (`collide`): checks board bounds and existing filled cells for a given shape/offset.
- **Wall kicks** (`tryRotate`): after rotating, tries offsets `[0, -1, 1, -2, 2]` and takes the first that doesn't collide. This is a simplified kick table, not the full SRS kick data.
- **Game loop** (`loop`): driven by `requestAnimationFrame`; accumulates elapsed time in `dropAccum` and advances the piece when it exceeds `dropInterval`.
- **Locking/clearing**: `lockPiece` → `merge` (bake piece into `board`) → `clearLines` (bottom-up scan, splice + unshift empty row) → `spawn` (promote `next` to `current`, generate new `next`; if the new piece immediately collides, `endGame()` fires).
- **Scoring**: `LINE_SCORES = [0, 100, 300, 500, 800]` multiplied by `level`; hard drop adds 2 points/row dropped, soft drop adds 1 point/row.
- **Leveling/speed**: level = `floor(lines / 10) + 1`; `dropInterval = max(100, 1000 - (level-1) * 90)` ms.
- **Ghost piece**: `ghostY()` projects the current piece straight down to its landing row; drawn at `globalAlpha = 0.2`.
- **Rendering**: `draw()` clears and redraws the grid, locked board, ghost, and current piece every frame onto `#board`; `drawNext()` renders the preview piece centered in a 4×4 cell onto `#next-canvas`.
- **Input**: a single `keydown` listener switches on `e.code` (arrows for move/soft-drop, `ArrowUp`/`KeyX` for rotate, `Space` for hard drop, `KeyP` for pause). Input is ignored while `paused` or `gameOver`.

## Tunable constants (top of `game.js`)

`COLS`, `ROWS`, `BLOCK`, `COLORS`, `LINE_SCORES`, initial `dropInterval`. If `COLS`/`ROWS`/`BLOCK` change, the `#board` canvas `width`/`height` in `index.html` must be updated to match (`COLS × BLOCK`, `ROWS × BLOCK`).
