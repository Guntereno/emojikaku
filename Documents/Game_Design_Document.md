# Emojikaku — Game Design Document

## 1. Overview

| Item | Detail |
|------|--------|
| **Working title** | *Emojikaku* |
| **Genre** | Puzzle — logic / spatial division |
| **Platform(s)** | Mobile (iOS / Android) as primary; web PWA as secondary |
| **Target audience** | Ages 10+, puzzle enthusiasts, casual commuters |
| **Single-player / Multiplayer** | Single-player, asynchronous leaderboards optional |


### 1.1 Elevator pitch

A standard game of Shikaku — a rectangle is subdivided into rectangular regions, each containing exactly one number whose value equals the region's area — reimagined with emojis in place of numbers. Emojis are assigned randomly to values per puzzle, serving purely as obfuscation. Players must deduce which emoji maps to which number by analysing region sizes alone.

### 2. Core Gameplay

#### 2.1 Shikaku rules (reference)

1. A grid contains several cells, each **optionally** labelled with a positive integer (**one number per region only**, placed at the top-left / anchor cell). Other cells in a region are blank.
2. Divide the entire grid into rectangular regions so that:
   - Every region is a solid rectangle (width × height ≥ 1).
   - Each region contains exactly one numbered cell (**the anchor**).
3. The area (number of cells) of a region must equal the number in its **anchor** cell.

#### 2.2 Emoji twist — how it differs

| Standard Shikaku | Emojikaku |
|-----------------|---------------|
| Numbers are visible in **one cell per region** (the anchor). | One emoji replaces the number **per region**, shown on a single anchor cell placed at random within the region. All other cells in the region are blank. |
| The player reads the numbers directly from anchor cells. | The mapping from emojis → values is **hidden and randomized per puzzle**. Players must deduce which emoji maps to which area by drawing regions. |

#### 2.3 Drawing regions

- **Drag** across adjacent cells on the grid to draw a rectangular region. The preview updates live as you drag.
- **Click an existing player region** (any cell within it) to remove that region. This is the only way to correct mistakes — there is no per-cell "mark" or value assignment step.
- Regions must be perfect rectangles and cannot overlap with other player regions.
- A region must contain **at least one emoji** and never two **different** emojis, but several cells showing the *same* emoji may share one region. Draws that break these rules are rejected on release, which is why the uniqueness proof (§3.3) has to rule out that freedom explicitly.

#### 2.4 Completion check

- The puzzle is validated **only after all cells are covered** by player-drawn regions.
- When the last cell is placed, the game checks every region: each must be a valid rectangle whose area matches the hidden solution value of its cells.
- If correct → 🎉 success screen with time and stats.
- If incorrect → 😕 failure message explaining that some regions don't match, with the option to try again.
- There is no incremental validation or per-region confirmation — players must rely entirely on their own reasoning.

#### 2.5 Emoji‑value mapping

- Region values can be **any positive integer** (matching real Shikaku rules). In a standard Shikaku grid, the number in each region equals that region's area — so a 4×3 rectangle has value 12. Multiple regions may share the same value; this is completely normal and expected.
- On puzzle generation, each distinct value present in the grid receives a **fully random** emoji drawn from a pool of available emojis. The assignment is purely random — no semantic relationship between an emoji's appearance and the value it represents.
- The mapping is consistent for that entire puzzle (e.g. 💩 might always equal 7 for that puzzle, even if two separate regions both have value 7) but hidden until the puzzle is completed or surrendered.

> **Design principle:** Emojis are used to *obfuscate* numbers, not to hint at them. Each distinct value gets its own emoji (so two regions with value 4 share the same emoji but are spatially distinct). Players must rely entirely on Shikaku logic (region sizes) to deduce which emoji corresponds to which value. The visual randomness of emojis adds a layer of cognitive dissonance that makes solved grids feel surprising ("Wait, the poop emoji was 4?!").

### 3. Puzzle Generation

#### 3.1 Generation algorithm (top-down)

```
GeneratePuzzle(difficulty):
    plan ← planFor(difficulty)          // cap ladder, tries per cap, difficulty target
    best ← none

    for cap in plan.caps:
        for attempt in 1..plan.tries:
            // 1. Tile the grid with rectangles, none larger than `cap`.
            division ← RandomTiling(gridWidth, gridHeight, cap)

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
    return FallbackPuzzle(difficulty)
```

