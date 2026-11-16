# Emojikaku — Game Design Document

**Status of this document:** normative. It describes the game that exists in `index.html`.
Anything the game does *not* do lives in §10 (Future ideas) or §11 (Known gaps) and is
labelled as such. If this document and `index.html` ever disagree, the code is right and
this document is stale.

## 1. Overview

| Item | Detail |
|------|--------|
| **Working title** | *Emojikaku* |
| **Genre** | Puzzle — logic / spatial division |
| **Platform** | Web page. One self-contained `index.html`; opens by double-clicking the file. Desktop (mouse + keyboard) and mobile (touch) browsers. |
| **Distribution** | Local file only. No PWA manifest, no service worker, no app store build, no server, no build step, no dependencies. |
| **Screen sizes** | Any viewport from ~320 px wide upward; the layout reflows at 700 px (side-by-side → stacked). |
| **Single-player / Multiplayer** | Single-player. No accounts, no sessions, no leaderboards. |
| **Session length** | One puzzle at a time; a puzzle is not saved and cannot be resumed after leaving the game screen. |

### 1.1 Elevator pitch

A standard game of Shikaku — a rectangle is subdivided into rectangular regions, each containing exactly one number whose value equals the region's area — reimagined with emojis in place of numbers. Emojis are assigned randomly to values per puzzle, serving purely as obfuscation. Players must deduce which emoji maps to which number by analysing region sizes alone.

### 1.2 Design principles (as built)

1. **Every puzzle is provably unique.** No puzzle may require guessing. This is enforced by a prover at generation time, not by level design (§3.3).
2. **Every puzzle has a provable first move.** The player is never facing a wall (§3.3).
3. **Difficulty is measured, not assumed.** The prover's search cost is the dial; board size is not (§4.1).
4. **Emojis obfuscate; they never hint.** No semantic relationship between glyph and value (§2.5).
5. **Minimal aids.** Undo, Clear, Reveal and a passive guesses readout. No hints (§10.2).
6. **Nothing to install or launch.** A single HTML file that runs offline.

## 2. Core Gameplay

### 2.1 Shikaku rules (reference)

1. A rectangular grid has some cells labelled with an emoji (**one anchor per region**, placed anywhere inside the region — not pinned to the top-left). Other cells are blank.
2. Divide the entire grid into rectangular regions so that:
   - Every region is a solid rectangle (width × height ≥ 1).
   - Each region contains emojis of one value only, and at least one anchor cell.
3. The area (number of cells) of a region must equal the hidden value behind its emoji.

### 2.2 Emoji twist — how it differs

| Standard Shikaku | Emojikaku |
|-----------------|---------------|
| Numbers are visible in **one cell per region** (the anchor). | One emoji replaces the number **per region**, shown on a single anchor cell placed at a random cell *within* the region. All other cells in that region are blank. |
| The player reads the numbers directly from anchor cells. | The mapping from emojis → values is **hidden and randomized per puzzle**. Players must deduce which emoji maps to which area by drawing regions. |
| A region is legal once it has the right area around its number. | A region is legal-looking as soon as it is a rectangle covering at least one emoji and no *other* emoji. Correctness is invisible until the board is complete (§2.4). |

### 2.3 Drawing regions — the interaction grammar

Input is pointer events (mouse, touch, pen) on the grid container. A press moves past a 20 px threshold before it counts as a drag; below that it is a tap.

| Gesture | Result |
|---------|--------|
| **Drag from a cell** (≥ 20 px) | A live preview rectangle shows the bounding box of start cell → current cell. On release the rectangle is committed if it passes the legality rules below; otherwise the release is silently ignored (nothing is drawn, no message). |
| **Tap an empty, unoccupied cell that holds an emoji** | Immediately creates a 1×1 region there and runs the completion check. |
| **Tap a blank cell that is not in a region** | Nothing. |
| **Tap inside an existing region** (on the region fill, or on the cell underneath it) | The whole region is removed. This is the primary correction gesture. |
| **Press on an existing region and drag away** | The region is removed and the drag is discarded — a drag never re-draws over an existing region. |
| **`Ctrl`/`Cmd` + `Z`** | Undo, same as the ↩ Undo button. |
| **`Esc`** | Effectively nothing (it hides an unused panel — §11). |

