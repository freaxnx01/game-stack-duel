# Game Modes

Snapshot of the modes available from the Stack Duel main menu, as of `v0.2.0`
(source: `src/Stack Duel.dc.html` menu button list).

| Mode | Menu label | Players | Notes |
|---|---|---|---|
| Local hot-seat | START LOCAL MATCH | 2 | Two players on one keyboard. Best of 3 rounds. Default keys: P1 = WASD + Space (hard drop) + L-Shift (hold); P2 = Arrow Keys + Enter (hard drop) + R-Shift (hold). |
| Solo marathon | SOLO MARATHON | 1 | No opponent, no round limit — plays until top-out. Tracks a local best (lines/score) in `localStorage`. Uses the same keymap as Player 1 in local hot-seat (WASD + Space), since it reuses `players[0]`. |
| Online host | HOST ONLINE GAME | 2 or 3 | Manual WebRTC P2P; host creates the offer and picks 2- or 3-player mode before a guest connects. |
| Online join | JOIN ONLINE GAME | 2 or 3 | Manual WebRTC P2P; guest pastes the host's offer/answer codes. Always uses Arrow Keys + Space (hard drop) + Shift (hold), regardless of local hot-seat remaps. |

`CONTROLS` is a settings screen (remap keys for P1/P2), not a game mode.

## Known gap

Solo Marathon has no dedicated keymap — it currently inherits whatever Player 1's
local-hot-seat keys are (default WASD + Space). There's no in-menu way to use the
Arrow-Keys + Space scheme in solo without going into CONTROLS and manually
remapping P1. See tracker for a proposed fix.
