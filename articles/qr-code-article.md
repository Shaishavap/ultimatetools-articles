Building a QR Code Generator with 10 Content Types and Scan Analytics


========== TAGS (max 4) ==========

javascript, webdev, nextjs, tutorial


========== BODY (paste in dev.to body editor — raw markdown) ==========

QR codes are everywhere, but most generators only handle URLs. We built one that supports 10 content types — WhatsApp, WiFi, vCard, GPS, Email, UPI and more — with dynamic short links, scan analytics (device, browser, location), and custom dot styles. Here's how it's structured.

---

## Content Types

Each QR type encodes a differently formatted string. The browser renders a live preview as the user fills in the fields.

```typescript
type QrType =
  | "URL" | "TEXT" | "PHONE" | "UPI"
  | "WHATSAPP" | "WIFI" | "EMAIL"
  | "VCARD" | "GPS" | "PDF";

function buildQrContent(type: QrType, fields: Record<string, string>): string {
  switch (type) {
    case "WHATSAPP":
      return `https://wa.me/${fields.phone}?text=${encodeURIComponent(fields.message)}`;

    case "WIFI":
      return `WIFI:T:${fields.encryption};S:${fields.ssid};P:${fields.password};;`;

    case "EMAIL":
      return `mailto:${fields.email}?subject=${encodeURIComponent(fields.subject)}&body=${encodeURIComponent(fields.body)}`;

    case "VCARD":
      return [
        "BEGIN:VCARD", "VERSION:3.0",
        `FN:${fields.name}`,
        `TEL:${fields.phone}`,
        `EMAIL:${fields.email}`,
        `ORG:${fields.company}`,
        `URL:${fields.website}`,
        "END:VCARD"
      ].join("\n");

    case "GPS":
      return `geo:${fields.lat},${fields.lng}`;

    case "UPI":
      return `upi://pay?pa=${fields.upiId}&pn=${encodeURIComponent(fields.name)}&am=${fields.amount}&cu=INR`;

    default:
      return fields.content || "";
  }
}
```

---

## Rendering with qr-code-styling

We use [qr-code-styling](https://github.com/kozakdenys/qr-code-styling) for the visual output — it supports custom dot shapes, corner styles, colors, and logo embedding that plain `<canvas>` QR libraries don't.

It's an imperative DOM library, so it needs careful handling in React:

```typescript
const qrRef = useRef<HTMLDivElement>(null);
const qrInstance = useRef<QRCodeStyling | null>(null);

useEffect(() => {
  // Dynamic import — avoids SSR issues
  import('qr-code-styling').then(({ default: QRCodeStyling }) => {
    if (qrInstance.current) {
      // Update existing instance rather than recreating
      qrInstance.current.update({ data: content, ...styleOptions });
    } else {
      qrInstance.current = new QRCodeStyling({ data: content, ...styleOptions });
      if (qrRef.current) {
        qrInstance.current.append(qrRef.current);
      }
    }
  });
}, [content, styleOptions]);
```

Key points:
- Dynamic import prevents SSR crash (the library accesses `document` on init)
- Store the instance in `useRef`, not `useState` — you don't want re-renders when it changes
- Call `.update()` instead of recreating — avoids the DOM element being replaced on every keystroke

---

## Dynamic QR Codes and Short Links

Static QR codes encode the destination directly. Dynamic codes point to a short link (`/q/[shortId]`) that redirects to the real URL — meaning you can update the destination without regenerating the image.

The redirect route records a scan before redirecting:

```typescript
// app/q/[shortId]/route.ts
export async function GET(req: NextRequest, { params }: { params: { shortId: string } }) {
  const qrCode = await prisma.qRCode.findUnique({
    where: { shortId: params.shortId }
  });

  if (!qrCode) return NextResponse.redirect('/404');

  // Check expiry and scan limit
  if (qrCode.expiresAt && new Date() > qrCode.expiresAt) {
    return NextResponse.redirect('/expired');
  }
  if (qrCode.scanLimit && qrCode.scanCount >= qrCode.scanLimit) {
    return NextResponse.redirect('/limit-reached');
  }

  // Record the scan asynchronously
  const ip = req.headers.get('x-forwarded-for')?.split(',')[0] ?? '';
  const ua = req.headers.get('user-agent') ?? '';
  const { device, browser, os } = parseUA(ua);
  const { country, city } = await getGeoFromIp(ip);

  await prisma.$transaction([
    prisma.scan.create({
      data: { qrCodeId: qrCode.id, ip, country, city, userAgent: ua, device, browser, os }
    }),
    prisma.qRCode.update({
      where: { id: qrCode.id },
      data: { scanCount: { increment: 1 } }
    })
  ]);

  return NextResponse.redirect(qrCode.content);
}
```

---

## Scan Analytics

Each scan record captures:

```typescript
// ua-parser-js for device/browser/OS
function parseUA(ua: string) {
  const parser = new UAParser(ua);
  return {
    device: parser.getDevice().type ?? 'desktop',
    browser: parser.getBrowser().name ?? 'Unknown',
    os: parser.getOS().name ?? 'Unknown',
  };
}