**Legality rules applied on release** (`commitDrawnRegion`):

- The selected cells must form a perfect rectangle (cell count = bounding-box area).
- The rectangle must not overlap any existing region.
- It must contain at least one emoji.
- It must not contain two **different** emojis — several cells showing the *same* emoji may share one region.

Those last two rules are exactly the freedom the player has, so the uniqueness prover (§3.3) has to rule the same freedom out; the generator and the prover read one shared "every legal rectangle" index so they can never disagree about what a player is allowed to draw.

**Feedback while drawing:** the preview is a translucent accent-coloured rectangle with a 2.5 px accent border. It does not indicate legality, does not snap, and does not show the area.

### 2.4 Completion check

- The puzzle is validated **only when every cell of the grid is covered** by player-drawn regions. There is no incremental or per-region validation.
- When the last cell is placed, each region is checked: it must be a rectangle, all of its cells must share one solution value, and its area must equal that value.
- **Correct** → 🎉 "Puzzle Solved!" screen with solve time, best-time marker, ▶ Next Puzzle and Menu buttons. The solve is recorded in stats (§6.1).
- **Incorrect** → 😕 "Not Quite Right" screen. The button offers **↩ Resume**, which keeps the regions that were correct, deletes the ones that were wrong, resets the undo history, and restarts the timer. The removed wrong regions are marked on the grid with a 4 px red outline (white stroke in dark mode); that outline is dismissed by the next interaction with the grid.
- Because the generator proves uniqueness, the "Not Quite Right" path is only reachable through player error, and it is not recorded anywhere.

### 2.5 Emoji–value mapping

- Region values can be **any positive integer** (real Shikaku values). Multiple regions may share a value; equal values share one emoji, different values never do.
- At generation each distinct value gets a **fully random** emoji from a 60-glyph pool: 10 faces, 10 animals, 10 foods, 10 nature, 10 objects, 10 hearts/stars. Assignment is a shuffled permutation — no semantic filtering. Measured distinct values per puzzle: 2–5 (Easy), 3–6 (Medium), 4–7 (Hard), 5–9 (Expert), so the pool never runs short.
- The mapping is consistent for the whole puzzle.

**What is and is not printed:** the hidden value behind an emoji is never shown as a number — not during play, not on the success screen, and never as a label next to an emoji. What *is* printed is the **area of every player-drawn region**: a small centred number (only when the area is greater than 1, so 1×1 regions show nothing). That number is the player's own rectangle size, which is the tool they use to reason about values. After Reveal, the solution regions are drawn as player regions, so their area labels coincide with the values.

> **Design principle:** Emojis are used to *obfuscate* numbers, not to hint at them. Players must rely entirely on Shikaku logic (region sizes) to deduce which emoji corresponds to which value. The visual randomness of emojis adds a layer of cognitive dissonance that makes solved grids feel surprising ("Wait, the poop emoji was 4?!").

## 3. Puzzle Generation

Generation happens synchronously in the browser at the moment Play is pressed, and always yields a puzzle that has been *proven* to have exactly one solution.

### 3.1 Generation algorithm (top-down)

```
GeneratePuzzle(difficulty):
    plan ← GEN_PLAN[difficulty]         // cap ladder, tries per cap, target proof cost
    best ← none

    for cap in plan.caps:
        for attempt in 1..plan.tries:
            w, h ← random size inside the tier's grid range

            // 1. Tile the grid with rectangles, none larger than `cap`.
            division ← RandomTiling(w, h, cap)      // needs at least 3 regions

            // 2. Manufacture a starting truth: a 1×1 region whose every neighbour
            //    is pinned as its own region's anchor. No rectangle larger than a
            //    single cell can then cover it, so the player can prove that
            //    emoji is a 1 before drawing anything.
            pins ← FindOpening(division)            // none → this division offers no opening
            if pins = none: continue
            anchors ← PlaceAnchors(division, pins)  // pinned cells first, the rest random

            // 3. Confirm the clue set really does force that emoji, rather than
            //    trusting the construction.
            if not HasForcedCell(anchors): continue

            // 4. Reject the puzzle unless the clues leave exactly one division.
            proof ← CountDeducibleDivisions(anchors)
            if proof.aborted or proof.sols ≠ 1: continue

            // 5. Keep candidates at this tier's difficulty. `proof.nodes` counts how
            //    much case-analysis was needed to see the division is the only one;
            //    the tier accepts the first candidate within ±35% (log gap ≤ 0.30) of
            //    its target and otherwise remembers the closest one.
            gap ← |ln(proof.nodes / plan.targetNodes)|
            if gap ≤ 0.30: return BuildPuzzle(division, anchors)
            best ← the closer of (best, this candidate)

            if time spent > 800 ms: stop sampling          // latency guard

    if best ≠ none: return BuildPuzzle(best)

    // Escape hatch, almost never reached: pin every anchor to its region's
    // top-left corner — the plainest division there is — and ship what proves.
    // (cap ladder 6 → 4, 20 attempts)
    return FallbackPuzzle(difficulty)
```