`BuildPuzzle` then labels the division: the distinct region values are collected, a same-sized subset of the emoji pool is drawn uniformly at random, and a shuffled permutation assigns emoji to value. There is no semantic filtering — assignments are purely random for maximum obfuscation. Equal values share an emoji; different values never do.

#### 3.2 Backtracking constraints

- At each recursion step, try all possible rectangular placements that:
  - Cover an unvisited cell.
  - Do not exceed the grid dimensions.
  - Can lead to a complete tiling of the entire grid.
- Standard Shikaku generation heuristics apply (first-empty-cell strategy: always place the rectangle covering the first unplaced cell, which biases toward well-formed, connected puzzles).
- **Fisher-Yates shuffle of candidate placements**: at each recursion step every fitting rectangle is offered in random order, so consecutive puzzles differ even on identical grid dimensions.
- **Region-area caps are a sampling knob, not the difficulty dial.** Each tier walks a short ladder of caps — easy 8 → 6 → 4, medium 8 → 10 → 6, hard and expert 8 → 6 — cheapest and most promising first, and stops as soon as a candidate is in its band. The old large rungs (12 and 14 cells) were dropped: on a 9×9 grid and up, ~90% of their divisions could not be proven unique inside the node budget, so they cost time and produced nothing.
- **Opening pins are anchored deliberately.** The boxed 1×1 region and every neighbour region keep their pinned cell as the emoji position; all remaining regions place their anchor on a random cell of the region, so players cannot infer where a region begins. Only the fallback pins every region to its top-left corner.

#### 3.3 Quality checks

These checks screen every candidate inside the generation loop — a puzzle is shown only once it passes:

| Check | Description | Pass criterion |
|-------|-------------|----------------|
| **Uniqueness** | The division is uniquely determined by logic alone — no guessing required. | Exactly one division survives the search. Two divisions — or a search that runs out of node budget and therefore proves nothing — rejects the puzzle. |
| **Prover's rule set** | Only the rules the player is taught are applied: rectangles tiling the grid, each region holding emojis of a single value (two cells of the same emoji may share a region, as the drawing rules allow), equal emojis sharing one area, different emojis having different areas. | The prover never assumes the region-size cap or the opening construction, neither of which the player can know. |
| **Opening move** | Some emoji is provable on sight, so the puzzle is never a wall. | A legal-rectangle scan leaves at least one anchored cell covered only by rectangles of one area. Built by the box-in, then verified rather than trusted. |
| **Difficulty band** | The puzzle sits near its tier's target instead of anywhere in the tier's natural range. | First candidate within ±35% of the target prover cost is accepted outright. |
| **Emoji uniqueness** | No two values share the same emoji within a puzzle. | Enforced by the permutation step. |

Uniqueness is a hard gate, not a preference: a puzzle is shown only after it is proven, because ambiguous puzzles push players into trial-and-error. Proving is the expensive part, so generation is bounded by a 200,000-node search budget and an 800 ms wall-clock guard. Measured in-browser, per puzzle: ≈0.5 ms (easy), ≈3 ms (medium), ≈22 ms (hard), ≈106 ms mean / 262 ms at p95 (expert).

### 4. Difficulty System

#### 4.1 Factors that influence difficulty

| Factor | Impact |
|--------|--------|
| **Proof cost — the dial** | The prover's node count: how much case-analysis it takes to see that the division is the only one. Each tier samples until a candidate lands near its target, so this, not the region sizes, is what sets a tier. |
| **Grid size** | 3×3 → easiest; 14×14 → expert. Grid range is also the visible difference between the top two tiers, whose proof targets differ by only ~4×. |
| **Number of regions** | More regions = more constraints = easier per-region, but harder global logic. A consequence of the cap ladder and grid size, not aimed at directly. |
| **Constraint density** | Regions that force early deductions vs. those left until the end. Measured difficulty covers this implicitly: a puzzle whose deductions arrive late costs the prover more. |
| **Max region area** | Now only a sampling knob. Large caps leave the division under-constrained, so the proof drags or runs out of budget; the tiers therefore sample small caps, where a proof is affordable and the difficulty can be chosen. |
| **Forced opening** | No longer a difficulty lever — every puzzle is guaranteed at least one emoji that is provable on sight, so "no way in" is not a thing the tiers vary. |

Size is no guide to difficulty: two 90-cell puzzles measured 2,800 and 109,000 proof nodes. That is why the tiers sample rather than tune dimensions.

