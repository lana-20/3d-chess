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
- `vibium press <key> canvas` after selecting a piece deselects it — **every** navigation key after selection must use JS dispatch via `vibium eval`
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

### 2. Select + first navigation key (one eval with setTimeout)

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
| PageUp | `PageUp` | 33 |
| PageDown | `PageDown` | 34 |

### 2b. Additional navigation keys (each in its own eval)

Every key after the first also requires `vibium eval` with `focus()`. Use one eval per key:

```sh
vibium eval 'document.querySelector("canvas").focus(); setTimeout(function(){document.querySelector("canvas").dispatchEvent(new KeyboardEvent("keydown",{"key":"ArrowRight","code":"ArrowRight","keyCode":39,"bubbles":true}))},100); "nav"'
vibium sleep 400
```

Check selection is still active after each key if uncertain. If SELECTION disappears, the cursor landed on an invalid destination — navigate back to the piece and try again.

### 3. Confirm the move

```sh
vibium eval 'document.querySelector("canvas").focus(); ["keydown","keyup"].forEach(function(t){document.querySelector("canvas").dispatchEvent(new KeyboardEvent(t,{"key":" ","code":"Space","keyCode":32,"which":32,"bubbles":true,"cancelable":true}))})'
vibium sleep 500
vibium text  # MOVES count should increment, PLAYER should switch
```

**Watch out**: if the confirm Space lands on a square occupied by a friendly piece, that piece gets selected instead of the move executing. MOVES count will NOT increment. Deselect (Escape) and start over — but note that Escape teleports the cursor to center (Level C, Rank 3, File c).

---

## Piece Movement Notes

### Unicorn
Moves along **space diagonals** — all 3 coordinates (level, rank, file) change by exactly ±1 simultaneously. Example valid moves from Be1: Ad2 (level-1, rank+1, file-1), Cd2 (level+1, rank+1, file-1). Cannot move to a square occupied by a friendly piece.

When planning a unicorn move, verify the target square is empty. Use the select+navigate+confirm pattern, but be aware that if the destination is blocked by a friendly piece, the confirm will re-select that piece instead of moving.

### Rooks
Move along straight lines (rank, file, or level only — one axis at a time). May have limited valid moves if blocked by own pawns on adjacent squares.

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

**White: Be1 → Ad2** (unicorn developing move — all 3 coords change by 1)
1. Navigate cursor to Level B, Rank 1, File e
2. Select + PageDown (level B→A) in one eval
3. ArrowDown (rank 1→2) in separate eval
4. ArrowRight (file e→d) in separate eval
5. Confirm with Space (requires Ad2 to be empty)

---

## Key Gotchas

1. **Cursor starts at Level C (center)**, not Level E. From center: Level A = 2× PageDown, Level E = 2× PageUp. From Level E to Level A = 4× PageDown. Always count from your current level.

2. **`vibium press Space canvas` does not work** for piece selection — use JS dispatch.

3. **`vibium press <key> canvas` after selection deselects the piece** — the canvas loses focus between vibium commands. Every key after selection (not just the first) must use `vibium eval` with `canvas.focus()`. Pattern: one eval per key, each starting with `document.querySelector("canvas").focus(); setTimeout(dispatch, 100)`.

4. **No top-level `const`/`let`/`var` in vibium eval** — they cause a "script exception". Use direct method chains or wrap everything in an IIFE.

5. **Outer single quotes for vibium eval** — outer double quotes can be stripped by the shell for certain JS content.

6. **Camera default is ~135°, not 0°** — diagonal movement at default causes parity trap. Rotate to 180° (verified cardinal) before navigating.

7. **Parity trap at 135°**: diagonal movement means only squares where `(rank+file)` has same parity as start are reachable. Fix: rotate camera until ArrowUp changes only rank.

8. **After selection, cursor moves freely to any square** — NOT restricted to valid moves. The cursor does NOT deselect when passing through invalid squares. Only pressing Space (confirm) on an invalid destination deselects. Check `vibium text` before confirming.

9. **Confirming on a friendly piece re-selects it instead of moving** — if Space is pressed to confirm a move and the cursor is on a square occupied by your own piece, that piece gets selected and MOVES does NOT increment. Deselect with Escape and start over (but see gotcha #10).

10. **Escape teleports cursor to Level C, Rank 3, File c** — pressing Escape to deselect jumps the cursor to center regardless of where it was. Navigate back to the target square after deselecting.

11. **When a piece is selected, the CURSOR section disappears** from `vibium text` output — the POSITION field under SELECTION shows the piece location, not the cursor position.

12. **Clicking the canvas resets the game** if you click outside the active board area. Use "NEW GAME" button or key `N`.

13. **React fiber state traversal returns 'not found'** — the game state structure changed. Rely solely on `vibium text` for game state; do not use React fiber traversal.

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
