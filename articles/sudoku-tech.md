# Building a Sudoku Game in React — Backtracking Solver, Puzzle Generation, Notes Mode, and Hints

Sudoku has more implementation depth than it looks. You need a solver (backtracking), a puzzle generator (solved board → remove cells → ensure unique solution), difficulty levels, notes mode (pencil marks), hints, and a highlight system for related cells.

Here's how I built the [Sudoku game](https://ultimatetools.io/tools/fun-tools/sudoku/) in React with all of these working correctly.

## The Backtracking Solver

Everything starts here. The solver fills a 9×9 grid following Sudoku rules: each number 1–9 appears exactly once per row, column, and 3×3 box.

```typescript
type Grid = (number | null)[][]

function solve(grid: Grid): boolean {
  const empty = findEmpty(grid)
  if (!empty) return true // no empty cells = solved

  const [row, col] = empty

  for (let num = 1; num <= 9; num++) {
    if (isValid(grid, row, col, num)) {
      grid[row][col] = num
      if (solve(grid)) return true
      grid[row][col] = null // backtrack
    }
  }
  return false
}

function findEmpty(grid: Grid): [number, number] | null {
  for (let r = 0; r < 9; r++)
    for (let c = 0; c < 9; c++)
      if (grid[r][c] === null) return [r, c]
  return null
}

function isValid(grid: Grid, row: number, col: number, num: number): boolean {
  // Check row
  if (grid[row].includes(num)) return false

  // Check column
  if (grid.some(r => r[col] === num)) return false

  // Check 3×3 box
  const boxRow = Math.floor(row / 3) * 3
  const boxCol = Math.floor(col / 3) * 3
  for (let r = boxRow; r < boxRow + 3; r++)
    for (let c = boxCol; c < boxCol + 3; c++)
      if (grid[r][c] === num) return false

  return true
}
```

## Generating a Valid Solved Board

Start from an empty grid, fill it with a randomized solver:

```typescript
function generateSolvedGrid(): Grid {
  const grid: Grid = Array.from({ length: 9 }, () => Array(9).fill(null))
  fillGrid(grid)
  return grid
}

function fillGrid(grid: Grid): boolean {
  const empty = findEmpty(grid)
  if (!empty) return true

  const [row, col] = empty
  const nums = shuffle([1, 2, 3, 4, 5, 6, 7, 8, 9]) // randomize order

  for (const num of nums) {
    if (isValid(grid, row, col, num)) {
      grid[row][col] = num
      if (fillGrid(grid)) return true
      grid[row][col] = null
    }
  }
  return false
}
```

Shuffling the number order before trying each position produces a different valid Sudoku solution every time, rather than always getting the same "solved board" pattern.

## Generating a Puzzle by Removing Cells

Take the solved board, remove cells one at a time while ensuring the puzzle still has a unique solution:

```typescript
const DIFFICULTY_CLUES = {
  easy:   45,  // 36 cells removed
  medium: 35,  // 46 removed
  hard:   26,  // 55 removed
}

function generatePuzzle(difficulty: keyof typeof DIFFICULTY_CLUES): {
  puzzle: Grid
  solution: Grid
} {
  const solution = generateSolvedGrid()
  const puzzle = solution.map(row => [...row]) as Grid

  const targetClues = DIFFICULTY_CLUES[difficulty]
  const positions = shuffle(
    Array.from({ length: 81 }, (_, i) => [Math.floor(i / 9), i % 9] as [number, number])
  )

  let cluesLeft = 81

  for (const [r, c] of positions) {
    if (cluesLeft <= targetClues) break

    const backup = puzzle[r][c]
    puzzle[r][c] = null

    // Check if puzzle still has a unique solution
    if (countSolutions(puzzle) === 1) {
      cluesLeft--
    } else {
      puzzle[r][c] = backup // restore — removing this cell breaks uniqueness
    }
  }

  return { puzzle, solution }
}
```

## Counting Solutions for Uniqueness Check

