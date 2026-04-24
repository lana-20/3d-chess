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
- White pawns: increase rank (ArrowDown at 180° camera, see below)
- Black pawns: decrease rank (ArrowUp at 180° camera, see below)

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
  PIECE: <type>
  COLOR: <color>
  POSITION: LEVEL / RANK / FILE
  VALID MOVES COUNT: <n>
  STATUS: READY
CURSOR
  LEVEL: A–E
  RANK:  1–5
  FILE:  a–e
```

When a piece is selected, the CURSOR section disappears and is replaced by SELECTION with the piece's POSITION — the cursor position is no longer shown separately.

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

**Critical constraints**:
- `vibium press Space canvas` does NOT reliably trigger piece selection — use JS dispatch
- `vibium press <key> canvas` between select and navigate will deselect the piece — all movement keys after selection must also be JS dispatch
- Modifier+key combos like `Ctrl+Right` cannot be sent via `vibium press` at all

---

## vibium eval Rules

`vibium eval` evaluates JS in the browser page. Two rules that cause silent failures:

**1. Use outer single quotes** — outer double quotes are sometimes stripped by the shell:
```sh
vibium eval 'document.title'   # works
vibium eval "document.title"   # may fail depending on JS content
```

**2. No top-level `const`/`let`/`var`** — they throw a script exception:
```sh
vibium eval 'const c = document.querySelector("canvas"); c.focus()'  # FAILS
vibium eval 'document.querySelector("canvas").focus()'               # works
```

Inside an IIFE, `const`/`let` are fine:
```sh
vibium eval '(function(){ const c = document.querySelector("canvas"); return c.tagName; })()'  # works
```

---

## Camera-Relative Navigation

**Arrow keys depend on camera angle.** The default camera angle is approximately **2.356 radians (~135°)**, giving diagonal movement.

**Always rotate camera to cardinal before navigating.** The target is 180° (flipped cardinal), which proved reachable and usable. At 180°:
- **ArrowUp** → rank-1 (toward rank 1 / White's side)
- **ArrowDown** → rank+1 (toward rank 5 / Black's side)
- **ArrowLeft** → file+1 (toward file e)
- **ArrowRight** → file-1 (toward file a)

Rotate camera with repeated Ctrl+ArrowRight dispatches (one at a time), then verify by pressing ArrowUp and checking if only rank changes (no file change). Parity problem at 135°: ArrowUp changes both rank AND file, blocking access to half the board.

With camera angle S (radians), ArrowUp moves: rank += round(cos(S)), file -= round(sin(S)).

---

## Camera Rotation

Dispatch one Ctrl+ArrowRight at a time and verify after each batch:

```sh
vibium eval 'document.querySelector("canvas").focus(); document.querySelector("canvas").dispatchEvent(new KeyboardEvent("keydown",{"key":"ArrowRight","code":"ArrowRight","ctrlKey":true,"keyCode":39,"bubbles":true,"cancelable":true}))'
```

Counter-clockwise (Ctrl+ArrowLeft):
```sh
vibium eval 'document.querySelector("canvas").focus(); document.querySelector("canvas").dispatchEvent(new KeyboardEvent("keydown",{"key":"ArrowLeft","code":"ArrowLeft","ctrlKey":true,"keyCode":37,"bubbles":true,"cancelable":true}))'
```

**Verify cardinal movement**: navigate cursor (without selection) and confirm ArrowUp changes only rank, not file. The exact number of rotations needed from default varies — always test.

---

## Making a Move — Proven Pattern

### 1. Navigate to the piece (cursor only, no selection)

`vibium press` is safe here since no piece is selected yet.

```sh
# Cursor starts at Level C, Rank 3, File c on a new game
# Navigate to Level A (2 PageDown) or Level E (2 PageUp)
vibium press PageDown canvas && vibium sleep 200
vibium press PageDown canvas && vibium sleep 200

