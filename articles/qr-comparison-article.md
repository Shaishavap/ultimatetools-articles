I Built a Free Dynamic QR Code Generator — Here's What Competitors Charge For

========== TAGS (max 4) ==========

webdev, opensource, javascript, productivity

========== BODY (paste in dev.to body editor — raw markdown) ==========

I needed dynamic QR codes for a project. The kind where you can change the destination URL after printing, track who scanned it, and see analytics like device type and location.

So I looked at pricing from existing services. Here's what I found.

---

## What "free" actually means at most QR services

Most QR code generators advertise a free tier. But the limits make it unusable for anything real:

| Feature | Typical Free Tier | What you actually need |
|---|---|---|
| Dynamic QR codes | 1-2 codes | More than 2 |
| Analytics retention | 7-14 days | At least 30+ days |
| Watermark | Forced on free | No watermark |
| CSV export | Paid only | Yes, for reporting |
| Custom short URLs | Paid only | Nice to have |
| Scan limits | Often capped | Unlimited |

To unlock what most people consider basic features — no watermark, analytics beyond 14 days, CSV export — you're looking at **$5-50/month** on the low end, and **$50-100/month** for business plans.

For generating a QR code. That redirects to a URL.

---

## What I built instead

I built a [Dynamic QR Code Generator](https://ultimatetools.io/tools/misc-tools/qr-code-generator/) as part of UltimateTools — and made everything free. No "upgrade to unlock" walls.

Here's what's included at zero cost:

### Unlimited dynamic QR codes

Create as many as you need. Each one gets a short redirect URL (`ultimatetools.io/q/your-slug`) that you can edit anytime without reprinting the code.

### Full scan analytics — no time limit

Every scan is tracked with:
- **Device type** — mobile, tablet, or desktop
- **Browser and OS** — Chrome, Safari, iOS, Android, etc.
- **Location** — country, state, and city (via IP geolocation)
- **Timestamp** — when each scan happened

No 14-day window. No 90-day cap. Your data stays as long as your QR code exists.

### Custom short URLs

Instead of a random string like `/q/k8f2x9`, set a branded slug: `/q/spring-sale` or `/q/menu`. Most services charge for this.

### Expiry dates and scan limits

Set an optional expiry date or maximum scan count. When the limit is hit, scanners see a clean error page instead of a broken redirect. Useful for limited-time promotions or event tickets.

### No watermark

The QR code you download is clean. No branding overlay, no "powered by" text. Most free tiers force a watermark that you can only remove by paying.

### CSV export

Download your scan data as CSV for reporting or analysis. Again, typically a paid feature elsewhere.

### 10 content types

Not just URLs. Generate QR codes for:
- WhatsApp messages
- WiFi credentials
- Email (pre-filled to, subject, body)
- vCards (contact cards)
- Phone numbers
- GPS coordinates
- UPI payments
- Plain text
- PDF links

### Visual customization

- Foreground and background colors
- Dot styles: square, rounded, dots, classy
- Corner patterns
- Logo upload (centered)
- Error correction levels (L, M, Q, H)
- Download as PNG (up to 2048px), SVG, or PDF

---

## The comparison nobody asked for (but you need)

Here's a real side-by-side with a popular QR code service's pricing:

| Feature | Competitor (Free) | Competitor (₹400/mo) | UltimateTools (Free) |
|---|---|---|---|
| Dynamic QR codes | 2 (with ads) | 5 (ad-free) | **Unlimited** |
| Analytics retention | 14 days | 60 days | **Unlimited** |
| Watermark-free | No | Yes | **Yes** |
| CSV export | No | Yes | **Yes** |
| Custom slugs | No | No | **Yes** |
| Expiry/scan limits | No | No | **Yes** |
| Content types | Limited | Limited | **10 types** |
| GPS location tracking | No | Checkbox (extra) | **Included** |

Their business plan at **₹4,509/month** ($54/mo) adds features like API access and integrations — but for most use cases (restaurant menus, event links, marketing campaigns), the free tier on UltimateTools already covers everything.

---

## How it works under the hood

If you're curious about the technical implementation:

**Static QR codes** are generated entirely in the browser using `qr-code-styling`. The data is embedded directly in the image — no server involved.

**Dynamic QR codes** use a redirect architecture:

```
QR Code → ultimatetools.io/q/abc123
                    ↓
         Server validates (paused? expired? scan limit?)
                    ↓
         Records scan (IP, device, location via ip-api.com)
                    ↓
         307 redirect → final destination URL
```

The scan recording uses a Prisma transaction to atomically create the scan log and increment the denormalized counter. Geolocation lookup has a 2-second timeout — if the API is slow, the scan is still recorded without location data, and the redirect happens immediately.

I wrote a detailed technical deep-dive here: [How Dynamic QR Codes Work: Redirect Routes, Scan Tracking, and Analytics in Next.js](https://dev.to/shaishav_patel_271fdcd61a/how-dynamic-qr-codes-work-redirect-routes-scan-tracking-and-analytics-in-nextjs-2l)

---

## Why give it away for free?

UltimateTools is a collection of 35+ browser-based tools — PDF processing, image editing, text utilities, developer tools, and more. The QR code generator is one tool in the suite.

The goal is to build something genuinely useful, not to gate basic features behind a paywall. Dynamic QR codes are not expensive to serve. The redirect route adds ~100-200ms of latency per scan. The database storage is minimal. There's no technical reason to charge $50/month for this.

---

## Try it

Create a dynamic QR code in 30 seconds: [QR Code Generator](https://ultimatetools.io/tools/misc-tools/qr-code-generator/)

- Static codes work without an account
- Dynamic codes require a free signup (for analytics and editing)
- No credit card, no trial period, no upgrade prompts