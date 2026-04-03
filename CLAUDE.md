# CLAUDE.md

**This file is the source of truth for this project.** Keep it updated whenever behavior, architecture, or conventions change. When in doubt, trust this file over comments in code or other docs. If you make a change that contradicts something here, update this file.

---

## What This Is

Plates is a barbell weight plate calculator. Given a target weight, a bar weight, and an inventory of available plates, it calculates the optimal plate loading configuration — the smallest achievable weight that is >= the target.

## Tech Stack

- **Single-file app**: Everything lives in `index.html` (HTML + CSS + JS, ~1000 lines)
- **No frameworks, no dependencies, no build step**
- **Vanilla JavaScript** — no libraries
- **Custom CSS** with pixel-art inspired retro design
- **Font**: VT323 (embedded as base64 data URL in the HTML)
- **Favicon**: Embedded SVG data URL
- **Fully offline** — zero network requests after initial load

## How to Run

Open `index.html` in a browser. That's it. No server, no build, no install.

## Project Structure

```
index.html              The entire app
SPECIFICATION.md        Detailed spec (may be slightly outdated — trust CLAUDE.md and the code)
VT323.woff2             Fallback font file (not currently referenced; font is base64-embedded)
```

## Core Concepts

- **Unit-agnostic**: No kg/lbs — just numbers. User picks a unit and stays consistent.
- **Symmetric loading**: Plates go on both sides equally. Only pairs are usable (5 plates = 4 usable).
- **Formula**: `Total = Bar + (2 x sum of plates per side)`
- **Optimization priority**: (1) minimize overshoot, (2) minimize plate count

## UI Overview

- **Inputs**: Total (target weight) and Bar weight on one row, with clear (x) buttons
- **Results**: Achieved weight, per-side breakdown, total plate count, overshoot (red, only if non-zero)
- **Multiple solutions**: All equally-optimal combos shown as separate "Per side:" lines
- **Plate inventory table**: Weight (read-only), Available (editable, step 2), Remove button
- **Add plate row**: At bottom of table with Weight input, Qty input, + button
- **Reset button**: Returns everything to defaults, clears localStorage
- **Dark/light mode**: Automatic via `prefers-color-scheme`
- **Max width**: 420px, mobile-first responsive design

## Data Persistence

- **localStorage key**: `platesAppSettings.v7`
- Saved on every change (debounced)
- State restored on page load

### Stored structure:
```json
{
  "targetWeight": "112",
  "barWeight": 25,
  "plates": [
    { "id": "p_abc123_def456", "weight": 25, "available": 4 }
  ]
}
```

## Default Values

- Bar weight: 25
- Target weight: 112
- Plates: 25(4), 20(4), 15(4), 10(4), 5(8), 2.5(8), 1.5(8), 1.25(8)

## Algorithm

1. Validate inputs (finite, non-negative)
2. Calculate need per side: `ceil(max(0, target - bar) / 2)`
3. Build plate types (filter to >=2 available, calc maxPerSide, sort desc)
4. Feasibility check (can max plates reach target?)
5. Greedy upper bound (largest plates first)
6. DFS optimization with pruning (finds all equally-optimal solutions)
7. Returns all solutions with same overshoot and plate count

**Floating-point handling**: All calculations use centi-units (x100) internally to avoid precision errors.

## Input Handling

- Comma normalized to period (locale support: `2,5` -> `2.5`)
- Whitespace stripped
- Invalid inputs handled silently (no error messages — just shows "—")
- Escape key clears focused input
- Available count rounds DOWN to nearest even on blur
- Adding a plate with existing weight replaces the count (not additive)
- Inventory always sorted by weight descending

## Edge Cases

- **Target < bar**: Shows bar weight as minimum, with explanatory note
- **Empty inputs**: Shows "—"
- **No solution**: Displayed when plates can't reach target
- **Odd plate counts**: Note shown that only pairs are usable

## Styling

- Pixel-art retro aesthetic with VT323 monospace font
- Light mode: beige background (#e8e4d9), dark text, green accents (#4a7c59), red danger (#a63d40)
- Dark mode: dark blue background (#1a1a2e), light text, lighter green (#7ec8a3), lighter red (#e76f6f)
- Box-shadow depth on buttons, 2px translate on active
- Debounced recalculation (~60ms)