// ip-api.com for geolocation — free, no API key, 45 req/min
async function getGeoFromIp(ip: string) {
  const isPrivate = /^(10\.|172\.(1[6-9]|2\d|3[01])\.|192\.168\.|127\.|::1)/.test(ip);
  if (isPrivate) return { country: 'Local', city: '' };

  try {
    const res = await fetch(`http://ip-api.com/json/${ip}?fields=country,city`,
      { signal: AbortSignal.timeout(2000) }
    );
    const data = await res.json();
    return { country: data.country ?? '', city: data.city ?? '' };
  } catch {
    return { country: '', city: '' };
  }
}
```

The analytics dashboard aggregates scans by date (trend chart), device type, browser, and country using `recharts` BarChart and PieChart.

---

## SVG vs PNG Download

For print use, SVG is far better — infinitely scalable, no pixelation. `qr-code-styling` exposes `getRawData('svg')` for this:

```typescript
async function handleDownload(format: 'svg' | 'png') {
  if (format === 'svg') {
    const blob = await qrInstance.current?.getRawData('svg');
    if (!blob) return;
    const url = URL.createObjectURL(blob);
    const a = document.createElement('a');
    a.href = url; a.download = 'qr-code.svg';
    a.click(); URL.revokeObjectURL(url);
    return;
  }

  // PNG at high resolution — create a separate instance at download size
  const QRCodeStyling = (await import('qr-code-styling')).default;
  const highResQr = new QRCodeStyling({ ...options, width: 1024, height: 1024 });
  const blob = await highResQr.getRawData('png');
  // ... same download pattern
}
```

The live preview renders at ~300px. The download creates a fresh instance at 1024px so the PNG is print-ready without upscaling artifacts.

---

## Custom Slugs

Users can set a custom short ID (`/q/my-brand`) instead of the auto-generated random string. Validation before saving:

```typescript
const SLUG_REGEX = /^[a-z0-9-]{3,30}$/;

if (customSlug) {
  if (!SLUG_REGEX.test(customSlug)) {
    return NextResponse.json({ error: 'Slug must be 3-30 chars, lowercase letters, numbers and hyphens only.' }, { status: 400 });
  }
  const existing = await prisma.qRCode.findUnique({ where: { shortId: customSlug } });
  if (existing) {
    return NextResponse.json({ error: 'This slug is already taken.' }, { status: 409 });
  }
}
```

---

## Try It

Live at [QR Code Generator](https://ultimatetools.io/tools/misc-tools/qr-code-generator/) — static codes are free with no account, dynamic codes include analytics and editing.

What QR type do you find most underused? WiFi codes are criminally underrated in my opinion — drop your thoughts in the comments.