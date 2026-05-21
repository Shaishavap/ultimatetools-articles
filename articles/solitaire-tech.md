# Building Klondike Solitaire in React — Card State, Move Validation, and the Fisher-Yates Shuffle

Solitaire is one of those games that looks simple until you try to implement it. The rules are well-known, but representing the full game state, validating every possible move, and keeping UI in sync with that state all have non-obvious pieces.

Here's how the [Solitaire](https://ultimatetools.io/tools/fun-tools/solitaire/) game at Ultimate Tools is built.

## Card Representation

Every card needs a suit, rank, and whether it's face-up:

```typescript
type Suit = 'spades' | 'hearts' | 'diamonds' | 'clubs';
type Rank = 1 | 2 | 3 | 4 | 5 | 6 | 7 | 8 | 9 | 10 | 11 | 12 | 13;

type Card = {
  suit: Suit;
  rank: Rank;   // 1 = Ace, 11 = Jack, 12 = Queen, 13 = King
  faceUp: boolean;
};
```

Color is derived from suit rather than stored:

```typescript
const isRed = (suit: Suit): boolean =>
  suit === 'hearts' || suit === 'diamonds';
```

## Game State Shape

The full game state for Klondike Solitaire:

```typescript
type GameState = {
  tableau: Card[][];    // 7 columns
  foundations: Card[][]; // 4 piles, one per suit
  stock: Card[];         // face-down draw pile
  waste: Card[];         // face-up played cards from stock
};
```

The `tableau` is 7 arrays. Each column array is ordered bottom-to-top — index 0 is the base card, the last element is the top (playable) card.

## Dealing: Fisher-Yates Shuffle

Dealing starts with a shuffled 52-card deck. The standard Fisher-Yates algorithm:

```typescript
const createDeck = (): Card[] => {
  const suits: Suit[] = ['spades', 'hearts', 'diamonds', 'clubs'];
  const deck: Card[] = [];

  for (const suit of suits) {
    for (let rank = 1; rank <= 13; rank++) {
      deck.push({ suit, rank: rank as Rank, faceUp: false });
    }
  }

  return shuffle(deck);
};

const shuffle = <T>(array: T[]): T[] => {
  const arr = [...array];
  for (let i = arr.length - 1; i > 0; i--) {
    const j = Math.floor(Math.random() * (i + 1));
    [arr[i], arr[j]] = [arr[j], arr[i]];
  }
  return arr;
};
```

Fisher-Yates is O(n) and produces a uniformly random permutation. The naive approach — `array.sort(() => Math.random() - 0.5)` — is biased and slower due to comparison overhead.

## Initial Deal

After shuffling, deal into the 7 tableau columns:

```typescript
const deal = (deck: Card[]): GameState => {
  const tableau: Card[][] = Array.from({ length: 7 }, () => []);
  let deckIndex = 0;

  for (let col = 0; col < 7; col++) {
    for (let row = 0; row <= col; row++) {
      const card = { ...deck[deckIndex++] };
      card.faceUp = row === col; // only the top card starts face-up
      tableau[col].push(card);
    }
  }

  return {
    tableau,
    foundations: [[], [], [], []],
    stock: deck.slice(deckIndex).map(c => ({ ...c, faceUp: false })),
    waste: [],
  };
};
```

Column 0 gets 1 card, column 1 gets 2, ..., column 6 gets 7. Only the top card in each column is face-up.

## Move Validation

### Tableau-to-Tableau

A card (or stack) can move onto a tableau column if:
1. The destination column is empty → only a King can move there
2. The destination top card is face-up, one rank higher, and opposite color

```typescript
const canMoveToTableau = (card: Card, targetColumn: Card[]): boolean => {
  if (targetColumn.length === 0) {
    return card.rank === 13; // King only
  }

  const target = targetColumn[targetColumn.length - 1];
  return (
    target.faceUp &&
    card.rank === target.rank - 1 &&
    isRed(card.suit) !== isRed(target.suit) // opposite color
  );
};
```

### Tableau-to-Foundation

A card moves to a foundation pile if:
1. The foundation is empty → only an Ace
2. The foundation top card is the same suit and one rank lower

```typescript
const canMoveToFoundation = (card: Card, foundation: Card[]): boolean => {
  if (foundation.length === 0) {
    return card.rank === 1; // Ace only
  }

  const top = foundation[foundation.length - 1];
  return card.suit === top.suit && card.rank === top.rank + 1;
};
```

## Moving a Stack

In Klondike, you can move a sequence of face-up cards as a group. When the user clicks a face-up card that isn't the top card, they're selecting the entire sub-stack from that card to the top:

```typescript
const getMovableStack = (column: Card[], fromIndex: number): Card[] => {
  return column.slice(fromIndex);
};

const handleTableauClick = (colIndex: number, cardIndex: number) => {
  const column = state.tableau[colIndex];
  const card = column[cardIndex];

  if (!card.faceUp) return; // can't select face-down cards

  const stack = getMovableStack(column, cardIndex);
  setSelectedStack({ colIndex, cardIndex, cards: stack });
};
```

When the user then clicks a destination column, validate using the bottom card of the stack (the card that will sit on the destination):

```typescript
const canPlaceStack = (stack: Card[], targetColumn: Card[]): boolean =>
  canMoveToTableau(stack[0], targetColumn);
```

## Flipping the Top Card After a Move

After removing cards from a tableau column, the new top card (if face-down) should flip:

```typescript
const flipTopCard = (column: Card[]): Card[] => {
  if (column.length === 0) return column;
  const top = column[column.length - 1];
  if (top.faceUp) return column;

  return [
    ...column.slice(0, -1),
    { ...top, faceUp: true },
  ];
};
```

This returns a new array (immutably) with the top card flipped — compatible with React state updates.

## Stock and Waste

Drawing from the stock moves the top card to the waste pile:

```typescript
const drawFromStock = (state: GameState): GameState => {
  if (state.stock.length === 0) {
    // Recycle waste back into stock (face-down, reversed)
    return {
      ...state,
      stock: [...state.waste].reverse().map(c => ({ ...c, faceUp: false })),
      waste: [],
    };
  }

  const [top, ...remainingStock] = state.stock;
  return {
    ...state,
    stock: remainingStock,
    waste: [...state.waste, { ...top, faceUp: true }],
  };
};
```

When the stock is empty, clicking it resets the waste pile back to the stock (face-down). Standard Klondike allows unlimited recycles.

## Win Condition

The game is won when all four foundations are complete (13 cards each):

```typescript
const isWon = (state: GameState): boolean =>
  state.foundations.every(f => f.length === 13);
```

## Key Gotchas

| Problem | Solution |
|---|---|
| Stack moves need bottom-card validation | Validate `stack[0]`, not the clicked card |
| Tableau column is empty | Only Kings allowed; check `column.length === 0` separately |
| Stock exhausted | Recycle waste in reverse order, face-down |
| Top card doesn't flip after move | `flipTopCard()` after every tableau removal |
| `sort(() => Math.random() - 0.5)` shuffle | Use Fisher-Yates — `sort`-based shuffle is biased |

The full game is live at the [Solitaire](https://ultimatetools.io/tools/fun-tools/solitaire/) tool — try to beat it on the first draw.