# Navigate rank/file (at 180° camera)
vibium press ArrowDown canvas   # rank+1 (toward rank 5)
vibium press ArrowUp canvas     # rank-1 (toward rank 1)
vibium press ArrowLeft canvas   # file+1 (toward file e)
vibium press ArrowRight canvas  # file-1 (toward file a)
```

Always verify with `vibium text`.

### 2. Select + navigate to destination (one eval, one setTimeout)

**This is the critical pattern.** Select and the first navigation key must happen in the same JS execution to keep canvas focus:

```sh
vibium eval 'document.querySelector("canvas").focus(); ["keydown","keyup"].forEach(function(t){document.querySelector("canvas").dispatchEvent(new KeyboardEvent(t,{"key":" ","code":"Space","keyCode":32,"which":32,"bubbles":true,"cancelable":true}))}); setTimeout(function(){document.querySelector("canvas").dispatchEvent(new KeyboardEvent("keydown",{"key":"ArrowDown","code":"ArrowDown","keyCode":40,"bubbles":true}))},100); "dispatched"'
vibium sleep 500
vibium text  # confirm SELECTION still shows piece, VALID MOVES > 0, STATUS: READY
```

Replace `ArrowDown`/`keyCode:40` with the correct direction key:
| Direction | key | keyCode |
|-----------|-----|---------|
| ArrowUp | `ArrowUp` | 38 |
| ArrowDown | `ArrowDown` | 40 |
| ArrowLeft | `ArrowLeft` | 37 |
| ArrowRight | `ArrowRight` | 39 |

For a level change (PageUp/PageDown) instead of an arrow:
```sh
vibium eval 'document.querySelector("canvas").focus(); ["keydown","keyup"].forEach(function(t){document.querySelector("canvas").dispatchEvent(new KeyboardEvent(t,{"key":" ","code":"Space","keyCode":32,"which":32,"bubbles":true,"cancelable":true}))}); setTimeout(function(){document.querySelector("canvas").dispatchEvent(new KeyboardEvent("keydown",{"key":"PageDown","code":"PageDown","keyCode":34,"bubbles":true}))},100); "dispatched"'
```

If the selection disappears after this eval, the destination was not a valid move. Navigate cursor back to the piece and try a different direction.

### 3. Confirm the move

```sh
vibium eval 'document.querySelector("canvas").focus(); ["keydown","keyup"].forEach(function(t){document.querySelector("canvas").dispatchEvent(new KeyboardEvent(t,{"key":" ","code":"Space","keyCode":32,"which":32,"bubbles":true,"cancelable":true}))})'
vibium sleep 500
vibium text  # MOVES count should increment, PLAYER should switch
```

---

## Proven Moves

**White: Ab2 → Ab3** (pawn forward one rank)
1. Navigate cursor to Level A, Rank 2, File b
2. Select + ArrowDown (rank+1): `key:"ArrowDown", keyCode:40`
3. Confirm with Space

**Black: Eb4 → Eb3** (pawn forward one rank)
1. Navigate cursor to Level E, Rank 4, File b
2. Select + ArrowUp (rank-1): `key:"ArrowUp", keyCode:38`
3. Confirm with Space

---

## Key Gotchas

1. **Cursor starts at Level C (center)**, not Level E. To reach Level A press PageDown **2 times**; to reach Level E press PageUp **2 times**.

2. **`vibium press Space canvas` does not work** for piece selection — use JS dispatch.

3. **`vibium press <key> canvas` deselects the piece** when used after selection. The canvas loses focus between separate vibium commands. Solution: chain select + navigate in one `vibium eval` using `setTimeout`.

4. **No top-level `const`/`let`/`var` in vibium eval** — they cause a "script exception". Use direct method chains or wrap everything in an IIFE.

5. **Outer single quotes for vibium eval** — outer double quotes can be stripped by the shell for certain JS content.

6. **Camera default is ~135°, not 0°** — diagonal movement at default causes parity trap. Rotate to 180° (verified cardinal) before navigating.

7. **Parity trap at 135°**: diagonal movement means only squares where `(rank+file)` has same parity as start are reachable. Fix: rotate camera until ArrowUp changes only rank.

8. **After selection, cursor moves freely to any square** — NOT restricted to valid moves. Landing on an invalid destination deselects the piece. If selection disappears after the select+navigate eval, the move was invalid.

9. **When a piece is selected, the CURSOR section disappears** from `vibium text` output — the POSITION field under SELECTION shows the piece location, not the cursor.

10. **Clicking the canvas resets the game** if you click outside the active board area. Use "NEW GAME" button or key `N`.

11. **React fiber state traversal** returns 'not found' — the game state structure may have changed. Rely on `vibium text` for game state.

12. **React fiber key** is dynamically named. Always look it up with `Object.keys(canvas).find(k => k.startsWith("__reactFiber"))` — don't hardcode it.

---

## Full Game Loop (Proven Pattern)

```sh
vibium go https://testtrack.org/3d-chess
vibium find text "NEW GAME" && vibium click @e1 && vibium sleep 500

