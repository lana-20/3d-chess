---
name: 3d-chess
description: Play 3D chess (5×5×5 Raumschach) at testtrack.org/3d-chess using vibium browser automation. Use when the user wants to play, watch, or automate moves in the 3D chess game.
---

# 3D Chess — Raumschach at testtrack.org/3d-chess

**Game**: 5×5×5 Raumschach three-dimensional chess  
**URL**: `https://testtrack.org/3d-chess`  
**Rendering**: WebGL canvas — squares are NOT in the DOM, only status text is

---

## Quick Start

```sh
vibium go https://testtrack.org/3d-chess
vibium find text "NEW GAME"
vibium click @e1
vibium sleep 500
vibium screenshot -o ~/Pictures/Vibium/chess.png
```

**Critical**: Always start via the "NEW GAME" button. Clicking directly on the canvas is unreliable and may deactivate the game.

---

## Board Layout

- **5 levels**: A (bottom, White's home) → E (top, Black's home)
- **5 ranks**: 1 (White's side) → 5 (Black's side) — displayed as Rank 1–5
- **5 files**: a–e (left to right)
- Coordinates written as `LevelRankFile`, e.g. `Ae2` = Level A, Rank 2, File e

**White starts on Levels A–B. Black starts on Levels D–E.**

Pawns move toward the opponent's home level:
- White pawns move toward Level E (PageUp to raise level, or increase rank)
- Black pawns move toward Level A (PageDown to lower level, or decrease rank)

---

## Reading Game State

The DOM has a text overlay with live game state. Read it with:

```sh
vibium text
```

Look for these fields in the output:
```
PLAYER: WHITE or BLACK
MOVES:  <number>
SELECTION
  <piece info or NO SELECTION>
CURSOR
  LEVEL: A–E
  RANK:  1–5
  FILE:  a–e
```

To get full React game state (piece positions, valid moves):
```sh
vibium eval "JSON.stringify((() => { const canvas = document.querySelector('canvas'); const key = Object.keys(canvas).find(k => k.startsWith('__reactFiber')); let f = canvas[key]; while(f) { const s = f.memoizedState?.memoizedState; if(s && s.pieces) return s; f = f.return; } return null; })()||'not found')"
```

Get current camera angle (radians):
```sh
vibium eval "(() => { const canvas = document.querySelector('canvas'); const key = Object.keys(canvas).find(k => k.startsWith('__reactFiber')); let f = canvas[key]; while(f) { const s = f.memoizedState?.memoizedState; if(s && typeof s.cameraAngle === 'number') return s.cameraAngle; f = f.return; } return 'not found'; })()"
```

---

## Keyboard Controls

| Key | Effect |
|-----|--------|
| Arrow keys | Move cursor on current level (camera-relative — see below) |
| PageUp | Move cursor up one level (toward E) |
| PageDown | Move cursor down one level (toward A) |
| Space or Enter | Select piece OR confirm move |
| Escape | Deselect piece |
| Ctrl+Left / Ctrl+Right | Rotate camera |
| N | New game |

**Important**: `vibium press Space canvas` does NOT reliably trigger piece selection — use JS dispatch instead (see below). Modifier+key combos like `Ctrl+Right` cannot be sent via `vibium press` at all.

---

## Camera-Relative Navigation

**Arrow keys depend on camera angle.** The default camera angle is approximately **2.356 radians (~135°)**, NOT 0.

At 135° (default), arrow keys move diagonally:
- **ArrowUp** → rank+1, file+1
- **ArrowDown** → rank-1, file-1
- **ArrowLeft** → rank+1, file-1
- **ArrowRight** → rank-1, file+1

**Parity problem**: Diagonal movement means you can only reach squares where `(rank+file)` has the same parity as the starting square. This makes roughly half the board unreachable from any given cursor position.

At 0° (after rotating camera), arrow keys are cardinal:
- **ArrowUp** → rank+1
- **ArrowDown** → rank-1
- **ArrowLeft** → file-1
- **ArrowRight** → file+1

**Always rotate camera to 0° before navigating** by dispatching Ctrl+ArrowRight via JS (see below). Check the angle first.

With camera angle S (radians), ArrowUp moves: rank += round(cos(S)), file -= round(sin(S)).

---

## JS Dispatch — Required for Space and Modifier Keys

`vibium press Space canvas` does not properly focus and trigger piece selection. Use this instead:

