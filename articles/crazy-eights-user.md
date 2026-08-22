# Play Crazy Eights Online with Friends — Free, No App (the Game UNO Is Based On)

If you've ever tried to play Crazy Eights online with friends, you know the drill: the good versions live inside app stores, want an account, and seat you with strangers before you can play with the one person you actually opened the app for. So we built the version we wanted: a [free Crazy Eights table](https://ultimatetools.io/tools/fun-tools/crazy-eights/) that works like a group chat — create a table, share the link, and 2–6 people play from their own phones. No app, no signup, no ads between turns.

## Thirty-second rules refresher

Crazy Eights is the public-domain classic UNO was built on, played with a standard 52-card deck:

- Match the top discard's **suit or rank**, or play an **8** (wild) and call the next suit.
- Can't play? Draw one card — if it's playable you can play it immediately, otherwise your turn passes.
- First empty hand wins. Everyone else's leftovers count against them (8 = 50 points, faces = 10, ace = 1).

If your group knows UNO, they already know this game — minus the trademark and the app install.

## How a table works

1. **Create a table** — one tap, you get a private link.
2. **Share it** on WhatsApp or any chat. Friends open it and join with just a nickname.
3. **Deal** — the server shuffles and deals. Your hand shows only on *your* screen; everyone else sees your card count.
4. **Play** — tap a highlighted card to play it, tap the deck to draw. Playing an 8 pops a suit picker, and the called suit shows as a chip on every screen.
5. **Win, gloat, rematch** — hands are revealed with the points tally, and the host deals again with one tap.

The nice touches are the boring ones: refresh mid-game and your hand comes back; the draw pile automatically recycles the discard pile when it empties; and if someone wanders off to make chai, the host can skip their turn after 45 seconds so the table never stalls.

## The interesting technical bit: secret hands without WebSockets

This runs on plain shared hosting — no WebSocket server, no Redis, no database. Every table is one JSON file; clients poll it every couple of seconds, and moves are serialized through an advisory lockfile so two simultaneous plays can't corrupt the game.

The part that made Crazy Eights different from our earlier multiplayer games (Bingo, typing races): **hands are secret**. Broadcasting the same room state to every poller — fine for Bingo — would leak everyone's cards. So each player gets a private token at join, and the server renders a *per-player view*: your full hand, everyone else's counts, and never the deck order. All moves are validated server-side against your actual hand and the top discard, so a modified client can't play cards it doesn't hold. Optimistic-concurrency versioning turns double-taps and races into clean "table refreshed" responses instead of duplicated moves.

Turn-based games, it turns out, are a perfect match for humble polling: a 1–2 second delay between your friend's move and your screen is invisible in a card game, and the whole thing runs free forever because there's no real-time infrastructure to pay for.

## What it doesn't do (honestly)

- No accounts means no saved match history or rankings — tables live for 6 hours, then vanish.
- It's the *classic* Crazy Eights ruleset: no skips, draw-twos, or reverses. If you want UNO's action cards, the official UNO app is the honest recommendation.
- You bring your own players via the link — there's no matchmaking with strangers, by design.

## Related Tools

- [Host a multiplayer Bingo night with shared rooms](https://ultimatetools.io/tools/fun-tools/bingo/) — every phone gets a unique card; same share-link flow, up to a whole classroom.
- [Race friends in a live typing speed test](https://ultimatetools.io/tools/fun-tools/typing-speed-test/) — create a room and race the same passage in real time.
- [Play classic Klondike Solitaire in the browser](https://ultimatetools.io/tools/fun-tools/solitaire/) — for when it's just you and a coffee.

Deal the first hand now — free, no app, no account: [Crazy Eights Online](https://ultimatetools.io/tools/fun-tools/crazy-eights/)