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
vibium find text "NEW GAME" && vibium click @e1 && vibium sleep 800
# Game shows "GAME READY - Click anywhere to begin" — activate with JS:
vibium eval 'document.querySelector("canvas").dispatchEvent(new MouseEvent("click",{"bubbles":true,"cancelable":true,"view":window}))' && vibium sleep 800
vibium text  # should show PLAYER: WHITE, MOVES: 0
```

**Critical**: After clicking NEW GAME the game shows a "GAME READY" splash. `vibium click` on the canvas fails ("element is obscured"). `canvas.click()` via eval may return null without effect. Use `dispatchEvent(new MouseEvent("click",{bubbles:true,cancelable:true,view:window}))` — this reliably returns `true`. **Sometimes two calls are needed**: the first call may return `true` but leave the splash up if the game isn't fully initialized. Call it a second time with 800ms+ sleep and it will activate. Always verify with `vibium text` that PLAYER/MOVES are visible before proceeding.

---

## Board Layout

- **5 levels**: A (bottom, White's home) → E (top, Black's home)
- **5 ranks**: 1 (White's side) → 5 (Black's side)
- **5 files**: a–e (left to right)
- Coordinates: `LevelRankFile` — e.g. `Ae2` = Level A, Rank 2, File e

**White starts on Levels A–B. Black starts on Levels D–E.**

### Initial piece setup (fully confirmed)

```
Level B rank 1 (White):  Ba1=Bishop  Bb1=Unicorn  Bc1=Queen   Bd1=Bishop  Be1=Unicorn
Level A rank 1 (White):  Aa1=Rook    Ab1=Knight   Ac1=King    Ad1=Knight  Ae1=Rook
Level A rank 2 (White):  Aa2=Pawn    Ab2=Pawn     Ac2=Pawn    Ad2=Pawn    Ae2=Pawn
Level B rank 2 (White):  Ba2=Pawn    Bb2=Pawn     Bc2=Pawn    Bd2=Pawn    Be2=Pawn