```sh
vibium eval "const c=document.querySelector('canvas');c.focus();['keydown','keyup'].forEach(t=>c.dispatchEvent(new KeyboardEvent(t,{key:' ',code:'Space',keyCode:32,which:32,bubbles:true,cancelable:true})))"
```

For Ctrl+ArrowRight (rotate camera clockwise):
```sh
vibium eval "const c=document.querySelector('canvas');c.focus();c.dispatchEvent(new KeyboardEvent('keydown',{key:'ArrowRight',code:'ArrowRight',ctrlKey:true,keyCode:39,bubbles:true,cancelable:true}))"
```

For Ctrl+ArrowLeft (rotate camera counter-clockwise):
```sh
vibium eval "const c=document.querySelector('canvas');c.focus();c.dispatchEvent(new KeyboardEvent('keydown',{key:'ArrowLeft',code:'ArrowLeft',ctrlKey:true,keyCode:37,bubbles:true,cancelable:true}))"
```

**Quote escaping**: `vibium eval` takes a double-quoted string. Use only single quotes inside the JS. No inner double quotes — they will break the command.

---

## Making a Move — Step by Step

### 0. Rotate camera to 0° first

Check the angle, then rotate if needed:
```sh
vibium eval "(() => { const canvas = document.querySelector('canvas'); const key = Object.keys(canvas).find(k => k.startsWith('__reactFiber')); let f = canvas[key]; while(f) { const s = f.memoizedState?.memoizedState; if(s && typeof s.cameraAngle === 'number') return s.cameraAngle; f = f.return; } return 'not found'; })()"
```

Each Ctrl+ArrowRight dispatch rotates by ~π/4. From 135° (2.356 rad), you need 3 presses to reach ~0°:
```sh
vibium eval "const c=document.querySelector('canvas');c.focus();c.dispatchEvent(new KeyboardEvent('keydown',{key:'ArrowRight',code:'ArrowRight',ctrlKey:true,keyCode:39,bubbles:true,cancelable:true}))"
# repeat 3× then re-check angle
```

### 1. Navigate to the piece

Cursor starts at **Level C, Rank 3, File c** on a new game (center of board). Use PageUp/PageDown to change level, arrows for rank/file.

```sh
# Navigate to Level A from default Level C (2 PageDown presses)
vibium press PageDown canvas && vibium sleep 200
vibium press PageDown canvas && vibium sleep 200

# Navigate rank/file (at camera angle 0°)
vibium press ArrowUp canvas    # rank +1
vibium press ArrowDown canvas  # rank -1
vibium press ArrowRight canvas # file +1
vibium press ArrowLeft canvas  # file -1
```

Check position after each step:
```sh
vibium text
```

### 2. Select the piece

Use JS dispatch (not `vibium press Space canvas`):
```sh
vibium eval "const c=document.querySelector('canvas');c.focus();['keydown','keyup'].forEach(t=>c.dispatchEvent(new KeyboardEvent(t,{key:' ',code:'Space',keyCode:32,which:32,bubbles:true,cancelable:true})))"
vibium sleep 300
vibium text  # confirm SELECTION shows piece name, VALID MOVES > 0
```

### 3. Navigate to destination

Use arrow/PageUp/PageDown. **Arrow keys move the cursor freely to any square — NOT restricted to valid moves.** Landing on an invalid square deselects the piece. Check `vibium text` often to verify cursor position and that selection is still active.

### 4. Confirm the move

```sh
vibium eval "const c=document.querySelector('canvas');c.focus();['keydown','keyup'].forEach(t=>c.dispatchEvent(new KeyboardEvent(t,{key:' ',code:'Space',keyCode:32,which:32,bubbles:true,cancelable:true})))"
vibium sleep 500
vibium text  # MOVES count should increment, PLAYER should switch
```

---

## Getting Valid Move Targets via JS

After selecting a piece, read its valid moves directly from React state:

```sh
vibium eval "JSON.stringify((() => { const canvas = document.querySelector('canvas'); const key = Object.keys(canvas).find(k => k.startsWith('__reactFiber')); let f = canvas[key]; while(f) { const s = f.memoizedState?.memoizedState; if(s && s.selectedSquare) return {selected: s.selectedSquare, validMoves: s.validMoves}; f = f.return; } return null; })()||{})"
```

This returns `selectedSquare` and `validMoves` array — use these to plan navigation to the correct destination.

---

## Key Gotchas

1. **Cursor starts at Level C (center)**, not Level E. To reach Level A, press PageDown **2 times**. To reach Level E, press PageUp **2 times**.

