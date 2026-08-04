# Solo Marathon dedicated keymap — design spec

Issue: [#4 — default Solo Marathon to Arrow Keys + Space](https://github.com/freaxnx01/game-stack-duel/issues/4)

## Summary

The original Solo Marathon spec ([2026-08-03-single-player-mode-design.md](2026-08-03-single-player-mode-design.md),
"Controls" section) decided solo would reuse Player 1's local-hot-seat keymap
(`this.maps[0]`, default WASD + Space). That decision is superseded here:
testers expect Solo Marathon to default to Arrow Keys (movement) + Space (hard
drop), and — worse than a wrong hint label — Arrow keys are currently
non-functional in solo, since `keyDown`/`keyUp` only check `this.maps[0]`
(the WASD map) for the sole active player.

This gives solo its own, independently remappable third keymap, stored
alongside the existing Player 1/Player 2 maps.

Out of scope: any change to local hot-seat (P1/P2) or online (`NETMAP`)
keymaps; a context-aware CONTROLS screen that hides panels based on how it was
entered (always shows all three panels, per house pattern).

## Data model & migration

`maps` grows from `[p1, p2]` to `[p1, p2, solo]`. `defaultMaps()` gains a
third entry:

```js
defaultMaps() {
  return [
    { KeyA: 'left', KeyD: 'right', KeyS: 'soft', KeyW: 'rot', Space: 'hard', ShiftLeft: 'hold' },
    { ArrowLeft: 'left', ArrowRight: 'right', ArrowDown: 'soft', ArrowUp: 'rot', Enter: 'hard', ShiftRight: 'hold' },
    { ArrowLeft: 'left', ArrowRight: 'right', ArrowDown: 'soft', ArrowUp: 'rot', Space: 'hard', ShiftRight: 'hold' }
  ];
}
```

(Arrow Keys for movement + Space for hard drop, per the tester request;
`ShiftRight` for hold — not explicitly requested, but consistent with the
"arrow cluster + nearby modifier" feel of the existing P2 map.)

`loadMaps()` currently parses the saved `stackDuelKeys` array as-is. It gains
a backfill: if the parsed array has fewer than 3 entries, append
`defaultMaps()[2]` so a returning player's saved P1/P2 remaps are preserved
exactly, and solo simply appears with its new default on first load after the
update.

```js
loadMaps() {
  const def = this.defaultMaps();
  try {
    const raw = localStorage.getItem('stackDuelKeys');
    if (raw) {
      const parsed = JSON.parse(raw);
      if (Array.isArray(parsed) && parsed.length >= 2) {
        if (parsed.length < 3) parsed.push(def[2]);
        return parsed;
      }
    }
  } catch (e) {}
  return def;
}
```

`assignKey(player, action, code)` currently rebuilds a hardcoded 2-slot array
(`const maps = [this.maps[0], this.maps[1]]`) before writing the edited map
back — that must become a 3-slot copy (`[...this.maps]`) so a solo remap
(`player === 2`) persists instead of being dropped.

`resetKeys()` needs no change — it already does `this.maps = this.defaultMaps()`
wholesale, which now naturally resets all three maps together (matching the
single existing "RESET TO DEFAULTS" button, which has no per-panel variant
today).

## Runtime wiring (gameplay + hint text)

`keyDown`/`keyUp` loop `for (i = 0; i < players.length; i++)` and read
`this.maps[i][code]`. In solo, `players.length === 1`, so `i` is always `0`
and it reads the P1 map — the bug. Fix with a small helper, used in place of
direct `this.maps[i]` reads in both handlers:

```js
mapFor(i) { return (this.playerCount === 1 && i === 0) ? this.maps[2] : this.maps[i]; }
```

The on-screen hint text under the solo board has the same problem: `hintStr(0)`
is built from `keyFor(0, act)`, which reads `this.maps[0]` directly. `hintStr`
needs the equivalent `mapFor`-based lookup so the displayed hint always matches
what actually works.

`keyFor(i, action)` itself — used verbatim by the CONTROLS screen's per-panel
remap rows — is **not** touched. Its `i` is a literal UI panel index (0 = P1
panel, 1 = P2 panel, 2 = SOLO panel), not a gameplay-context-dependent player
index, so it stays as direct `this.maps[i]` reads.

Local 2-player hot-seat and online (`NETMAP`) paths are unaffected — `mapFor`
only special-cases `playerCount === 1 && i === 0`; `netMode` already takes a
separate branch upstream of any `maps` lookup.

## CONTROLS screen — new SOLO panel

A third always-visible remap panel, alongside the existing P1 and P2 panels,
sourced the same way they are: `keyRows(2)` (already index-generic) added to
`renderVals()` as `soloKeys`, with remap buttons calling `startListen(2, act)`
and letting the existing `assignKey(2, act, code)` (via `keyListen.player`)
handle the write.

Labeled **`SOLO`** — not `P3`. This repo already uses "P3" for the third
*online* player in 3-player P2P mode (see `isThreePlayer` in the versus
layout); reusing that label here for an unrelated single-player keymap would
be a real point of confusion between two independent features.

The template gets a third column reusing the existing per-panel markup
pattern (heading + `ACTLIST` rows + remap buttons), retargeted to
`soloKeys`/index `2`. No changes to the panel's internal row markup — it's
already fully driven by `keyRows(i)` / `startListen(i, act)`.

## Build

This is a bundled (dc-tool) game: `src/Stack Duel.dc.html` is the editable
source; `index.html` is generated and must never be hand-edited directly.
There is no external bundler CLI — regenerating `index.html` is a manual
merge: copy the current source content in, then re-apply the fixed set of
known `index.html`-only additions (favicon link, `version.js` include, the
`game-nav` footer block, the version-badge self-healing script). All logic
changes above (`defaultMaps`, `loadMaps`, `assignKey`, `mapFor`, `hintStr`,
`renderVals`'s `soloKeys`) go into `src/Stack Duel.dc.html`; `index.html`
must be re-synced from it before the fix is actually shippable (GitHub Pages
serves `index.html`, not the source file).

## Testing

Manual verification only (no test harness in this repo):

- Fresh `localStorage` (no `stackDuelKeys`): Solo Marathon's on-screen hint
  reads the Arrow Keys + Space scheme; Arrow keys actually move/rotate/soft-drop,
  Space hard-drops, R-Shift holds.
- Simulated returning player (seed `localStorage` with a 2-entry
  `stackDuelKeys` array before load): after load, P1/P2 keys are unchanged,
  solo still gets the Arrow + Space default (migration backfill), and no
  console errors.
- CONTROLS screen: the SOLO panel remaps independently — remapping a SOLO key
  doesn't affect P1/P2 and vice versa; a remap persists across reload.
- RESET TO DEFAULTS resets all three panels (P1, P2, SOLO) together.
- Local 2-player hot-seat still works unaffected (`playerCount` stays 2 there,
  `mapFor` only special-cases the `playerCount === 1` path).
- Empty console, no errors, per the existing manual checklist.
