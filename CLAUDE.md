# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project

Cube Tac Toe — a browser game combining a 3×3 Rubik's Cube with Tic Tac Toe. The entire game is a single self-contained `index.html` file with inline CSS and JavaScript. There is no build step, no package manager, and no dependencies. To run: open `index.html` in any modern browser.

## Game rules (encoded in the code, important when changing logic)

- Player 1 = X, Player 2 = O.
- Each turn has two phases: **place** (click an empty sticker), then **rotate** (click one move button). Both are mandatory.
- Win = 3 marks in a row, column, or diagonal on a single 3×3 face.
- Win is checked **only after the rotation completes**, never immediately after placement. This is a deliberate rule — do not move the `checkWin()` call.

## Architecture

### State model
The cube is represented as a flat 54-element array, `state`, where each entry is `null | 'X' | 'O'`. Faces are indexed F=0, B=1, L=2, R=3, U=4, D=5 with offsets `face * 9`. Sticker indices 0–8 within a face are in reading order (top-left to bottom-right when looking at that face from outside).

### Move permutations
The `MOVES` table is the most critical and error-prone part of the codebase. Each of the 12 moves (U/U', D/D', L/L', R/R', F/F', B/B') defines:
- `face` — which face's 9 stickers self-rotate
- `dir` — `+1` (CW from outside) or `-1` (CCW)
- `cycles` — three 4-element edge cycles `[a,b,c,d]` listing flat indices on the four neighbouring faces that rotate together

`applyCycle(arr, [a,b,c,d], +1)` sends `a→b→c→d→a` (i.e., `new[b] = old[a]`). The CW and CCW variants share the same cycle indices and only flip `dir`. When changing these arrays, verify against standard Rubik's cube notation — a single wrong index silently corrupts the cube state.

### Rendering
CSS 3D transforms only — no canvas, no WebGL, no Three.js. The cube structure is `#scene` (perspective container) → `#cube-wrapper` (carries drag rotation) → `#cube` → 6 `.face` divs → 9 `.sticker` divs per face. Each face's transform places it at `translateZ(90px)` after a face-specific rotation. `transform-style: preserve-3d` must remain on both `#cube-wrapper` and `#cube`.

### Drag rotation
`pointerdown` on `#scene` starts drag; `pointermove`/`pointerup` are on `document` so the pointer can leave the scene without losing the drag. `dragMoved` is set when total displacement exceeds 5px and is checked by `onStickerClick` to suppress clicks that were really drags. `rotX` is clamped to ±89° to avoid gimbal flipping.

### Turn state machine
`phase` is `'place' | 'rotate' | 'gameover'`. Move buttons are disabled (`b.disabled = phase !== 'rotate'`) so the UI enforces the two-phase turn structure. `updateStatus()` is the single source of truth for both the status text and button enabled state — call it after every state mutation.