#### 4.2 Difficulty tiers

Grid ranges, region counts and cost ranges are what the current generator actually produces, measured over 60 puzzles per tier. Cost ranges are the 5th–95th percentile of prover nodes for the puzzles that were actually shown; tier targets are 45, 1,000, 22,000 and 80,000 nodes.

| Tier | Grid range | Regions | Shown cost p5–p95 | Description |
|------|-----------|---------|-------------------|-------------|
| 🟢 Easy | 3×3 – 5×5 | 5–15 | 33–63 | First-time friendly; a visible truth and short divisions. |
| 🔵 Medium | 6×6 – 8×8 | 9–24 | 597–2,024 | Comfortable Shikaku with moderate deduction chains. |
| 🟣 Hard | 9×9 – 10×10 | 19–33 | 14,700–35,000 | Requires systematic logic; ambiguous emoji labels add cognitive friction. |
| 🔴 Expert | 11×11 – 14×14 | 26–60 | 55,000–139,000 | For veterans; long deduction chains and many region interactions. |

Before banding, Hard puzzles ranged over 2,900–112,600 nodes — a forty-fold spread that made the tier name mean something different on consecutive puzzles. Banding holds each tier inside roughly a 2× window, and on these samples the tiers do not touch: Hard's p95 sits below Expert's p5.

#### 4.3 Recommended daily/weekly content

- **Daily puzzle**: One pre-selected medium/hard puzzle with a unique emoji mapping, reset every 24 h.
- **Weekly challenge**: Two expert puzzles (one mid-week, one weekend special).

### 5. Player Interface

#### 5.1 Main screens

```
┌──────────────────────────────────────┐
│ [Logo]          ☰  🏆  ⚙️          │
│                                      │
│         ┌─────┬─────┬─────┐          │
│         │  a  │  b  │  c  │          │
│    ┌────┼─────┼─────┼─────┤          │
│    │ d  │ e   │  f  │  g  │ ← Grid   │
│    ├────┼─────┼─────┼─────┤          │
│    │  h │  b  │  a  │ e   │          │
│    └────┴─────┴─────┴─────┘          │
│                                      │
│  🖊️ Draw lines        ⏮ Undo      │
│                                      │
│  Difficulty: 🔵 Medium       ▶ Next │
└──────────────────────────────────────┘

```

#### 5.2 Controls & interactions

| Action | Input | Effect |
|--------|-------|--------|
| **Start drawing** | Touch / click on a cell and drag (or single-tap two opposite corners). | Begins selection of a rectangular region. |
| **Complete region** | Release finger / lift mouse when the rectangle is correct. | Region is drawn and stored. No immediate validation — regions are only checked when all cells are covered. |
| **Cancel draw** | Tap away from the current rectangle or press Esc/Back. | Deselects without committing. |
| **Undo** | Button or swipe gesture (or Ctrl+Z on desktop). | Reverses the last region placement; any regions drawn *inside* an undone region are also removed. |
| **Reveal solution**  | Button (shows divider). | Reveals all remaining divisions and maps; can be used only after giving up the puzzle. |

There is no hint mechanic. Undo and Reveal are the only aids, and Reveal ends the puzzle.

#### 5.3 Region management

There is no value-assignment or emoji-reveal mechanic. Players interact with regions directly:

| Action | Input | Effect |
|--------|-------|--------|
| **Remove a region** | Tap any cell within an existing player-drawn region. | The entire region is removed and can be redrawn. |
| **Undo** | Button or swipe gesture (or Ctrl+Z on desktop). | Reverses the last region placement; any regions drawn *inside* an undone region are also removed. |

The absence of a scratchpad or value-declaration system means players must reason entirely in their head or use external notes. This is intentional — it preserves the pure logic-puzzle feel.

### 6. Progression & Economy

#### 6.1 Player stats

| Stat | Purpose |
|------|---------|
| **Puzzles solved** | Overall experience / level. |
| **Streak** | Consecutive days with at least one solve. |
| **Personal best** | Fastest solve time per grid size / difficulty. |
| **Weird facts** | Counts of memorable emoji-to-value combinations discovered on solved puzzles — surfaced as shareable cards at puzzle completion. |

#### 6.2 In-game currency (optional)

