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
Level E  ·  ·  ·  ·  ·    ← Black's home
Level D  ·  ·  ·  ·  ·
Level C  ·  ·  ·  ·  ·    ← center (cursor starts here)
Level B  ·  ·  ·  ·  ·
Level A  ·  ·  ·  ·  ·    ← White's home

Files:   a  b  c  d  e
Ranks:   1 → 5 (White → Black)
```

Coordinates: `LevelRankFile` — e.g. `Ae2` = Level A, Rank 2, File e.

## Key Discoveries

| Finding | Detail |
|---------|--------|
| Default camera angle | ~135° (not 0°) — arrow keys are **diagonal** by default |
| Cursor start | Level C, Rank 3, File c (center) — not Level E |
| Space key | `vibium press Space canvas` doesn't work — must use JS `KeyboardEvent` dispatch |
| All keys post-selection | `vibium press <key> canvas` deselects the piece — every navigation key after selection must use `vibium eval` with `canvas.focus()` |
| Cursor moves freely | After selecting, cursor navigates to any square; deselection only happens when Space (confirm) is pressed on an invalid destination |
| Confirm on friendly piece | If confirm Space lands on a square occupied by your own piece, it re-selects that piece instead of moving — MOVES count stays the same |
| Escape teleports cursor | Pressing Escape to deselect jumps cursor to Level C, Rank 3, File c regardless of where it was |
| Parity trap | At 135°, only squares where `(rank+file)` parity matches the start are reachable — rotate camera to 180° first |
| Camera rotation | Ctrl+Arrow cannot be sent via `vibium press` — requires JS dispatch |
| Unicorn movement | Moves along space diagonals: all 3 coords (level, rank, file) change by exactly ±1 simultaneously |
| React fiber | Returns 'not found' — game state structure changed. Use only `vibium text` for state |

## Requirements

- [Claude Code](https://claude.ai/code)
- [vibium](https://vibium.app) CLI installed and authenticated
- A browser open and accessible to vibium