# Rotate camera to cardinal (repeat until ArrowUp is pure rank change, no file change)
vibium eval 'document.querySelector("canvas").focus(); document.querySelector("canvas").dispatchEvent(new KeyboardEvent("keydown",{"key":"ArrowRight","code":"ArrowRight","ctrlKey":true,"keyCode":39,"bubbles":true,"cancelable":true}))' && vibium sleep 200
# ... repeat as needed, then test with vibium press ArrowUp canvas + vibium text

# === WHITE'S TURN ===
# Navigate to White pawn at Ab2
vibium press PageDown canvas && vibium sleep 200  # Level C → B
vibium press PageDown canvas && vibium sleep 200  # Level B → A
vibium press ArrowRight canvas && vibium sleep 200  # file c → b (ArrowRight = file-1)
vibium press ArrowUp canvas && vibium sleep 200     # rank 3 → 2 (ArrowUp = rank-1)
vibium text  # verify CURSOR: LEVEL A, RANK 2, FILE b

# Select pawn + navigate to Ab3 (ArrowDown = rank+1)
vibium eval 'document.querySelector("canvas").focus(); ["keydown","keyup"].forEach(function(t){document.querySelector("canvas").dispatchEvent(new KeyboardEvent(t,{"key":" ","code":"Space","keyCode":32,"which":32,"bubbles":true,"cancelable":true}))}); setTimeout(function(){document.querySelector("canvas").dispatchEvent(new KeyboardEvent("keydown",{"key":"ArrowDown","code":"ArrowDown","keyCode":40,"bubbles":true}))},100); "dispatched"'
vibium sleep 500
vibium text  # verify SELECTION: PAWN, VALID MOVES: 1, STATUS: READY

# Confirm
vibium eval 'document.querySelector("canvas").focus(); ["keydown","keyup"].forEach(function(t){document.querySelector("canvas").dispatchEvent(new KeyboardEvent(t,{"key":" ","code":"Space","keyCode":32,"which":32,"bubbles":true,"cancelable":true}))})'
vibium sleep 500
vibium text  # MOVES: 1, PLAYER: BLACK

vibium screenshot -o ~/Pictures/Vibium/chess-white-move.png

# === BLACK'S TURN ===
# Navigate to Black pawn at Eb4
vibium press PageUp canvas && vibium sleep 200  # ×4 to reach Level E
vibium press PageUp canvas && vibium sleep 200
vibium press PageUp canvas && vibium sleep 200
vibium press PageUp canvas && vibium sleep 200
vibium press ArrowRight canvas && vibium sleep 200  # file c → b (ArrowRight = file-1)
vibium press ArrowDown canvas && vibium sleep 200   # rank 3 → 4 (ArrowDown = rank+1)
vibium text  # verify CURSOR: LEVEL E, RANK 4, FILE b

# Select pawn + navigate to Eb3 (ArrowUp = rank-1)
vibium eval 'document.querySelector("canvas").focus(); ["keydown","keyup"].forEach(function(t){document.querySelector("canvas").dispatchEvent(new KeyboardEvent(t,{"key":" ","code":"Space","keyCode":32,"which":32,"bubbles":true,"cancelable":true}))}); setTimeout(function(){document.querySelector("canvas").dispatchEvent(new KeyboardEvent("keydown",{"key":"ArrowUp","code":"ArrowUp","keyCode":38,"bubbles":true}))},100); "dispatched"'
vibium sleep 500
vibium text  # verify SELECTION: PAWN, VALID MOVES: 1, STATUS: READY

# Confirm
vibium eval 'document.querySelector("canvas").focus(); ["keydown","keyup"].forEach(function(t){document.querySelector("canvas").dispatchEvent(new KeyboardEvent(t,{"key":" ","code":"Space","keyCode":32,"which":32,"bubbles":true,"cancelable":true}))})'
vibium sleep 500
vibium text  # MOVES: 2, PLAYER: WHITE

vibium screenshot -o ~/Pictures/Vibium/chess-black-move.png
```

---

## Screenshots

```sh
vibium screenshot -o ~/Pictures/Vibium/chess-state.png
```
