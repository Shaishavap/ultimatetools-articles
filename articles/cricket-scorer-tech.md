# Building a Real-Time Cricket Scorer with Next.js 15, Prisma, and Canvas Scorecards

I play box cricket every other Friday. Six overs. Ten players. One scorer typing scores into a group chat while trying to watch the match.

The tooling for this is terrible. So I built a proper one: a real-time scoring app where the scorer scores on their phone, everyone else watches live on a shared link, and the scorecard gets WhatsApp-shared as a generated image at the end.

Here's how it's built — the architecture, the tricky parts, and the specific choices I made for a Hostinger VPS deployment.

**Live:** [Box Cricket Scorer — free, no login](https://ultimatetools.io/cricket/)

---

## Architecture Overview

```
Creator phone           → POST /api/cricket/          → creates match row
Scorer phone            → PATCH /api/cricket/[id]/    → updates innings JSON
Viewers (everyone else) → GET /api/cricket/[id]/      → polls every 5 seconds
End of match            → Canvas API generates scorecard PNG → navigator.share()
```

**Key choices:**

- **Polling, not WebSockets** — score updates are infrequent (one ball every ~30 seconds). WebSockets add server complexity for no real gain here. 5-second polling is perfectly adequate.
- **Single JSON column per innings** — the entire innings state (runs, wickets, overs, balls, batters, bowlers) is one JSON blob. No relational joins needed per request.
- **No auth** — creator gets a `shareId` (6-char public ID) + `editToken` (UUID in localStorage). Edit access is verified by sending `editToken` in the request header.

---

## Data Model

```prisma
model CricketMatch {
  id          String    @id @default(cuid())
  shareId     String    @unique   // 6-char public ID
  editToken   String               // UUID — grants edit access
  lockToken   String?              // current active editor (handover)
  lockExpiry  DateTime?
  handoverPin String?              // 4-digit PIN for "pass the phone"
  pinExpiry   DateTime?

  team1Name   String
  team2Name   String
  totalOvers  Int
  status      String    @default("innings1")

  innings1    Json?
  innings2    Json?

  createdAt   DateTime  @default(now())
  expiresAt   DateTime  // auto-expire after 24h

  @@index([shareId])
  @@index([expiresAt])
}
```

The innings JSON structure:

```typescript
interface InningsData {
  runs: number;
  wickets: number;
  currentOver: number;
  currentBalls: number;       // legal balls in current over
  currentOverBalls: string[]; // ["0","1","W","nb","nb4","wd"]
  extras: { wides: number; noBalls: number };
  activeBatters: BatterStats[];
  currentBowler: BowlerStats | null;
  overs: OverSummary[];       // completed overs archive
  dismissals: Dismissal[];
  bowlers: BowlerStats[];     // completed bowler figures
}
```

---

## The `processBall()` Pure Function

All scoring logic lives in one pure function. No side effects — it takes the current innings state and a ball string, returns the new state.

```typescript
export function processBall(
  innings: InningsData,
  ball: string,           // "0"|"1"|"2"|"3"|"4"|"6"|"wd"|"nb"|"W"
  totalOvers: number,
  wicketType?: string,    // "Caught"|"Bowled"|"Run Out"|...
  nbExtraRuns?: number,   // runs scored off the no-ball
  dismissedBatterIdx?: number, // for run-outs (striker or non-striker)
): InningsData {
  const inns: InningsData = JSON.parse(JSON.stringify(innings)); // deep clone
  const strikerIdx = inns.activeBatters.findIndex(b => b.onStrike);
  const bowler = inns.currentBowler!;

  if (ball === "nb") {
    const nbRuns = nbExtraRuns ?? 0;
    const totalExtra = 1 + nbRuns;
    inns.runs += totalExtra;
    inns.extras.noBalls += totalExtra;
    bowler.runs += totalExtra;
    bowler.noBalls += 1;
    // Store as "nb", "nb1", "nb4" etc. for display
    inns.currentOverBalls.push(nbRuns > 0 ? `nb${nbRuns}` : "nb");
    // NOT a legal ball — don't increment currentBalls

  } else if (ball === "W") {
    // Run Out can dismiss either batter
    const si = (wicketType === "Run Out" && dismissedBatterIdx !== undefined)
      ? dismissedBatterIdx : strikerIdx;

    inns.wickets += 1;
    inns.currentBalls += 1;
    bowler.balls += 1;
    if (wicketType !== "Run Out") bowler.wickets += 1;

    const b = inns.activeBatters[si];
    inns.dismissals.push({
      name: b.name || "?",
      runs: b.runs, balls: b.balls,
      fours: b.fours, sixes: b.sixes,
      how: wicketType ?? "Out", bowler: bowler.name,
    });

    // Dismissed slot inherits the onStrike flag
    // so the incoming batter starts at the same end
    const wasOnStrike = b.onStrike;
    inns.activeBatters[si] = {
      name: "", runs: 0, balls: 0, fours: 0, sixes: 0, onStrike: wasOnStrike
    };

  } else {
    const runs = parseInt(ball, 10);
    inns.runs += runs;
    inns.currentBalls += 1;
    inns.activeBatters[strikerIdx].runs += runs;
    inns.activeBatters[strikerIdx].balls += 1;
    if (runs === 4) inns.activeBatters[strikerIdx].fours += 1;
    if (runs === 6) inns.activeBatters[strikerIdx].sixes += 1;
    bowler.runs += runs;
    bowler.balls += 1;
    inns.currentOverBalls.push(ball);
    // Odd runs = strike rotates
    if (runs % 2 === 1) {
      inns.activeBatters = inns.activeBatters.map(b => ({ ...b, onStrike: !b.onStrike }));
    }
  }

  // Over complete (6 legal balls)?
  if (inns.currentBalls >= 6) {
    const overRuns = inns.currentOverBalls.reduce((s, b) => {
      if (b === "W") return s;
      if (b === "wd") return s + 1;
      if (b.startsWith("nb")) {
        const extra = parseInt(b.slice(2), 10) || 0;
        return s + 1 + extra;
      }
      return s + (parseInt(b, 10) || 0);
    }, 0);

    inns.overs.push({
      over: inns.currentOver,
      runs: overRuns,
      wickets: inns.currentOverBalls.filter(b => b === "W").length,
      balls: [...inns.currentOverBalls],
    });

    // Archive bowler figures
    const finishedBowler = { ...bowler, overs: bowler.overs + 1, balls: 0 };
    const idx = inns.bowlers.findIndex(b => b.name === bowler.name);
    if (idx >= 0) inns.bowlers[idx] = finishedBowler;
    else inns.bowlers.push(finishedBowler);

    inns.currentOver += 1;
    inns.currentBalls = 0;
    inns.currentOverBalls = [];
    inns.currentBowler = null;

    // Strike rotates at end of over
    if (inns.wickets < 10 && inns.currentOver <= totalOvers) {
      inns.activeBatters = inns.activeBatters.map(b => ({ ...b, onStrike: !b.onStrike }));
    }
  }

  return inns;
}
```

This pure function makes the scoring logic fully testable without any DB or React involved.

---

## The API Routes

### Create match — `POST /api/cricket/`

```typescript
export async function POST(req: Request) {
  const { team1Name, team2Name, totalOvers } = await req.json();

  const shareId = nanoid(6);   // e.g. "abc123"
  const editToken = randomUUID();
  const expiresAt = new Date(Date.now() + 24 * 60 * 60 * 1000); // 24h

  const match = await prisma.cricketMatch.create({
    data: {
      shareId, editToken, team1Name, team2Name,
      totalOvers, status: "innings1",
      innings1: emptyInnings(), innings2: null,
      expiresAt,
    },
  });

  return Response.json({ shareId: match.shareId, editToken: match.editToken });
}
```

The client stores `editToken` in `localStorage`. It's never in the URL.

### Update score — `PATCH /api/cricket/[shareId]/`

```typescript
export async function PATCH(req: Request, { params }) {
  const { shareId } = await params;
  const editToken = req.headers.get("x-edit-token");
  const lockToken = req.headers.get("x-lock-token");

  const match = await prisma.cricketMatch.findUnique({
    where: { shareId }
  });

  // Verify edit access
  const hasAccess =
    match.editToken === editToken ||
    (match.lockToken === lockToken &&
     match.lockExpiry && new Date(match.lockExpiry) > new Date());

  if (!hasAccess) return Response.json({ error: "Unauthorized" }, { status: 401 });

  const body = await req.json();
  const updated = await prisma.cricketMatch.update({
    where: { shareId },
    data: body,
  });

  return Response.json(updated);
}
```

### Poll for updates — `GET /api/cricket/[shareId]/`

```typescript
export async function GET(req: Request, { params }) {
  const { shareId } = await params;
  const match = await prisma.cricketMatch.findUnique({ where: { shareId } });

  if (!match || new Date() > match.expiresAt) {
    return Response.json({ error: "Not found" }, { status: 404 });
  }

  return Response.json(match);
}
```

Expired matches naturally 404. No cleanup cron needed.

---

## Pass the Phone — PIN Handover

The "Hand Over Scoring" flow:

1. Editor taps "Hand Over" → `POST /api/cricket/[id]/handover/` with their `editToken`
2. Server generates a 4-digit PIN, sets `pinExpiry = now + 2 minutes`
3. New scorer opens the view link, taps "Take Over", enters the PIN
4. `PUT /api/cricket/[id]/handover/` validates the PIN, issues a new `lockToken`, sets `lockExpiry = now + 90 seconds`
5. Previous editor's token is still valid (editToken never expires), but the lockToken is now in the new scorer's localStorage

```typescript
// Claim the PIN
export async function PUT(req: Request, { params }) {
  const { pin } = await req.json();
  const match = await prisma.cricketMatch.findUnique({
    where: { shareId: params.shareId }
  });

  if (
    !match.handoverPin ||
    match.handoverPin !== pin ||
    !match.pinExpiry ||
    new Date() > match.pinExpiry
  ) {
    return Response.json({ error: "Invalid or expired PIN" }, { status: 400 });
  }

  const lockToken = randomUUID();
  const lockExpiry = new Date(Date.now() + 90 * 1000);

  await prisma.cricketMatch.update({
    where: { shareId: params.shareId },
    data: { lockToken, lockExpiry, handoverPin: null, pinExpiry: null },
  });

  return Response.json({ lockToken });
}
```

---

## Canvas Scorecard Generation

The end-of-match scorecard is a 700×~700px PNG generated entirely in the browser using the Canvas API. No server-side image generation needed.

**Two-column layout:**
- Left: batting (name + dismissal inline, R(B), 4s, 6s)
- Right: bowling (name, O, R, W, ECO)
- Vertical divider at x=378

**Key technique — dynamic height pre-calculation:**

Before creating the canvas, calculate the height based on the data:

```typescript
function blockHeight(innings: InningsData | null): number {
  if (!innings) return 0;
  const batters = Math.min(
    (innings.dismissals?.length ?? 0) +
    (innings.activeBatters?.filter(b => b.name)?.length ?? 0),
    12
  );
  const bowlers = Math.min(innings.bowlers?.length ?? 0, 8);
  return TEAM_H + SECT_H + Math.max(batters, bowlers) * ROW_H + EXTRA_H;
}

const H = HEADER_H + blockHeight(innings1) + INS_GAP + blockHeight(innings2) + FOOTER_H;
canvas.height = Math.max(H, 400);
```

**Text truncation using measureText:**

```typescript
function trunc(text: string, maxPx: number): string {
  if (ctx.measureText(text).width <= maxPx) return text;
  let t = text;
  while (t.length > 1 && ctx.measureText(t + "…").width > maxPx) t = t.slice(0, -1);
  return t + "…";
}
```

**QR code in the footer:**

Using `qr-code-styling` (already a project dependency) to generate a scannable QR pointing to `ultimatetools.io/cricket`. The QR has:
- White background + 8px white quiet zone (crucial for scanners to detect boundaries)
- Square dots (sharper edges after WhatsApp JPEG compression)
- Error correction "M" (keeps QR density low for better scannability on a 160px canvas element)

```typescript
const qr = new QRCodeStyling({
  width: QR_SIZE * 4, height: QR_SIZE * 4,  // render at 4x, draw at QR_SIZE
  data: "https://ultimatetools.io/cricket",
  dotsOptions: { type: "square", color: "#1e293b" },
  backgroundOptions: { color: "#ffffff" },
  cornersSquareOptions: { type: "extra-rounded", color: "#4f46e5" },
  qrOptions: { errorCorrectionLevel: "M" },
});
const blob = await qr.getRawData("png");
// Draw white quiet zone first, then QR on top
ctx.fillStyle = "#ffffff";
ctx.fillRect(QR_X - 8, QR_Y - 8, QR_SIZE + 16, QR_SIZE + 16);
ctx.drawImage(img, QR_X, QR_Y, QR_SIZE, QR_SIZE);
```

**Sharing the image:**

```typescript
async function shareToWhatsApp() {
  const blob = await fetch(dataUrl).then(r => r.blob());
  const file = new File([blob], "cricket-scorecard.png", { type: "image/png" });

  const isMobile = /Android|iPhone|iPad|iPod/i.test(navigator.userAgent);
  if (isMobile && navigator.share) {
    await navigator.share({ files: [file], title: "Cricket Scorecard" });
  } else {
    // Desktop fallback: download
    const a = document.createElement("a");
    a.href = dataUrl; a.download = "cricket-scorecard.png"; a.click();
  }
}
```

---

## Hiding Nav/Footer on the Match Page

The match page is a full-screen experience — no top nav, no footer, no padding container. But the root layout in Next.js always renders.

Solution: a client component that reads the current path and conditionally renders:

```typescript
// components/ConditionalNav.tsx
"use client";
import { usePathname } from "next/navigation";

const MATCH_PAGE = /^\/cricket\/[a-z0-9]+\/?$/i;

export function ConditionalNav({ children }) {
  const path = usePathname();
  if (MATCH_PAGE.test(path)) return null;
  return <>{children}</>;
}

export function ConditionalWrapper({ children }) {
  const path = usePathname();
  if (MATCH_PAGE.test(path)) return <>{children}</>;
  return (
    <div className="mx-auto max-w-screen-2xl px-6 py-6 lg:py-8">
      {children}
    </div>
  );
}
```

Used in `app/layout.tsx`:

```tsx
<ConditionalNav><TopNav /></ConditionalNav>
<main className="flex-1">
  <ConditionalWrapper>{children}</ConditionalWrapper>
  <ConditionalNav><Footer /></ConditionalNav>
</main>
```

---

## Deployment Notes (Hostinger VPS)

This app runs on a Hostinger VPS Node.js hosting. A few constraints worth knowing:

**Prisma with MariaDB adapter:**
The project uses Prisma 7.7.0 with `@prisma/adapter-mariadb` instead of the default MySQL connector. Standard Prisma MySQL connector has issues on Hostinger's MariaDB version.

**No WebSockets:**
Hostinger VPS doesn't support persistent WebSocket connections well. Polling was the right call architecturally anyway — cricket scores update infrequently.

**24-hour auto-expiry without a cron:**
Expired matches return 404 on the next GET. Rows are never explicitly deleted — they just become permanently 404. The DB stays small because match data is tiny (~2–5KB per match JSON).

---

## What I'd Change

**If this grew to 1000 concurrent matches:**
- Replace polling with Server-Sent Events (SSE) — much lower overhead than polling, no WebSocket complexity
- Add a cleanup cron to actually delete expired rows

**If I wanted player persistence:**
- Add a simple "save my team" feature using localStorage with a QR-shareable team JSON

For now, for a Friday box cricket match with 22 players, polling + one JSON row per match is exactly right.

---

## Try It

**[Box Cricket Scorer — build your own live scorecard](https://ultimatetools.io/cricket/)**

The full code for the scoring logic, API routes, canvas generation, and handover PIN flow is all in the Next.js app. If you're building something similar — a live poll, a shared game score, any multi-device real-time-ish feature without WebSockets — the polling + single JSON column pattern is worth considering.