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
| **Distribution** | Local file only. No PWA manifest, no service worker, no app store build, no server, no build step, no dependencies. A shared puzzle link carries only a `#tier-seed` fragment (§6.3), so it resolves against whatever URL the file is opened from; a recipient needs their own copy of the file until the game is hosted. |
| **Screen sizes** | Any viewport from ~320 px wide upward; the layout reflows at 700 px (side-by-side → stacked). |
| **Single-player / Multiplayer** | Single-player. No accounts, no sessions, no leaderboards. |
| **Session length** | One puzzle at a time; a puzzle is not saved and cannot be resumed after leaving the game screen. A puzzle link (§6.3) regenerates the same board rather than resuming it. |

### 1.1 Elevator pitch

A standard game of Shikaku — a rectangle is subdivided into rectangular regions, each containing exactly one number whose value equals the region's area — reimagined with emojis in place of numbers. Emojis are assigned randomly to values per puzzle, serving purely as obfuscation. Players must deduce which emoji maps to which number by analysing region sizes alone.

### 1.2 Design principles (as built)

1. **Every puzzle is provably unique.** No puzzle may require guessing. This is enforced by a prover at generation time, not by level design (§3.3).
2. **Every puzzle has a provable first move.** The player is never facing a wall (§3.3).
3. **Difficulty is measured, not assumed.** The rectangles the player has to rule out are the dial; the prover's search cost and board size are not (§4.1).
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
- **Correct** → 🎉 "Puzzle Solved!" screen with solve time, best-time marker, ▶ Next Puzzle, 📋 Copy result and Menu buttons. The solve is recorded in stats (§6.1) and the copy button puts a challenge line plus a link to this exact puzzle on the clipboard (§6.3).
- **Incorrect** → 😕 "Not Quite Right" screen. The button offers **↩ Resume**, which keeps the regions that were correct, deletes the ones that were wrong, resets the undo history, and restarts the timer. The removed wrong regions are marked on the grid with a 4 px red outline (white stroke in dark mode); that outline is dismissed by the next interaction with the grid.
- Because the generator proves uniqueness, the "Not Quite Right" path is only reachable through player error, and it is not recorded anywhere.

### 2.5 Emoji–value mapping

- Region values can be **any positive integer** (real Shikaku values). Multiple regions may share a value; equal values share one emoji, different values never do.
- At generation each distinct value gets a **fully random** emoji from a 60-glyph pool: 10 faces, 10 animals, 10 foods, 10 nature, 10 objects, 10 hearts/stars. Assignment is a shuffled permutation — no semantic filtering. Measured distinct values per puzzle: 2–5 (Easy), 3–6 (Medium), 4–7 (Hard), 5–9 (Expert), so the pool never runs short.
- The mapping is consistent for the whole puzzle.

**What is and is not printed:** the hidden value behind an emoji is never shown as a number — not during play, not on the success screen, and never as a label next to an emoji. What *is* printed is the **area of every player-drawn region**: a small centred number (only when the area is greater than 1, so 1×1 regions show nothing). That number is the player's own rectangle size, which is the tool they use to reason about values. After Reveal, the solution regions are drawn as player regions, so their area labels coincide with the values.

> **Design principle:** Emojis are used to *obfuscate* numbers, not to hint at them. Players must rely entirely on Shikaku logic (region sizes) to deduce which emoji corresponds to which value. The visual randomness of emojis adds a layer of cognitive dissonance that makes solved grids feel surprising ("Wait, the poop emoji was 4?!").

## 3. Puzzle Generation

Generation happens synchronously in the browser at the moment Play is pressed, and always yields a puzzle that has been *proven* to have exactly one solution. It is a **pure function of `(seed, difficulty)`**: a 32-bit seed drives every random choice in the pipeline (`seedRng`, mulberry32), and the seed is the address-bar fragment when the page was opened from a puzzle link (§6.3), otherwise one `Math.random()` draw per puzzle. The same seed on the same engine therefore replays the same board.

### 3.1 Generation algorithm (top-down)

```
GeneratePuzzle(seed, difficulty):
    plan ← GEN_PLAN[difficulty]      // cap ladder, tries, workload band, ceiling, floor
    RNG ← SeedRng(seed)              // every random choice below draws from this stream
    nodes ← 0                        // prover work spent so far on this one puzzle
    best ← none

    for cap in plan.caps:
        for attempt in 1..plan.tries:      // sampling stops early if nodes > 500,000
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
            //    The prover keeps the single solution it found.
            proof ← CountDeducibleDivisions(anchors)   // { sols, nodes, aborted, path }
            nodes += proof.nodes
            if proof.aborted or proof.sols ≠ 1: continue

            // 5. A board whose clues barely pin anything down is too open for a tier,
            //    whatever the player's workload would be.
            if proof.nodes < plan.minNodes: continue

            // 6. Replay that one solution in board order and measure what the player
            //    is handed: `ruledOut` adds up every rectangle that was still legal
            //    but wrong, `widest` is how many the single busiest step offered.
            cost ← StepCost(proof.path)
            if cost.widest > plan.widest: continue   // too busy: discarded, not remembered

            // 7. Accept the workload this tier is named for (log gap ≤ 0.30, i.e. 35 %);
            //    otherwise remember the closest candidate.
            gap ← |ln(cost.ruledOut / plan.ruled)|
            if gap ≤ 0.30: return BuildPuzzle(division, anchors)
            best ← the closer of (best, this candidate)

    if best ≠ none: return BuildPuzzle(best)

    // Escape hatch, almost never reached: pin every anchor to its region's
    // top-left corner — the plainest division there is — and ship what proves.
    // (cap ladder 6 → 4, 20 attempts)
    return FallbackPuzzle(difficulty)
```

