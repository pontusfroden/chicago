# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project overview

A single-file PWA for the Swedish card game Chicago. Everything — HTML, CSS, and JS — lives in `index.html`. There is no build step, no package manager, no test suite, and no bundler. Open the file directly in a browser to run it.

## Architecture

The file is structured in three main sections:

1. **HTML/CSS** (top) — Inline `<style>` block with CSS custom properties for theming (dark default, `.light` override). Decorative background playing cards are static SVG elements.

2. **Static data** (start of `<script>`) — `HANDS` array defines all poker hands with their point values. `SUIT_RANK`, `SUIT_SYM`, card/deck helpers (`newDeck`, `shuffleDeck`, `cardHTML`) are defined here.

3. **App logic** (rest of `<script>`) — Two completely separate game modes share the same `render()` dispatcher and `<div id="app">` mount point.

### Two modes, two state objects

| Mode | State var | Prefix | Purpose |
|------|-----------|--------|---------|
| **Räkna** | `game`, `entry` | none | Manual score tracker — players enter hands and Chicago results each round |
| **Spela** | `sGame` | `s` | Full digital play — cards dealt, tricks played, hands evaluated automatically |

`mode` (string) drives which renderer is called. Navigation: `null` → home, `'rakna_rules'` → rule variant select, `'rakna'` → score tracker setup/scoreboard, `'spela'` → card game.

`rules` (`'standard'` | `'pers'`) only affects **Räkna** mode — specifically whether the 15-point minimum applies to Chicago declarations.

### Räkna (score tracker)

- `game` persists to `localStorage` via `save()` on every mutation.
- `entry` is the in-progress round overlay (`renderRoundEntry()`). It is `null` when no round is open.
- Scoring: `entryPts(i)` computes per-player delta — Phase 1+2 hands, best Phase 3 hand, last trick +5, Chicago ±15.
- Royal Straight Flush triggers instant win (score set to 52).

### Spela (card game)

`sGame.phase` is the state machine driver. Phases in order:

`deal_det` → `deal_det_reveal` → `view_hands` → `exchange` → `phase_result` → `chicago_decl` → `trick` → `round_summary` → (loop or `game_over`)

- All `sGame` functions are prefixed with `s` (e.g. `sPlayCard`, `sResolveTrick`, `sStartTrick`).
- `sGame` is **not** persisted to localStorage — losing the tab loses the game.
- The 15-point Chicago threshold lives in `sStartChicagoDecl()` and `sChicagoNo()` — these are independent of `rules`.

### Rendering pattern

All rendering is full DOM replacement via `innerHTML`. `render()` is the single entry point — it checks `mode`, `game`, and `entry` in sequence and calls the right `render*()` function. All event handlers are inline `onclick` strings calling global functions.

### Card animation (Spela trick phase)

When a card is played (`sPlayCard`):
1. Positions of existing fan cards are captured before `render()` (FLIP snapshot).
2. After `render()`, synchronously: new card hidden (`visibility:hidden`), existing cards frozen at old positions via `transform`.
3. In double-`requestAnimationFrame`: existing cards transition to correct positions; a `position:fixed` ghost card flies from the hand rect to the table rect, then is removed when `transitionend` fires.

Cards played in a trick accumulate as a fan in each player's seat (`sAllCards(playerIdx)` collects from `completedTricks` + current `trickPlays`, with a guard against double-counting when `trickState === 'won'`).

The game table is rendered as two SVG `<polygon>` elements (wood frame + felt) with HTML seat elements positioned absolutely on top — **not** using CSS `clip-path`, which would clip all descendant elements.

### PWA

Icons are canvas-generated at runtime and injected as blob URLs. The manifest is also a blob URL injected into `<link id="pwa-manifest">`. Wake Lock is requested when Spela starts and released on exit.

## GitHub

Remote: `https://github.com/pontusfroden/chicago` — branch `master`. Push directly (no PRs).
