# 3D Chess — Claude Code Skill

A Claude Code skill for playing **5×5×5 Raumschach** three-dimensional chess at [testtrack.org/3d-chess](https://testtrack.org/3d-chess) using [vibium](https://vibium.app) browser automation.

## Usage

```
/3d-chess
```

Claude will open the game, read board state via DOM overlay, and play moves using keyboard automation and JavaScript dispatch.

## How It Works

The game renders on a WebGL canvas — squares are not in the DOM. Claude navigates the board using:

- **`vibium press <key> canvas`** for arrow keys and level changes (PageUp/PageDown)
- **`vibium eval`** with JS `KeyboardEvent` dispatch for Space (select/confirm) and Ctrl+Arrow (camera rotation)
- **`vibium text`** to read the live status overlay (current player, cursor position, selection, valid move count)
- **React fiber traversal** via `vibium eval` to read piece positions and valid moves directly from game state

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
| Arrow keys post-selection | Move freely to **any** square, not just valid moves — landing on an invalid square deselects the piece |
| Parity trap | At 135°, only squares where `(rank+file)` parity matches the start are reachable — rotate camera to 0° first |
| Camera rotation | Ctrl+Arrow cannot be sent via `vibium press` — requires JS dispatch |

## Requirements

- [Claude Code](https://claude.ai/code)
- [vibium](https://vibium.app) CLI installed and authenticated
- A browser open and accessible to vibium
