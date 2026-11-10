# Cat Queens — a Free Meowdoku-Style Cat Logic Puzzle in the Browser (Daily Grid + 150 Levels, No App)

Cat Queens is a free cat logic puzzle you play in the browser: put one cat in every row, every column and every colour region of the grid, and never let two cats touch — not even at the corners. If you have seen Meowdoku-style grids going around this year, or LinkedIn's Queens before that, you already know the family; this is our take on it, built as a page rather than an app, and it is live at [Cat Queens](https://ultimatetools.io/tools/fun-tools/cat-queens/) with a shared daily grid, a 150-level campaign and unlimited free play.

This is the developer's version of the launch: the rules, one small grid solved by logic alone, and the part worth building — a generator that must promise a unique, logically solvable puzzle for every seed while running in the visitor's browser, because the site lives on shared hosting with no room for server-side generation.

## Four rules, one of which does most of the work

The grid is N×N and carved into N colour regions. You place exactly N cats so that:

1. every row has one cat,
2. every column has one cat,
3. every colour region has one cat,
4. no two cats touch — including diagonally.

Rules 1–3 are Sudoku logic: each row, column and region is a "house" that needs exactly one of something, and most of the play is elimination (our [Sudoku](https://ultimatetools.io/tools/fun-tools/sudoku/) runs on the same idea with numbers). Rule 4 makes the puzzle its own thing: a cat also kills the eight cells around it, including cells in *other* regions that nothing else would have touched. Reading the board as "which cell is attacked by every remaining candidate of this house" is where the game opens up.

## A 5×5 solved by logic alone

A 5×5 in the game's format, letters standing in for the colours:

```
A A B B B
C A B B B
C C B E B
C D D E E
C D E E E
```

Rows and columns are numbered 1–5 from the top left; (r, c) is row r, column c. Region A is (1,1), (1,2), (2,2); region D is (4,2), (4,3), (5,2); region C runs down the left side: (2,1), (3,1), (3,2), (4,1), (5,1). We will not guess once.

**1. Two regions, two columns.** Every cell of A and every cell of C sits in column 1 or column 2. Two regions that must supply two cats, both confined to the same two columns — so the cats of columns 1 and 2 *are* A's and C's, and no other region may put a cat there. Region D loses (4,2) and (5,2): its cat goes at (4,3).

**2. The fallout.** That cat wipes out the rest of row 4, the rest of column 3 and its eight neighbours. In region E, (4,4) and (4,5) are in row 4, (5,3) is in column 3, and (3,4) and (5,4) are neighbours of (4,3). Only (5,5) survives: E's cat.

**3. Row 3.** (3,5) is in column 5, (3,3) is in column 3, and (3,2) and (3,4) touch the cat at (4,3). That leaves (3,1). Cat.

**4. Row 2.** (2,1) shares column 1 with the new cat, (2,2) touches it diagonally, and (2,3) and (2,5) are in used columns. That leaves (2,4). Cat.

**5. Row 1.** Columns 1, 3, 4 and 5 are taken, so (1,2) is the last cell standing — in region A, exactly as step 1 predicted. Five cats, five deductions, zero guesses.

Step 1 — two regions locked inside two lines — is one of the deductions the generator looks for when it grades a puzzle.

## How a cat logic puzzle gets generated in your browser

There is no puzzle server: every daily and free-play grid is produced by TypeScript in the page, and the engine is pure — no `Date.now()`, no `Math.random()`, no React — so a seed string goes in and the same puzzle comes out on every device.

**Construction, not search.** Each attempt first picks the *answer*: a permutation of columns in which consecutive rows differ by at least two, so no two cats can touch. It then grows the colour regions outward from those cats, one region per cat, so every region is connected and holds exactly one solution cat.

**Uniqueness is built in, not repaired afterwards.** An unassigned cell can hold no cat, so assigning cells to regions can only *add* solutions, never remove one. The generator leans on that: regions grow outward one frontier cell at a time, and a cell is claimed only if a row-by-row bitmask search (`solutionThrough`) finds no alternative solution running through it. Grow-test-discard had collapsed as grids grew — in our prototype essentially no 8×8 attempt came out unique — but refusing exactly the cells that would open a second solution keeps every layout unique at every step. When every frontier cell would create a second solution, the growth kicks one cell of that alternative solution back out of its region and carries on; an attempt is abandoned after 3 × N kicks. A final `countSolutions(…, 2)` on the finished layout is a cheap safety net, not the mechanism.

**Difficulty is graded by a human-rule solver.** Unique is not the same as fair — a unique puzzle can still demand a lucky guess. So every candidate is solved a second time by `solveLogically`, which only knows moves a person could make, cheapest first:

```ts
export type RuleId =
    | "single"
    | "confine"
    | "subset2"
    | "subset3"
    | "attack2"
    | "attack3"
    | "attack4"
    | "bifurcate";
```

`single` is a house with one candidate left; `confine` is a region whose candidates all sit in one line (so the rest of that line is cleared), and the reverse; `subset2`/`subset3` is the move from the worked example — k houses locked inside k houses of another kind; the `attack` rules fire when every remaining candidate of a house attacks the same cell. Each rule has a cost, the score is the sum, and the tier follows from the most expensive rule needed and from the total score — a long chain of cheap steps is still graded Medium or Hard. Anything that needed `bifurcate` — a one-step "what if" — is Expert: the daily and free play never target it, and the campaign saves it for the final levels of the last two bands.

**Acceptance is counted, never timed.** The generator spends a fixed number of attempts on an exact tier match, then widens to a neighbouring tier, then to any tier without guessing, then to any unique puzzle a single "what if" can crack, and finally to a provably unique fallback. The caps are attempt counts, not milliseconds — that is what keeps the output identical on a phone and a desktop. Timing enters in exactly one place:

```ts
export interface GenerationJob {
    run(maxMs?: number): Puzzle | null;
    readonly attempts: number;
}
```

`run()` executes whole attempts until its budget is spent and returns `null` while work remains. The page calls it in short slices from a `setTimeout(0)` loop behind a square placeholder, so a 10×10 never freezes the tab, without a Web Worker. The budget decides *where the loop pauses*, never which puzzle you get.

**The seed is the contract.** Seeds look like `cq1|daily|2026-11-09`, `cq1|level|<size>|<n>` or `cq1|free|<size>|<token>`. The `cq1` prefix is the generator version; any change that could alter output for the same seed bumps it, and a saved daily carries the version, so a half-finished board is never resumed on top of a different grid. The 150 campaign levels skip the browser generator entirely: a script generated them once, curated them so the score rises through each band, and committed them as a small table that `getLevel(n)` decodes and validates.

## Daily grid, 150 levels, or just play

**Daily.** One grid per local date, identical for everyone; the size follows the weekday — 7×7 Monday and Tuesday, 8×8 Wednesday and Thursday, 9×9 Friday and Saturday, 8×8 Sunday. It resets at local midnight, the streak counts consecutive days and a missed day shows 0; a win gives you share text and an image card. It is the same habit as a [Wordle](https://ultimatetools.io/tools/fun-tools/wordle/) streak, with cats.

**Levels.** Six bands step up the grid: Kitten 1–8 (5×5), House Cat 9–28 (6×6), Alley Cat 29–63 (7×7), Tomcat 64–103 (8×8), Wildcat 104–133 (9×9) and Queen 134–150 (10×10). Levels unlock in order, solved levels can be replayed, and your best time per level is kept.

**Free play.** Any size from 5×5 to 10×10, as many grids as you like; the target tier follows the size — 5–6 Easy, 7–8 Medium, 9–10 Hard — and the solver's actual grade is shown once the grid is up.

## What the controls do — and what they won't do for you

- **Tap cycles** empty → X → cat; right-click toggles an X. Keyboard: arrows, Space or Enter to cycle, X for a mark, C or Q for a cat, Backspace to clear, U undo, H hint, N new puzzle, R reset.
- **Auto-X is on by default.** Placing a cat marks the rest of its row, column and colour region plus its eight neighbours — the consequences of cats you placed, never deductions you have not made. Switch it off for the purist experience.
- **Conflicts highlight live** — cats sharing a line or a region, or touching — with no lives and no lockout. A legal-but-wrong cat is *not* flagged; that is what the hint is for.
- **Hints are counted**, on the button and in the win panel. A hint first points at a wrong cat (or an X on a cat's cell); otherwise it shows the next forced deduction; it reveals a cat only when logic is exhausted.
- **Undo** steps back as far as you like. **The timer** starts on your first move, pauses when the tab is hidden and stops at the last cat.

## Honest limits

- Progress — settings, daily streak, level times — lives in this browser's localStorage only. No account, no sync between devices, no leaderboard, no multiplayer.
- Expert levels may need a one-step "what if". Everything else is reachable by pure logic, and the grader says so.
- Generation happens on your device, so a 10×10 takes a visible moment on a phone; that is why the daily tops out at 9×9.

Coming from a Meowdoku-style app or from LinkedIn's Queens? Same puzzle family, different packaging. Cat Queens is an independent game by Ultimate Tools. It is not affiliated with, endorsed by or connected to Oakever Games (makers of Meowdoku) or LinkedIn (makers of Queens); those names belong to their respective owners and appear here only to describe the puzzle family.

## Related Tools

- [play a daily Sudoku online with a built-in solver](https://ultimatetools.io/tools/fun-tools/sudoku/) — the same one-per-house elimination with numbers instead of cats, plus a solver for checking a grid.
- [clear a Minesweeper board by deduction, 9×9 to 20×20](https://ultimatetools.io/tools/fun-tools/minesweeper/) — the other browser puzzle here where deduction beats clicking and hoping; the first click is always safe.
- [play Wordle unlimited with 4 to 7 letter words](https://ultimatetools.io/tools/fun-tools/wordle/) — a second daily streak to keep next to the cats, with an unlimited mode for practice.

Place the first cat — free, in the browser, no app to install: [Cat Queens](https://ultimatetools.io/tools/fun-tools/cat-queens/)