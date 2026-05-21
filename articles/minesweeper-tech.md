# Building Minesweeper in React — BFS Flood Fill, Safe First Click, and Responsive Scale-to-Fit

Minesweeper looks trivial to implement until you actually start. Then you run into: guaranteed safe first click, the flood-fill cascade that reveals large empty regions, and the scaling problem — the board needs to fit a phone screen without the cells getting unusably tiny.

Here's how I built [Minesweeper](https://ultimatetools.io/tools/fun-tools/minesweeper/) in React with all three solved properly.

## Board Representation

Each cell is a plain object:

```typescript
interface Cell {
  isMine: boolean
  isRevealed: boolean
  isFlagged: boolean
  adjacentMines: number  // 0–8
}

type Board = Cell[][]
```

The board is a 2D array indexed as `board[row][col]`. For an Easy (9×9, 10 mines) game:

```typescript
function createEmptyBoard(rows: number, cols: number): Board {
  return Array.from({ length: rows }, () =>
    Array.from({ length: cols }, () => ({
      isMine: false,
      isRevealed: false,
      isFlagged: false,
      adjacentMines: 0,
    }))
  )
}
```

## Safe First Click — Placing Mines After the First Move

The classic Minesweeper rule: the first cell you click is always safe, and ideally reveals a large empty region. Implement this by placing mines *after* the first click, excluding the clicked cell and its neighbors.

```typescript
function placeMines(
  board: Board,
  mineCount: number,
  safeRow: number,
  safeCol: number
): Board {
  const rows = board.length
  const cols = board[0].length

  // Cells excluded from mine placement (clicked cell + 8 neighbors)
  const excluded = new Set<string>()
  for (let dr = -1; dr <= 1; dr++) {
    for (let dc = -1; dc <= 1; dc++) {
      excluded.add(`${safeRow + dr},${safeCol + dc}`)
    }
  }

  // Collect eligible positions
  const eligible: [number, number][] = []
  for (let r = 0; r < rows; r++) {
    for (let c = 0; c < cols; c++) {
      if (!excluded.has(`${r},${c}`)) eligible.push([r, c])
    }
  }

  // Fisher-Yates shuffle, take first mineCount
  for (let i = eligible.length - 1; i > 0; i--) {
    const j = Math.floor(Math.random() * (i + 1))
    ;[eligible[i], eligible[j]] = [eligible[j], eligible[i]]
  }

  const newBoard = board.map(row => row.map(cell => ({ ...cell })))
  eligible.slice(0, mineCount).forEach(([r, c]) => {
    newBoard[r][c].isMine = true
  })

  return computeAdjacentCounts(newBoard)
}
```

## Computing Adjacent Mine Counts

After mines are placed, compute the `adjacentMines` value for every non-mine cell:

```typescript
function computeAdjacentCounts(board: Board): Board {
  const rows = board.length
  const cols = board[0].length

  for (let r = 0; r < rows; r++) {
    for (let c = 0; c < cols; c++) {
      if (board[r][c].isMine) continue
      let count = 0
      for (let dr = -1; dr <= 1; dr++) {
        for (let dc = -1; dc <= 1; dc++) {
          if (dr === 0 && dc === 0) continue
          const nr = r + dr, nc = c + dc
          if (nr >= 0 && nr < rows && nc >= 0 && nc < cols) {
            if (board[nr][nc].isMine) count++
          }
        }
      }
      board[r][c].adjacentMines = count
    }
  }
  return board
}
```

## BFS Flood Fill Cascade

When you click a cell with `adjacentMines === 0`, the standard Minesweeper behavior is to auto-reveal all connected zero-cells and their numbered neighbors. This is a BFS traversal:

```typescript
function revealFrom(board: Board, startRow: number, startCol: number): Board {
  const rows = board.length
  const cols = board[0].length
  const newBoard = board.map(row => row.map(cell => ({ ...cell })))

  const queue: [number, number][] = [[startRow, startCol]]
  const visited = new Set<string>([`${startRow},${startCol}`])

  while (queue.length > 0) {
    const [r, c] = queue.shift()!
    const cell = newBoard[r][c]

    if (cell.isFlagged || cell.isMine) continue

    cell.isRevealed = true

    // Only continue BFS from zero-cells
    if (cell.adjacentMines === 0) {
      for (let dr = -1; dr <= 1; dr++) {
        for (let dc = -1; dc <= 1; dc++) {
          const nr = r + dr, nc = c + dc
          const key = `${nr},${nc}`
          if (
            nr >= 0 && nr < rows &&
            nc >= 0 && nc < cols &&
            !visited.has(key) &&
            !newBoard[nr][nc].isRevealed
          ) {
            visited.add(key)
            queue.push([nr, nc])
          }
        }
      }
    }
  }

  return newBoard
}
```

