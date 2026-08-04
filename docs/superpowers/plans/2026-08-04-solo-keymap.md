# Solo Marathon Dedicated Keymap Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Give Solo Marathon its own default keymap (Arrow Keys + Space), independently remappable, instead of silently reusing (and being non-functional under) Player 1's local-hot-seat keys.

**Architecture:** `maps` grows from a 2-entry array (`[p1, p2]`) to 3 (`[p1, p2, solo]`). A new `mapFor(i)` helper resolves which map gameplay code should read for a given player-array index, special-casing `playerCount === 1` (solo) to redirect to `maps[2]`. The CONTROLS screen gets a third, always-visible remap panel ("SOLO") that edits `maps[2]` the same way the existing P1/P2 panels edit `maps[0]`/`maps[1]`.

**Tech Stack:** Single-file "DC" component (`src/Stack Duel.dc.html`, Handlebars-style `{{ }}` interpolation + `sc-if`/`sc-for` directives). No build step, no test framework — this repo verifies by opening the game in a browser. `index.html` is a generated file with **no external bundler CLI** (see Global Constraints).

## Global Constraints

- Repo is a bundled (dc-tool) game: edit **only** `src/Stack Duel.dc.html` for all logic/markup changes in Tasks 1–3.
- There is **no external `dc-tool` binary** — do not search for one or skip re-bundling if one isn't found. Re-bundling `index.html` is a confirmed-working **manual merge**, done once already in this repo (2026-08-04, commit `eb62a45`): copy the full current `src/Stack Duel.dc.html` content into `index.html`, then re-apply exactly these 4 known `index.html`-only additions:
  1. `<link rel="icon" href="favicon.png" sizes="32x32" type="image/png">` inside `<head>`
  2. `<script src="./version.js"></script>` immediately after `<body>`
  3. The `game-nav` `<nav>` block + `buttons.github.io/buttons.js` `<script>`, inserted just before `</x-dc>`
  4. The version-badge self-healing IIFE `<script>` block, appended at the very end before `</body>`
  Verify the merge by running `diff "src/Stack Duel.dc.html" index.html` afterward — the only differences should be exactly those 4 additions (nothing else added or missing). This step is **not optional** — GitHub Pages serves `index.html`, not the source file, so skipping it means the fix never ships (this is exactly how issue #4 happened in the first place).
- No automated test suite exists. Each task's "manual verification" step describes what a **human, in a real browser**, should check. If implementing in a headless/CI sandbox with a display available (confirmed possible in this repo via `python3 -m http.server` + Playwright — see Task 5), actually run it; if no display/browser tooling is available at all, read the diff against this plan's Find/Replace blocks instead and leave the playtest as a PR checklist item for the human reviewer.
- `localStorage` key convention: mirror the existing `stackDuelKeys` pattern (JSON-encoded, defensive `try/catch` parse, safe default on any failure) — see `loadMaps()`/`saveMaps()` at `src/Stack Duel.dc.html:486-508`.
- Keep local hot-seat (P1/P2) and online (`NETMAP`) keymap behavior byte-for-byte unchanged — every new branch is additive and gated on `playerCount === 1`, never a change to existing 2/3-player code paths.
- Commit after every task.

---

## File Structure

Everything lives in `src/Stack Duel.dc.html` (single file). No new files are created except the CHANGELOG entry. The tasks below touch these regions, in this order:

1. Keybindings persistence — `defaultMaps()`, `loadMaps()`, `assignKey()` (lines ~480–550)
2. `keyFor`/`keyForMap` split + new `mapFor(i)` helper (lines ~516–520)
3. Gameplay key handling — `keyDown()`, `keyUp()` (lines ~1123–1187)
4. `renderVals()` — `hintStr`, `soloKeys`/`editSolo` additions (lines ~1450–1483)
5. CONTROLS screen template — new SOLO panel (lines ~239–269)
6. `CHANGELOG.md`

---

### Task 1: Third keymap slot — defaults, migration, and remap persistence

**Files:**
- Modify: `src/Stack Duel.dc.html:480-497` (`defaultMaps`, `loadMaps`)
- Modify: `src/Stack Duel.dc.html:538-549` (`assignKey`)

**Interfaces:**
- Produces: `this.maps` is now always a 3-element array `[p1Map, p2Map, soloMap]`. `this.defaultMaps()[2]` is `{ ArrowLeft: 'left', ArrowRight: 'right', ArrowDown: 'soft', ArrowUp: 'rot', Space: 'hard', ShiftRight: 'hold' }`.
- Consumes: nothing new — this is the foundation task. Downstream tasks (2–4) read `this.maps[2]` and call `this.assignKey(2, action, code)`.

- [ ] **Step 1: Add the solo default and relax the migration check**

Find (`src/Stack Duel.dc.html:479-498`):

```js
  /* ---------- keybindings persistence ---------- */
  defaultMaps() {
    return [
      { KeyA: 'left', KeyD: 'right', KeyS: 'soft', KeyW: 'rot', Space: 'hard', ShiftLeft: 'hold' },
      { ArrowLeft: 'left', ArrowRight: 'right', ArrowDown: 'soft', ArrowUp: 'rot', Enter: 'hard', ShiftRight: 'hold' }
    ];
  }
  loadMaps() {
    const def = this.defaultMaps();
    try {
      const raw = localStorage.getItem('stackDuelKeys');
      if (!raw) return def;
      const parsed = JSON.parse(raw);
      if (Array.isArray(parsed) && parsed.length === 2 && parsed[0] && parsed[1]
        && typeof parsed[0] === 'object' && typeof parsed[1] === 'object') {
        return [Object.assign({}, parsed[0]), Object.assign({}, parsed[1])];
      }
    } catch (e) {}
    return def;
  }
```

Replace with:

```js
  /* ---------- keybindings persistence ---------- */
  defaultMaps() {
    return [
      { KeyA: 'left', KeyD: 'right', KeyS: 'soft', KeyW: 'rot', Space: 'hard', ShiftLeft: 'hold' },
      { ArrowLeft: 'left', ArrowRight: 'right', ArrowDown: 'soft', ArrowUp: 'rot', Enter: 'hard', ShiftRight: 'hold' },
      { ArrowLeft: 'left', ArrowRight: 'right', ArrowDown: 'soft', ArrowUp: 'rot', Space: 'hard', ShiftRight: 'hold' }
    ];
  }
  loadMaps() {
    const def = this.defaultMaps();
    try {
      const raw = localStorage.getItem('stackDuelKeys');
      if (!raw) return def;
      const parsed = JSON.parse(raw);
      if (Array.isArray(parsed) && parsed.length >= 2 && parsed[0] && parsed[1]
        && typeof parsed[0] === 'object' && typeof parsed[1] === 'object') {
        const maps = [Object.assign({}, parsed[0]), Object.assign({}, parsed[1])];
        maps.push(parsed[2] && typeof parsed[2] === 'object' ? Object.assign({}, parsed[2]) : def[2]);
        return maps;
      }
    } catch (e) {}
    return def;
  }
```

(`parsed.length >= 2` — was `=== 2` — plus the explicit `parsed[2]` fallback is the migration: a saved 2-entry blob from before this change gets `def[2]` appended, a 3-entry blob keeps its saved solo map, and P1/P2 are read exactly as before either way.)

- [ ] **Step 2: Make `assignKey` write to any of the 3 slots**

Find (`src/Stack Duel.dc.html:538-546`):

```js
  assignKey(player, action, code) {
    const m = Object.assign({}, this.maps[player]);
    for (const c in m) if (m[c] === action) delete m[c];
    delete m[code];
    m[code] = action;
    const maps = [this.maps[0], this.maps[1]];
    maps[player] = m;
    this.maps = maps;
    this.saveMaps();
    this.snd('rotate');
    this.setState({ keyListen: null });
  }
```

Replace with:

```js
  assignKey(player, action, code) {
    const m = Object.assign({}, this.maps[player]);
    for (const c in m) if (m[c] === action) delete m[c];
    delete m[code];
    m[code] = action;
    const maps = [...this.maps];
    maps[player] = m;
    this.maps = maps;
    this.saveMaps();
    this.snd('rotate');
    this.setState({ keyListen: null });
  }
```

- [ ] **Step 3: Manual verification**

Open `src/Stack Duel.dc.html` in a browser with a clean `localStorage` (DevTools → Application → clear site data, or `localStorage.clear()` in the console). No visible change yet (nothing reads `maps[2]` until Task 3) — just confirm the page loads with no console errors, and `localStorage.getItem('stackDuelKeys')` is still absent until CONTROLS or a game is opened.

Then in the console, seed a **stale 2-entry** blob to check the migration path directly:

```js
localStorage.setItem('stackDuelKeys', JSON.stringify([
  { KeyA: 'left', KeyD: 'right', KeyS: 'soft', KeyW: 'rot', Space: 'hard', ShiftLeft: 'hold' },
  { ArrowLeft: 'left', ArrowRight: 'right', ArrowDown: 'soft', ArrowUp: 'rot', Enter: 'hard', ShiftRight: 'hold' }
]));
location.reload();
```

After reload, in the console: `this` isn't directly accessible, so instead open CONTROLS from the menu once Task 5 lands the SOLO panel — for now, just confirm no console error appears on load with this 2-entry blob present (proves `loadMaps()` doesn't throw on the old shape).