| Currency | Earned by | Spent on |
|----------|-----------|----------|
| 💎 Gems | Daily login, streaks, leaderboard finish | Themed emoji packs (cosmetic only). |

### 7. Visual & Audio Direction

#### 7.1 Art style

- **Clean, minimalist grid** with generous cell padding — similar to *Unblock Me* / *Flow Free* aesthetics.
- **Background**: soft off-white or dark mode variant; no texture.
- **Line colour**: thin, high-contrast stroke (e.g. `#2D3142` on light, `#E8E0D5` on dark).
- **Region fills**: pastel wash corresponding to the emoji's assigned hue category (warm / cool / neutral) — not tied to value.

#### 7.2 Emoji presentation

- Emojis rendered at a readable size (~40 % of cell height).
- On tap, emojis receive a brief scale-up animation with a subtle glow on correct region completion.
- "Confirmed" emojis (player-declared via double-tap/long-press) get a thin ring overlay so the player can see their scratchpad state at a glance.
- Emoji pool drawn from a diverse set of categories (faces, animals, objects, symbols, food, nature) purely for visual variety — no semantic intent.

### 8. Onboarding & Tutorials

#### 8.1 First-time flow

```
Screen 1 — What is Shikaku?
    • "Divide the grid into rectangles!"
    • Animated diagram of a solved 3×3 Shikaku (with numbers shown).

Screen 2 — Meet the Emojis
    • "Now the numbers are hidden behind emojis!"
    • Show a solved grid side-by-side: one with numbers, one with emojis. "The same puzzle — different skin."
    • Explain that each emoji consistently represents one number per puzzle (e.g. 💩 is always 4 or always 2, never both).
    • Teach that the value must be deduced from region size alone.
    • Show a 3×3 puzzle with emoji overlay + brief hint on how to use area to deduce mappings.

Screen 3 — How to draw
    • Interactive mini-game: drag a rectangle of the correct size (with value shown temporarily so player learns the mechanic).

Screen 4 — Scratchpad
    • Teach double-tap / long-press to declare what an emoji means.
    • Show how marks persist across attempts within the same puzzle.

Screen 5 — First real puzzle (easy, 4×4)
    • Full gameplay — no assistive aids; the player reasons from region sizes alone.
```

### 9. Technical Notes

#### 9.1 Architecture overview

```
┌─────────────────┐      ┌──────────────────┐      ┌────────────────┐
│   UI Layer       │◄────►│  Game Logic Core  │◄────►│ Puzzle Generator│
│  (React Native / │      │  - Validation     │      │  (backtracking) │
│   Flutter)       │      │  - State machine  │      │                 │
└─────────────────┘      │  - Undo/redo stack│      │                 │
                         └──────────────────┘      └────────────────┘
                                 ▲
                                 │
                         ┌──────────────────┐
                         │   Persistence     │
                         │  (SQLite / Room)  │
                         └──────────────────┘
```

#### 9.3 Recommended tech stack

| Layer | Recommendation |
|-------|----------------|
| Prototype | **Single `index.html` file** with inline `<style>` and `<script>`. Zero setup — just open in a browser for instant feedback. Uses vanilla JS + HTML5 Canvas (or DOM-based grid). |
| Puzzle generation | Local backtracking engine in JavaScript, running entirely in the browser. No server required. |
| Persistence | **`localStorage`** / **IndexedDB** via the Web Storage API — stores progress, preferences, stats directly in the browser. |
| Analytics | (Out of scope for prototype; defer until post-launch.) |

> **Rationale:** Starting with a single HTML file gives the fastest possible feedback loop during prototyping — no build tools, no bundlers, no server. The entire game runs locally when you double-click `index.html`. Once the core mechanics are solid, the logic can be extracted into a proper mobile framework (Flutter / React Native) if desired.

### 10. Future / Post-Launch Ideas

| Feature | Priority | Notes |
|---------|----------|-------|
| **Emoji themes** | Low | Switch emojis per season (e.g., Halloween 🎃, Christmas 🎄). Purely cosmetic — values remain the same. |
| **Editor mode** | Medium | Let players design and share their own Shikaku grids; system auto-generates emoji mappings for shared puzzles. |
| **Multiplayer (async)** | Low | Two players solve the same grid; results compared by time. |
| **Collaborative solver** | Low | Invite a friend to view your puzzle state via link and comment on strategy. |
| **API / widget** | Low | "Puzzle of the day" home-screen widget for mobile. |
