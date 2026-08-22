# Play Kaali Teeri Online — the Punjabi Card Game with a Secret Partner (Free, with Bots)

If you grew up around a Punjabi card table, you know Kaali Teeri — the "black three." If you didn't, here's the pitch: it's a trick-taking game where the winning bidder **calls a card, and whoever holds it becomes their secret teammate**. Nobody knows who's allied with whom until that card hits the table, and by then half the game may already be decided. It's the most social mechanic in card games, and it barely exists on the web — so we built [Kaali Teeri online](https://ultimatetools.io/tools/fun-tools/kaali-teeri/): free, in the browser, 4–6 players via a share link, **with computer players so even one person can play**. No app, no signup.

## Sixty-second rules

- Standard deck; **exactly 250 points**: every A/K/Q/J/10 is worth 10, every 5 is worth 5, and the **3♠ is worth 30 — and is permanently the highest trump**, whatever suit gets named. The game is literally named after it.
- Players **bid 150–250** for the right to name the trump suit (*rung*) and call the partner card. A pass is final. A max 250 bid ends the auction on the spot and calls *two* partner cards.
- Tricks work like Hearts or Spades: follow the led suit if you can, highest trump wins, winner leads next.
- The bidding team must capture at least the bid in points. Make it and every team member scores the bid; fail and the defenders do.

The masterstroke: the bidder can call a card **from their own hand** — and secretly play alone while everyone else "helps" an ally who doesn't exist.

## The build: bots without a scheduler

The interesting engineering constraint: this runs on plain shared hosting — no WebSockets, no job queues, no database. Rooms are JSON files; clients poll; moves serialize through an advisory lockfile.

So how do computer players take their turns with **no server-side timers at all**? Synchronously. Every human action — a bid, a card, a skip — runs inside a locked mutation, and at the tail of that mutation the server simply plays every consecutive bot turn until it's a human's move again (or the round ends). One atomic write covers the human's move plus the whole bot cascade. A full 4-player round with three bots resolves in about 14 human interactions, and the bots cost zero infrastructure.

Two design rules made this trustworthy:

- **Fairness is enforced structurally.** Bot decision functions receive only what a human in that seat would see: their own hand plus the public table. The secret team assignments and other players' hands are never passed in — a bot learns it's the bidder's partner the same way you would, by holding the called card.
- **Hidden state stays server-side.** Each player gets a private token at join; the server renders a per-player view (your hand, everyone else's card *counts*, the public trick). Team membership is only ever revealed the way the real game reveals it: when the called card is played.

And honestly stated on the page: the bots are casual level — good for learning the game and filling the fourth seat, not for out-thinking your grandmother.

## Handling real people

Turn-based multiplayer with strangers to nobody — it's family — still needs guardrails: a stuck turn can be skipped after 45–90 seconds (auto-pass in the auction, cheapest legal card in play), and anyone who leaves mid-round is **replaced by a bot who finishes their hand** — so the round always completes, the 250 points always add up, and the partner reveal always happens. A running scoreboard keeps totals across rematches at the same table.

## What it doesn't do

- No accounts: tables live 6 hours, no saved history, no rankings.
- One documented variant (rules table on the page) — if your family plays double-deck with 8 players, that's a future version.
- No matchmaking — you bring your own players via the link, or play the bots.

## Related Tools

- [Play Crazy Eights online with friends](https://ultimatetools.io/tools/fun-tools/crazy-eights/) — the UNO-style shedding game on the same share-link tables, 2–6 players.
- [Host a multiplayer Bingo night with shared rooms](https://ultimatetools.io/tools/fun-tools/bingo/) — every phone gets a unique card; up to a whole classroom.
- [Race friends in a live typing speed test](https://ultimatetools.io/tools/fun-tools/typing-speed-test/) — same rooms, different adrenaline.

Deal the first hand — free, no app, and the bots are always awake: [Kaali Teeri Online](https://ultimatetools.io/tools/fun-tools/kaali-teeri/)