2. **`vibium press Space canvas` does not work** for piece selection — use the JS KeyboardEvent dispatch instead (see above).

3. **Camera default is ~135°, not 0°** — arrow keys are diagonal at default. Rotate to 0° before navigating to avoid the parity problem.

4. **Parity trap at 135°**: From a square with odd `(rank+file)` sum, you can only reach other odd-sum squares. This can make an entire side of the board unreachable. Fix: rotate camera to 0°.

5. **After selection, arrow keys move freely to any square** — not just valid moves. Landing on an invalid destination deselects the piece. Navigate carefully and verify cursor position with `vibium text` before confirming.

6. **Clicking the canvas resets the game** if you click outside the active board area. Always use "NEW GAME" button or keyboard key `N`.

7. **Ctrl+key combos cannot be sent via `vibium press`** — use `vibium eval` with a JS KeyboardEvent dispatch.

8. **Black pawns move via PageDown** (decrease level). A Black pawn at Level E (e.g. Ee4) moves to Level D (De4) — use PageDown after selecting.

9. **React fiber key** is dynamically named (observed as `__reactFiber$y77y7d6kqmr`). Always look it up with `Object.keys(canvas).find(k => k.startsWith('__reactFiber'))` — don't hardcode it.

---

## Full Game Loop (Camera-Corrected)

```sh
vibium go https://testtrack.org/3d-chess
vibium find text "NEW GAME" && vibium click @e1 && vibium sleep 500

# Rotate camera to 0° (3 Ctrl+ArrowRight presses from default 135°)
vibium eval "const c=document.querySelector('canvas');c.focus();c.dispatchEvent(new KeyboardEvent('keydown',{key:'ArrowRight',code:'ArrowRight',ctrlKey:true,keyCode:39,bubbles:true,cancelable:true}))" && vibium sleep 200
vibium eval "const c=document.querySelector('canvas');c.focus();c.dispatchEvent(new KeyboardEvent('keydown',{key:'ArrowRight',code:'ArrowRight',ctrlKey:true,keyCode:39,bubbles:true,cancelable:true}))" && vibium sleep 200
vibium eval "const c=document.querySelector('canvas');c.focus();c.dispatchEvent(new KeyboardEvent('keydown',{key:'ArrowRight',code:'ArrowRight',ctrlKey:true,keyCode:39,bubbles:true,cancelable:true}))" && vibium sleep 200

# Verify angle is near 0
vibium eval "(() => { const canvas = document.querySelector('canvas'); const key = Object.keys(canvas).find(k => k.startsWith('__reactFiber')); let f = canvas[key]; while(f) { const s = f.memoizedState?.memoizedState; if(s && typeof s.cameraAngle === 'number') return s.cameraAngle; f = f.return; } return 'not found'; })()"

# === WHITE'S TURN ===
vibium text  # read state

# Navigate to Level A from center (2 PageDown)
vibium press PageDown canvas && vibium sleep 200
vibium press PageDown canvas && vibium sleep 200

# Navigate to pawn at Ae2 (from Ac3: ArrowDown once = rank 2, ArrowRight twice = file e)
vibium press ArrowDown canvas && vibium sleep 200
vibium press ArrowRight canvas && vibium sleep 200
vibium press ArrowRight canvas && vibium sleep 200
vibium text  # verify CURSOR: LEVEL A, RANK 2, FILE e

# Select pawn
vibium eval "const c=document.querySelector('canvas');c.focus();['keydown','keyup'].forEach(t=>c.dispatchEvent(new KeyboardEvent(t,{key:' ',code:'Space',keyCode:32,which:32,bubbles:true,cancelable:true})))"
vibium sleep 300
vibium text  # verify SELECTION: PAWN, VALID MOVES: ...

# Move to Ae3 (rank +1)
vibium press ArrowUp canvas && vibium sleep 200
vibium text  # verify cursor at rank 3

# Confirm
vibium eval "const c=document.querySelector('canvas');c.focus();['keydown','keyup'].forEach(t=>c.dispatchEvent(new KeyboardEvent(t,{key:' ',code:'Space',keyCode:32,which:32,bubbles:true,cancelable:true})))"
vibium sleep 500
vibium text  # MOVES: 1, PLAYER: BLACK

vibium screenshot -o ~/Pictures/Vibium/chess-white-move.png
```

---

## Screenshots

```sh
vibium screenshot -o ~/Pictures/Vibium/chess-state.png
```
