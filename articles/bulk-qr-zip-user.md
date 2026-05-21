# Generate Bulk QR Codes From a CSV File — Free, No Signup, ZIP Download

Generating QR codes one at a time stops scaling fast. Ten codes: fine. A hundred product SKUs, a list of event attendees, a set of table menus — you need a way to generate them all at once and download the whole batch in one click.

The [Bulk QR Code Generator](https://ultimatetools.io/tools/misc-tools/bulk-qr-code-generator/) accepts a CSV file and outputs every QR code as a named PNG inside a single ZIP download. Up to 500 codes per upload. No account, no per-code pricing, no watermarks.

---

## The CSV Format

Two required columns: `name` and `url`.

```csv
name,url
Product A,https://yoursite.com/product-a
Product B,https://yoursite.com/product-b
Store London,https://yoursite.com/store/london
Store Manchester,https://yoursite.com/store/manchester
```

The `name` column becomes the filename for each QR code PNG inside the ZIP:
- `Product A` → `Product A.png`
- `Store London` → `Store London.png`

Use descriptive names — SKU numbers, location codes, event IDs — so the ZIP is organized and usable immediately after download.

**Create your CSV in any spreadsheet app:** Google Sheets, Excel, LibreOffice Calc. Add the `name` and `url` columns, fill in your data, export as CSV (File → Download → CSV).

---

## Step-by-Step

**1. Prepare your CSV**
Open any spreadsheet. Column A: `name`. Column B: `url`. Fill in rows. Export as `.csv`.

**2. Upload**
Go to the [free Bulk QR Code Generator](https://ultimatetools.io/tools/misc-tools/bulk-qr-code-generator/) and upload your CSV. The tool parses it instantly.

**3. Preview**
A preview grid shows the first batch of QR codes with their names. Verify a few to confirm everything encoded correctly before downloading.

**4. Download ZIP**
Click **Download All as ZIP**. One ZIP file, one PNG per row, named by your `name` column. Open the ZIP and your QR codes are ready to use.

---

## What Goes in the URL Column

The `url` field accepts any QR code payload — it doesn't have to be a web URL:

| Type | Example value |
|---|---|
| Web URL | `https://yoursite.com/page` |
| WhatsApp link | `https://wa.me/1234567890` |
| Email | `mailto:contact@company.com` |
| Phone | `tel:+1234567890` |
| Plain text | Any string — address, serial number, product code |
| vCard | Full vCard string for contact QR codes |

The generator encodes whatever string you put in the `url` column. Any QR scanner on the other end decodes it.

---

## Use Cases

**Retail / inventory**
One QR per product SKU linking to the product page, spec sheet, or reorder form. Label products during packing.

**Events**
One QR per attendee linking to their digital ticket or check-in confirmation. Print and include in the welcome pack.

**Restaurant table menus**
One QR per table linking to the table-specific order form or menu. Generate for every table in one upload.

**Marketing — UTM tracking**
One QR per channel (print, email, billboard, social) all pointing to the same landing page but with different UTM parameters (`?utm_source=print`, `?utm_source=email`). Track which physical channel drove traffic.

**Asset tagging**
Label physical assets — laptops, equipment, office furniture — with QR codes that link to their inventory record. Generate codes for the full asset list in one CSV upload.

---

## Free, No Signup, No Watermark

The [Bulk QR Code Generator](https://ultimatetools.io/tools/misc-tools/bulk-qr-code-generator/) generates up to 500 QR codes per CSV. No account required. No watermarks on the PNG output. No per-code fee.

Everything runs in your browser — the CSV data and generated QR codes never leave your device. The ZIP downloads directly without passing through any server.