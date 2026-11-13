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
| Numbers are visible in **one cell per region** (the anchor). | One emoji replaces the number **per region**, shown on a single anchor cell (top-left of the region). All other cells in the region are blank. |
| The player reads the numbers directly from anchor cells. | The mapping from emojis → values is **hidden and randomized per puzzle**. Players must deduce which emoji maps to which area by drawing regions. |

#### 2.3 Drawing regions

- **Drag** across adjacent cells on the grid to draw a rectangular region. The preview updates live as you drag.
- **Click an existing player region** (any cell within it) to remove that region. This is the only way to correct mistakes — there is no per-cell "mark" or value assignment step.
- Regions must be perfect rectangles and cannot overlap with other player regions.

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
GenerateShikaku(gridWidth, gridHeight):
    // 1. Create a valid Shikaku division using backtracking.
    division ← SolveBacktrack(gridWidth, gridHeight)

    // 2. Extract the unique value from each region's anchor cell.
    values ← { region.value for region in division }   // e.g. {1,3,4,2,5}

    // 3. Pick a random subset of emojis (same size as |values|)
    //    chosen uniformly from the available emoji pool. No semantic
    //    filtering — assignments are purely random for maximum obfuscation.
    emojiPool ← GetRandomEmojis(|values|)

    // 4. Randomly assign emojis to values (permutation).
    mapping ← Shuffle(emojiPool)   // maps value → emoji

    return GridWithEmojis(division, mapping)
```

#### 3.2 Backtracking constraints

- At each recursion step, try all possible rectangular placements that:
  - Cover an unvisited cell.
  - Do not exceed the grid dimensions.
  - Can lead to a complete tiling of the entire grid.
- Standard Shikaku generation heuristics apply (first-empty-cell strategy: always place the rectangle covering the first unplaced cell, which biases toward well-formed, connected puzzles).
- **Largest-first ordering with shuffle**: candidate rectangles are tried in descending size order so large regions get placed before being broken into 1×1 cells; a Fisher-Yates shuffle then randomises the order so consecutive puzzles differ even on identical grid dimensions.
- **No upper cap on region area** — any positive integer area is valid, as long as regions tile the entire grid without overlap.

#### 3.3 Quality checks

After generation, run these filters:

| Check | Description | Pass criterion |
|-------|-------------|----------------|
| **Uniqueness** | The division is uniquely determined by logic alone — no guessing required. | Single solution found via constraint propagation. |
| **Emoji uniqueness** | No two values share the same emoji within a puzzle. | Enforced by permutation step. |

### 4. Difficulty System

#### 4.1 Factors that influence difficulty

| Factor | Impact |
|--------|--------|
| **Grid size** | 3×3 → easiest; 12×12+ → expert |
| **Number of regions** | More regions = more constraints = generally easier per-region, but harder global logic. Balanced by grid size. |
| **Constraint density** | Regions that force early deductions vs. those left until the end. Puzzles with few "forced moves" at start are harder. |

#### 4.2 Difficulty tiers

| Tier | Grid range | Regions | Description |
|------|-----------|---------|-------------|
| 🟢 Easy | 3×3 – 5×5 | ≤ 8 | First-time friendly; straightforward divisions. |
| 🔵 Medium | 6×6 – 8×8 | 9–16 | Comfortable Shikaku experience with moderate deduction chains. |
| 🟣 Hard | 9×9 – 10×10 | 17–25 | Requires systematic logic; ambiguous emoji labels add cognitive friction. |
| 🔴 Expert | 11×11+ | 26+ | For veterans; long deduction chains and many region interactions needed. |

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
