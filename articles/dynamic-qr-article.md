How Dynamic QR Codes Work: Redirect Routes, Scan Tracking, and Analytics in Next.js

========== TAGS (max 4) ==========

javascript, webdev, nextjs, tutorial

========== BODY (paste in dev.to body editor — raw markdown) ==========

Static QR codes embed data directly — scan one and you get a URL, a WiFi password, or a phone number baked into the image. But you can't change them after printing. Dynamic QR codes solve this by encoding a short redirect URL instead of the final destination. The server handles the redirect, tracks who scanned it, and lets you change where it points — without reprinting the code.

I built this into [QR Code Generator](https://ultimatetools.io/tools/misc-tools/qr-code-generator/). Here's how the redirect, tracking, and analytics pipeline works.

---

## The architecture in one diagram

```
User scans QR code
        ↓
QR contains: ultimatetools.io/q/abc123
        ↓
GET /q/abc123 (Next.js route)
        ↓
┌─────────────────────────────┐
│ 1. Fetch QR record from DB  │
│ 2. Check: paused? expired?  │
│    scan limit reached?      │
│ 3. Parse IP + User-Agent    │
│ 4. Geo-lookup (ip-api.com)  │
│ 5. Record scan in DB        │
│ 6. 307 redirect → dest URL  │
└─────────────────────────────┘
        ↓
User lands on final destination
```

The QR code itself never changes. Only the server-side mapping changes.

---

## Database schema

Two tables drive the system:

```prisma
model QRCode {
  id        String   @id @default(cuid())
  shortId   String   @unique        // "abc123" — the redirect slug
  userId    String                   // owner (NextAuth)
  type      String                   // URL, WHATSAPP, WIFI, etc.
  title     String
  content   String   @db.Text       // final destination URL
  isDynamic Boolean  @default(true)
  isPaused  Boolean  @default(false)
  scans     Int      @default(0)    // denormalized counter
  expiresAt DateTime?               // optional expiry
  scanLimit Int?                    // optional max scans
  settings  String   @db.Text       // JSON: colors, dot style, logo
  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt
  scanLogs  Scan[]
}

model Scan {
  id         String   @id @default(cuid())
  qrCodeId   String
  ip         String?
  country    String?
  state      String?
  city       String?
  userAgent  String?  @db.Text
  deviceType String?  // mobile | tablet | desktop
  browser    String?
  os         String?
  scannedAt  DateTime @default(now())
  qrCode     QRCode   @relation(fields: [qrCodeId], references: [id])
}
```

The `scans` field on `QRCode` is denormalized — instead of counting `Scan` records every time, we increment it atomically during each scan. Fast reads for the dashboard, no `COUNT(*)` queries.

---

## The redirect route

This is where every scan hits. The entire flow lives in a single `GET` handler at `/q/[shortId]`:

```ts
// app/q/[shortId]/route.ts
import { NextRequest, NextResponse } from "next/server";
import { prisma } from "@/lib/prisma";
import { UAParser } from "ua-parser-js";

export async function GET(
    req: NextRequest,
    { params }: { params: Promise<{ shortId: string }> }
) {
    const { shortId } = await params;

    const qrCode = await prisma.qRCode.findUnique({
        where: { shortId },
    });

    if (!qrCode) {
        return NextResponse.json(
            { error: "QR Code not found" },
            { status: 404 }
        );
    }
```

### Validation gates

Before redirecting, the route checks three conditions:

```ts
    // Gate 1: Owner paused the link
    if (qrCode.isPaused) {
        return htmlPage("⏸️", "This link is currently paused",
            "The owner has temporarily paused this link.");
    }

    // Gate 2: Link has expired
    if (qrCode.expiresAt && new Date() > qrCode.expiresAt) {
        return htmlPage("⏰", "This link has expired",
            "This QR code has passed its expiry date.");
    }

    // Gate 3: Scan limit reached
    if (qrCode.scanLimit !== null && qrCode.scans >= qrCode.scanLimit) {
        return htmlPage("🚫", "Scan limit reached",
            "This QR code has reached its maximum scans.");
    }
```

Each gate returns a styled HTML page (not a JSON error) because the person scanning is a real user on a phone, not an API consumer. The pages use inline CSS — no external stylesheets that might fail to load.

---

## Scan tracking pipeline

After validation passes, we extract everything we can from the request:

### Step 1: IP and User-Agent

```ts
const ip = req.headers.get("x-forwarded-for")?.split(",")[0].trim()
    || req.headers.get("x-real-ip")
    || null;
const userAgent = req.headers.get("user-agent") || null;
```

`x-forwarded-for` can contain multiple IPs if the request passed through proxies. We take the first one — that's the client's original IP.

### Step 2: Device, browser, and OS parsing

```ts
let deviceType: string | null = null;
let browser: string | null = null;
let os: string | null = null;

if (userAgent) {
    const ua = new UAParser(userAgent);
    deviceType = ua.getDevice().type || "desktop";
    browser = ua.getBrowser().name || null;
    os = ua.getOS().name || null;
}
```

`UAParser` returns `undefined` for desktop devices (because desktops don't send a device type in the UA string). We default to `"desktop"` — if it's not a phone or tablet, it's a desktop.

### Step 3: Geolocation from IP

```ts
let country: string | null = null;
let state: string | null = null;
let city: string | null = null;

const isPrivateIp = !ip ||
    /^(127\.|192\.168\.|10\.|172\.(1[6-9]|2\d|3[01])\.)/.test(ip);

if (!isPrivateIp) {
    try {
        const geo = await fetch(
            `http://ip-api.com/json/${ip}?fields=status,country,regionName,city`,
            { signal: AbortSignal.timeout(2000) }
        ).then(r => r.json());

        if (geo?.status === "success") {
            country = geo.country ?? null;
            state = geo.regionName ?? null;
            city = geo.city ?? null;
        }
    } catch {
        // Geo lookup failed — scan is still recorded without location
    }
}
```

Key decisions:

- **ip-api.com** — free tier, no API key, 45 requests/minute. Good enough for QR scan volumes.
- **2-second timeout** — if the geo API is slow, we don't block the redirect. The scan gets recorded without location data.
- **Private IP skip** — `192.168.x.x`, `10.x.x.x`, `127.0.0.1` are local addresses. No point querying geolocation for them.
- **Silent failure** — geo is nice-to-have, not critical. The `catch` block is intentionally empty.

### Step 4: Atomic database write

```ts
await prisma.$transaction([
    prisma.scan.create({
        data: {
            qrCodeId: qrCode.id,
            ip, country, state, city,
            userAgent, deviceType, browser, os
        },
    }),
    prisma.qRCode.update({
        where: { id: qrCode.id },
        data: { scans: { increment: 1 } }
    }),
]);
```

This is a Prisma transaction — both the scan log creation and the counter increment either both succeed or both roll back. Without the transaction, you could have a scan recorded but the counter not incremented (or vice versa) if the server crashes mid-write.

### Step 5: Redirect

```ts
let redirectUrl = qrCode.content;
if (redirectUrl.startsWith("/")) {
    const origin = req.nextUrl.origin;
    redirectUrl = `${origin}${redirectUrl}`;
}

