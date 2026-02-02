# Plates - Application Specification

A barbell weight plate calculator that determines which plates to load on a barbell to achieve a target weight.

## Overview

**Purpose:** Given a target total weight, a bar weight, and an inventory of available plates, the application calculates the optimal plate loading configuration. It finds the smallest achievable weight that is ≥ the target.

**Unit-agnostic:** The app does not display or enforce any unit (kg, lbs, or otherwise). All numbers are just numbers. Users pick one unit system and use it consistently. The app works identically for any unit.

**Design philosophy:** Keep everything as simple as possible. No unnecessary features, no complex configuration, no over-engineering. A focused tool that does one thing well.

---

## Core Concepts

### Symmetric Loading
Plates must be loaded symmetrically on a barbell - the same plates go on each side. Therefore:
- Only **pairs** of plates can be used
- If you have 5 plates of a certain weight, only 4 are usable (2 per side)
- Odd single plates are effectively ignored in calculations

### Weight Calculation
```
Total Weight = Bar Weight + (2 × Sum of plates per side)
```

### Optimization Goals (in priority order)
1. **Minimize overshoot** - Get as close to target as possible without going under
2. **Minimize plate count** - Among solutions with equal overshoot, prefer fewer total plates

---

## User Interface

### Inputs Section

#### Total (Target Weight)
- Text input with decimal keyboard on mobile
- Accepts comma as decimal separator (for locales that use comma)
- Clear button (×) to reset the field
- Pressing Escape while focused clears the field

#### Bar Weight
- Text input with decimal keyboard on mobile
- Accepts comma as decimal separator
- Clear button (×) to reset the field
- Pressing Escape while focused clears the field

### Result Section

Displays calculation results in real-time (debounced ~60ms):

**When valid inputs:**
- **Achieved total** - The actual weight achievable (bolded, prominent)
- **Per side** - Plate breakdown for one side (e.g., "25 + 10 + 5")
- **Total plates** - Total number of plates used (both sides combined)
- **Over by** - Amount exceeding target (shown in red/danger color, only if not exact)

**When showing multiple solutions:**
- All equally-optimal combinations are displayed
- Each shown as a separate "Per side:" line

**Special states:**
- Empty inputs → Shows "—"
- Target below bar weight → Shows bar weight as minimum achievable with explanatory note
- No solution possible → Shows "No solution"
- Odd plates exist → Shows note that only pairs are usable

### Plates Section

A table showing the plate inventory:

| Weight | Available | Action |
|--------|-----------|--------|
| 25     | 4         | × (remove) |
| 20     | 4         | × (remove) |
| ...    | ...       | ... |

**Plate rows:**
- Weight is displayed (read-only)
- Available count is editable (number input, step of 2)
- Remove button (×) deletes the plate type entirely

**Available count behavior:**
- While typing: accepts any value
- On blur (leaving field): rounds DOWN to nearest even number
- Reason: If the gym has 5 plates of a weight, only 4 are usable (2 pairs). Rounding down reflects reality - you can't use that 5th plate.

**Add plate row (at bottom):**
- Weight input (text, decimal keyboard)
- Available input (number)
- Add button (+)

**Adding plates:**
- If weight already exists, **replaces** the available count (not additive)
- If new weight, adds to inventory
- Inventory is always sorted by weight descending
- After adding, weight input clears and receives focus for quick entry of multiple plates

### Footer

**Reset button:**
- Returns all values to defaults
- Clears localStorage
- Resets inventory to default plate set

---

## Default Values

### Default Bar Weight
```
25
```

### Default Target Weight
```
112
```

### Default Plate Inventory
| Weight | Available |
|--------|-----------|
| 25     | 4         |
| 20     | 4         |
| 15     | 4         |
| 10     | 4         |
| 5      | 8         |
| 2.5    | 8         |
| 1.5    | 8         |
| 1.25   | 8         |

Note: The 1.5 and 1.25 weights are intentional - these are common in some gyms for fine-grained adjustments.

---

## Data Persistence

- All state is saved to `localStorage`
- Storage key: `weightsAppSettings.v7`
- Saved on every change (debounced)
- State restored on page load

