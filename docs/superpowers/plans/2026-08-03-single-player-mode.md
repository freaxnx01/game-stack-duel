# Solo Marathon Mode Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add a one-board "SOLO MARATHON" mode to Stack Duel — no opponent, no garbage, speed ramps every 10 lines as today, ends on top-out, with a persisted best score.

**Architecture:** `playerCount = 1` reuses the existing generic round/physics/HUD loop unchanged. Only the two-player-specific paths — end-of-round detection, the versus HUD half of the template, and the phase-transition/copy logic — get a new `solo`-aware branch alongside the existing 2/3-player branches.

**Tech Stack:** Single-file "DC" component (`src/Stack Duel.dc.html`, Handlebars-style `{{ }}` interpolation + `sc-if`/`sc-for` directives, no build step other than the repo's own dc-tool bundler). No test framework — this repo verifies by opening the game in a browser.

## Global Constraints

- Repo is a bundled (dc-tool) game: edit **only** `src/Stack Duel.dc.html`. Never hand-edit the generated `index.html` / `src/support.js` (CLAUDE.md, "Bundled game (dc-tool)").
- No automated test suite exists. Every task's verification step is: open `src/Stack Duel.dc.html` directly in a browser (per README: "Open the `.dc.html` directly in a browser to run from source") and manually exercise the described behavior.
- `localStorage` key convention: mirror the existing `stackDuelKeys` pattern (JSON-encoded, defensive `try/catch` parse, safe default on any failure) — see `loadMaps()`/`saveMaps()` at `src/Stack Duel.dc.html:481-494`.
- Keep versus (local 2P/3P, online P2P) behavior byte-for-byte unchanged — every new branch must be additive and gated on `playerCount === 1` / a new `ph === 'soloover'` check, never a change to the existing 2/3-player code paths.
- Commit after every task.

---

### Task 1: Menu entry point — `openSolo()` handler + SOLO MARATHON button

**Files:**
- Modify: `src/Stack Duel.dc.html:550` (add handler near `chooseHostPlayerCount`)
- Modify: `src/Stack Duel.dc.html:1390-1402` (`btn` style map + `menuButtons` array)
- Modify: `src/Stack Duel.dc.html:28` (menu screen's static two-player-only tagline — leave; out of scope) — **no change here**, listed only to confirm it was checked and is unrelated to the `menuButtons` array being modified.

**Interfaces:**
- Produces: `this.openSolo` — instance method, no args, no return value. Sets `this.playerCount = 1` and calls `this.startRound()`.
- Consumes: `this.startRound()` (existing, unchanged), `this.playerCount` (existing field, default `2`).

- [ ] **Step 1: Add the `openSolo` handler**

In `src/Stack Duel.dc.html`, immediately after the `chooseHostPlayerCount` method (around line 550):

```js
  chooseHostPlayerCount = (n) => { this.playerCount = n; this.setState({}); };
  openSolo = () => {
    this.ensureAC();
    this.playerCount = 1;
    this.startRound();
  };
```

**Step 2: Add a green button style and the menu button**

Find:

```js
    const btn = {
      dark: btnBase + '; background: #1D1D1B; color: #F6F6F3',
      blue: btnBase + '; background: #4E7FBE; color: #FFFFFF',
      orange: btnBase + '; background: #C97A3D; color: #FFFFFF',
      ghost: btnBase + '; background: #FFFFFF; color: #3A3A36; border: 1px solid #D8D8D1'
    };
    const menuButtons = [
      { lab: 'START LOCAL MATCH', style: btn.dark, onClick: this.advance },
      { lab: 'HOST ONLINE GAME', style: btn.blue, onClick: this.openHost },
      { lab: 'JOIN ONLINE GAME', style: btn.orange, onClick: this.openJoin },
      { lab: 'CONTROLS', style: btn.ghost, onClick: this.openControls }
    ];
```

Replace with:

```js
    const btn = {
      dark: btnBase + '; background: #1D1D1B; color: #F6F6F3',
      blue: btnBase + '; background: #4E7FBE; color: #FFFFFF',
      orange: btnBase + '; background: #C97A3D; color: #FFFFFF',
      green: btnBase + '; background: #5A9E6F; color: #FFFFFF',
      ghost: btnBase + '; background: #FFFFFF; color: #3A3A36; border: 1px solid #D8D8D1'
    };
    const menuButtons = [
      { lab: 'START LOCAL MATCH', style: btn.dark, onClick: this.advance },
      { lab: 'SOLO MARATHON', style: btn.green, onClick: this.openSolo },
      { lab: 'HOST ONLINE GAME', style: btn.blue, onClick: this.openHost },
      { lab: 'JOIN ONLINE GAME', style: btn.orange, onClick: this.openJoin },
      { lab: 'CONTROLS', style: btn.ghost, onClick: this.openControls }
    ];
```

**Step 3: Manual verification**

Open `src/Stack Duel.dc.html` in a browser. On the main menu, confirm a green "SOLO MARATHON" button appears between "START LOCAL MATCH" and "HOST ONLINE GAME". Click it — the countdown ("3, 2, 1, GO") should play and a single board should start dropping pieces (the second board will still be visible and playable with P2's keys until Task 6 hides it — that's expected at this point in the plan).

**Step 4: Commit**

```bash
cd /tmp/stack-duel-check
git add "src/Stack Duel.dc.html"
git commit -m "feat: add solo marathon menu entry point"
```

---

### Task 2: Solo end-of-round detection + result state

**Files:**
- Modify: `src/Stack Duel.dc.html:406-413` (`state` — add `soloResult`)
- Modify: `src/Stack Duel.dc.html:1006-1025` (`markOut` / `_endRound` — add `_endSolo` branch)

**Interfaces:**
- Consumes: `this.playerCount` (Task 1), `this.players[0]` (existing per-player working object: `{ idx, score, lines, level, incoming, out, ... }`), `this.updateSoloBest(lines, score)` (Task 3 — **stub it as returning `false` in this task**, Task 3 replaces the stub).
- Produces: `this._endSolo()` — instance method, no args. `state.soloResult` — `{ lines: number, score: number, level: number, beat: boolean }`, read by Task 5's `renderVals()`. `state.phase` value `'soloover'`, consumed by Task 4 (`advance()`, `keyDown()`) and Task 5 (`renderVals()`).

- [ ] **Step 1: Add `soloResult` to initial state**

Find:

```js
  state = {
    phase: 'menu', round: 1, wins: [0, 0], lastWinner: 0, countdown: '3', muted: false,
    keyListen: null, netStatus: '', netStatusKind: '',
    players: [
      { score: 0, lines: 0, level: 1, incoming: 0, out: false },
      { score: 0, lines: 0, level: 1, incoming: 0, out: false }
    ]
  };
```

Replace with:

```js
  state = {
    phase: 'menu', round: 1, wins: [0, 0], lastWinner: 0, countdown: '3', muted: false,
    keyListen: null, netStatus: '', netStatusKind: '',
    soloResult: { lines: 0, score: 0, level: 1, beat: false },
    players: [
      { score: 0, lines: 0, level: 1, incoming: 0, out: false },
      { score: 0, lines: 0, level: 1, incoming: 0, out: false }
    ]
  };
```

**Step 2: Add a temporary `updateSoloBest` stub (replaced in Task 3)**

Immediately before `markOut` (around line 1006), add:

```js
  updateSoloBest(lines, score) { return false; } // TEMP stub — replaced in Task 3
```

**Step 3: Branch `markOut` into `_endSolo` for solo mode, add `_endSolo`**

Find:

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

Replace with:

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

**Step 4: Manual verification**

Open `src/Stack Duel.dc.html` in a browser, click "SOLO MARATHON", play until the board tops out. The overlay should reappear (kicker/title still say whatever `ph === 'roundover'`/default copy shows right now — that's expected, Task 5 fixes the copy) but the game must **not** crash and must **not** show "takes the round" 2-player phrasing crash (open devtools console, confirm no uncaught exceptions).

**Step 5: Commit**

```bash
cd /tmp/stack-duel-check
git add "src/Stack Duel.dc.html"
git commit -m "feat: detect solo game-over and stop the round without a winner"
```

---

### Task 3: Solo best-score persistence

**Files:**
- Modify: `src/Stack Duel.dc.html:442-460` (`componentDidMount` — load best score)
- Modify: `src/Stack Duel.dc.html` (near `loadMaps`/`saveMaps`, ~line 481-494 — add `loadSoloBest`/`saveSoloBest`/`updateSoloBest`, replacing the Task 2 stub)

**Interfaces:**
- Consumes: `localStorage` key `stackDuelSoloBest`.
- Produces: `this.soloBest` — `{ lines: number, score: number }`, read by Task 5's `renderVals()` for the "BEST: N LINES" menu/game-over copy.

- [ ] **Step 1: Add persistence methods, replacing the Task 2 stub**

Find the Task 2 stub:

```js
  updateSoloBest(lines, score) { return false; } // TEMP stub — replaced in Task 3
```

Replace with:

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

**Step 2: Load `this.soloBest` on mount**

Find, inside `componentDidMount()`:

```js
    if (!this.maps) this.maps = this.loadMaps();
    this.hostCode = ''; this.joinAnswer = '';
```

Replace with:

```js
    if (!this.maps) this.maps = this.loadMaps();
    this.soloBest = this.loadSoloBest();
    this.hostCode = ''; this.joinAnswer = '';
```

**Step 3: Manual verification**

Open `src/Stack Duel.dc.html` in a browser with devtools open. Play a solo round, deliberately clear a few lines, then top out. Run `localStorage.getItem('stackDuelSoloBest')` in the console — confirm it now holds `{"lines":N,"score":M}` matching the run. Reload the page and play a shorter run (fewer lines) — confirm `stackDuelSoloBest` is unchanged (not overwritten by a worse run).

**Step 4: Commit**

```bash
cd /tmp/stack-duel-check
git add "src/Stack Duel.dc.html"
git commit -m "feat: persist best solo marathon score to localStorage"
```

---

### Task 4: Phase-transition wiring — `advance()` and `keyDown()`

**Files:**
- Modify: `src/Stack Duel.dc.html:1069-1081` (`advance()`)
- Modify: `src/Stack Duel.dc.html:1110-1113` (`keyDown()` menu/roundover/matchover branch)

**Interfaces:**
- Consumes: `state.phase === 'soloover'` (Task 2).
- Produces: clicking the on-screen "PLAY AGAIN" button (Task 5 wires `advance` to it, same as existing "NEXT ROUND"/"REMATCH" buttons) or pressing Enter/Space on the solo game-over screen both call `this.advance()` → `this.startRound()`, starting a fresh solo round (since `this.playerCount` is still `1` from the previous round).

- [ ] **Step 1: Add the `soloover` branch to `advance()`**

Find:

```js
  advance = () => {
    if (performance.now() - this.overlayAt < 450) return;
    this.ensureAC();
    const ph = this.state.phase;
    if (this.netMode) {
      if (this.myIdx !== 0) return; // only host controls round flow
      if (ph === 'roundover' || ph === 'matchover') this.netAdvance();
      return;
    }
    if (ph === 'menu') { this.setState({ round: 1, wins: this.players.map(() => 0) }); this.startRound(); }
    else if (ph === 'roundover') { this.setState({ round: this.state.round + 1 }); this.startRound(); }
    else if (ph === 'matchover') { this.setState({ round: 1, wins: this.players.map(() => 0) }); this.startRound(); }
  };
```

Replace with:

```js
  advance = () => {
    if (performance.now() - this.overlayAt < 450) return;
    this.ensureAC();
    const ph = this.state.phase;
    if (this.netMode) {
      if (this.myIdx !== 0) return; // only host controls round flow
      if (ph === 'roundover' || ph === 'matchover') this.netAdvance();
      return;
    }
    if (ph === 'menu') { this.setState({ round: 1, wins: this.players.map(() => 0) }); this.startRound(); }
    else if (ph === 'roundover') { this.setState({ round: this.state.round + 1 }); this.startRound(); }
    else if (ph === 'matchover') { this.setState({ round: 1, wins: this.players.map(() => 0) }); this.startRound(); }
    else if (ph === 'soloover') { this.startRound(); }
  };
```

**Step 2: Let Enter/Space advance past the solo game-over screen**

Find:

```js
    if (ph === 'menu' || ph === 'roundover' || ph === 'matchover') {
      if (code === 'Enter' || code === 'Space') this.advance();
      return;
    }
```

Replace with:

```js
    if (ph === 'menu' || ph === 'roundover' || ph === 'matchover' || ph === 'soloover') {
      if (code === 'Enter' || code === 'Space') this.advance();
      return;
    }
```

**Step 3: Manual verification**

Open `src/Stack Duel.dc.html`, play solo to a top-out, then press Space (or Enter) — confirm a new solo round starts immediately (countdown plays again) without needing to click anything.

**Step 4: Commit**

```bash
cd /tmp/stack-duel-check
git add "src/Stack Duel.dc.html"
git commit -m "feat: wire keyboard/advance handling for solo game-over screen"
```

---

### Task 5: Solo game-over copy + guard 2-player stat access

**Files:**
- Modify: `src/Stack Duel.dc.html:1364-1388` (`renderVals()` — `ph` copy branches)
- Modify: `src/Stack Duel.dc.html:1424-1435` (`renderVals()` — return object, `p2Score`/`p2Lines`/`p2Level`)
- Modify: `src/Stack Duel.dc.html:1447` (`showButton` condition)

**Interfaces:**
- Consumes: `state.soloResult` (Task 2), `this.soloBest` (Task 3), `state.phase === 'soloover'`.
- Produces: `overlayKicker`/`overlayTitle`/`overlaySub`/`buttonLabel` values consumed by the existing overlay template (`src/Stack Duel.dc.html:192-230`, unchanged — it already renders whatever `renderVals()` returns).

- [ ] **Step 1: Fix the crash — guard `p2Score`/`p2Lines`/`p2Level` for `playerCount === 1`**

`state.players` has length `this.playerCount` (set by `syncHud()`), so in solo mode `s.players[1]` is `undefined`. Find:

```js
      p1Score: s.players[0].score, p1Lines: s.players[0].lines, p1Level: s.players[0].level,
      p2Score: s.players[1].score, p2Lines: s.players[1].lines, p2Level: s.players[1].level,
```

Replace with:

```js
      p1Score: s.players[0].score, p1Lines: s.players[0].lines, p1Level: s.players[0].level,
      p2Score: s.players[1] ? s.players[1].score : 0,
      p2Lines: s.players[1] ? s.players[1].lines : 0,
      p2Level: s.players[1] ? s.players[1].level : 1,
```

(This mirrors the existing `p3Score`/`p3Lines`/`p3Level` guard pattern a few lines below, at `src/Stack Duel.dc.html:1467-1469`.)

**Step 2: Add the `soloover` copy branch and a menu best-score line**

Find:

```js
    if (ph === 'menu') {
      kicker = 'TWO PLAYER · FIRST TO 2 ROUNDS';
      title = 'STACK DUEL';
      sub = 'Clear lines to dump garbage on your opponent. Play local hot-seat or online against a friend — remap keys under CONTROLS.';
    } else if (ph === 'countdown') {
```

Replace with:

```js
    if (ph === 'menu') {
      kicker = 'TWO PLAYER · FIRST TO 2 ROUNDS';
      title = 'STACK DUEL';
      sub = 'Clear lines to dump garbage on your opponent. Play local hot-seat or online against a friend, or go solo marathon — remap keys under CONTROLS.'
        + (this.soloBest && this.soloBest.lines > 0 ? '  ·  SOLO BEST: ' + this.soloBest.lines + ' LINES' : '');
    } else if (ph === 'soloover') {
      kicker = 'SOLO MARATHON';
      title = 'GAME OVER';
      sub = s.soloResult.lines + ' LINES · LEVEL ' + s.soloResult.level
        + (s.soloResult.beat ? '  ·  NEW BEST!' : '  ·  BEST: ' + this.soloBest.lines + ' LINES');
      button = 'PLAY AGAIN';
    } else if (ph === 'countdown') {
```

**Step 3: Show the game-over button on the solo screen**

Find:

```js
      showButton: (ph === 'roundover' || ph === 'matchover') && !showWaitHost,
```

Replace with:

```js
      showButton: (ph === 'roundover' || ph === 'matchover' || ph === 'soloover') && !showWaitHost,
```

**Step 4: Manual verification**

Open `src/Stack Duel.dc.html`, click "SOLO MARATHON" (this run's `soloBest.lines` should be `0`, so the menu subtitle should **not** show a "SOLO BEST" clause yet). Clear at least one line, then deliberately top out. Confirm the overlay reads "SOLO MARATHON" / "GAME OVER" / "`N` LINES · LEVEL `L` · NEW BEST!" with a "PLAY AGAIN" button. Click "PLAY AGAIN", top out again immediately without clearing lines — confirm this time it shows "· BEST: `N` LINES" (not "NEW BEST!"). Go back to the main menu (reload the page) — confirm the menu subtitle now shows "SOLO BEST: `N` LINES".

**Step 5: Commit**

```bash
cd /tmp/stack-duel-check
git add "src/Stack Duel.dc.html"
git commit -m "feat: add solo game-over copy and fix 2-player stat guard"
```

---

### Task 6: Hide the opponent's board/panel in solo mode

**Files:**
- Modify: `src/Stack Duel.dc.html:34` (outer boards flex container — centering)
- Modify: `src/Stack Duel.dc.html:75-148` (middle round/dots column + P2 board/panel — wrap in `sc-if`)
- Modify: `src/Stack Duel.dc.html:1424-1483` (`renderVals()` — add `showVersusHud`, `boardsJustify`)

**Interfaces:**
- Consumes: `this.playerCount === 1` at render time.
- Produces: `showVersusHud: boolean` (template `sc-if` gate), `boardsJustify: string` (`'center'` or `'flex-start'`), both new keys in the object `renderVals()` returns.

- [ ] **Step 1: Add `showVersusHud` and `boardsJustify` to `renderVals()`**

Find:

```js
      isThreePlayer: this.playerCount === 3,
      boardW, boardH,
```

Replace with:

```js
      isThreePlayer: this.playerCount === 3,
      showVersusHud: this.playerCount !== 1,
      boardsJustify: this.playerCount === 1 ? 'center' : 'flex-start',
      boardW, boardH,
```

**Step 2: Center the boards row**

Find:

```html
    <div style="position: relative">
      <div style="display: flex; gap: 24px; align-items: stretch">
```

Replace with:

```html
    <div style="position: relative">
      <div style="display: flex; gap: 24px; align-items: stretch; justify-content: {{ boardsJustify }}">
```

**Step 3: Wrap the round/dots column + Player 2's board/panel in `sc-if value="{{ showVersusHud }}"`**

This is the contiguous block from the round-column's opening `<div>` (currently line 75) through Player 2's panel's closing `</div>` (currently line 148) — everything between Player 1's board column and the Player-3-only block. Find the opening of the round column:

```html
        <div style="width: 118px; display: flex; flex-direction: column; align-items: center; justify-content: space-between; padding: 30px 0 24px">
          <div style="display: flex; flex-direction: column; align-items: center; gap: 2px">
            <div style="font-size: 10px; font-weight: 600; letter-spacing: 0.2em; color: #9A9A93">ROUND</div>
```

Insert `<sc-if value="{{ showVersusHud }}" hint-placeholder-val="{{ true }}">` immediately before this `<div style="width: 118px; ...">` line.

Then find the closing of Player 2's panel — the block ends right before the Player-3 `sc-if`:

```html
            <div style="display: flex; flex-direction: column; align-items: flex-start; gap: 2px">
              <div style="font-size: 10px; font-weight: 600; letter-spacing: 0.16em; color: #9A9A93">LEVEL</div>
              <div style="font-family: ui-monospace, 'SF Mono', Menlo, Consolas, monospace; font-size: 15px; font-weight: 600">{{ p2Level }}</div>
            </div>
          </div>
        </div>

        <sc-if value="{{ isThreePlayer }}" hint-placeholder-val="{{ false }}">
```

Insert `</sc-if>` immediately after the `</div>` that closes Player 2's panel (the blank line right before `<sc-if value="{{ isThreePlayer }}"`), so the result reads:

```html
            <div style="display: flex; flex-direction: column; align-items: flex-start; gap: 2px">
              <div style="font-size: 10px; font-weight: 600; letter-spacing: 0.16em; color: #9A9A93">LEVEL</div>
              <div style="font-family: ui-monospace, 'SF Mono', Menlo, Consolas, monospace; font-size: 15px; font-weight: 600">{{ p2Level }}</div>
            </div>
          </div>
        </div>
        </sc-if>

        <sc-if value="{{ isThreePlayer }}" hint-placeholder-val="{{ false }}">
```

**Step 4: Manual verification**

Open `src/Stack Duel.dc.html`. Start "SOLO MARATHON" — confirm only Player 1's panel + board is visible, centered on the page (no ROUND counter, no win dots, no Player 2 board/panel). Play to a top-out, confirm the game-over overlay from Task 5 still renders correctly centered over the single board. Then go back to the menu and start "START LOCAL MATCH" — confirm the full 2-player layout (both boards, ROUND counter, win dots) is completely unchanged. Then start a 3-player host game locally (or at minimum inspect that `isThreePlayer` still works) — confirm Player 3's board/panel still appears as before.

**Step 5: Commit**

```bash
cd /tmp/stack-duel-check
git add "src/Stack Duel.dc.html"
git commit -m "feat: show only the solo player's board in solo marathon mode"
```

---

### Task 7: Full regression pass + re-bundle for publish

**Files:** none (verification + build step only)

**Interfaces:** none — this task only exercises the finished feature and prepares it for shipping.

- [ ] **Step 1: Full manual regression pass**

Open `src/Stack Duel.dc.html` in a browser and walk through, in order:

1. Menu → SOLO MARATHON → play, clear lines, top out → confirm game-over copy, best-score tracking (per Task 3/5), PLAY AGAIN restart, and Space/Enter restart (Task 4) all work.
2. Menu → START LOCAL MATCH → play a full best-of-3 with two keyboards (or one person on both keymaps) → confirm garbage rows, round/match-over screens, and win dots are unaffected.
3. Menu → HOST ONLINE GAME (or CONTROLS → remap a key, then RESET TO DEFAULTS) → confirm these screens still open/close correctly and are visually unaffected by the `boardsJustify`/`showVersusHud` changes.
4. Resize the browser window narrow/wide during a solo round — confirm the single centered board doesn't overlap the page edges awkwardly (no functional requirement, just a sanity check).

Fix anything broken before proceeding — do not commit over a known regression.

**Step 2: Re-bundle for the generated files**

Per CLAUDE.md's bundled-game convention, `index.html` and `src/support.js` are generated from `src/Stack Duel.dc.html` and must never be hand-edited. Follow this repo's existing publish workflow (see `README.md`'s "Publishing" section / `CLAUDE.md`) to regenerate `index.html` from the updated source before this change is considered shippable — this plan's tasks only touch the source file.

**Step 3: Update the issue**

Once the regression pass and re-bundle are done, comment on GitHub issue #3 (`freaxnx01/game-stack-duel`) confirming solo marathon mode is implemented and ready for review/publish.
