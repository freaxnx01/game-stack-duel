# Stack Duel

**▶ Play it:** https://github.freaxnx01.ch/game-stack-duel/

Two-player Tetris battle on one keyboard. Clear lines to dump garbage rows on your opponent — top out and you lose the round. Best of 3.

**Play:** open `index.html` in any browser, or visit the GitHub Pages URL once published.

## Controls

| Action | Player 1 | Player 2 |
|---|---|---|
| Move | A / D | ← / → |
| Rotate | W | ↑ |
| Soft drop | S | ↓ |
| Hard drop | Space | Enter |
| Hold | Left Shift | Right Shift |

`P` pause · `M` sound on/off

## Rules

- Clearing 2/3/4 lines sends 1/2/4 garbage rows to the opponent
- Incoming garbage can be cancelled by clearing lines before it lands
- Speed increases every 10 lines
- First to 2 round wins takes the match

## Files

- `index.html` — the complete game, fully self-contained (no dependencies, no build step). This is what GitHub Pages serves.
- `src/` — original editable source (`Stack Duel.dc.html` + `support.js` runtime). Open the `.dc.html` directly in a browser to run from source.

## Publishing

See `CLAUDE.md` — open this folder in Claude Code and say "publish this".