If even the escape hatch fails, `startPuzzle` shows an alert and generates again. Measured across 240 sampled generations (60 per tier) every puzzle landed inside its ±35 % band, so neither the remembered-best path nor the corner-pinned escape hatch was needed.

`BuildPuzzle` then labels the division: the distinct region values are collected, a same-sized subset of the emoji pool is drawn uniformly at random, and a shuffled permutation assigns emoji to value. Equal values share an emoji; different values never do.

### 3.2 Tiling and anchoring constraints

- **First-empty-cell tiling:** `RandomTiling` always places the rectangle covering the lowest-numbered unplaced cell, so every division is enumerated once and the result is always a complete tiling.
- **Fisher-Yates shuffle of candidate placements:** at each recursion step every fitting rectangle is offered in random order, so consecutive puzzles differ even on identical grid dimensions.
- **Region-area caps are a sampling knob, not the difficulty dial.** Each tier walks a short ladder of caps — Easy 8 → 6 → 4, Medium 6 → 4 → 8, Hard 6 → 8 → 10, Expert 10 → 8 → 12 — cheapest first. Yield drops as the cap rises (on 7×7: 21 of 45 sampled divisions proved unique at cap 6 versus 12 of 42 at cap 10), so small caps come first for speed, not because they are easier for the player.
- **Opening pins are anchored deliberately.** The boxed 1×1 region and every neighbour region keep their pinned cell as the emoji position; all remaining regions place their anchor on a random cell of the region, so players cannot infer where a region begins. Only the fallback pins every region to its top-left corner.

### 3.3 Quality checks

These checks screen every candidate inside the generation loop — a puzzle is shown only once it passes:

| Check | Description | Pass criterion |
|-------|-------------|----------------|
| **Uniqueness** | The division is uniquely determined by logic alone — no guessing required. | Exactly one division survives the search. Two divisions — or a search that runs out of node budget and therefore proves nothing — rejects the puzzle. |
| **Prover's rule set** | Only the rules the player is taught are applied: rectangles tiling the grid, each region holding emojis of a single value (two cells of the same emoji may share a region, as the drawing rules allow), equal emojis sharing one area, different emojis having different areas. | The prover never assumes the region-size cap or the opening construction, neither of which the player can know. |
| **Opening move** | Some emoji is provable on sight, so the puzzle is never a wall. | A legal-rectangle scan leaves at least one anchored cell covered only by rectangles of one area. Built by the box-in, then verified rather than trusted. |
| **Difficulty band** | The puzzle sits near its tier's target instead of anywhere in the tier's natural range. | First candidate within ±35% of the target prover cost is accepted outright. |
| **Emoji uniqueness** | No two values share the same emoji within a puzzle. | Enforced by the permutation step. |

Uniqueness is a hard gate, not a preference: a puzzle is shown only after it is proven, because ambiguous puzzles push players into trial-and-error. Proving is the expensive part, so generation is bounded by a 200,000-node search budget and an 800 ms wall-clock guard.