Level E rank 4 (Black):  Ea4=Pawn    Eb4=Pawn     Ec4=Pawn    Ed4=Pawn    Ee4=Pawn
Level D rank 4 (Black):  Da4=Pawn    Db4=Pawn     Dc4=Pawn    Dd4=Pawn    De4=Pawn
Level E rank 5 (Black):  Ea5=Rook    Eb5=Knight   Ec5=King    Ed5=Knight  Ee5=Rook
Level D rank 5 (Black):  Da5=Bishop  Db5=Unicorn  Dc5=Queen   Dd5=Bishop  De5=Unicorn
```

White Level A rank 1 ↔ Black Level E rank 5 (exact mirror).  
White Level B rank 1 ↔ Black Level D rank 5 (exact mirror).  
The Queen is NOT on Level A — it's on Level B (White: Bc1, Black: Dc5).

### Initial pawn setup
- White: Level A Rank 2 (files a–e) **and** Level B Rank 2 (files a–e)
- Black: Level E Rank 4 (files a–e) **and** Level D Rank 4 (files a–e)

White pawns advance by increasing rank; Black pawns advance by decreasing rank.  
At 180° camera: White advances with **ArrowDown**, Black advances with **ArrowUp**.

### Promotion (confirmed)
- White: pawn at Rank 5 on **Level A** promotes (e.g. Aa5 → auto-Queen)
- Black: pawn at Rank 1 on **Level E** promotes (e.g. Ea1 → Queen dialog / auto-Queen)

Level B/D pawns — promotion rules not yet confirmed; they showed VALID MOVES: 2 at Rank 2 suggesting a cross-level advance option exists.

---

## Reading Game State

```sh
vibium text
```

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

When a piece is selected, the CURSOR section disappears — POSITION under SELECTION shows the piece's original square, not the cursor.

---

## Settings Panel

The settings panel (right column) has two useful controls:

**Default Promotion** — piece type for auto-promote (Queen by default).

**Auto Promote** — toggle ON to skip the promotion dialog and auto-promote to the default piece. Enable at game start to avoid dialog interruptions:

```sh
vibium eval '(function(){ var b = Array.from(document.querySelectorAll("button")).find(function(x){return x.textContent.trim()==="OFF";}); if(b) b.click(); })()'
```

---

## Pawn Promotion Dialog

When AUTO PROMOTE is OFF and a pawn reaches the promotion square, a modal dialog appears. The `vibium text` output includes the dialog options. To click a piece:

```sh
# Click Queen (button index 7 in document order)
vibium eval '(function(){ document.querySelectorAll("button")[7].dispatchEvent(new MouseEvent("click",{"bubbles":true,"cancelable":true,"view":window})); })()'
```

Button order: 0=back, 1=theme, 2=NEW GAME, 3=RESET, 4=FULLSCREEN, 5=DefaultPromotion dropdown, 6=AUTO PROMOTE toggle, **7=♛Queen**, 8=♜Rook, 9=♝Bishop, 10=♞Knight, 11=🦄Unicorn, 12=Close.

If the dialog persists after `.click()`, use `dispatchEvent(new MouseEvent(..., {view:window}))` on the correct button index.

**Best practice: enable AUTO PROMOTE before the game starts.**

---

## Keyboard Controls

| Key | Effect |
|-----|--------|
| Arrow keys | Move cursor on current level (camera-relative — see below) |
| PageUp | Move cursor up one level (toward E) |
| PageDown | Move cursor down one level (toward A) |
| Space or Enter | Select piece OR confirm move |
| Escape | Deselect / reset cursor to C,3,c |
| Ctrl+Left / Ctrl+Right | Rotate camera |
| N | New game |

**Critical constraints**:
- `vibium press Space canvas` does NOT reliably trigger piece selection — use JS dispatch
- `vibium press <key> canvas` after selecting a piece deselects it — every navigation key after selection must use `vibium eval` with `canvas.focus()`
- Modifier+key combos like `Ctrl+Arrow` cannot be sent via `vibium press` — use JS dispatch

---

## vibium eval Rules

**1. Outer single quotes** — outer double quotes are sometimes stripped by the shell.

**2. No top-level `const`/`let`/`var`** — they throw a script exception:
```sh
vibium eval 'const c = document.querySelector("canvas"); c.focus()'  # FAILS
vibium eval 'document.querySelector("canvas").focus()'               # works
```

Inside an IIFE, `const`/`let` are fine.

---

## Camera Rotation — Getting to 180°

**Arrow keys depend on camera angle.** Default is ~135° (diagonal movement). Target is **180°**, where:
- **ArrowUp** → rank−1 only (no file change)
- **ArrowDown** → rank+1 only
- **ArrowLeft** → file+1 (toward e)
- **ArrowRight** → file−1 (toward a)

The number of Ctrl+Arrow steps to reach 180° is **non-deterministic** across sessions — the rotation state is not reset predictably by NEW GAME. **Always verify empirically.**

### Rotation dispatch (one at a time)

```sh
# Ctrl+ArrowLeft
vibium eval 'document.querySelector("canvas").focus(); document.querySelector("canvas").dispatchEvent(new KeyboardEvent("keydown",{"key":"ArrowLeft","code":"ArrowLeft","ctrlKey":true,"keyCode":37,"bubbles":true,"cancelable":true}))'
# Ctrl+ArrowRight
vibium eval 'document.querySelector("canvas").focus(); document.querySelector("canvas").dispatchEvent(new KeyboardEvent("keydown",{"key":"ArrowRight","code":"ArrowRight","ctrlKey":true,"keyCode":39,"bubbles":true,"cancelable":true}))'
```

### Verification pattern

After each rotation batch, test the camera direction:

```sh
vibium press Escape canvas && vibium sleep 200   # reset cursor to C,3,c
vibium press ArrowUp canvas && vibium sleep 300
vibium text
# Read CURSOR. From C,3,c:
#   rank 3→2, file c→c  = 180° ✓  (desired)
#   rank 3→3, file c→b  = 90°  (rotate more)
#   rank 2→3, file c→b  = 135° (default — rotate more)
#   rank 4→4, file c→c  = 0°   (halfway — rotate more)
#   rank 4→4, file c→d  = 315° (overshot — rotate back)
```

### Rotation recipes (session observations — always verify, never rely on step counts)

**Session 1** (starting from fresh game):
1. 8× Ctrl+ArrowLeft → test → ~315°
2. 12× Ctrl+ArrowRight → test → ~0°
3. 8× Ctrl+ArrowLeft → test → ~90°
4. 8× Ctrl+ArrowLeft → test → ~135°
5. 1× Ctrl+ArrowRight → test → **180° ✓**

**Session 2** (game already open, camera at ~315° on load):
1. 3× Ctrl+ArrowLeft → test → **180° ✓** (direct)

Key lesson: use small batches (1–4 steps) once you're close, and test after each. The starting angle varies — always test first before rotating.

---

## Escape Behavior

Pressing Escape **always resets the cursor to Level C, Rank 3, File c** — whether or not a piece is selected. The CURSOR section is hidden after Escape; pressing any navigation key reveals it starting from C,3,c.

Use this as a reliable cursor reset point before navigating to a known position.

---

## Making a Move — Proven Pattern

### 1. Navigate to the piece (no selection active)

`vibium press` is safe before selection.

```sh
# From center C,3,c: Level A = 2× PageDown, Level E = 4× PageUp from A (or 2× from C)
vibium press PageDown canvas && vibium sleep 200
vibium press PageDown canvas && vibium sleep 200
# At 180° camera:
vibium press ArrowDown canvas   # rank+1
vibium press ArrowUp canvas     # rank-1
vibium press ArrowLeft canvas   # file+1 (toward e)
vibium press ArrowRight canvas  # file-1 (toward a)
```

Always verify with `vibium text` before selecting.

### 2a. Select + first navigation key (combined eval)

The preferred pattern — select and first nav key in one JS execution:

```sh
vibium eval 'document.querySelector("canvas").focus(); ["keydown","keyup"].forEach(function(t){document.querySelector("canvas").dispatchEvent(new KeyboardEvent(t,{"key":" ","code":"Space","keyCode":32,"which":32,"bubbles":true,"cancelable":true}))}); setTimeout(function(){document.querySelector("canvas").dispatchEvent(new KeyboardEvent("keydown",{"key":"ArrowDown","code":"ArrowDown","keyCode":40,"bubbles":true}))},100); "dispatched"'
vibium sleep 500
vibium text  # confirm SELECTION still shows piece, VALID MOVES > 0, STATUS: READY
```

### 2b. Alternative: bare select, then navigate separately

Also works — select without navigation, then navigate with eval:

```sh
vibium eval 'document.querySelector("canvas").focus(); ["keydown","keyup"].forEach(function(t){document.querySelector("canvas").dispatchEvent(new KeyboardEvent(t,{"key":" ","code":"Space","keyCode":32,"which":32,"bubbles":true,"cancelable":true}))})'
vibium sleep 500
vibium text  # verify SELECTION shows piece
# Then navigate:
vibium eval 'document.querySelector("canvas").focus(); setTimeout(function(){document.querySelector("canvas").dispatchEvent(new KeyboardEvent("keydown",{"key":"ArrowDown","code":"ArrowDown","keyCode":40,"bubbles":true}))},100); "nav"'
vibium sleep 400
```

### 2c. Additional navigation keys

Each additional key requires its own eval with `focus()`:

```sh
vibium eval 'document.querySelector("canvas").focus(); setTimeout(function(){document.querySelector("canvas").dispatchEvent(new KeyboardEvent("keydown",{"key":"ArrowRight","code":"ArrowRight","keyCode":39,"bubbles":true}))},100); "nav"'
vibium sleep 400
```

Key codes: ArrowUp=38, ArrowDown=40, ArrowLeft=37, ArrowRight=39, PageUp=33, PageDown=34.

### 3. Confirm the move

```sh
vibium eval 'document.querySelector("canvas").focus(); ["keydown","keyup"].forEach(function(t){document.querySelector("canvas").dispatchEvent(new KeyboardEvent(t,{"key":" ","code":"Space","keyCode":32,"which":32,"bubbles":true,"cancelable":true}))})'
vibium sleep 500
vibium text  # MOVES count increments, PLAYER switches
```

**Watch out**: confirming on a friendly piece re-selects it instead of moving (MOVES stays the same). Escape and start over.

---

## Piece Movement Notes

### Unicorn
Moves along **3D space diagonals** — all 3 coordinates change by exactly ±1 simultaneously. **This variant also includes same-level 2D diagonal movement** (level unchanged, rank±1, file±1).

Evidence: From a corner/edge (Be1, De5): VALID MOVES: 3. From a center square (Cd2, Cd4): **VALID MOVES: 9**. The jump from 3 to 9 is only explained by same-level diagonal moves being included alongside pure 3D diagonals.

**Starting positions**: White unicorn at **Be1** (Level B, Rank 1, File e). Black unicorn at **De5** (Level D, Rank 5, File e) — confirmed.

From Be1 (Level B, Rank 1, File e):
- **Ad2** — blocked by White pawn at initial setup (Level A, Rank 2 has pawns on all files)
- **Cd2** — confirmed valid ✓ (Level C, Rank 2, File d)

**Capture confirmed**: Black unicorn Cd4 → De3 captures White unicorn at De3 (Level+1, Rank-1, File+1 = all 3 coords ±1 ✓).

When planning a unicorn move: verify target is empty (or is an opponent piece to capture). If confirmation re-selects a friendly piece, try an alternative square.

### Queen
Moves like a standard 3D queen — straight lines along any axis and diagonals in any plane, including cross-level diagonals.

Confirmed moves:
- Aa5 → Ea1: Level+4, Rank-4, File constant = 2D diagonal across all 5 levels ✓
- Db5 → Eb4: Level+1, Rank-1, File-1 = pure 3D space diagonal (all ±1) ✓

Navigate by combining PageUp/PageDown and Arrow keys after selection.

### Knight
Moves in a 3D L-shape: changes exactly **2 of the 3 coordinates** (level, rank, file), with magnitudes **2+1** in the changed dimensions. The third coordinate stays fixed.

Confirmed moves:
- Ab1 → Cb2: level+2, rank+1, file=0 ✓ (White knight opening)
- Eb5 → Cb4: level-2, rank-1, file=0 ✓ (Black knight mirror)

Other valid jumps from a knight at (L, R, F): any combination where exactly one of level/rank/file stays fixed, and the other two change by 2+1 (or 1+2). This gives up to 16 possible squares (8 directions × 2 axis pairs), minus those off the board or occupied by friendly pieces.

VALID MOVES scaling: **5 from corner/edge** (Ab1, Eb5), **14 from center** (Bd3 confirmed).

### Rooks
Move along straight lines (one axis at a time). May be blocked by own pawns early in the game.

---

## Proven Moves

**White pawn: Ab2 → Ab3**
1. Navigate to A,2,b
2. Select + ArrowDown (rank+1)
3. Confirm

**Black pawn: Eb4 → Eb3**
1. Navigate to E,4,b
2. Select + ArrowUp (rank−1)
3. Confirm

**White unicorn: Be1 → Cd2** (not Ad2 — blocked by pawn)
1. Navigate to B,1,e
2. Select + PageUp (level B→C)
3. ArrowDown (rank 1→2) in separate eval
4. ArrowRight (file e→d) in separate eval
5. Confirm

**White queen (cross-level capture): Aa5 → Ea1**
1. Select queen at A,5,a
2. 4× PageUp in separate evals (level A→E)
3. 4× ArrowUp in separate evals (rank 5→1)
4. Confirm

**Black pawn: Ed4 → Ed3** (Level E pawn, VALID MOVES: 1)
1. Navigate to E,4,d
2. Select + ArrowUp (rank−1)
3. Confirm

**Black pawn: Dd4 → Dd3** (Level D pawn, VALID MOVES: 2 — cross-level advance option exists)
1. Navigate to D,4,d
2. Select + ArrowUp (rank−1)
3. Confirm (moves to D,3,d as expected)

**Black unicorn: De5 → Cd4** (confirmed starting position)
1. Navigate to D,5,e
2. Select + PageDown (level D→C)
3. ArrowUp (rank 5→4) in separate eval
4. ArrowRight (file e→d) in separate eval
5. Confirm

**White unicorn: Cd2 → De3** (aggressive push into Black's Level D)
1. Navigate to C,2,d
2. Select + PageUp (level C→D)
3. ArrowDown (rank 2→3) in separate eval
4. ArrowLeft (file d→e) in separate eval
5. Confirm

**Black unicorn capture: Cd4 → De3** (captures White unicorn)
1. Navigate to C,4,d
2. Select + PageUp (level C→D)
3. ArrowUp (rank 4→3) in separate eval
4. ArrowLeft (file d→e) in separate eval
5. Confirm — captures opponent piece, MOVES increments

**White unicorn: Cd2 → Dc3** (level+1, rank+1, file-1 = all ±1 ✓)
1. Navigate to C,2,d
2. Select + PageUp (level C→D)
3. ArrowDown (rank 2→3) in separate eval
4. ArrowRight (file d→c) in separate eval
5. Confirm

**White unicorn: Dc3 → Eb4** (captures Black pawn, puts King at Ec5 in check via same-level diagonal)
1. Navigate to D,3,c
2. Select + PageUp (level D→E)
3. ArrowDown (rank 3→4) in separate eval
4. ArrowRight (file c→b) in separate eval
5. Confirm — pawn captured, Black King at Ec5 checked (Eb4→Ec5 is same-level diagonal)

**Black Queen: Db5 → Eb4** (3D diagonal capture, level+1, rank-1, file-1 = all ±1 ✓)
1. Navigate to D,5,b
2. Select + PageUp (level D→E)
3. ArrowUp (rank 5→4) in separate eval
4. Confirm — captures opponent piece

**White Knight chain: Bd3 → Dd4 → Eb4**
- Bd3→Dd4: level+2, rank+1, file=d ✓ (enters Black territory)
- Dd4→Eb4: level+1, rank=4, file-2 ✓ (threatens Cc4 unicorn via level-2, file+1)

**White pawn: Ab3 → Ab4** (VALID MOVES: 2 at rank 3 — cross-level option still available)
1. Navigate to A,3,b
2. Select + ArrowDown (rank+1)
3. Confirm

---

## Key Gotchas

1. **"GAME READY" splash after NEW GAME** — `vibium click` on canvas fails ("element is obscured"). `canvas.click()` via eval may return null without effect. Use `canvas.dispatchEvent(new MouseEvent("click",{bubbles:true,cancelable:true,view:window}))` — reliably returns `true` and dismisses the splash.

2. **Camera rotation is non-deterministic** — the rotation state persists or varies across sessions. Always verify by testing ArrowUp movement, not by counting steps.

3. **Escape resets cursor to C,3,c** — always, even with no piece selected. Use this as a reliable reset.

4. **Cursor starts at C,3,c on new game** (center). Level A = 2× PageDown; Level E = 2× PageUp (or 4× from A).

5. **`vibium press Space canvas` does not work** for selection — use JS dispatch.

6. **All keys after selection must use vibium eval** — `vibium press <key> canvas` deselects the piece.

7. **Camera default ~135°** causes parity trap — only squares with matching `(rank+file)` parity are reachable. Rotate to 180° first.

8. **After selection, cursor moves freely** — not restricted to valid squares. Confirming on an invalid square deselects; confirming on a friendly piece re-selects it.

9. **Ad2 is occupied by a White pawn** at game start — unicorn from Be1 cannot go there.

10. **VALID MOVES: 2 for pawns persists mid-game** — both Level B/D pawns at rank 2 AND Level A pawns at rank 3 show VALID MOVES: 2. The cross-level advance option is not a one-time starting bonus; it remains available as the pawn advances. Confirmed: White pawn at Ab3 (Level A rank 3) still has VALID MOVES: 2. Promotion rules for Level B/D pawns still unconfirmed.

11. **Promotion dialog buttons** — use button index 7 (♛Queen) via `dispatchEvent(new MouseEvent("click",{bubbles:true,cancelable:true,view:window}))`. Enable AUTO PROMOTE at game start to avoid the dialog entirely.

12. **React fiber returns 'not found'** — use only `vibium text` for game state.

13. **Clicking the canvas outside the board** may reset the game. Use NEW GAME button or key `N`.

---

## Full Game Loop (Proven Pattern)

```sh
vibium go https://testtrack.org/3d-chess && vibium sleep 1500