If even the escape hatch fails, `startPuzzle` moves to the next seed in a bounded four-seed ladder (`seed`, `seed + 1`, `seed + 2`, `seed + 3`) and only then gives up: an alert and a return to the menu, never a silent re-entry into generation. The ladder is arithmetic, so two machines opening the same link fail at the same point. Measured across 480 sampled generations at these constants on the seeded build (120 fixed seeds per tier, headless Edge, 2026-11-17): every shipped puzzle re-proved unique and kept its forced opening, and 479 replay inside their ±35 % workload band when the measurement is repeated afterwards — the one that did not was shipped by the closest-miss rule in step 7. Neither the escape hatch nor the cost guard fired: the heaviest generation spent 133,854 of its 500,000 prover nodes, and re-running the same seed list in a second browser process reproduced the same boards and the same node counts.

`BuildPuzzle` then labels the division: the distinct region values are collected, a same-sized subset of the emoji pool is drawn uniformly at random, and a shuffled permutation assigns emoji to value. Equal values share an emoji; different values never do.

### 3.2 Tiling and anchoring constraints

- **First-empty-cell tiling:** `RandomTiling` always places the rectangle covering the lowest-numbered unplaced cell, so every division is enumerated once and the result is always a complete tiling.
- **Fisher-Yates shuffle of candidate placements:** at each recursion step every fitting rectangle is offered in random order, so consecutive puzzles differ even on identical grid dimensions.
- **Region-area caps set both the odds of a unique division and how the board reads.** Each tier walks a short ladder of caps — Easy 8 → 6 → 4, Medium 6 → 4 → 8, Hard 5 → 6 → 8, Expert 6 → 8 → 10 — cheapest and most promising first. Re-measured on 60 sampled divisions per board and cap: on 7×7, 24 of 60 proved unique at cap 5 against 14 of 60 at cap 8; on 8×8, 30 of 60 at cap 6 against 27 of 60 at cap 8. On the big boards the small cap is no longer only a cheap first try, it is part of the difficulty: a division cut under a small cap breaks into more, shorter regions, so its busiest step offers fewer rectangles (7×7 median 9 at cap 5 against 12 at cap 8; 8×8 median 11 at cap 6 against 13 at cap 10).
- **Opening pins are anchored deliberately.** The boxed 1×1 region and every neighbour region keep their pinned cell as the emoji position; all remaining regions place their anchor on a random cell of the region, so players cannot infer where a region begins. Only the fallback pins every region to its top-left corner.

### 3.3 Quality checks

These checks screen every candidate inside the generation loop — a puzzle is shown only once it passes:

| Check | Description | Pass criterion |
|-------|-------------|----------------|
| **Uniqueness** | The division is uniquely determined by logic alone — no guessing required. | Exactly one division survives the search. Two divisions — or a search that runs out of node budget and therefore proves nothing — rejects the puzzle. |
| **Prover's rule set** | Only the rules the player is taught are applied: rectangles tiling the grid, each region holding emojis of a single value (two cells of the same emoji may share a region, as the drawing rules allow), equal emojis sharing one area, different emojis having different areas. | The prover never assumes the region-size cap or the opening construction, neither of which the player can know. |
| **Opening move** | Some emoji is provable on sight, so the puzzle is never a wall. | A legal-rectangle scan leaves at least one anchored cell covered only by rectangles of one area. Built by the box-in, then verified rather than trusted. |
| **Difficulty band** | The candidate hands the player the workload the tier is named for, and never one brutal step. | Its one solution is replayed in board order and every still-legal rectangle counted: the wrong-but-legal ones must add up to within 35 % of the tier's figure, and the busiest single step must not exceed the tier's ceiling. A candidate over the ceiling is discarded, not remembered. `proof.nodes` only has to clear a floor, so a tier cannot ship a board the clues barely pin down. |
| **Emoji uniqueness** | No two values share the same emoji within a puzzle. | Enforced by the permutation step. |

