# Solo Marathon mode — design spec

Issue: [#3 — Add single-player mode](https://github.com/freaxnx01/game-stack-duel/issues/3)

## Summary

Stack Duel is currently versus-only (local hot-seat or WebRTC P2P, garbage-row
attacks, best of 3). This adds a solo marathon mode: one board, no opponent, no
garbage. Speed still ramps every 10 lines as it does today. The run ends when
the player tops out. Score is lines cleared + level reached, with a persisted
best result shown on the menu and on the game-over screen.

Out of scope: AI opponent, garbage attacks against a scripted target, sprint
(fixed line-count) or timed variants, any server-side leaderboard.

## Menu entry point

Add a third top-level button on the `menu` phase, alongside the existing
`START LOCAL MATCH` / `HOST ONLINE GAME` / `JOIN ONLINE GAME`:

- **`SOLO MARATHON`** → calls a new `openSolo()` handler.

`openSolo()` sets `this.playerCount = 1` and calls `this.startRound()`
directly — it skips the local/host/join setup screens entirely, since solo
mode has no setup (no player count choice, no invite code).

## Round flow

`startRound()`, `spawn()`, the physics/loop, line-clear, and level-speed logic
are all unchanged and already loop over `this.players` generically — they work
as-is for `playerCount === 1`.

The only place that assumes ≥2 players is the end-of-round check in
`markOut()`:

```js
markOut(idx) {
  if (!this.playerAlive[idx]) return;
  this.playerAlive[idx] = false;
  if (this.players[idx]) this.players[idx].out = true;
  const survivors = [];
  for (let i = 0; i < this.playerAlive.length; i++) if (this.playerAlive[i]) survivors.push(i);
  if (survivors.length === 1) this._endRound(survivors[0]);
}
```

With one player, topping out drops `survivors.length` to `0`, so this never
fires — solo mode needs its own branch, checked before the existing one:

```js
markOut(idx) {
  if (!this.playerAlive[idx]) return;
  this.playerAlive[idx] = false;
  if (this.players[idx]) this.players[idx].out = true;
  if (this.playerCount === 1) { this._endSolo(); return; }
  const survivors = [];
  for (let i = 0; i < this.playerAlive.length; i++) if (this.playerAlive[i]) survivors.push(i);
  if (survivors.length === 1) this._endRound(survivors[0]);
}
```

`_endSolo()` mirrors `_endRound()`'s shape (stop the round, play the `lose`
sound, set `overlayAt`, sync HUD) but has no winner/wins-counter and instead
updates the best-score record:

```js
_endSolo() {
  if (!this.roundActive) return;
  this.roundActive = false;
  const p = this.players[0];
  const beat = this.updateSoloBest(p.lines, p.score);
  this.snd('lose');
  this.overlayAt = performance.now();
  this.syncHud();
  this.setState({ phase: 'soloover', soloResult: { lines: p.lines, score: p.score, level: p.level, beat } });
}
```

Round/win counters (`state.wins`, `state.round`) are not touched by the solo
path — they stay at their existing defaults and are simply unused while
`playerCount === 1`.

## `advance()` branch

`advance()` (the phase-transition handler bound to click/Enter) gets one new
branch:

```js
else if (ph === 'soloover') { this.startRound(); }
```

(no round/wins reset needed — solo doesn't use them)

## Layout

Solo mode reuses the existing single-board layout math
(`boardW = this.playerCount === 3 ? 230 : 300`, so `playerCount === 1` already
resolves to the 300px board) with only one board rendered: the render path
that lays out per-player panels iterates `this.players`/`this.playerCount`, so
for `playerCount === 1` only one board panel appears with no code changes
needed there. The wins-dots UI (`dots(i)`) is skipped for solo — it's already
gated implicitly by only being rendered per opponent-vs-opponent phases
(`roundover`/`matchover`); the new `soloover` phase simply doesn't render it.

## Copy (`renderVals()`)

New `ph === 'soloover'` branch, following the existing pattern for
`roundover`/`matchover`:

```js
} else if (ph === 'soloover') {
  kicker = 'SOLO MARATHON';
  title = 'GAME OVER';
  sub = s.soloResult.lines + ' LINES · LEVEL ' + s.soloResult.level
    + (s.soloResult.beat ? '  ·  NEW BEST!' : '  ·  BEST: ' + this.soloBest.lines + ' LINES');
  button = 'PLAY AGAIN';
}
```

The `menu` phase kicker/sub also picks up the persisted best, e.g. appending
`SOLO BEST: N LINES` under the existing menu copy when `this.soloBest.lines >
0`.

## Controls

Solo mode uses Player 1's configured keymap (`this.maps[0]`), same as local
hot-seat — no `NETMAP`, no second keymap involved, remappable via the existing
CONTROLS screen without change.

## High score persistence

New `localStorage` key `stackDuelSoloBest`, mirroring the existing
`stackDuelKeys` load/save pattern:

```js
loadSoloBest() {
  try {
    const raw = localStorage.getItem('stackDuelSoloBest');
    if (!raw) return { lines: 0, score: 0 };
    const parsed = JSON.parse(raw);
    if (parsed && typeof parsed.lines === 'number' && typeof parsed.score === 'number') return parsed;
  } catch (e) {}
  return { lines: 0, score: 0 };
}
saveSoloBest() { try { localStorage.setItem('stackDuelSoloBest', JSON.stringify(this.soloBest)); } catch (e) {} }
updateSoloBest(lines, score) {
  const beat = lines > this.soloBest.lines;
  if (beat) { this.soloBest = { lines, score }; this.saveSoloBest(); }
  return beat;
}
```

`this.soloBest` is loaded once in `componentDidMount()`, same lifecycle spot
`this.maps` is loaded today.

## Build

This is a bundled (dc-tool) game: `src/Stack Duel.dc.html` is the editable
source, `index.html` and `src/support.js` are generated — per CLAUDE.md, never
hand-edit the generated files. All logic changes above (`openSolo`,
`markOut`/`_endSolo`, `advance`, `renderVals`, solo-best persistence) go into
`src/Stack Duel.dc.html`, following the existing versus logic (`topOut`,
`markOut`, `_endRound`, `advance`, `renderVals`) already authored there. After
editing, re-bundle to regenerate `index.html` per this repo's existing publish
step (see CLAUDE.md's publish workflow) before considering the change shippable.

## Testing

Manual verification only (no test harness in this repo):

- Menu shows `SOLO MARATHON` button; clicking it starts a round with one
  board, no opponent panel, no wins dots.
- Playing until topping out shows the `soloover` screen with correct
  lines/level, `PLAY AGAIN` restarts a fresh solo round.
- Beating the previous best updates `stackDuelSoloBest` in `localStorage` and
  shows "NEW BEST!"; a worse run shows the persisted "BEST: N LINES" instead.
- Local 2P and online modes are unaffected (`playerCount` stays 2 or 3 there,
  `markOut`'s existing branch still fires for them).