- [ ] **Step 4: Commit**

```bash
cd "/home/freax/repos/github/freaxnx01/public/game-stack-duel"
git add "src/Stack Duel.dc.html"
git commit -m "feat(controls): add solo keymap slot with migration"
```

---

### Task 2: `mapFor(i)` resolver + `keyFor`/`keyForMap` split

**Files:**
- Modify: `src/Stack Duel.dc.html:516-520` (`keyFor`)

**Interfaces:**
- Produces: `this.mapFor(i)` — returns the actual keymap object to use for gameplay-context player index `i`: `this.NETMAP` if `this.netMode`, else `this.maps[2]` if `this.playerCount === 1 && i === 0` (solo), else `this.maps[i]`. `this.keyForMap(m, action)` — the label-lookup body factored out of `keyFor`, taking a resolved map object directly.
- Consumes: `this.maps` (3-element array, from Task 1), `this.playerCount`, `this.netMode`, `this.NETMAP` (all pre-existing).

- [ ] **Step 1: Split `keyFor` and add `mapFor`**

Find (`src/Stack Duel.dc.html:516-520`):

```js
  keyFor(i, action) {
    const m = this.netMode ? this.NETMAP : this.maps[i];
    for (const c in m) if (m[c] === action) return this.keyLabel(c);
    return '—';
  }
```