Uniqueness is a hard gate, not a preference: a puzzle is shown only after it is proven, because ambiguous puzzles push players into trial-and-error. Proving is the expensive part, so generation is bounded twice over — a 200,000-node budget for proving one candidate, and a 500,000-node budget shared by all candidates in one generation. There is no wall-clock guard: a time limit would make the accepted candidate depend on machine speed, and a seeded puzzle has to come out the same on every machine (§6.3).

**Measured yield and cost** (headless Edge, 2026-11-16; 60 sampled divisions per board and per cap, using that tier's cap ladder). "No opening" covers candidates with no boxed 1×1 and clue sets that do not actually force it; the workload columns describe only the divisions that proved unique, before any banding:

| Board | Caps tried | Proved unique | Ambiguous (≥ 2 divisions) | No opening | Ran out of budget | Natural prover cost (median) | Natural workload, ruled-out rectangles (median / worst) | Natural busiest step (median / worst) |
|-------|-----------|---------------|------------------------------|------------|-------------------|------------------------------|----------------------------------------------------------|----------------------------------------|
| 5×6 | 6 / 4 / 8 | 22 / 28 / 20 | 20 / 23 / 27 | 18 / 9 / 13 | 0 | 197 / 121 / 199 nodes | 12 / 32, 11 / 22, 13 / 19 | 7 / 18, 7 / 19, 8 / 13 |
| 7×7 | 5 / 6 / 8 | 24 / 22 / 14 | 30 / 31 / 32 | 6 / 7 / 14 | 0 | 503 / 1,113 / 904 nodes | 16 / 28, 18 / 33, 22 / 31 | 9 / 13, 11 / 17, 12 / 18 |
| 8×8 | 6 / 8 / 10 | 30 / 27 / 30 | 24 / 26 / 26 | 6 / 7 / 4 | 0 | 1,651 / 8,229 / 6,764 nodes | 21 / 55, 30 / 55, 30 / 44 | 11 / 23, 12 / 25, 13 / 23 |

Three consequences. Roughly a third of random divisions are ambiguous and must be thrown away, so sampling is not optional. The first cap in each big-board ladder is both the cheapest to prove and the least busy for the player — 8×8 at cap 6 costs a median 1,651 nodes against 8,229 at cap 8, and its busiest step offers a median 11 rectangles against 12 — which is why the ladders start there. And the natural spread on one fixed board is enormous: the unique 8×8 divisions sampled here carry workloads of 21 to 30 ruled-out rectangles with single steps reaching 25, so a tier that does not band ships whatever the sampler happens to find.

The 200,000-node budget has never been reached on boards this size — the worst sampled proof was 59,735 nodes (8×8 at cap 10) — and the opening stage rejected 6 of 60 candidates at each big board's first cap, so both checks stay as guards rather than filters.

## 4. Difficulty System

### 4.1 Factors that influence difficulty

| Factor | Impact |
|--------|--------|
| **Workload band — the dial** | What the player is handed: every rectangle that was still legal but wrong, added up across the whole solution, plus a ceiling on how many the single busiest step may offer. Each tier samples until a candidate lands in its band, so this, not the region sizes or the prover's cost, is what sets a tier. |
| **Prover cost — a floor, not a dial** | How much case-analysis it takes to see that the division is the only one. It stopped being the dial when it was re-measured: on 8×8 the sampled caps span a median 1,651 – 8,229 nodes while the workload a player actually meets moves only from 21 to 30. It measures how bushy the proof search is, not how much reasoning the board asks for. Each tier now only has to clear a node floor, which keeps a tier from shipping a board the clues barely pin down. |
| **Grid size** | 3×3 → easiest; 8×8 → expert. Grid size is a *ceiling* on achievable difficulty, not a difficulty setting: on 8×8 the provable divisions sampled in §3.3 run to prover costs of 1,651 to 8,229 nodes and workloads of 21 to 30 on one fixed board, a spread no tier name carries without banding. |
| **Number of regions** | More regions = more constraints = easier per-region, but more drawing. A consequence of the cap ladder and grid size, not aimed at directly, and it is the price of the new dial: easing the big boards raised the median region count from 16 to 19 on Hard and from 18 to 21 on Expert. |
| **Constraint density** | Regions that force early deductions vs. those left until the end. The workload band covers part of this: a puzzle whose deductions arrive late leaves more open steps, and its busiest step gets wider, which is why the ceiling exists as well as the total. |
| **Max region area** | A sampling knob with a visible side effect. Large caps leave the division under-constrained, so fewer candidates prove unique; and a small cap bounds both the biggest region and the widest step, so the big boards sample small caps first — Expert's median largest region fell from 9 cells to 6 once its ladder became 6 → 8 → 10. |
| **Forced opening** | Not a difficulty lever — every puzzle is guaranteed at least one emoji that is provable on sight, so "no way in" is not something the tiers vary. |

Size alone is no guide to difficulty, which is why the tiers sample rather than tune dimensions. Banding is what makes a tier name mean the same thing twice: the unbandied 8×8 divisions sampled in §3.3 carry workloads of 21 to 30 ruled-out rectangles with single steps reaching 25, while shown Expert puzzles land in 15 – 24 with no step wider than 10.

### 4.2 Difficulty tiers

The configuration columns are the values in `DIFFS` and `GEN_PLAN`. The measurement columns are what 120 generated puzzles per tier actually contained, measured afterwards by a probe that re-proves each shipped puzzle and replays its one solution (headless Edge, 2026-11-17). Generation is now seeded, so those samples are a fixed list of seeds per tier and the whole table is reproducible, not a fresh draw each time it is measured.

| Tier | Grid range | Cap ladder | Tries / cap | Workload target | Busiest step ceiling | Prover-node floor | Regions med (range) | Largest region med (max) | Shown workload p5 – p90 (max) | Shown busiest step med / max | Generation time med / p95 |
|------|-----------|-----------|-------------|-----------------|----------------------|-------------------|--------------------|--------------------------|-------------------------------------|-------------------------------|---------------------------|
| 🟢 Easy | 3×3 – 5×5 | 8 → 6 → 4 | 16 | 6 | 6 | 20 | 7 (4 – 12) | 5 (max 8) | 5 – 8 (max 9) | 5 / 6 | 0.1 ms / 0.6 ms |
| 🔵 Medium | 5×5 – 6×6 | 6 → 4 → 8 | 24 | 14 | 9 | 120 | 12 (8 – 18) | 6 (max 6) | 11 – 16 (max 18) | 7 / 9 | 0.2 ms / 0.8 ms |
| 🟣 Hard | 7×7 only | 5 → 6 → 8 | 24 | 16 | 9 | 300 | 19 (16 – 27) | 5 (max 6) | 12 – 20 (max 21) | 8 / 9 | 0.3 ms / 1.2 ms |
| 🔴 Expert | 8×8 only | 6 → 8 → 10 | 40 | 20 | 10 | 1,000 | 21 (18 – 27) | 6 (max 8) | 15 – 24 (max 26) | 8 / 10 | 1.5 ms / 4.8 ms |

Against the previous dial, measured the same way on 60 puzzles per tier: the worst single step a player met fell from 30 rectangles to 9 on Hard and from 23 to 10 on Expert, the heaviest total workload from 39 to 21 and from 60 to 26, and Expert's biggest region from a median 9 cells to 6. The price is that a board is cut into more pieces — a median 19 regions on Hard against 16, and 21 on Expert against 18 — which is the trade the band pays for.

Because generation is a synchronous main-thread task, its cost is a UX budget: the heaviest tier needed a median 1.5 ms and a p95 of 4.8 ms, and the worst of the 480 sampled generations took 12.2 ms, so no worker, spinner or incremental generation is needed. **Never grow the grid to add difficulty** — grids stay at or below 8×8; a bigger board buys raw proof cost at the price of a readable cell on a phone, and the tiers would lose their measured bands.

Tier descriptions:

| Tier | Description |
|------|-------------|
| 🟢 Easy | First-time friendly; a visible truth and short divisions. |
| 🔵 Medium | Comfortable Shikaku with moderate deduction chains. |
| 🟣 Hard | Requires systematic logic; ambiguous emoji labels add cognitive friction. |
| 🔴 Expert | For veterans; long deduction chains and many region interactions. |

## 5. Player Interface

### 5.1 Screens

Five screens, all in one document, toggled by a single `.screen.active` class. Everything in the header is always available.

```
menu ──▶ game ──▶ complete ──▶ game (next puzzle) │ menu
  │                 │
  └──▶ tutorial ────┘ (ends at menu)

header: 🏆 ──▶ stats ──▶ back returns to the screen you came from
```

A puzzle link (§6.3) is the one way to arrive at **game** without passing through the menu.

Game screen as laid out (row mode, viewport wider than 700 px):

```
┌──────────────────────────────────────────────────────────────────┐
│ Emojikaku                                              🌙  🏆  │
│                                                                  │
│ ← Menu              🟣 Hard                   1:24.06            │
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

The filled rectangle is a player-drawn region of area 4 — the centred `4` is its area (§2.5). Between ← Menu and the timer the header names the tier the board was generated at, using the menu chips' token as plain text (`DIFF_TAG`), so the player always knows which band they are in; it is not a control and cannot be changed mid-puzzle. The timer reads `m:ss.hh` — whole minutes, seconds and hundredths, e.g. `1:24.06` — because an Easy board is routinely decided in a couple of seconds and the hundredths are what separate two solves of it (§6.1). Below 700 px the guesses panel stacks under the grid and the grid takes the full width.

### 5.2 Header and menu

- **Header:** the wordmark `Emojikaku`, then the theme button (shows 🌙 while dark, ☀️ while light; one click toggles, on by default), 🏆 (stats).
- **Menu:** the mark — a 96 px inline SVG drawn from the board's own geometry: three rounded blocks that are the regions of a solved 4×4 grid (areas 6, 2 and 8), with the top-left one an accented tile tipped 6° counter-clockwise and carrying a cut-out smiling face. It is decoration, not text (`aria-hidden="true"`), and paints only from the theme tokens: `--fg` for the blocks, `--accent` for the tipped tile, `--bg` for the face, so the page colour shows through it in either theme. The face is a detail of the tile, not a glyph — nothing in the mark is text, and no emoji is used. Below the mark: the one-line pitch ("Divide the grid into rectangles. Each emoji hides a number — deduce which is which."), four difficulty chips (🟢 Easy, 🔵 Medium, 🟣 Hard, 🔴 Expert, Easy preselected), ▶ Play, ✨ Tutorial. Difficulty choice is per-session and is not remembered between loads.

### 5.3 Controls & interactions

| Action | Input | Effect |
|--------|-------|--------|
| **Draw a region** | Drag across cells; release. | Commits the rectangle if legal (§2.3). No validation until the board is complete. |
| **Draw a 1×1 region** | Tap an emoji-bearing, unclaimed cell. | Commits immediately and runs the completion check. |
| **Remove a region** | Tap anywhere inside it, or press on it and drag away. | Removes the region. |
| **Undo** | ↩ button, or `Ctrl`/`Cmd` + `Z`. | Restores the previous saved state. The history is a stack of whole snapshots of the region list, taken before each create or remove. |
| **Clear** | 🗑 button. | Deletes every drawn region. Not itself undoable, and it does not clear the history (§11). |
| **Reveal** | 🔍 button, behind a confirm ("Reveal the solution? This ends the puzzle."). | Draws every solution region with dividers and area labels, disables Undo/Reveal/Clear and the guesses panel, stops the timer. The puzzle is over; there is no way back into it except ← Menu. The emoji→value mapping is never labelled — it is implied by region area. |

There is no hint mechanic. Undo, Clear and Reveal are the only aids. The only setting is the header theme button; there is no option to display the hidden numbers at any point, including after Reveal.

There is no multi-select, no region resizing, no rotation, no swipe gesture, and no "tap two opposite corners" drawing mode.

### 5.4 Guesses panel

A read-only scratchpad the game builds for you from what you have drawn — the player never types or declares anything.

- One row per distinct emoji on the board, in the order the emojis appear on the grid (top-to-bottom, left-to-right). The order is fixed for the whole puzzle and never derived from the hidden values, so it cannot rank them. 160 px panel beside the grid (full width below the grid under 700 px).
- Per row, a summary of the regions drawn for that emoji: `—` when nothing is drawn; `6` for one region; `2 ×2` for two same-sized regions (hover/long-press title lists the sizes); `×3 ⚠` in the accent colour when the same emoji has regions of *different* sizes.
- A row is highlighted amber when two **different** emojis have guessed the same size — the visual cue for "these two cannot both be right".
- The panel dims and stops updating after Reveal.

### 5.5 Stats screen (🏆)

Four difficulty tabs (🟢 Easy / 🔵 Medium / 🟣 Hard / 🔴 Expert), the current menu difficulty preselected, and three cards for the selected tab: **Solved**, **Best**, **Average**. Empty tiers show `—` for both times. Back returns to the screen the player was on.

### 5.6 Completion

- **Success:** 🎉, "Puzzle Solved!", the tier the puzzle was generated at (the same plain-text token as §5.1, which is what makes "that tier's record" below interpretable), a card reading "Well done.", the line `Solved in m:ss.hh`, suffixed ` — new best!` when the time beats that tier's record, then ▶ Next Puzzle (same difficulty, new puzzle), 📋 Copy result (§6.3) and Menu.
- **Failure:** 😕, "Not Quite Right", the same tier line, the explanation that correct regions are kept and incorrect ones removed, then ↩ Resume (§2.4). No time is shown or recorded, and there is no Copy result button: with no time there is no score to challenge anyone with.

## 6. Progression & Persistence

### 6.1 Stats

Stats are the entire progression system. There is no XP, no levels, no unlocks, no currency, no streaks.

Per difficulty tier, the game keeps `{ n, best, total }`:

- Recorded **only on a correct completion**, filed under the difficulty the puzzle was *generated at*, so changing the menu selection mid-puzzle cannot misfile a solve.
- `n` — solves at that tier. `best` — fastest solve, and `total` — the sum of solve times used to show an average; both are integer **centiseconds** (1/100 s), never floating-point seconds. The average is `round(total / n)` in the same unit.
- Failed attempts, Reveal and abandoned puzzles are not recorded at all.
- Solve times are wall-clock centiseconds on the game screen: the timer starts when a puzzle is generated, pauses at the completion check, resumes on Resume, and stops on Reveal or ← Menu. A revealed or abandoned puzzle can therefore never be a "best". The stop happens before the time is read, so the solve cannot be filed with a stale hundredth.
- Elapsed time is measured against `performance.now()` from the moment the clock was started, not by counting display ticks. **Why:** a backgrounded tab throttles `setInterval` to roughly one callback per second, so a tick counter would under-count any solve that spent time in another tab — and could not resolve hundredths at all. What polls is only the *display*, every 20 ms, so the number on screen can trail real time by about 2 cs; the value a solve is filed with is committed from `performance.now()` at the pause, so it is exact to the centisecond.

### 6.2 Persistence

One `localStorage` entry, key `emojikaku`, holding `{ unit, stats, darkMode }`. `unit: 'cs'` says the times in `stats` are centiseconds; a save written before hundredths existed has no `unit` and its whole-second times are multiplied by 100 once, while loading. That is the one migration, and it is safe because it rescales the same measurements rather than inventing data — unlike save data from other earlier versions (lifetime solved counter, streak, per-puzzle time list, facts), which is still discarded on load rather than migrated. Written on every successful solve and on every theme change; read once at load. Difficulty selection, tutorial progress and in-progress puzzle state are not persisted.

There is no account, sync, export or cloud storage.

### 6.3 Sharing a puzzle

Every puzzle has an address: the URL's fragment names its tier and the seed that generated it.

```
index.html#hard-1z141z3          // tier, then the seed in base36 (7 digits at most)
```

- **Written on every puzzle.** Play, ▶ Next Puzzle and a deep link all go through `startPuzzle`, which rewrites the fragment with `location.replace` once the board exists — deliberately not `history.replaceState`, which throws on a `file://` page: `location.replace` to a fragment-only target neither reloads nor adds a history entry, so Back still leaves the game rather than walking back through a trail of puzzles. The address bar therefore always describes the board on screen.
- **Read once at boot.** A well-formed fragment — one of `easy` / `medium` / `hard` / `expert`, then 1 – 7 base36 characters decoding to at most 2³² − 1 — goes straight into that puzzle: the menu chip moves to the link's tier and the timer starts, exactly as if Play had been pressed at that difficulty. Anything else (a typo, an unknown tier, an out-of-range seed) is ignored and the page boots to the menu as before. A link seeds exactly one puzzle; the next ▶ Next Puzzle draws its own seed and rewrites the address bar.
- **The copy button** on the 🎉 screen puts two lines on the clipboard — `navigator.clipboard.writeText`, with a selection-based `document.execCommand('copy')` fallback for engines lacking the API or refusing it without focus — and swaps its own label to ✓ Copied / ⚠ Copy failed for 1.5 s. The two lines, with the solved puzzle's tier and time filled in:

  ```
  I solved a 🟣 Hard Emojikaku puzzle in 2:41.06 — can you beat it?
  file:///D:/dev/Shikaku/index.html#hard-1z141z3
  ```

  The tier token and the time come from the puzzle just solved; the link is the page's own URL with that puzzle's fragment, so it is correct wherever the file happens to be open.
- **A link is not an answer key.** The fragment carries the seed and nothing else. The recipient regenerates the board through the same generator, which still proves uniqueness before shipping, so nothing about the solution, the sender's regions or their mistakes travels with the link.
- **Solving a shared puzzle is an ordinary solve.** It files under the tier the link named (§6.1); nothing about sharing is stored (§6.2). There is no comparison with the sender, no leaderboard, and no marker that a puzzle arrived by link.
- **While the game is a local file, the link's path is the sender's** (§1). A recipient needs their own copy of `index.html`; only the fragment has to match, so links become universal the day the file is hosted.
- **Reproduction is same-engine.** The seed is a 32-bit integer and every draw comes from mulberry32, whose arithmetic is integer-exact; reproduction was measured to be identical across separate browser processes on Chromium/Edge, the engine this project already assumes (§9.1). The accept test uses `Math.log`, which is not IEEE-mandated, so a different engine could in principle land on the other side of a workload threshold sitting within 1 ulp of a band edge.
- **Seeds are for sharing, not for secrecy.** 2³² values are trivially enumerable.

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
| Region fills | 9-pastel palette at 0.45 alpha | same hues at 0.30 alpha |

- Type stack: Segoe UI / Roboto / system. Timer and stats use tabular numerals.
- Elevation: three shadow levels on cards and buttons; buttons lift 1–2 px on hover and settle on press.
- Motion: 0.3 s fade + 10 px rise on screen change; a 0.5 s scale bounce on the completion emoji. No animation on the grid itself.
- No audio of any kind: no sounds, no haptics.

### 7.2 Grid rendering

The grid is an inline **SVG**, rebuilt from scratch on every change (no per-cell DOM nodes, no canvas):

- Layer 0 grid background (rounded 6 px), layer 1 cell rectangles (rounded 3 px, translucent white on the grid background), layer 2 player/solution region fills with their area labels, layer 2b draw-preview and incorrect outlines, layer 3 emoji glyphs on top so they are never hidden.
- Cell size is computed to fit the viewport: available width minus the guesses panel (row mode) or the full width (column mode), available height minus 56 px header and 72 px control bar. Cell gap 2 px, outer padding 4 px. A span of n cells takes n cells plus (n-1) gaps plus the padding, and that whole span is what has to fit, so the board never overshoots the column it sits in (which would clip it). Re-computed on a 150 ms-debounced window resize.
- Region colours cycle through the nine pastels by creation order, so adjacent regions rarely collide.
- Area labels use the theme's foreground colour at full opacity, so they read against the region fill in both themes.
- Every colour is read from the theme when the SVG is built and baked into it, so a theme change repaints the board (an in-game toggle keeps the same regions) and repaints any tutorial diagram on screen.
- The draw-preview and incorrect outlines are painted as part of these layers and styled by class, not by per-region attributes, so their 4 px strokes are theme-correct everywhere they appear: white in dark mode, `#b71c1c` in light (§2.4).
- One renderer, `drawBoard(svg, model)`, paints the playable grid **and** the tutorial diagrams (§8.1). It sets both `width`/`height` and a matching `viewBox`: the game pins the pixel size because the pointer math assumes 1 CSS px = 1 SVG user unit, while the slides let CSS scale the same board down in a narrow column.

### 7.3 Emoji presentation

- Emoji glyphs are SVG text centred in the cell at 50% of the cell size (minimum 8 px), so they stay readable on an 8×8 board at phone width.
- No tap animation, no glow, no ring overlay, no per-emoji "confirmed" state — the emoji is decoration, and the guesses panel (§5.4) is where state is shown.
- Pool: 60 glyphs across six categories, chosen for visual variety, never for meaning (§2.5).

## 8. Onboarding & Tutorials

### 8.1 First-time flow as built

There is no forced first-run tour. Tutorial is an optional ✨ button on the menu, and playing immediately without it is supported.

```
Step 1 — What is Shikaku?      🧩   plain Shikaku rules in three sentences, drawn as a pair:
                                    the puzzle (four number clues on an empty grid) →
                                    the same grid divided into fills; the clue digit already
                                    states the area, so no extra labels are painted.
Step 2 — The Emoji Twist       🎲   the same division as a pair: "what you see" (four emoji
                                    clues) → "what is hidden" (fills plus area numbers), with
                                    one mapping line: 🍎 = 6   🌸 = 3   ⭐ = 2   ⚽ = 1.
Step 3 — How to Draw           ✋   pair: an accent draw-preview around the 6-cell rectangle
                                    mid-drag → a 4 px outline around a 4-cell rectangle that
                                    holds two emojis (🌸 and ⚽), i.e. what gets rejected.
Step 4 — Ready!                🚀   "Use ↩ Undo if you make a mistake. Click Play and
                                    have fun!" → returns to the menu.
```

← Back / Next → navigation; the last button reads "Let's Play! 🎉" and returns to the menu rather than starting a puzzle.

The three diagrams are the **same hand-authored 4×3 board** — regions of area 6, 3, 2 and 1, one clue each, with the clues deliberately *not* in a region's corner cell so the picture does not teach a false rule (§2.1). It is painted by `drawBoard()`, the game's own board renderer (§7.2), at a fixed 38 px cell, so the tutorial can never drift from the look of a real board. The board is fixed rather than generated: it must fit two-across in the tutorial column and stay identical between visits, and the Easy generator is not obliged to produce a four-value layout. Its emojis come from four different pool categories and carry no meaning (§2.5).

### 8.2 What onboarding does not do

It is painted-and-static only: the diagrams are real SVG boards, but nothing on them can be dragged, nothing animates, and they are not a puzzle — the example board is never playable (§10.2). There is no scratchpad lesson (there is nothing to teach there), and the tutorial is never shown automatically. An interactive first-puzzle walkthrough is future work (§10).

## 9. Technical Notes (as built)

### 9.1 Architecture

```
┌────────────────────────────────────────────────────────────────┐
│  index.html  — single file, no build, no dependencies, no server│
│                                                                │
│  <style>      CSS custom properties, light + dark tokens       │
│  <body>       five .screen divs + header                        │
│  <script>     one classic script; every helper is global        │
│                                                                │
│    state        one mutable object `S` (screens, puzzle, draws,│
│                 history, guesses, stats, timer, dark mode)      │
│    generation   seedRng (mulberry32) → randomTiling →           │
│                 findOpening → placeAnchors → buildRectIndex →   │
│                 countDeducibleDivisions → stepCost              │
│    rendering    SVG rebuilt per change; cell size fit to screen │
│    input        pointer events on the grid container + keyboard │
│    rules        commitDrawnRegion / validateSolution / prover   │
│                 share one rectangle model                       │
│    linking      '#' + tier + '-' + seed(base36) ↔ shown board   │
│    persistence  localStorage 'emojikaku'                        │
└────────────────────────────────────────────────────────────────┘
```

- **No framework, no modules, no bundler.** A single classic `<script>`; top-level `const`/`function` names are global, which is why the game is easy to probe from a browser console and why dead markup still has live event handlers (§11).
- **Generation and proving are synchronous** and cost a median 1.5 ms per puzzle on the heaviest tier, the worst of 480 sampled generations being 12.2 ms (§4.2); no Web Worker is warranted, and the 500,000-node budget shared by one generation is what bounds that tail. No wall-clock guard: generation must replay identically for a given seed (§6.3).
- **The prover is also the difficulty meter.** `countDeducibleDivisions` returns `{ sols, nodes, aborted, path }`: `sols` gates the puzzle, `nodes` only floors a tier, and the replayed `path` feeds `stepCost`, whose `{ ruledOut, widest }` is what the tiers band on (§3.3, §4.1).
- **Browser assumptions:** Pointer Events, CSS custom properties, `color-mix()`, inline SVG, `localStorage`, `location.replace` with a fragment-only URL, `performance.now()` as the only time source for the clock, and `navigator.clipboard` (falling back to a selection + `document.execCommand('copy')`). Developed and measured in Chromium/Edge.
- **Touch gestures:** `touch-action: none` on the grid container, plus a named non-passive `touchstart` handler that suppresses default touch behaviour while the board is still empty (it is attached and removed with the other grid listeners).
- **No reactive state.** `S` is one plain mutable object; every mutation is followed by an explicit `render*()` call. There is no diffing, no virtual DOM and no state library.
- **Failure handling:** generation failure walks a bounded four-seed ladder (§3.1) and then shows `alert()` and returns to the menu — it never re-enters `startPuzzle`. A malformed puzzle link is ignored rather than reported. `localStorage` read/write is wrapped in `try/catch` and silently ignored when storage is unavailable.

## 10. Future / Post-Launch Ideas

Nothing in this section is built. Each item is a proposal, not a description of the game.

### 10.1 Ideas

| Feature | Priority | Notes |
|---------|----------|-------|
| **Daily puzzle** | Medium | One pre-selected puzzle per 24 h, ranked against the player's own tier history. Generation is already seeded and link-addressable (§6.3); what is missing is a date → seed rule. |
| **Weekly challenge** | Low | Two expert puzzles (mid-week and weekend). Depends on the daily-puzzle machinery. |
| **Emoji themes** | Low | Seasonal pools (Halloween 🎃, Christmas 🎄). Purely cosmetic — one line of code, since the pool is already a flat array. |
| **Editor mode** | Medium | Players design and share grids; the system proves uniqueness (or rejects) and generates emoji mappings for shared puzzles. |
| **Multiplayer (async)** | Low | Two players solve the same seed; compare time. The seeds and the puzzle link are built (§6.3); what is missing is an identity. |
| **Collaborative solver** | Low | Share puzzle state by link; comment on strategy. |
| **API / widget** | Low | "Puzzle of the day" home-screen widget. |
| **App packaging / PWA** | Medium | Manifest + service worker so the single file installs offline; the code is already self-contained. The menu mark (§5.2) cannot be shipped as the icon unchanged: measured at 16 px its 4-unit gaps bleed shut and the whole thing fuses into a blob, and no face-like detail survives below ~40 px — so the icon needs a faceless, gap-surviving variant, not the same drawing smaller. |
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
| **Displaying the hidden numbers**, including after Reveal | Would remove the only thing the game asks the player to do. The "Show Numbers on Reveal" setting was a no-op and was deleted. The exemption is in-game only: the tutorial's example board (§8.1) prints plain numbers because nothing is hidden there — it is a diagram, and it is never playable. |
| **Lifetime solved counter, streak counter, "weird facts"** | The streak measured a habit the game does not support (no daily content); the facts card had no data behind it; a cross-tier solved count is not a useful number. Stats are per tier only (§6.1). |
| **Mistake-rate statistics** | With uniqueness guaranteed, "not quite right" is pure player error and not a signal worth surfacing. |
| **Per-cell marks / pencil candidates** | Shikaku's own notation does not help when the numbers are hidden; region sizes carry the reasoning. |

## 11. Known Gaps & Defects

Observed in the current build, in rough order of how much they cost the player:

| Item | Symptom | Where |
|------|---------|-------|
| **Reveal's confirmation toast never appears** | `showToast` appends to `getElementById('game-header')`, but the element is `class="game-header"` with no id, so the node is discarded. Reveal gives no feedback beyond the drawn regions. | `showToast()` |
| **Clear is not undoable** | Clear takes no history snapshot, so the next Undo returns to the state before the last create/remove rather than the board as it was before clearing. | Clear button handler |
| **Puzzles are lost silently** | ← Menu stops the timer and discards the board; Play always generates a new puzzle. There is no confirmation and no resume. The board itself is no longer lost — the address bar carries its link (§6.3), so reloading regenerates the same puzzle — but the drawn regions, undo history and timer are. | `btn-back`, `startPuzzle()` |
| **Filler copy on the success screen** | The completion card always reads "Well done." — a leftover of the deleted facts feature. | `showCompletion()` |

