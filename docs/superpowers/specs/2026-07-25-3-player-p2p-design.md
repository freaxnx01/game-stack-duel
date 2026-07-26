# 3-player P2P mode — design

Issue: [#1](https://github.com/freaxnx01/game-stack-duel/issues/1) — "Add MP P2P
co-op mode for 3 players".

## Problem

Stack Duel currently supports exactly 2 players: local hotseat, or P2P over a
single WebRTC data channel (`NetLink`, `index.html`) with a manual offer/answer
code exchange. There's no relay/mesh topology, no data model, and no UI for a
3rd player.

## Scope

Additive only. 2-player hotseat and 2-player P2P are unchanged. A 3rd option —
"3 players" — is added to the existing mode picker; the host chooses it before
generating the first connection code.

## Architecture

**Topology: host-relay star.** The host holds two `NetLink` instances (one per
guest) instead of one; each guest still holds exactly one `NetLink` connected
only to the host. The host is authoritative (as it already is in 2-player P2P)
and relays each guest's moves/attacks to the other player(s) — no guest ever
connects directly to another guest. This mirrors the host-relay pattern already
used elsewhere in the game catalog (Tschau Sepp), and needs only 2 WebRTC
connections total instead of 3 for a full mesh.

**Connection flow: sequential, one code at a time.** The host generates a
connection code for guest 1; guest 1 pastes it in and joins. Once connected,
the host generates a fresh code for guest 2, who joins the same way. This is
the existing 2-player host→guest exchange, repeated once more — no new UI
concept, no simultaneous-code juggling.

## Data model

`state.p1` / `state.p2` (two fixed keys, `{score, lines, level, incoming}`
each) becomes `state.players: [p1, p2, p3?]` — an array sized to the match
(2 or 3 entries). `state.wins` becomes an array of the same length, replacing
the current `wins: [0, 0]` tuple. Every place in `index.html` currently
indexing `p1`/`p2` directly (rendering, garbage delivery, round-end checks,
canvas refs `cvB1`/`cvB2`/`cvH1`/`cvH2`/`cvN1`/`cvN2`, `ACCENTS` color array)
gets rewritten to index the array by player slot instead. This is the largest
mechanical part of the change — the 2-player case becomes simply the
`players.length === 2` case of the same array-driven code, not a separate path.

## Gameplay rules

- **Garbage targeting:** on a line clear, the attack targets one randomly
  chosen surviving opponent (never the sender). In 2-player this reduces to
  today's behavior (only one possible target). In 3-player, each attack still
  goes to exactly one board, chosen at random among the other survivors.
- **Garbage cancellation:** unchanged — each player's own incoming queue can
  still be cancelled by clearing lines before the garbage lands, independent
  of how many opponents there are.
- **Round win condition:** a round ends when only one player hasn't topped
  out — that player wins the round. This generalizes the existing 2-player
  "opponent tops out, you win" rule to N players via the same "last one
  standing" check.
- **Match win condition:** unchanged — first player to win 2 rounds takes the
  match.
- **Mid-round disconnect:** a guest who disconnects mid-round is marked out
  for the rest of that round (their board freezes and greys out); the
  remaining players finish the round normally under the existing "last one
  standing" rule above. This only matters for 3-player matches — in 2-player,
  a disconnect already ends the match today and that behavior is unchanged.

## UI layout

Extend the existing split-screen layout: a 3rd Tetris board is added
alongside the current two, in the same visual style. Each board's on-screen
width narrows to fit three side by side (currently sized for two).

## Testing

Manual, buildless static site (matches the existing test gate for this repo):
- 3-client P2P: host + 2 guests connect via the sequential code exchange,
  confirm all three boards render and update live for all three clients.
- Garbage targeting: trigger line clears from each of the 3 players in turn,
  confirm the attack lands on exactly one of the two other (surviving) boards
  each time, never on the sender's own board.
- Round/match win: play a round to only one survivor remaining, confirm that
  player is credited the round win; play to 2 round wins, confirm the match
  ends and the correct player is declared the match winner.
- Disconnect mid-round: close one guest's tab mid-round, confirm their board
  freezes/greys out and the remaining two players' round continues and
  resolves normally.
- Regression: confirm 2-player hotseat and 2-player P2P still work exactly as
  before (mode picker still offers "2 players" as today's default path).