Replace with:

```js
  keyFor(i, action) {
    const m = this.netMode ? this.NETMAP : this.maps[i];
    return this.keyForMap(m, action);
  }
  keyForMap(m, action) {
    for (const c in m) if (m[c] === action) return this.keyLabel(c);
    return '—';
  }
  mapFor(i) {
    if (this.netMode) return this.NETMAP;
    return (this.playerCount === 1 && i === 0) ? this.maps[2] : this.maps[i];
  }
```

(`keyFor(i, action)` keeps its existing behavior unchanged — it's used by the CONTROLS screen's `keyRows`/`editRows`, where `i` is a literal panel index (0=P1, 1=P2, 2=SOLO) that must **not** be redirected by `playerCount`. `mapFor(i)` is the new gameplay-context resolver, used by Task 3's `keyDown`/`keyUp` and Task 4's `hintStr`.)

- [ ] **Step 2: Manual verification**

Open the file in a browser, open DevTools console, and confirm no syntax/reference errors on load (e.g. run `typeof window.GAME_VERSION` or just check the console is clean — `mapFor`/`keyForMap` aren't called by anything yet, so behavior is unchanged).

- [ ] **Step 3: Commit**

```bash
cd "/home/freax/repos/github/freaxnx01/public/game-stack-duel"
git add "src/Stack Duel.dc.html"
git commit -m "feat(controls): add mapFor keymap resolver for solo"
```

---

### Task 3: Fix gameplay key handling in solo — `keyDown`, `keyUp`

**Files:**
- Modify: `src/Stack Duel.dc.html:1123-1160` (`keyDown`)
- Modify: `src/Stack Duel.dc.html:1174-1187` (`keyUp`)

**Interfaces:**
- Consumes: `this.mapFor(i)` (Task 2).
- Produces: no new interface — this is the fix that makes Arrow Keys + Space actually move pieces in solo.

- [ ] **Step 1: Use `mapFor` in `keyDown`**

Find (`src/Stack Duel.dc.html:1152-1160`):

```js
    if (ph !== 'playing' || !this.roundActive) return;
    for (let i = 0; i < this.players.length; i++) {
      if (this.netMode && i !== this.myIdx) continue; // net: only local player's keys
      if (this.playerAlive && !this.playerAlive[i]) continue;
      const act = this.netMode ? this.NETMAP[code] : this.maps[i][code];
      if (!act) continue;
      this.doAction(this.players[i], act);
    }
  }
```

Replace with:

```js
    if (ph !== 'playing' || !this.roundActive) return;
    for (let i = 0; i < this.players.length; i++) {
      if (this.netMode && i !== this.myIdx) continue; // net: only local player's keys
      if (this.playerAlive && !this.playerAlive[i]) continue;
      const act = this.mapFor(i)[code];
      if (!act) continue;
      this.doAction(this.players[i], act);
    }
  }
```

- [ ] **Step 2: Use `mapFor` in `keyUp`**

Find (`src/Stack Duel.dc.html:1174-1187`):

```js
  keyUp(e) {
    if (!this.players) return;
    for (let i = 0; i < this.players.length; i++) {
      if (this.netMode && i !== this.myIdx) continue;
      if (this.playerAlive && !this.playerAlive[i]) continue;
      const act = this.netMode ? this.NETMAP[e.code] : this.maps[i][e.code];
      if (!act) continue;
      const p = this.players[i];
      if (act === 'left' || act === 'right') {
        const dir = act === 'left' ? -1 : 1;
        if (p.das && p.das.dir === dir) p.das = null;
      } else if (act === 'soft') { p.soft = false; }
    }
  }
```

Replace with:

```js
  keyUp(e) {
    if (!this.players) return;
    for (let i = 0; i < this.players.length; i++) {
      if (this.netMode && i !== this.myIdx) continue;
      if (this.playerAlive && !this.playerAlive[i]) continue;
      const act = this.mapFor(i)[e.code];
      if (!act) continue;
      const p = this.players[i];
      if (act === 'left' || act === 'right') {
        const dir = act === 'left' ? -1 : 1;
        if (p.das && p.das.dir === dir) p.das = null;
      } else if (act === 'soft') { p.soft = false; }
    }
  }
```

- [ ] **Step 3: Manual verification**

Open the file in a browser, clear `localStorage`, click **SOLO MARATHON**. During the countdown/play:

- Press `ArrowLeft`/`ArrowRight` — the piece moves.
- Press `ArrowUp` — the piece rotates.
- Press `ArrowDown` — the piece soft-drops (faster fall while held).
- Press `Space` — the piece hard-drops immediately.
- Press `ShiftRight` — the piece holds/swaps.
- Press `KeyW`/`KeyA`/`KeyS`/`KeyD` (P1's old scheme) — these should now do **nothing** in solo (confirms the redirect to `maps[2]`, not a fallback to both).

Then start a **local 2-player match** (`START LOCAL MATCH`) and confirm both P1 (WASD+Space) and P2 (Arrows+Enter) keys still work exactly as before — this path doesn't go through the solo redirect (`playerCount` is `2`).

- [ ] **Step 4: Commit**

```bash
cd "/home/freax/repos/github/freaxnx01/public/game-stack-duel"
git add "src/Stack Duel.dc.html"
git commit -m "fix(controls): route solo gameplay input through mapFor"
```

---

### Task 4: Fix the on-screen hint text under the solo board

**Files:**
- Modify: `src/Stack Duel.dc.html:1463-1466` (`hintStr`)

**Interfaces:**
- Consumes: `this.mapFor(i)` (Task 2), `this.keyForMap(m, action)` (Task 2).
- Produces: `p1Hint` (already an existing `renderVals()` output) now correctly reflects solo's Arrow+Space scheme when `playerCount === 1`, instead of always showing P1's WASD scheme.

- [ ] **Step 1: Route `hintStr` through `mapFor`**

Find (`src/Stack Duel.dc.html:1463-1466`):

```js
    const hintStr = (i) => {
      const k = (a) => this.keyFor(i, a);
      return k('left') + ' ' + k('right') + ' move · ' + k('rot') + ' rotate · ' + k('soft') + ' soft · ' + k('hard') + ' hard · ' + k('hold') + ' hold';
    };
```

Replace with:

```js
    const hintStr = (i) => {
      const m = this.mapFor(i);
      const k = (a) => this.keyForMap(m, a);
      return k('left') + ' ' + k('right') + ' move · ' + k('rot') + ' rotate · ' + k('soft') + ' soft · ' + k('hard') + ' hard · ' + k('hold') + ' hold';
    };
```

(`p1Hint: hintStr(0)`, `p2Hint: hintStr(1)`, `p3Hint: hintStr(2)` — all three call sites are unchanged; only `hintStr(0)`'s resolved map changes, and only when `playerCount === 1`.)

- [ ] **Step 2: Manual verification**

Open the file in a browser, clear `localStorage`, click **SOLO MARATHON**. The hint text under the board should read `← → move · ↑ rotate · ↓ soft · SPACE hard · R-SHIFT hold` — not the old `A D move · W rotate · S soft · SPACE hard · L-SHIFT hold`.

Start a local 2-player match and confirm P1's hint still reads the WASD scheme and P2's still reads the Arrow+Enter scheme (unaffected).

- [ ] **Step 3: Commit**

```bash
cd "/home/freax/repos/github/freaxnx01/public/game-stack-duel"
git add "src/Stack Duel.dc.html"
git commit -m "fix(controls): show solo's own keymap in the on-screen hint"
```

---

### Task 5: SOLO panel in the CONTROLS screen

**Files:**
- Modify: `src/Stack Duel.dc.html:1450-1483` (`renderVals` — add `soloKeys`/`editSolo`)
- Modify: `src/Stack Duel.dc.html:239-269` (CONTROLS template — add third column)

**Interfaces:**
- Produces: `renderVals()` output gains `soloKeys` (array, same shape as `p1Keys`/`p2Keys`) and `editSolo` (array, same shape as `edit1`/`edit2`), both built from index `2`.
- Consumes: `keyRows(i)`, `editRows(i)` (existing, already index-generic — no changes needed to their definitions), `this.startListen`, `this.assignKey` (existing, already index-generic per Task 1).

- [ ] **Step 1: Add `soloKeys`/`editSolo` to `renderVals()`**

Find (`src/Stack Duel.dc.html:1481-1483`):

```js
      p1Hint: hintStr(0), p2Hint: hintStr(1),
      p1Keys: keyRows(0), p2Keys: keyRows(1),
      edit1: editRows(0), edit2: editRows(1),
```

Replace with:

```js
      p1Hint: hintStr(0), p2Hint: hintStr(1),
      p1Keys: keyRows(0), p2Keys: keyRows(1), soloKeys: keyRows(2),
      edit1: editRows(0), edit2: editRows(1), editSolo: editRows(2),
```

- [ ] **Step 2: Add the third CONTROLS panel to the template**

Find (`src/Stack Duel.dc.html:239-269`, the full `showControlsEditor` block):

```html
            <sc-if value="{{ showControlsEditor }}" hint-placeholder-val="{{ false }}">
              <div style="display: flex; flex-direction: column; align-items: center; gap: 16px">
                <div style="font-size: 30px; font-weight: 800; letter-spacing: 0.05em">CONTROLS</div>
                <div style="font-size: 13px; color: #6E6E68; line-height: 1.5">Click a slot, then press the key you want. ESC cancels. M / P are reserved.</div>
                <div style="display: flex; gap: 40px; margin-top: 4px">
                  <div style="display: flex; flex-direction: column; gap: 9px; align-items: flex-start">
                    <div style="display: flex; align-items: center; gap: 7px; margin-bottom: 3px">
                      <div style="width: 8px; height: 8px; background: #4E7FBE"></div>
                      <div style="font-size: 11px; font-weight: 700; letter-spacing: 0.14em">{{ p1NameU }}</div>
                    </div>
                    <sc-for list="{{ edit1 }}" as="d" hint-placeholder-count="6">
                      <div style="display: flex; align-items: center; gap: 12px; width: 200px; justify-content: space-between"><span style="font-size: 11px; color: #6E6E68; letter-spacing: 0.08em">{{ d.lab }}</span><button onClick="{{ d.onClick }}" style="font-family: ui-monospace, 'SF Mono', Menlo, monospace; font-size: 11px; background: {{ d.bg }}; border: 1px solid {{ d.bd }}; border-radius: 3px; padding: 4px 10px; color: {{ d.fg }}; cursor: pointer; min-width: 78px">{{ d.disp }}</button></div>
                    </sc-for>
                  </div>
                  <div style="width: 1px; background: #E0E0DA"></div>
                  <div style="display: flex; flex-direction: column; gap: 9px; align-items: flex-start">
                    <div style="display: flex; align-items: center; gap: 7px; margin-bottom: 3px">
                      <div style="width: 8px; height: 8px; background: #C97A3D"></div>
                      <div style="font-size: 11px; font-weight: 700; letter-spacing: 0.14em">{{ p2NameU }}</div>
                    </div>
                    <sc-for list="{{ edit2 }}" as="d" hint-placeholder-count="6">
                      <div style="display: flex; align-items: center; gap: 12px; width: 200px; justify-content: space-between"><span style="font-size: 11px; color: #6E6E68; letter-spacing: 0.08em">{{ d.lab }}</span><button onClick="{{ d.onClick }}" style="font-family: ui-monospace, 'SF Mono', Menlo, monospace; font-size: 11px; background: {{ d.bg }}; border: 1px solid {{ d.bd }}; border-radius: 3px; padding: 4px 10px; color: {{ d.fg }}; cursor: pointer; min-width: 78px">{{ d.disp }}</button></div>
                    </sc-for>
                  </div>
                </div>
                <div style="display: flex; gap: 12px; margin-top: 6px">
                  <button onClick="{{ resetKeys }}" style="font: inherit; font-size: 11px; font-weight: 700; letter-spacing: 0.14em; background: #FFFFFF; color: #3A3A36; border: 1px solid #D8D8D1; padding: 11px 20px; cursor: pointer; border-radius: 2px" style-hover="border-color: #B9B9B2">RESET TO DEFAULTS</button>
                  <button onClick="{{ backToMenu }}" style="font: inherit; font-size: 11px; font-weight: 700; letter-spacing: 0.16em; background: #1D1D1B; color: #F6F6F3; border: none; padding: 11px 24px; cursor: pointer; border-radius: 2px" style-hover="background: #3A3A36">DONE</button>
                </div>
              </div>
            </sc-if>
```

Replace with:

```html
            <sc-if value="{{ showControlsEditor }}" hint-placeholder-val="{{ false }}">
              <div style="display: flex; flex-direction: column; align-items: center; gap: 16px">
                <div style="font-size: 30px; font-weight: 800; letter-spacing: 0.05em">CONTROLS</div>
                <div style="font-size: 13px; color: #6E6E68; line-height: 1.5">Click a slot, then press the key you want. ESC cancels. M / P are reserved.</div>
                <div style="display: flex; gap: 40px; margin-top: 4px">
                  <div style="display: flex; flex-direction: column; gap: 9px; align-items: flex-start">
                    <div style="display: flex; align-items: center; gap: 7px; margin-bottom: 3px">
                      <div style="width: 8px; height: 8px; background: #4E7FBE"></div>
                      <div style="font-size: 11px; font-weight: 700; letter-spacing: 0.14em">{{ p1NameU }}</div>
                    </div>
                    <sc-for list="{{ edit1 }}" as="d" hint-placeholder-count="6">
                      <div style="display: flex; align-items: center; gap: 12px; width: 200px; justify-content: space-between"><span style="font-size: 11px; color: #6E6E68; letter-spacing: 0.08em">{{ d.lab }}</span><button onClick="{{ d.onClick }}" style="font-family: ui-monospace, 'SF Mono', Menlo, monospace; font-size: 11px; background: {{ d.bg }}; border: 1px solid {{ d.bd }}; border-radius: 3px; padding: 4px 10px; color: {{ d.fg }}; cursor: pointer; min-width: 78px">{{ d.disp }}</button></div>
                    </sc-for>
                  </div>
                  <div style="width: 1px; background: #E0E0DA"></div>
                  <div style="display: flex; flex-direction: column; gap: 9px; align-items: flex-start">
                    <div style="display: flex; align-items: center; gap: 7px; margin-bottom: 3px">
                      <div style="width: 8px; height: 8px; background: #C97A3D"></div>
                      <div style="font-size: 11px; font-weight: 700; letter-spacing: 0.14em">{{ p2NameU }}</div>
                    </div>
                    <sc-for list="{{ edit2 }}" as="d" hint-placeholder-count="6">
                      <div style="display: flex; align-items: center; gap: 12px; width: 200px; justify-content: space-between"><span style="font-size: 11px; color: #6E6E68; letter-spacing: 0.08em">{{ d.lab }}</span><button onClick="{{ d.onClick }}" style="font-family: ui-monospace, 'SF Mono', Menlo, monospace; font-size: 11px; background: {{ d.bg }}; border: 1px solid {{ d.bd }}; border-radius: 3px; padding: 4px 10px; color: {{ d.fg }}; cursor: pointer; min-width: 78px">{{ d.disp }}</button></div>
                    </sc-for>
                  </div>
                  <div style="width: 1px; background: #E0E0DA"></div>
                  <div style="display: flex; flex-direction: column; gap: 9px; align-items: flex-start">
                    <div style="display: flex; align-items: center; gap: 7px; margin-bottom: 3px">
                      <div style="width: 8px; height: 8px; background: #5A9E6F"></div>
                      <div style="font-size: 11px; font-weight: 700; letter-spacing: 0.14em">SOLO</div>
                    </div>
                    <sc-for list="{{ editSolo }}" as="d" hint-placeholder-count="6">
                      <div style="display: flex; align-items: center; gap: 12px; width: 200px; justify-content: space-between"><span style="font-size: 11px; color: #6E6E68; letter-spacing: 0.08em">{{ d.lab }}</span><button onClick="{{ d.onClick }}" style="font-family: ui-monospace, 'SF Mono', Menlo, monospace; font-size: 11px; background: {{ d.bg }}; border: 1px solid {{ d.bd }}; border-radius: 3px; padding: 4px 10px; color: {{ d.fg }}; cursor: pointer; min-width: 78px">{{ d.disp }}</button></div>
                    </sc-for>
                  </div>
                </div>
                <div style="display: flex; gap: 12px; margin-top: 6px">
                  <button onClick="{{ resetKeys }}" style="font: inherit; font-size: 11px; font-weight: 700; letter-spacing: 0.14em; background: #FFFFFF; color: #3A3A36; border: 1px solid #D8D8D1; padding: 11px 20px; cursor: pointer; border-radius: 2px" style-hover="border-color: #B9B9B2">RESET TO DEFAULTS</button>
                  <button onClick="{{ backToMenu }}" style="font: inherit; font-size: 11px; font-weight: 700; letter-spacing: 0.16em; background: #1D1D1B; color: #F6F6F3; border: none; padding: 11px 24px; cursor: pointer; border-radius: 2px" style-hover="background: #3A3A36">DONE</button>
                </div>
              </div>
            </sc-if>
```

(`SOLO` uses `#5A9E6F` — the same green already used for the `SOLO MARATHON` menu button and the 3-player-mode P3 accent color, kept consistent within this file. It reads "SOLO", not "P3" — this repo already uses "P3" for the third *online* player in 3-player P2P mode, and reusing that label here for the unrelated solo keymap would be confusing.)

- [ ] **Step 3: Manual verification**

Open the file in a browser, clear `localStorage`, click **CONTROLS** from the main menu. Confirm three panels appear side by side: `{{ p1NameU }}` (blue dot), `{{ p2NameU }}` (orange dot), and `SOLO` (green dot) — each listing MOVE LEFT/RIGHT/ROTATE/SOFT DROP/HARD DROP/HOLD with their bound keys, SOLO showing `←`/`→`/`↑`/`↓`/`SPACE`/`R-SHIFT`.

Click the SOLO panel's HARD DROP slot, press `KeyJ` — the slot should update to show `J`. Click DONE, then re-open CONTROLS — the SOLO HARD DROP slot should still show `J` (persisted), and P1/P2 slots should be untouched. Click **RESET TO DEFAULTS** — all three panels (including SOLO) return to their default keys.

Then start **SOLO MARATHON** and confirm the remapped key (`J` in this example) now hard-drops instead of `Space` — proves the CONTROLS panel and the live gameplay path (`mapFor`, Task 3) read the same `maps[2]`.

- [ ] **Step 4: Commit**

```bash
cd "/home/freax/repos/github/freaxnx01/public/game-stack-duel"
git add "src/Stack Duel.dc.html"
git commit -m "feat(controls): add SOLO panel to the controls screen"
```

---

### Task 6: CHANGELOG entry

**Files:**
- Modify: `CHANGELOG.md`

**Interfaces:** none — documentation only.

- [ ] **Step 1: Add an entry under `[Unreleased]`**

Find (`CHANGELOG.md`):

```markdown
## [Unreleased]

### Added

- 3-player P2P mode: host-relay star topology, sequential invite-code exchange for two guests, random-opponent garbage targeting, and last-player-standing round wins.
```

Replace with:

```markdown
## [Unreleased]

### Added

- 3-player P2P mode: host-relay star topology, sequential invite-code exchange for two guests, random-opponent garbage targeting, and last-player-standing round wins.
- Solo Marathon: dedicated Arrow Keys + Space default keymap (previously silently reused, and non-functional under, Player 1's local-hot-seat keys), independently remappable via a new SOLO panel in Controls.
```

- [ ] **Step 2: Commit**

```bash
cd "/home/freax/repos/github/freaxnx01/public/game-stack-duel"
git add CHANGELOG.md
git commit -m "docs(changelog): note solo marathon dedicated keymap"
```

---

### Task 7: Static self-review, re-bundle `index.html`, and open the PR

**Files:**
- Read: full diff of Tasks 1–6
- Modify: `index.html` (re-bundled from `src/Stack Duel.dc.html`, per Global Constraints)

> Verification here is a **static read of the diff**, cross-checked against Tasks 1–6, followed by an actual re-bundle (this is a manual merge, not a missing-tool skip — see Global Constraints).

- [ ] **Step 1: Static self-review of every task's diff**

Re-read `git diff main..HEAD -- "src/Stack Duel.dc.html"` end to end and check it against Tasks 1–6, one at a time:

- Every `Find`/`Replace` block landed exactly as specified (no partial edits, no leftover old code).
- No accidental double-application and no unrelated lines changed.
- `git log --oneline main..HEAD` shows one commit per task.

Fix anything that doesn't match before proceeding.

- [ ] **Step 2: Re-bundle `index.html`**

Follow the manual-merge process from Global Constraints: copy `src/Stack Duel.dc.html`'s full content into `index.html`, then re-apply the 4 known additions (favicon link, `version.js` include, `game-nav` block, version-badge script). Verify with:

```bash
cd "/home/freax/repos/github/freaxnx01/public/game-stack-duel"
diff "src/Stack Duel.dc.html" index.html
```

The output should show **only** the 4 known additions — nothing from Tasks 1–6 should appear as a diff (meaning the merge picked up all the source changes correctly).

If a real browser + display is available in this environment, also do a quick smoke test of the merged `index.html` (not just the source file): serve it with `python3 -m http.server` and repeat Task 5's CONTROLS + SOLO MARATHON verification against `http://localhost:<port>/index.html` this time, confirming zero console errors.

Commit:

```bash
git add index.html
git commit -m "fix: rebundle index.html with solo dedicated keymap"
```

- [ ] **Step 3: Push the branch and open the draft PR**

```bash
git push -u origin HEAD
gh pr create --draft \
  --title "feat(controls): default Solo Marathon to Arrow Keys + Space" \
  --body "Closes #4

## Summary
- Solo Marathon gets its own default keymap (Arrow Keys + Space + R-Shift hold) via a new 3rd \`maps\` slot, instead of silently reusing (and being non-functional under) Player 1's WASD scheme
- New always-visible SOLO panel in the CONTROLS screen, independently remappable and persisted
- On-screen hint text under the solo board now reflects the actual solo keymap
- Existing saved keymaps migrate automatically (2-entry blobs get the solo default appended, P1/P2 remaps untouched)
- Local hot-seat (2P) and online (P2P) keymaps are unaffected — every change is gated on \`playerCount === 1\`

## Self-review performed
- Diffed every task's changes in \`src/Stack Duel.dc.html\` against the plan's Find/Replace blocks — all landed as specified, one commit per task
- \`index.html\` re-bundled and diff-verified against \`src/Stack Duel.dc.html\` (only the known 4 additions differ)

## Still needed before merge (human/browser step)
- [ ] Open the game in a real browser and play through: SOLO MARATHON (Arrow Keys + Space move/rotate/drop/hold correctly, hint text matches), CONTROLS (SOLO panel remaps independently of P1/P2, RESET TO DEFAULTS resets all three), local 2P match (unaffected)"
```

This is the task that actually matters for pipeline recovery: **do not end the run without having pushed the branch and opened this PR** — a finished diff that's never pushed is indistinguishable from no progress at all once the sandbox is torn down.