# Start game
vibium find text "NEW GAME" && vibium click @e1 && vibium sleep 800
vibium eval 'document.querySelector("canvas").dispatchEvent(new MouseEvent("click",{"bubbles":true,"cancelable":true,"view":window}))' && vibium sleep 800

# Enable auto-promote to skip promotion dialogs
vibium eval '(function(){ var b = Array.from(document.querySelectorAll("button")).find(function(x){return x.textContent.trim()==="OFF";}); if(b) b.click(); })()'
vibium sleep 300

# Rotate camera to 180° — use test-and-adjust (see Camera Rotation section)
# Test pattern: Escape + ArrowUp, check if rank-1 only (no file change)
vibium eval 'document.querySelector("canvas").focus(); document.querySelector("canvas").dispatchEvent(new KeyboardEvent("keydown",{"key":"ArrowLeft","code":"ArrowLeft","ctrlKey":true,"keyCode":37,"bubbles":true,"cancelable":true}))' && vibium sleep 150
# ... repeat and test until ArrowUp gives rank-1, file unchanged

# === WHITE'S TURN ===
vibium press PageDown canvas && vibium sleep 200  # C → B
vibium press PageDown canvas && vibium sleep 200  # B → A
vibium press ArrowRight canvas && vibium sleep 200  # file c → b
vibium press ArrowUp canvas && vibium sleep 200     # rank 3 → 2
vibium text  # verify CURSOR: LEVEL A, RANK 2, FILE b

vibium eval 'document.querySelector("canvas").focus(); ["keydown","keyup"].forEach(function(t){document.querySelector("canvas").dispatchEvent(new KeyboardEvent(t,{"key":" ","code":"Space","keyCode":32,"which":32,"bubbles":true,"cancelable":true}))}); setTimeout(function(){document.querySelector("canvas").dispatchEvent(new KeyboardEvent("keydown",{"key":"ArrowDown","code":"ArrowDown","keyCode":40,"bubbles":true}))},100); "dispatched"'
vibium sleep 500
vibium text  # SELECTION: PAWN, VALID MOVES: 1, STATUS: READY

vibium eval 'document.querySelector("canvas").focus(); ["keydown","keyup"].forEach(function(t){document.querySelector("canvas").dispatchEvent(new KeyboardEvent(t,{"key":" ","code":"Space","keyCode":32,"which":32,"bubbles":true,"cancelable":true}))})'
vibium sleep 500
vibium text  # MOVES: 1, PLAYER: BLACK
```

---

## Screenshots

```sh
vibium screenshot -o ~/Pictures/Vibium/chess-state.png
```