BFS (not DFS) matters here for two reasons: it reveals cells in "waves" outward from the click, which matches the visual expectation, and it avoids stack overflow on large boards that DFS recursion would risk.

## Win and Loss Detection

**Loss**: clicked a mine → reveal all mines, game over.

**Win**: every non-mine cell is revealed.

```typescript
function checkWin(board: Board): boolean {
  return board.every(row =>
    row.every(cell => cell.isMine || cell.isRevealed)
  )
}
```

Check win after every reveal. When won, auto-flag all remaining mines:

```typescript
function flagAllMines(board: Board): Board {
  return board.map(row =>
    row.map(cell => cell.isMine ? { ...cell, isFlagged: true } : cell)
  )
}
```

## Responsive Scale-to-Fit with ResizeObserver

The board has fixed pixel-per-cell dimensions (e.g., 32px per cell for Expert = 30×16 = 960px wide). On a 375px mobile screen, that doesn't fit.

The solution: measure the container width and apply a CSS `transform: scale()` to shrink the board to fit, without changing cell sizes or reflowing.

```typescript
const containerRef = useRef<HTMLDivElement>(null)
const boardRef = useRef<HTMLDivElement>(null)
const [scale, setScale] = useState(1)

const CELL_SIZE = 32

useEffect(() => {
  const observer = new ResizeObserver(entries => {
    const containerW = entries[0].contentRect.width
    const boardW = cols * CELL_SIZE
    const newScale = Math.min(1, containerW / boardW)
    setScale(newScale)
  })

  if (containerRef.current) observer.observe(containerRef.current)
  return () => observer.disconnect()
}, [cols])
```

Apply it in JSX:

```tsx
<div ref={containerRef} style={{ width: '100%' }}>
  <div
    ref={boardRef}
    style={{
      transform: `scale(${scale})`,
      transformOrigin: 'top center',
      width: cols * CELL_SIZE,
      // Compensate height so layout doesn't collapse
      marginBottom: (scale - 1) * rows * CELL_SIZE,
    }}
  >
    {/* board cells */}
  </div>
</div>
```

The `marginBottom` compensation is important. When `scale < 1`, the element's layout box stays at full size but the visual box is smaller — leaving a gap below. The negative margin (since `scale - 1` is negative) pulls subsequent elements up to close the gap.

## Number Colors

The classic Minesweeper color scheme for adjacent counts:

```typescript
const numberColors: Record<number, string> = {
  1: '#2563eb', // blue
  2: '#16a34a', // green
  3: '#dc2626', // red
  4: '#1e3a8a', // dark blue
  5: '#991b1b', // dark red
  6: '#0891b2', // cyan
  7: '#000000', // black
  8: '#6b7280', // gray
}
```

These match the original Windows Minesweeper colors players are familiar with.

## Right-Click Flag on Desktop, Long-Press on Mobile

```tsx
function handleCellInteraction(
  e: React.MouseEvent | React.TouchEvent,
  row: number,
  col: number
) {
  if ('button' in e && e.button === 2) {
    // Right-click = flag
    e.preventDefault()
    toggleFlag(row, col)
  }
}

// Long press for mobile
let longPressTimer: ReturnType<typeof setTimeout>

function handleTouchStart(row: number, col: number) {
  longPressTimer = setTimeout(() => toggleFlag(row, col), 500)
}

function handleTouchEnd() {
  clearTimeout(longPressTimer)
}
```

On mobile, a 500ms touch hold flags/unflags the cell. Short tap reveals it. This matches how Minesweeper typically works on touch devices.

## Difficulty Config

```typescript
const DIFFICULTIES = {
  easy:   { rows: 9,  cols: 9,  mines: 10 },
  medium: { rows: 16, cols: 16, mines: 40 },
  hard:   { rows: 16, cols: 30, mines: 99 },
} as const
```

The Expert grid (16×30) is 960px wide at 32px/cell — wider than most phones. That's exactly where the `ResizeObserver` scale kicks in and makes it playable on any screen.

---

Play it: [Minesweeper → ultimatetools.io](https://ultimatetools.io/tools/fun-tools/minesweeper/)

*Part of [Ultimate Tools](https://ultimatetools.io/) — free browser-based tools and games.*