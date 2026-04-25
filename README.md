# 3D Chess — Claude Code Skill

A Claude Code skill for playing **5×5×5 Raumschach** three-dimensional chess at [testtrack.org/3d-chess](https://testtrack.org/3d-chess) using [vibium](https://vibium.app) browser automation.

## Usage

```
/3d-chess
```

Claude will open the game, read board state via DOM overlay, and play moves using keyboard automation and JavaScript dispatch.

## How It Works

The game renders on a WebGL canvas — squares are not in the DOM. Claude navigates the board using:

- **`vibium press <key> canvas`** for arrow keys and level changes (PageUp/PageDown) — only safe before a piece is selected
- **`vibium eval`** with JS `KeyboardEvent` dispatch for Space (select/confirm), Ctrl+Arrow (camera rotation), and **all keys after selection**
- **`vibium text`** to read the live status overlay (current player, cursor position, selection, valid move count)

## Board

```
Level E  ·  ·  ·  ·  ·    ← Black's home (pawns on E4 and D4)
Level D  ·  ·  ·  ·  ·
Level C  ·  ·  ·  ·  ·    ← center (cursor starts here)
Level B  ·  ·  ·  ·  ·
Level A  ·  ·  ·  ·  ·    ← White's home (pawns on A2 and B2)

Files:   a  b  c  d  e
Ranks:   1 → 5 (White → Black)
```

Coordinates: `LevelRankFile` — e.g. `Ae2` = Level A, Rank 2, File e.

**Starting piece layout (fully confirmed):**

| File | a | b | c | d | e |
|------|---|---|---|---|---|
| Level B rank 1 (White) | Bishop | Unicorn | Queen | Bishop | Unicorn |
| Level A rank 1 (White) | Rook | Knight | **King** | Knight | Rook |
| Level E rank 5 (Black) | Rook | Knight | **King** | Knight | Rook |
| Level D rank 5 (Black) | Bishop | Unicorn | Queen | Bishop | Unicorn |

White Level A ↔ Black Level E (exact mirror). White Level B ↔ Black Level D (exact mirror). The **Queen is on Level B/D**, not Level A/E. The **King is on Level A/E at file c**.

**Pawn setup**: White has pawns on Level A rank 2 AND Level B rank 2 (all files a–e). Black has pawns on Level E rank 4 AND Level D rank 4. This gives each pawn VALID MOVES: 2 at the start (same-level advance + cross-level advance).

**Promotion squares**: White promotes at rank 5 on Level A (e.g. Aa5). Black promotes at rank 1 on Level E (e.g. Ea1).

## Key Discoveries

| Finding | Detail |
|---------|--------|
| "GAME READY" splash after NEW GAME | `vibium click` on canvas fails ("element is obscured"). `canvas.click()` may return null without effect. Use `canvas.dispatchEvent(new MouseEvent("click",{bubbles:true,cancelable:true,view:window}))` — reliably returns `true` |
| Auto-promote setting | Settings panel has an AUTO PROMOTE toggle (default OFF); enable it at game start to skip promotion dialogs |
| Promotion dialog buttons | Button index 7 = ♛Queen; use `dispatchEvent(new MouseEvent("click",{bubbles:true,cancelable:true,view:window}))` — plain `.click()` doesn't work |
| Default camera angle | ~135° — arrow keys are diagonal by default; rotate to 180° before navigating |
| Camera rotation non-deterministic | Steps to reach 180° vary by session — always verify empirically with Escape + ArrowUp + read cursor, not by counting |
| Cursor start | Level C, Rank 3, File c (center) |
| Escape always resets cursor | Pressing Escape jumps cursor to Level C, Rank 3, File c regardless of selection state |
| Space key | `vibium press Space canvas` doesn't work — must use JS `KeyboardEvent` dispatch |
| All keys post-selection | `vibium press <key> canvas` deselects the piece — every navigation key after selection must use `vibium eval` with `canvas.focus()` |
| Cursor moves freely after selection | After selecting, cursor navigates to any square; deselection only happens when Space (confirm) is pressed on an invalid destination |
| Confirm on friendly piece | Lands on own piece → that piece gets re-selected instead of moving; MOVES count stays the same |
| Full piece layout confirmed | See Board section above for complete starting positions of all 20 pieces per side |
| Knight movement (3D L-shape) | Changes exactly 2 of 3 coordinates (level/rank/file), magnitudes 2+1. Confirmed: Ab1→Cb2 (+2 level, +1 rank, 0 file) and Eb5→Cb4 (-2 level, -1 rank, 0 file) |
| Unicorn starting positions | White: Be1 (Level B, Rank 1, File e). Black: **De5** (Level D, Rank 5, File e) — confirmed |
| Unicorn Ad2 blocked at game start | Level A rank 2 has White pawns on all files — unicorn from Be1 must go to Cd2 (PageUp) not Ad2 (PageDown) |
| Unicorn includes same-level diagonals | From corner (Be1/De5): VALID MOVES: 3. From center (Cd2/Cd4): VALID MOVES: **9** — confirms this variant's unicorn moves on same-level 2D diagonals too |
| Unicorn capture confirmed | Unicorn can capture opponent pieces: Black Cd4 → De3 captured White unicorn (Level+1, Rank-1, File+1 ✓) |
| Queen 3D diagonal confirmed | Db5→Eb4 is valid (level+1, rank-1, file-1 = pure 3D space diagonal, all ±1) ✓ |
| Queen cross-level diagonal | Aa5 → Ea1 is valid (4 levels + 4 ranks along file a = confirmed long-range diagonal capture) |
| Queen Level-File diagonal confirmed | Bc1→De1 valid (rank 1 fixed, 2 steps of Level+1, File+1) — queen uses Level-File 2D diagonal plane ✓ |
| Pawn diagonal capture confirmed | Black pawn Dd4→De3 (rank-1, file+1, same level) captures White unicorn — same as standard chess diagonal pawn capture ✓ |
| Pawn VALID MOVES: 3 = capture available | Dd4 showed VALID MOVES: 3 (straight advance + cross-level advance + diagonal capture) vs. VALID MOVES: 2 with no capture target |
| Pawn VALID MOVES: 2 persists mid-game | Level A pawn at rank 3 (Ab3) still has VALID MOVES: 2 — cross-level advance option is not a one-time bonus from starting rank |
| Bishop movement fully confirmed | Three 2D plane types: Level-Rank (file fixed), Level-File (rank fixed), Rank-File (level fixed). VALID MOVES: 18–20 from Cc3 |
| Bishop Level-Rank diagonal check | Cc3→Dc4 (ΔL+1, Δr+1, file c fixed) captures pawn AND checks Black King at Ec5 via same diagonal direction ✓ |
| Knight VALID MOVES: 14 from center | Confirmed at Bd3 (Level B, rank 3, file d) — center knight has maximum mobility |
| Knight VALID MOVES: 15 from Dd3 | Level D center position gives 15 valid moves — more than 14 from Bd3 because fewer own pieces block paths from Level D |
| Knight VALID MOVES: 8 from Ed5 | Confirmed from Black home corner (Level E, rank 5, file d) |
| Piece on C,3,c causes Escape loop | Escape resets cursor to C,3,c; if a piece is there it auto-selects. Use eval ArrowUp immediately after Escape to move off |
| Mixing vibium press + eval causes cursor confusion | Mixing `vibium press` arrow keys with `vibium eval` navigation causes unexpected cursor jumps. Use eval-only throughout any sequence |
| React fiber | Returns 'not found' — use only `vibium text` for state |

## Camera Verification Pattern

```sh
vibium press Escape canvas && vibium sleep 200   # reset cursor to C,3,c
vibium press ArrowUp canvas && vibium sleep 300
vibium text
# From C,3,c at 180°: rank goes 3→2, file stays c  ✓
# Any file change means camera is not at 180° yet
```

## Requirements

- [Claude Code](https://claude.ai/code)
- [vibium](https://vibium.app) CLI installed and authenticated
- A browser open and accessible to vibium
