# Play Mindi Coat Online — the Card Game Where Trump Stays Hidden (Free, with Bots)

Depending on where you grew up you call it Mindi, Mindi Coat, Mendikot, or Dehla Pakad — "catch the ten." Same game: two partnerships, one deck, and only four cards that matter. It's everywhere in Maharashtra and Gujarat, it has millions of app installs, and it barely exists on the open web — so we built [Mindi Coat online](https://ultimatetools.io/tools/fun-tools/mindi-coat/): free, in the browser, 4 or 6 players via a share link, **with computer players so even one person can play**. No app, no signup.

Two things make it worth a look even if you've never heard of it.

## The tens are prizes, not weapons

Most trick-taking games reward high cards. This one rewards four *specific* cards. Capture three of the four tens and you win the hand — but a ten ranks exactly where it normally does, **under the jack**, so it loses almost every trick it's played into.

That single inversion changes the whole game. A ten in your hand is a liability you have to smuggle out. You don't lead it and you don't cover with it; you wait until your partner is already winning a trick and *then* quietly drop it on. Meanwhile every trick that has a ten sitting in it becomes a small war.

Sixty-second rules:

- **4 players (2v2) or 6 (3v3)**, partners alternating around the table.
- 4 players get 12 cards each, 6 get 8 — plus a **4-card kitty held back face down**. Both land on 48 + 4 = 52, so nothing is stripped.
- Ordinary ace-high ranking. Follow the led suit if you can; if you can't, play anything.
- **3+ tens wins the hand.** Tens 2-2? The team with more tricks takes it. All four tens is a **coat** — the result the game is named after.

## Nobody knows what trumps yet

Here's the mechanic I hadn't seen in a web card game: **play starts with no trump at all.** The first few tricks are pure follow-suit, highest card of the led suit wins, and nothing beats anything else.

Then someone runs out of the led suit. The top card of that face-down kitty is turned over, and its suit is trump for the rest of the hand — *effective immediately*, so it decides the trick already on the table. A hand you'd planned around one shape suddenly reorganises around a suit nobody could have prepared for.

It also creates a genuinely interesting read: the player about to run out of a suit is the player about to turn trump. Watch who's short and you can guess when the hand is going to change.

### One deal-time detail that turned out to matter

With four tens and a four-card kitty, a random deal buries one of the tens about 28% of the time — and a buried ten quietly breaks the win condition, because "3 of 4" stops being reachable and a coat becomes impossible.

So the deal guarantees a ten-free kitty. Not by swapping a ten out (which skews the distribution), but by **rejection sampling** — reshuffle until the kitty is clean:

```js
export function dealFor(playerCount, shuffle) {
    const handSize = handSizeFor(playerCount);
    for (let attempt = 0; attempt < 500; attempt++) {
        const deck = shuffle(buildDeck());
        const kitty = deck.slice(DEALT_TOTAL);      // the last 4
        if (kitty.some(isTen)) continue;            // all four tens must reach hands
        ...
    }
}
```

A random kitty is clean ~72% of the time — `C(48,4)/C(52,4)` — so this reshuffles about 1.4 times on average and stays perfectly uniform over all valid deals. Cheaper than being clever, and much easier to reason about. A test runs 800 deals across both table sizes and asserts no ten ever lands there.

## Hidden trump means hiding it properly

The kitty is the one piece of state that absolutely cannot leak. Three of its four cards stay face down all hand, and if a client could see them it could work out trump before the reveal — which would quietly ruin the entire game.

So the per-player view sends a **count and nothing else**:

```js
kittyCount: room.kitty.length,     // a count leaks nothing
trump: room.trump,                 // null while hidden
revealedTrumpCard: room.revealedTrumpCard,   // public only once turned
```

Everything else follows the same rule the engine already used for hands: each player gets a private token at join, and the server renders a per-viewer projection — your cards, everyone else's card *counts*, the public trick. There's an explicit test asserting one player's view never contains another player's cards, and that the string `"kitty"` never appears in a response.

The other subtle one: **a revoke must not be able to turn trump early.** Throwing off-suit while you still hold the led suit is illegal, and it has to be rejected *before* the reveal fires — otherwise a player could force trump over whenever it suited them. The check asks whether the hand is genuinely void, not just whether the card matches:

```js
export function revealsTrump(hand, trick, card) {
    if (trick.length === 0) return false;              // the leader sets the suit
    const led = suitOf(trick[0]);
    if (suitOf(card) === led) return false;            // followed suit
    return !hand.some(c => suitOf(c) === led);         // void, not a revoke
}
```

## Bots that understand what the game is about

The bots run **synchronously inside the same locked mutation** as the human move that exposed them — no timers, no scheduler, which matters because this runs on plain shared hosting with no WebSockets and no job queue. Rooms are JSON files, clients poll, moves serialise through an advisory lockfile.

One heuristic does most of the work of making them look like they know the game: **fight for tricks that contain a ten, and bank your own tens onto a trick your partner is already winning.** That's the actual craft of Mindi Coat, and it's about six lines.

Fairness is structural rather than promised: a bot's decision function receives only its own hand plus the public table. It never sees another hand, and it never sees the kitty — so a bot cannot know what trump will be either.

Stated honestly on the page: they're casual level. Good for learning the game or filling a seat, not for out-thinking a family that's played this for twenty years.

## What it doesn't do

- No accounts. Tables live 6 hours, no saved history, no rankings.
- Polling every second or two, not a live socket — fine for turn-based, and it's what keeps this free.
- **Exactly 4 or 6 players.** Partners alternate, so five seats would be three against two.
- One documented variant. Some tables name trump before play instead of hiding it; the page has a rules table so the referee's behaviour is never a surprise.

## Related Tools

- [Play Kaali Teeri — the 3 of Spades game with a secret partner](https://ultimatetools.io/tools/fun-tools/kaali-teeri/) — the other Indian partnership game on these tables, where the bidder calls a card and whoever holds it becomes their hidden teammate.
- [Play Crazy Eights online with friends](https://ultimatetools.io/tools/fun-tools/crazy-eights/) — the UNO-style shedding game on the same share-link rooms, 2–6 players.
- [Host a multiplayer Bingo night with shared rooms](https://ultimatetools.io/tools/fun-tools/bingo/) — every phone gets a unique card, up to a whole classroom.

Deal the first hand — free, no app, and the bots are always awake: [Mindi Coat Online](https://ultimatetools.io/tools/fun-tools/mindi-coat/)