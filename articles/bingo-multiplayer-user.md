# Free Multiplayer Bingo for Family Game Night — No App, No Login, Everyone Gets Their Own Card

Family game night used to mean printing Bingo cards, cutting them out, and hoping nobody grabbed the same one. Virtual game nights meant emailing PDFs that nobody bothered to open. There's a simpler way now.

The [multiplayer Bingo game](https://ultimatetools.io/tools/fun-tools/bingo/) at Ultimate Tools works like this: the host creates a room and shares one link. Everyone opens it on their phone, types a nickname, and gets a unique randomly-generated card — no two cards are the same. The host calls numbers, players tap to mark them, and the first to complete a line (or full house) hits the BINGO button. The server checks it. Winner announced on every device simultaneously.

No app. No account. No printed cards.

---

## How to set up a game in under a minute

**Step 1 — Create a room**

Go to the [online Bingo game](https://ultimatetools.io/tools/fun-tools/bingo/) and choose your win mode:

- **Line** — first player to complete any row, column, or diagonal wins
- **Full House** — first to mark all 24 numbers wins
- **Line + Full House** — two-phase game: line wins a prize, then the game continues until someone gets a full card

Click Create Room. You're now the host.

**Step 2 — Share the link**

A shareable link appears at the top of the host panel. Copy it and send it to players over WhatsApp, Slack, a group chat — anything. Players open it on their own device.

**Step 3 — Players join**

Each player opens the link, enters a nickname, and gets their own unique Bingo card. You can see players appear in the host panel as they join. There's no cap on players — the game works equally well with 2 or 20.

**Step 4 — Call numbers**

Click **Call Number** to draw the next number. It appears on every player's screen within 3 seconds. Players tap called numbers on their card to mark them — a green highlight confirms the mark.

Or turn on **Auto-Call** — the host panel has a toggle that draws numbers automatically at 3s, 5s, or 10s intervals. Useful when the host wants to focus on the room rather than the screen.

**Step 5 — Claim BINGO**

When a player completes a winning pattern, a large **BINGO!** button activates on their screen. They tap it. The server checks their card against the called numbers — if it's valid, the winner is announced on every device. If the claim is invalid (someone tapped the button too early), they get a quiet error and the game continues.

---

## The Bingo card — how it works

Every card follows standard US Bingo:

| Column | Range |
|--------|-------|
| B | 1 – 15 |
| I | 16 – 30 |
| N | 31 – 45 |
| G | 46 – 60 |
| O | 61 – 75 |

The center square (**N**, row 3) is always **FREE** — it counts as marked from the start. Winning patterns are any row, any column, or either diagonal. Full House requires all 24 remaining numbers marked.

Every player's card is generated randomly on the server at join time, so no two players have the same card layout.

---

## Line + Full House mode — the best format for groups

Standard Bingo ends the moment someone gets a line. With larger groups, that can feel fast. **Line + Full House** mode extends the game in two phases:

1. **Phase 1** — the first player to complete a line wins a line prize. The host announces it, but the game continues.
2. **Phase 2** — all remaining numbers keep getting called. The first player to fill their entire card wins the main prize.

This keeps everyone engaged after the first winner. Players who already got a line can still win the Full House.

---

## What happens if you refresh the page

Your card, nickname, and marked numbers are saved in your browser. Refreshing the page brings you straight back to your card — no need to rejoin. The host's room state and all called numbers are stored on the server, so nothing is lost even if the host briefly loses connection.

---

## Starting a new round without losing players

After a game ends, the host can click **New Round (same room)**. This resets all called numbers, regenerates everyone's cards, and restarts — players don't need to re-open the link or rejoin. Great for running multiple rounds in one session.

---

## Use cases

**Family game night** — Everyone is already on their phone. Create a room, drop the link into the family group chat, play directly from the sofa. No setup, no printed cards, no "can you read that again."

**Classroom** — Each student joins from their device. The teacher calls numbers manually and controls the pace. Works for vocabulary, number recognition, or just a Friday afternoon wind-down.

**Office party** — Drop the link in the work Slack channel. Remote and in-office colleagues play on equal footing — everyone has the same experience regardless of where they're sitting.

**Virtual game night** — Share the link in a video call. Everyone joins from their end and plays in real time alongside the video call.

---

## How it was built (the short version)

No WebSockets. The entire multiplayer experience runs on **3-second polling** — every player's device fetches the current room state from the server every 3 seconds. This approach is simpler to deploy, cheaper to run, and handles player disconnects gracefully. When a player refreshes or loses connection, the next poll brings them back to the correct state automatically.

Each room is a database row with a `calledNums` JSON array that grows one number at a time. Players have their own row with their `card` (a 5×5 number grid) and `markedNums` array. Win verification happens entirely server-side — when a player claims BINGO, the server fetches their card and marked numbers, checks all 12 possible winning patterns against the official called list, and returns `valid: true` or `valid: false`. The client can't fake a win.

Rooms expire after 6 hours. No cleanup job needed — expired rooms are just ignored.

---

Play it now: [Multiplayer Bingo](https://ultimatetools.io/tools/fun-tools/bingo/) — free, no login, works on any device. Other games on the same site: [Wordle](https://ultimatetools.io/tools/fun-tools/wordle/), [Memory Game](https://ultimatetools.io/tools/fun-tools/memory-game/), and [Whack-a-Mole](https://ultimatetools.io/tools/fun-tools/whack-a-mole/).