**Measured yield and cost** (headless Edge, 2026-11-16; 150 sampled divisions per board size, using that tier's cap ladder):

| Board | Proved unique | Ambiguous (≥ 2 divisions) | No opening available | Ran out of budget | Natural proof cost of the unique divisions |
|-------|---------------|---------------------------|----------------------|-------------------|--------------------------------------------|
| 5×6 | 55 / 150 | 54 | 41 | 0 | 44 – 1,107 nodes (median 150) |
| 7×7 | 50 / 150 | 81 | 19 | 0 | 89 – 5,567 nodes (median 826) |
| 8×8 | 49 / 150 | 80 | 21 | 0 | 357 – 35,884 nodes (median 4,179) |

Two consequences: roughly a third of random divisions are ambiguous and must be thrown away, and the 200,000-node budget has never been reached on boards this size — it sits about six times above the worst natural cost observed on 8×8. The forced-opening check rejected nothing that had an opening, so it stays as a guard rather than a filter.

## 4. Difficulty System

### 4.1 Factors that influence difficulty

| Factor | Impact |
|--------|--------|
| **Proof cost — the dial** | The prover's node count: how much case-analysis it takes to see that the division is the only one. Each tier samples until a candidate lands near its target, so this, not the region sizes, is what sets a tier. |
| **Grid size** | 3×3 → easiest; 8×8 → expert. Grid size is a *ceiling* on achievable difficulty, not a difficulty setting: on 8×8 the natural cost of a provable division spans 357 – 35,884 nodes, a hundred-fold spread on one fixed board. |
| **Number of regions** | More regions = more constraints = easier per-region, but harder global logic. A consequence of the cap ladder and grid size, not aimed at directly. |
| **Constraint density** | Regions that force early deductions vs. those left until the end. Measured difficulty covers this implicitly: a puzzle whose deductions arrive late costs the prover more. |
| **Max region area** | Only a sampling knob. Large caps leave the division under-constrained, so fewer candidates prove unique; the tiers therefore sample small caps first, where a proof is affordable and the difficulty can be chosen. |
| **Forced opening** | Not a difficulty lever — every puzzle is guaranteed at least one emoji that is provable on sight, so "no way in" is not something the tiers vary. |

Size alone is no guide to difficulty, which is why the tiers sample rather than tune dimensions. Banding is what makes a tier name mean the same thing twice: unbandied 8×8 candidates span 357 – 35,884 nodes, while shown Expert puzzles span 4,688 – 8,009 (a 1.7× window).

### 4.2 Difficulty tiers

The configuration columns are the values in `DIFFS` and `GEN_PLAN`. The measurement columns are what 60 generated puzzles per tier actually contained (headless Edge, 2026-11-16).

| Tier | Grid range | Cap ladder | Tries / cap | Target proof nodes | Cells | Regions | Emoji values | Shown proof cost p5 – p95 | Generation time med / p95 |
|------|-----------|-----------|-------------|--------------------|-------|---------|--------------|---------------------------|---------------------------|
| 🟢 Easy | 3×3 – 5×5 | 8 → 6 → 4 | 16 | 45 | 12 – 25 | 5 – 13 (med 8) | 2 – 5 | 34 – 59 | 0 ms / 1 ms |
| 🔵 Medium | 5×5 – 6×6 | 6 → 4 → 8 | 24 | 250 | 25 – 36 | 9 – 19 (med 12) | 3 – 6 | 188 – 327 | 0 ms / 1 ms |
| 🟣 Hard | 7×7 only | 6 → 8 → 10 | 24 | 1,000 | 49 | 13 – 22 (med 17) | 4 – 7 | 785 – 1,325 | 1 ms / 4 ms |
| 🔴 Expert | 8×8 only | 10 → 8 → 12 | 40 | 6,000 | 64 | 11 – 26 (med 18) | 5 – 9 | 4,688 – 8,009 | 6 ms / 29 ms |

Because generation is a synchronous main-thread task, its cost is a UX budget: even the worst tier in these samples needed 39 ms, so no worker, spinner or incremental generation is needed. **Never grow the grid to add difficulty** — grids stay at or below 8×8; a bigger board buys raw proof cost at the price of a readable cell on a phone, and the tiers would lose their measured bands.

Tier descriptions:

| Tier | Description |
|------|-------------|
| 🟢 Easy | First-time friendly; a visible truth and short divisions. |
| 🔵 Medium | Comfortable Shikaku with moderate deduction chains. |
| 🟣 Hard | Requires systematic logic; ambiguous emoji labels add cognitive friction. |
| 🔴 Expert | For veterans; long deduction chains and many region interactions. |

## 5. Player Interface

### 5.1 Screens

Six screens, all in one document, toggled by a single `.screen.active` class. Everything in the header is always available.

```
menu ──▶ game ──▶ complete ──▶ game (next puzzle) │ menu
  │                 │
  └──▶ tutorial ────┘ (ends at menu)

header: 🏆 ──▶ stats ──▶ back returns to the screen you came from
header: ⚙️ ──▶ settings ─▶ back returns to the screen you came from
```

Game screen as laid out (row mode, viewport wider than 700 px):

```
┌──────────────────────────────────────────────────────────────────┐
│ Emojikaku                                          🌙  🏆  ⚙️  │
│                                                                  │
│ ← Menu                                        1:24               │
│                                                                  │
│   ┌──────┬──────┬──────┬──────┐      ┌────────────────────┐      │
│   │ 🐼   │      │ ⭐   │      │      │ GUESSES            │      │
│   ├──────┼──────┼──────┼──────┤      │ 🐼          6      │      │
│   │      │  4   │      │ 🌵   │      │ 🌵          —      │      │
│   ├──────┼──────┼──────┼──────┤      │ ⭐          2 ×2   │      │
│   │ 🍎   │ 🍎   │      │      │      └────────────────────┘      │
│   └──────┴──────┴──────┴──────┘                                  │
│                                                                  │
│               ↩ Undo    🔍 Reveal    🗑 Clear                    │
└──────────────────────────────────────────────────────────────────┘
```

The filled rectangle is a player-drawn region of area 4 — the centred `4` is its area (§2.5). Below 700 px the guesses panel stacks under the grid and the grid takes the full width.

### 5.2 Header and menu

- **Header:** the wordmark `Emojikaku`, then 🌙 (instant dark/light toggle), 🏆 (stats), ⚙️ (settings).
- **Menu:** 🧩❓ mark, one-line pitch ("Divide the grid into rectangles. Each emoji hides a number — deduce which is which."), four difficulty chips (🟢 Easy, 🔵 Medium, 🟣 Hard, 🔴 Expert, Easy preselected), ▶ Play, ✨ Tutorial. Difficulty choice is per-session and is not remembered between loads.

### 5.3 Controls & interactions

| Action | Input | Effect |
|--------|-------|--------|
| **Draw a region** | Drag across cells; release. | Commits the rectangle if legal (§2.3). No validation until the board is complete. |
| **Draw a 1×1 region** | Tap an emoji-bearing, unclaimed cell. | Commits immediately and runs the completion check. |
| **Remove a region** | Tap anywhere inside it, or press on it and drag away. | Removes the region. |
| **Undo** | ↩ button, or `Ctrl`/`Cmd` + `Z`. | Restores the previous saved state. The history is a stack of whole snapshots of the region list, taken before each create or remove. |
| **Clear** | 🗑 button. | Deletes every drawn region. Not itself undoable, and it does not clear the history (§11). |
| **Reveal** | 🔍 button, behind a confirm ("Reveal the solution? This ends the puzzle."). | Draws every solution region with dividers and area labels, disables Undo/Reveal/Clear and the guesses panel, stops the timer. The puzzle is over; there is no way back into it except ← Menu. The emoji→value mapping is never labelled — it is implied by region area. |

There is no hint mechanic. Undo, Clear and Reveal are the only aids. Settings (⚙️) exposes **Dark Mode** only; there is no option to display the hidden numbers at any point, including after Reveal.

There is no multi-select, no region resizing, no rotation, no swipe gesture, and no "tap two opposite corners" drawing mode.

### 5.4 Guesses panel

A read-only scratchpad the game builds for you from what you have drawn — the player never types or declares anything.

- One row per distinct emoji on the board, 160 px panel beside the grid (full width below the grid under 700 px).
- Per row, a summary of the regions drawn for that emoji: `—` when nothing is drawn; `6` for one region; `2 ×2` for two same-sized regions (hover/long-press title lists the sizes); `×3 ⚠` in the accent colour when the same emoji has regions of *different* sizes.
- A row is highlighted amber when two **different** emojis have guessed the same size — the visual cue for "these two cannot both be right".
- The panel dims and stops updating after Reveal.
- Known defect: rows are ordered by the emoji's hidden value, which leaks the ranking of values (§11).

### 5.5 Stats screen (🏆)

Four difficulty tabs (🟢 Easy / 🔵 Medium / 🟣 Hard / 🔴 Expert), the current menu difficulty preselected, and three cards for the selected tab: **Solved**, **Best**, **Average**. Empty tiers show `—` for both times. Back returns to the screen the player was on.

### 5.6 Settings screen (⚙️)

One row: **Dark Mode**, a Material-style switch. Dark mode is on by default. There are no other options.

### 5.7 Completion

- **Success:** 🎉, "Puzzle Solved!", a card reading "Well done.", the line `Solved in m:ss`, suffixed ` — new best!` when the time beats that tier's record, then ▶ Next Puzzle (same difficulty, new puzzle) and Menu.
- **Failure:** 😕, "Not Quite Right", the explanation that correct regions are kept and incorrect ones removed, then ↩ Resume (§2.4). No time is shown or recorded.

## 6. Progression & Persistence

### 6.1 Stats

Stats are the entire progression system. There is no XP, no levels, no unlocks, no currency, no streaks.

Per difficulty tier, the game keeps `{ n, best, total }`:

- Recorded **only on a correct completion**, filed under the difficulty the puzzle was *generated at*, so changing the menu selection mid-puzzle cannot misfile a solve.
- `n` — solves at that tier. `best` — fastest solve in seconds. `total` — sum of solve times, used to show a rounded average.
- Failed attempts, Reveal and abandoned puzzles are not recorded at all.
- Solve times are wall-clock seconds on the game screen: the timer starts when a puzzle is generated, pauses at the completion check, resumes on Resume, and stops on Reveal or ← Menu. A revealed or abandoned puzzle can therefore never be a "best".

### 6.2 Persistence

One `localStorage` entry, key `emojikaku`, holding `{ stats, darkMode }`. Written on every successful solve and on every theme change; read once at load. Difficulty selection, tutorial progress and in-progress puzzle state are not persisted. Save data from earlier versions (lifetime solved counter, streak, per-puzzle time list, facts) is discarded on load rather than migrated.

There is no account, sync, export or cloud storage.

## 7. Visual & Audio Direction (as built)

### 7.1 Style

A Material-flavoured flat look built entirely from CSS custom properties, with light and dark palettes switched by a `body.dark` class.

| Token | Light | Dark |
|-------|-------|------|
| Background | `#F5F3EF` | `#1A1D23` |
| Surface (cards, buttons) | `#FFFFFF` | `#2A2E35` |
| Foreground | `#1D1B20` | `#E8E0D5` |
| Grid background | `#F1EFEA` | `#24272E` |
| Accent | `#E07A5F` | `#E07A5F` |
| Region fills | 9-pastel palette at 0.45 alpha | same hues at 0.30 alpha, drawn at 0.6 (0.7 once revealed) |

- Type stack: Segoe UI / Roboto / system. Timer and stats use tabular numerals.
- Elevation: three shadow levels on cards and buttons; buttons lift 1–2 px on hover and settle on press.
- Motion: 0.3 s fade + 10 px rise on screen change; a 0.5 s scale bounce on the completion emoji. No animation on the grid itself.
- No audio of any kind: no sounds, no haptics.

### 7.2 Grid rendering

The grid is an inline **SVG**, rebuilt from scratch on every change (no per-cell DOM nodes, no canvas):

- Layer 0 grid background (rounded 6 px), layer 1 cell rectangles (rounded 3 px, translucent white on the grid background), layer 2 player/solution region fills with their area labels, layer 3 emoji glyphs on top so they are never hidden.
- Cell size is computed to fit the viewport: available width minus the guesses panel (row mode) or the full width (column mode), available height minus 56 px header and 72 px control bar. Cell gap 2 px, outer padding 4 px. Re-computed on a 150 ms-debounced window resize.
- Region colours cycle through the nine pastels by creation order, so adjacent regions rarely collide.
- Area labels are drawn in a fixed `#2D3142` in both themes (§11).

### 7.3 Emoji presentation

- Emoji glyphs are SVG text centred in the cell at 50% of the cell size (minimum 8 px), so they stay readable on an 8×8 board at phone width.
- No tap animation, no glow, no ring overlay, no per-emoji "confirmed" state — the emoji is decoration, and the guesses panel (§5.4) is where state is shown.
- Pool: 60 glyphs across six categories, chosen for visual variety, never for meaning (§2.5).

## 8. Onboarding & Tutorials

### 8.1 First-time flow as built

There is no forced first-run tour. Tutorial is an optional ✨ button on the menu, and playing immediately without it is supported.

```
Step 1 — What is Shikaku?      🧩   plain Shikaku rules in three sentences.
Step 2 — The Emoji Twist       🎲   numbers become emojis, random mapping per puzzle,
                                    shown as a static side-by-side pair of 3×3 mini
                                    grids ("Numbers" vs "Emojis").
Step 3 — How to Draw           ✋   drag a rectangle, release when the area matches the
                                    hidden number; one emoji per region.
Step 4 — Ready!                🚀   "Use ↩ Undo if you make a mistake. Click Play and
                                    have fun!" → returns to the menu.
```

← Back / Next → navigation; the last button reads "Let's Play! 🎉" and returns to the menu rather than starting a puzzle.

### 8.2 What onboarding does not do

It is text-and-static-diagrams only: no interactive drawing exercise, no animated solved grid, no scratchpad lesson (there is nothing to teach there), and it is never shown automatically. An interactive first-puzzle walkthrough is future work (§10).

## 9. Technical Notes (as built)

### 9.1 Architecture

```
┌────────────────────────────────────────────────────────────────┐
│  index.html  — single file, no build, no dependencies, no server│
│                                                                │
│  <style>      CSS custom properties, light + dark tokens       │
│  <body>       six .screen divs + header + unused legacy panel  │
│  <script>     one classic script; every helper is global        │
│                                                                │
│    state        one mutable object `S` (screens, puzzle, draws,│
│                 history, guesses, stats, timer, dark mode)      │
│    generation   randomTiling → findOpening → placeAnchors →    │
│                 buildRectIndex → countDeducibleDivisions        │
│    rendering    SVG rebuilt per change; cell size fit to screen │
│    input        pointer events on the grid container + keyboard │
│    rules        commitDrawnRegion / validateSolution / prover   │
│                 share one rectangle model                       │
│    persistence  localStorage 'emojikaku'                        │
└────────────────────────────────────────────────────────────────┘
```

- **No framework, no modules, no bundler.** A single classic `<script>`; top-level `const`/`function` names are global, which is why the game is easy to probe from a browser console and why dead markup still has live event handlers (§11).
- **Generation and proving are synchronous** and run at up to ~40 ms per puzzle on the sampled tiers (§4.2), comfortably inside one frame budget's worth of patience; no Web Worker is warranted.
- **The prover is also the difficulty meter.** `countDeducibleDivisions` returns `{ sols, nodes, aborted }`; `sols` gates the puzzle and `nodes` sets its tier (§3.3, §4.1).
- **Browser assumptions:** Pointer Events, CSS custom properties, `color-mix()`, inline SVG and `localStorage`. Developed and measured in Chromium/Edge.
- **Touch gestures:** `touch-action: none` on the grid container, plus a named non-passive `touchstart` handler that suppresses default touch behaviour while the board is still empty (it is attached and removed with the other grid listeners).
- **No reactive state.** `S` is one plain mutable object; every mutation is followed by an explicit `render*()` call. There is no diffing, no virtual DOM and no state library.
- **Failure handling:** generation failure shows `alert()` and regenerates; `localStorage` read/write is wrapped in `try/catch` and silently ignored when storage is unavailable.

## 10. Future / Post-Launch Ideas

Nothing in this section is built. Each item is a proposal, not a description of the game.

### 10.1 Ideas

| Feature | Priority | Notes |
|---------|----------|-------|
| **Daily puzzle** | Medium | One pre-selected puzzle per 24 h, ranked against the player's own tier history. Would need seeded generation to be shareable. |
| **Weekly challenge** | Low | Two expert puzzles (mid-week and weekend). Depends on the daily-puzzle machinery. |
| **Emoji themes** | Low | Seasonal pools (Halloween 🎃, Christmas 🎄). Purely cosmetic — one line of code, since the pool is already a flat array. |
| **Editor mode** | Medium | Players design and share grids; the system proves uniqueness (or rejects) and generates emoji mappings for shared puzzles. |
| **Multiplayer (async)** | Low | Two players solve the same seed; compare time. Needs seeds and an identity. |
| **Collaborative solver** | Low | Share puzzle state by link; comment on strategy. |
| **API / widget** | Low | "Puzzle of the day" home-screen widget. |
| **App packaging / PWA** | Medium | Manifest + service worker so the single file installs offline; the code is already self-contained. |
| **Interactive onboarding** | Medium | The §8 walkthrough with a draggable practice 3×3 and a reveal-the-number teaching step. |
| **Audio & haptics** | Low | None today. Region commit, success, failure. |
| **💎 Gems economy** | Low | Cosmetic emoji packs earned by daily login / streaks / leaderboard placement. Only worth designing after retention is real. |
| **Leaderboards** | Low | Per-difficulty best times. Needs accounts, anti-cheat and a server, all of which the current design avoids. |

### 10.2 Deliberately out of scope

Recorded so they are not quietly re-added:

| Removed / rejected | Why |
|--------------------|-----|
| **Hints** | Every puzzle is provably solvable with the taught rules, so a hint is an admission that the prover is wrong. Removed with its button, per-puzzle counter and `hint()` helper. |
| **Scratchpad / value declaration** (double-tap or long-press an emoji to claim its value) | Duplicates what the guesses panel already derives from the player's own regions, and would have been an unvalidated second source of truth. |
| **Displaying the hidden numbers**, including after Reveal | Would remove the only thing the game asks the player to do. The "Show Numbers on Reveal" setting was a no-op and was deleted. |
| **Lifetime solved counter, streak counter, "weird facts"** | The streak measured a habit the game does not support (no daily content); the facts card had no data behind it; a cross-tier solved count is not a useful number. Stats are per tier only (§6.1). |
| **Mistake-rate statistics** | With uniqueness guaranteed, "not quite right" is pure player error and not a signal worth surfacing. |
| **Per-cell marks / pencil candidates** | Shikaku's own notation does not help when the numbers are hidden; region sizes carry the reasoning. |

## 11. Known Gaps & Defects

Observed in the current build, in rough order of how much they cost the player:

| Item | Symptom | Where |
|------|---------|-------|
| **Guesses list leaks the value ranking** | Rows are sorted by the emoji's hidden value, so the first row is always the smallest value. Careful players can exploit the order without solving anything. | `renderGuide()` |
| **Dark Mode off is not remembered** | `loadPersisted()` only acts when the stored flag is truthy, so a stored `false` is ignored and the game reopens in dark mode. | `loadPersisted()` |
| **Reveal's confirmation toast never appears** | `showToast` appends to `getElementById('game-header')`, but the element is `class="game-header"` with no id, so the node is discarded. Reveal gives no feedback beyond the drawn regions. | `showToast()` |
| **Area labels are unreadable in dark mode** | Region area numbers are hard-coded dark grey on a pale pastel fill, in both themes. | `renderGrid()` |
| **Clear is not undoable** | Clear takes no history snapshot, so the next Undo returns to the state before the last create/remove rather than the board as it was before clearing. | Clear button handler |
| **Puzzles are lost silently** | ← Menu stops the timer and discards the board; Play always generates a new puzzle. There is no confirmation and no resume. | `btn-back`, `startPuzzle()` |
| **Dead legacy UI** | A hidden "mark emoji as value" panel (markup + CSS + an `Esc` handler that only closes it) survives from the deleted scratchpad. It is invisible, unreachable, and still wired. | `#emoji-panel` |
| **Filler copy on the success screen** | The completion card always reads "Well done." — a leftover of the deleted facts feature. | `showCompletion()` |
| **Theme button label never changes** | The header button is permanently 🌙 and does not become ☀️ in light mode. | `btn-theme` |
| **Generation failure path recurses** | `alert()` then a recursive `startPuzzle()`; unbounded in principle, though the fallback has never failed in measurement. | `startPuzzle()` |
