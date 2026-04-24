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

**Pawn setup**: White has pawns on Level A rank 2 AND Level B rank 2 (all files a–e). Black has pawns on Level E rank 4 AND Level D rank 4. This gives each pawn VALID MOVES: 2 at the start (same-level advance + cross-level advance).

**Promotion squares**: White promotes at rank 5 on Level A (e.g. Aa5). Black promotes at rank 1 on Level E (e.g. Ea1).

## Key Discoveries

| Finding | Detail |
|---------|--------|
| "GAME READY" splash after NEW GAME | `vibium click` on canvas fails ("element is obscured") — must use `vibium eval 'document.querySelector("canvas").click()'` |
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
| Unicorn Ad2 blocked at game start | Level A rank 2 has White pawns on all files — unicorn from Be1 must go to Cd2 (PageUp) not Ad2 (PageDown) |
| Unicorn VALID MOVES: 3 | Third move from Be1 is likely Bd2 (same-level 2D diagonal), suggesting this implementation includes same-level diagonal movement |
| Queen cross-level diagonal | Aa5 → Ea1 is valid (4 levels + 4 ranks along file a = confirmed long-range diagonal capture) |
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