return NextResponse.redirect(redirectUrl);
```

The local path check handles internal redirects (like uploaded PDFs stored at `/uploads/file.pdf`). External URLs pass through unchanged.

---

## Short ID generation

When creating a dynamic QR code, the user can either set a custom slug or get a random one:

```ts
// Custom: user-defined, validated
const slugRegex = /^[a-z0-9_-]{1,30}$/;
if (customSlug && !slugRegex.test(customSlug)) {
    return NextResponse.json({ error: "Invalid slug" }, { status: 400 });
}

// Auto: 6-character random
const shortId = customSlug || Math.random().toString(36).substring(2, 8);
```

Custom slugs let users create branded short links: `ultimatetools.io/q/spring-sale` instead of `ultimatetools.io/q/k8f2x9`. The regex enforces lowercase, numbers, hyphens, and underscores only — no spaces, no special characters.

---

## Schema migration handling

One practical pattern worth sharing — the code handles missing database columns gracefully:

```ts
try {
    // Try with all fields (deviceType, browser, os)
    await prisma.$transaction([
        prisma.scan.create({
            data: { qrCodeId, ip, country, state, city,
                    userAgent, deviceType, browser, os },
        }),
        prisma.qRCode.update({ ... }),
    ]);
} catch (err) {
    // If columns don't exist yet, fall back to base fields
    if (err?.message?.includes("Unknown argument")) {
        await prisma.$transaction([
            prisma.scan.create({
                data: { qrCodeId, ip, country, state, city, userAgent },
            }),
            prisma.qRCode.update({ ... }),
        ]);
    } else {
        throw err;
    }
}
```

This lets you deploy code that references new columns before running the database migration. The scan still gets recorded with the fields that exist. No downtime, no deploy ordering headaches.

---

## The analytics dashboard

The raw scan data powers a dashboard with:

- **Total scans** and **unique visitors** (distinct IPs)
- **Daily trend chart** — bar chart using Recharts
- **Device breakdown** — mobile vs tablet vs desktop percentages
- **Location table** — city, state, country with scan counts
- **Date range filters** — 7, 14, 30, or 90 days
- **CSV export** — download all scan data

All queries filter by `userId` — you only see analytics for your own QR codes. The denormalized `scans` counter on `QRCode` means the dashboard list loads fast without aggregation queries.

---

## Error pages, not error JSON

When a dynamic QR code is paused, expired, or maxed out, the route returns a full HTML page — not a JSON response:

```ts
const htmlPage = (icon: string, title: string, message: string) =>
    new NextResponse(
        `<!DOCTYPE html><html>
         <head><style>/* inline CSS */</style></head>
         <body>
           <div class="card">
             <div class="icon">${icon}</div>
             <h1>${title}</h1>
             <p>${message}</p>
             <a href="https://ultimatetools.io">Go to UltimateTools.io</a>
           </div>
         </body></html>`,
        { status: 410, headers: { "Content-Type": "text/html" } }
    );
```

HTTP 410 (Gone) tells search engines and link checkers that this URL is intentionally unavailable — different from 404 (not found). The inline CSS ensures the page renders correctly even without external resources.

---

## Static vs dynamic: when to use which

| | Static | Dynamic |
|---|---|---|
| Data location | Embedded in QR image | Server-side mapping |
| Editable after print | No | Yes |
| Requires account | No | Yes |
| Scan tracking | No | Yes (IP, device, location) |
| Expiry/scan limits | No | Yes |
| Privacy | Maximum (no server) | Scans hit the server |
| Use case | WiFi passwords, vCards | Marketing campaigns, menus |

The tool supports both. Static codes are generated entirely in the browser using `qr-code-styling` — nothing touches the server. Dynamic codes require authentication and use the redirect architecture described above.

---

## Wrapping up

The dynamic QR code system comes down to three things:

1. **A redirect route** that validates, tracks, and redirects
2. **A database schema** that stores the mapping and scan logs
3. **An analytics layer** that aggregates the data into something useful

The redirect adds ~100-200ms of latency (database lookup + geo API), which is imperceptible to someone who just scanned a QR code with their phone camera.

Try it out: [QR Code Generator](https://ultimatetools.io/tools/misc-tools/qr-code-generator/)

Create a free account to try dynamic codes with scan tracking. Static codes work without signup.