The uniqueness check needs to count solutions but stop at 2 (we only care whether there's exactly 1):

```typescript
function countSolutions(grid: Grid, limit = 2): number {
  const empty = findEmpty(grid)
  if (!empty) return 1 // found a solution

  const [row, col] = empty
  let count = 0

  for (let num = 1; num <= 9; num++) {
    if (isValid(grid, row, col, num)) {
      grid[row][col] = num
      count += countSolutions(grid, limit)
      grid[row][col] = null
      if (count >= limit) return count // early exit
    }
  }
  return count
}
```

Stopping at `limit = 2` keeps this fast. We don't need to find all solutions — just confirm there isn't more than one.

## Cell State

Each cell stores the current value, whether it's a given (locked), and pencil notes:

```typescript
interface CellState {
  value: number | null
  given: boolean          // locked cell from puzzle generation
  notes: Set<number>      // pencil marks 1–9
  isError: boolean        // conflicts with row/col/box
}
```

Notes are stored as a `Set<number>` — fast add/delete/has. When a user enters a value in a cell, auto-clear that number from all notes in the same row, column, and box:

```typescript
function clearRelatedNotes(cells: CellState[][], row: number, col: number, num: number) {
  const boxRow = Math.floor(row / 3) * 3
  const boxCol = Math.floor(col / 3) * 3

  // Same row and column
  for (let i = 0; i < 9; i++) {
    cells[row][i].notes.delete(num)
    cells[i][col].notes.delete(num)
  }

  // Same 3×3 box
  for (let r = boxRow; r < boxRow + 3; r++)
    for (let c = boxCol; c < boxCol + 3; c++)
      cells[r][c].notes.delete(num)
}
```

## Error Detection

Highlight cells that conflict with the rules in real time:

```typescript
function computeErrors(cells: CellState[][]): boolean[][] {
  const errors = Array.from({ length: 9 }, () => Array(9).fill(false))

  for (let r = 0; r < 9; r++) {
    for (let c = 0; c < 9; c++) {
      const val = cells[r][c].value
      if (!val) continue

      // Check for conflict with any peer
      const peers = getPeers(r, c)
      for (const [pr, pc] of peers) {
        if (cells[pr][pc].value === val) {
          errors[r][c] = true
          break
        }
      }
    }
  }
  return errors
}

function getPeers(row: number, col: number): [number, number][] {
  const peers: [number, number][] = []
  const boxRow = Math.floor(row / 3) * 3
  const boxCol = Math.floor(col / 3) * 3

  for (let i = 0; i < 9; i++) {
    if (i !== col) peers.push([row, i])
    if (i !== row) peers.push([i, col])
  }
  for (let r = boxRow; r < boxRow + 3; r++)
    for (let c = boxCol; c < boxCol + 3; c++)
      if (r !== row || c !== col) peers.push([r, c])

  return peers
}
```

## Hint System

A hint reveals one incorrect or empty cell from the solution:

```typescript
function applyHint(cells: CellState[][], solution: Grid): CellState[][] | null {
  // Find cells that are wrong or empty
  const candidates: [number, number][] = []

  for (let r = 0; r < 9; r++) {
    for (let c = 0; c < 9; c++) {
      if (cells[r][c].given) continue
      if (cells[r][c].value !== solution[r][c]) {
        candidates.push([r, c])
      }
    }
  }

  if (candidates.length === 0) return null // already solved

  // Pick a random candidate
  const [r, c] = candidates[Math.floor(Math.random() * candidates.length)]

  const newCells = cells.map(row => row.map(cell => ({ ...cell, notes: new Set(cell.notes) })))
  newCells[r][c].value = solution[r][c]
  newCells[r][c].notes.clear()

  // Clear related notes for the revealed value
  clearRelatedNotes(newCells, r, c, solution[r][c]!)

  return newCells
}
```

## Cell Highlight: Row, Column, Box, and Same Number

When a cell is selected, highlight all related cells for visual guidance:

```typescript
function getHighlightClass(
  row: number,
  col: number,
  selectedRow: number | null,
  selectedCol: number | null,
  selectedValue: number | null,
  cells: CellState[][]
): string {
  if (selectedRow === null || selectedCol === null) return ''
  if (row === selectedRow && col === selectedCol) return 'selected'

  const sameValue = selectedValue && cells[row][col].value === selectedValue
  if (sameValue) return 'same-value'

  const sameRow = row === selectedRow
  const sameCol = col === selectedCol
  const sameBox =
    Math.floor(row / 3) === Math.floor(selectedRow / 3) &&
    Math.floor(col / 3) === Math.floor(selectedCol / 3)

  if (sameRow || sameCol || sameBox) return 'peer'

  return ''
}
```

Four highlight classes: `selected` (the active cell), `same-value` (all cells with the same number), `peer` (same row/col/box), and no class for unrelated cells.

## Win Detection

```typescript
function isSolved(cells: CellState[][]): boolean {
  return cells.every(row =>
    row.every(cell => cell.value !== null && !cell.isError)
  )
}
```

Check after every input. When the board is full and error-free, the game is won.

---

The most computationally expensive part is `generatePuzzle()` — the uniqueness check runs the solver repeatedly. On a fast machine this takes ~50–200ms. To avoid blocking the UI, run it in a `useEffect` or a Web Worker. For [our implementation](https://ultimatetools.io/tools/fun-tools/sudoku/), the generation runs asynchronously on page load.

---

Play it: [Sudoku → ultimatetools.io](https://ultimatetools.io/tools/fun-tools/sudoku/)

*Part of [Ultimate Tools](https://ultimatetools.io/) — free browser-based tools and games.*