### Stored Data Structure
```json
{
  "targetWeight": "112",
  "barWeight": 25,
  "plates": [
    { "id": "p_abc123_def456", "weight": 25, "available": 4 },
    ...
  ]
}
```

---

## Algorithm Details

### Internal Representation
- All calculations use "centi-units" (×100) to avoid floating-point errors
- Example: 2.5 is stored as 250 internally during calculation

### Solver Algorithm

1. **Validate inputs** - Check target and bar are finite, non-negative numbers

2. **Calculate need per side:**
   ```
   needTotal = max(0, target - bar)
   needPerSide = ceil(needTotal / 2)
   ```

3. **Build plate types array:**
   - Filter to plates with positive weight and at least 2 available (1 pair)
   - Calculate `maxPerSide = floor(available / 2)`
   - Sort by weight descending

4. **Check feasibility:**
   - Sum maximum possible weight per side
   - If less than needPerSide, return "no solution"

5. **Greedy upper bound:**
   - Take as many of largest plates as needed
   - Move to smaller plates until target reached
   - This gives an initial "best" to beat

6. **DFS optimization:**
   - Explore all combinations
   - Prune branches that can't improve on current best
   - Track multiple equally-optimal solutions
   - Optimization criteria: minimize overshoot, then minimize plate count

7. **Return all optimal solutions** with same overshoot and plate count

### Edge Cases

- **Target < Bar:** Returns bar weight as minimum achievable, with explanatory note. This is intentional - it's informative (tells user the minimum) rather than treating it as an error.
- **Negative inputs:** Treated as invalid
- **Empty inputs:** Shows placeholder ("—"), no calculation
- **Empty inventory:** "No solution" if target > bar; shows bar alone if target ≤ bar
- **No available plates with positive pairs:** Same as empty inventory

---

## Number Input Handling

### Locale Support
- Comma (`,`) is normalized to period (`.`) for decimal separator
- Whitespace is stripped
- Example: `"2,5"` → `2.5`

### Validation
- Must be a finite number
- Weights must be positive
- Available counts must be non-negative integers

### Invalid Input Handling
Invalid inputs are handled silently (no error messages or visual error states):
- Invalid target/bar (e.g., "abc") → shows "—" (no result)
- Invalid plate weight when adding → add is ignored, focus stays on input
- Invalid available count → treated as 0

This keeps the UI clean for mobile use. Users quickly learn to enter valid numbers.

---

## Styling

- Uses Pico CSS framework (v2.1.1)
- Supports light and dark modes (via `color-scheme` meta)
- Max width: 860px
- Responsive design for mobile (breakpoint at 520px)
- Color coding:
  - Green tint: add/constructive actions
  - Red tint: remove/destructive actions, overshoot warning

---

## Technical Notes

- Single HTML file, no build step, no frameworks
- Mobile-first design
- Decimal precision: 2 decimal places
- Works fully offline (only external dependency: Pico CSS from CDN)
- Debounced recalculation (~60ms) for responsive feel without excessive computation

---

## Wishlist for Rewrite

> Everything above describes the **current** application behavior. This section describes **changes** for the rewrite.

### Remove Pico CSS dependency
- Custom minimal CSS instead
- System font stack (no web fonts)
- Dark/light mode via `prefers-color-scheme` (automatic, no toggle)
- No color tints for buttons - just rely on × and + symbols being universally understood

### More compact UI

**Layout:**
- Target + Bar inputs on one line (if still clear what each is for - maybe inline labels or placeholders)
- Remove section headers ("Inputs", "Plates") - context is obvious
- Smaller app title, or remove entirely
- Keep clear (×) buttons - they're valuable because they clear the entire input AND pop up the keyboard, making it fast to enter a new value

**Results:**
- Tighter/more compact result display - just essentials

**Plate inventory:**
- Keep table format (clear for editing)
- Make rows tighter: smaller padding, smaller fonts, weight and count closer together

### General principles
- Everything as small as reasonably possible while staying usable
- Ideally fits on one screen without scrolling (but don't assume all screens)
- Minimal, beautiful, well thought out
