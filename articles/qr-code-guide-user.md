# The Complete Guide to QR Codes — How They Work, When to Use Them, and How to Generate Them Free

QR codes are everywhere — on packaging, menus, business cards, billboards, and event tickets. Most people know how to scan one. Fewer people know how to generate exactly the kind they need, or what the different formats are for.

This post covers how QR codes work, the different data types they support, customization options, and how to generate them for free without any account.

---

## How QR codes work

A QR code (Quick Response code) is a two-dimensional barcode that encodes data as a pattern of black and white squares. The data can be a URL, plain text, contact details, WiFi credentials, or any string of characters.

**Key structural elements:**
- **Finder patterns** — the three large square markers in the corners. Let scanners locate and orient the code.
- **Data area** — the matrix of small squares that encodes the actual content.
- **Error correction** — QR codes can be partially damaged or obscured and still scan correctly. Four levels: L (7% recovery), M (15%), Q (25%), H (30%).

Higher error correction = more redundant data = larger QR code for the same payload.

---

## QR code data types

### URL
The most common type. Encodes a full URL. When scanned, the device opens the URL in the default browser.

```
https://ultimatetools.io/tools/misc-tools/qr-code-generator/
```

### Plain text
Encodes a string of text. Scanners display the text directly without opening a browser.

```
Table 4 — WiFi password is: coffee2024
```

### WiFi credentials
Encodes WiFi network information using the `WIFI:` protocol. When scanned on iOS or Android, the device offers to join the network automatically.

```
WIFI:T:WPA;S:NetworkName;P:password123;;
```

**Parameters:** `T` = security type (WPA/WEP/nopass), `S` = SSID, `P` = password, `H` = hidden network.

### Contact (vCard)
Encodes a vCard string so scanning adds the contact directly to the device's address book.

```
BEGIN:VCARD
VERSION:3.0
FN:Jane Smith
ORG:Acme Corp
TEL:+1-555-0100
EMAIL:jane@acme.com
END:VCARD
```

### Email
Opens the email client with a pre-filled recipient and optional subject/body.

```
mailto:contact@example.com?subject=Hello&body=I%20saw%20your%20QR%20code
```

### Phone / SMS
Initiates a call or opens the SMS app.

```
tel:+15550100
sms:+15550100?body=Hi
```

---

## Static vs dynamic QR codes

**Static QR codes** — the destination URL or data is encoded permanently in the QR code pattern. Once printed, it can't be changed. Free to generate; standard for most uses.

**Dynamic QR codes** — a short URL redirect is encoded instead of the actual destination. The redirect can be updated after printing. Requires a service with a redirect infrastructure. Typically paid (QR Tiger, Bitly QR codes, etc.).

For most use cases, static QR codes are sufficient. You only need dynamic codes if you've already printed materials and need to change the destination — for example, a flyer with a QR code pointing to a campaign page that later moves.

---

## Customization options

### Colors
QR codes don't have to be black on white. Any high-contrast color combination works. Minimum recommended contrast ratio for reliable scanning: 4:1. Avoid light foreground on white background.

### Dot style
- **Square** — classic, maximum scanner compatibility
- **Rounded** — softer appearance, still highly scannable
- **Dots** — circular modules, modern look

### Size
For print: minimum 1cm × 1cm. For billboards or signage: scale up proportionally — the scan distance is the limiting factor.

### Format
- **PNG** — raster image, best for web and digital use
- **SVG** — vector, scales to any size without quality loss, best for print

---

## How to generate a QR code for free

The [QR Code Generator](https://ultimatetools.io/tools/misc-tools/qr-code-generator/) generates single QR codes in the browser. Choose data type, enter the content, customize colors and dot style, download as PNG or SVG. No account, no watermark.

For batch generation from a list of URLs or a CSV file, use the [Bulk QR Code Generator](https://ultimatetools.io/tools/misc-tools/bulk-qr-code-generator/) — generates up to 500 QR codes at once and downloads them as a ZIP.

Both tools run entirely in your browser. The content you encode never leaves your device.

---

## How to scan a QR code

**iOS** — Camera app (built-in since iOS 11). Point at the code, tap the notification.

**Android** — Camera app (built-in on most devices since Android 9), or Google Lens.

**Desktop** — Browser extensions (Chrome has a built-in "Use your phone" scan option), or use an [online QR scanner](https://ultimatetools.io/tools/misc-tools/qr-code-scanner/) to scan from an image file.

---

## Best practices

1. **Always test before printing.** Scan the QR code with at least two different devices before putting it on physical materials.

2. **Add a call to action.** "Scan to see the menu" or "Scan to get the discount code." People hesitate to scan QR codes without context.

3. **Keep URLs short.** Longer URLs = more data = denser QR code = harder to scan at small sizes. Use a short URL if the destination URL is very long.

4. **Don't put QR codes where phones can't connect.** Underground venues, parking garages, and dead zones don't work well for URL QR codes.

5. **For print, use SVG.** SVG scales cleanly to any print size without pixelation.

---

Generate yours at [ultimatetools.io/tools/misc-tools/qr-code-generator/](https://ultimatetools.io/tools/misc-tools/qr-code-generator/) — single QR codes free, no account, no watermark. For bulk generation: [ultimatetools.io/tools/misc-tools/bulk-qr-code-generator/](https://ultimatetools.io/tools/misc-tools/bulk-qr-code-generator/).