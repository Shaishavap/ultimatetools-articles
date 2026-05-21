# Building a Bulk QR Code Generator in the Browser — CSV Parsing, Batch Generation, and ZIP Download

The [Bulk QR Code Generator](https://ultimatetools.io/tools/misc-tools/bulk-qr-code-generator/) generates up to 500 QR codes from a CSV file and bundles them into a single ZIP download — all in the browser, no server involved. The three library choices that make this work: PapaParse for CSV, qrcode.js for generation, and JSZip for bundling.

---

## CSV Parsing With PapaParse

PapaParse handles the edge cases that break a naive `split(',')` approach: quoted fields with commas inside, multi-line values, inconsistent whitespace, and missing/extra columns.

```typescript
import Papa from 'papaparse';

interface CsvRow {
  name: string;
  url: string;
}

function parseCsv(file: File): Promise<CsvRow[]> {
  return new Promise((resolve, reject) => {
    Papa.parse<CsvRow>(file, {
      header: true,          // use first row as column names
      skipEmptyLines: true,  // ignore blank rows
      transformHeader: h => h.trim().toLowerCase(),
      complete: (results) => {
        const rows = results.data.filter(
          row => row.name?.trim() && row.url?.trim()
        );
        resolve(rows);
      },
      error: reject,
    });
  });
}
```

`transformHeader` normalizes column names — "Name", " name ", "NAME" all map to `"name"`. The `filter` step removes rows with empty name or url values (rows with data in other columns but not these two).

---

## Batch QR Generation With Progress

Generating 500 QR codes synchronously would freeze the UI. Process them in batches with `setTimeout` yield points so the browser can render progress updates between batches:

```typescript
import QRCode from 'qrcode';

interface QrResult {
  name: string;
  dataUrl: string;
}

async function generateQrCodes(
  rows: CsvRow[],
  onProgress: (completed: number, total: number) => void
): Promise<QrResult[]> {
  const results: QrResult[] = [];
  const BATCH_SIZE = 20;

  for (let i = 0; i < rows.length; i += BATCH_SIZE) {
    const batch = rows.slice(i, i + BATCH_SIZE);

    const batchResults = await Promise.all(
      batch.map(async row => ({
        name: row.name,
        dataUrl: await QRCode.toDataURL(row.url, {
          width: 300,
          margin: 2,
          errorCorrectionLevel: 'M',
        }),
      }))
    );

    results.push(...batchResults);
    onProgress(results.length, rows.length);

    // Yield to the event loop between batches
    if (i + BATCH_SIZE < rows.length) {
      await new Promise(resolve => setTimeout(resolve, 0));
    }
  }

  return results;
}
```

`Promise.all` runs a batch of 20 in parallel. `setTimeout(resolve, 0)` after each batch yields control back to the browser for one event loop tick — enough for React to re-render the progress bar with the updated count.

`errorCorrectionLevel: 'M'` is a good default — 15% error correction. Level `'H'` (30%) makes the QR denser and harder to scan quickly; level `'L'` (7%) is too fragile for printed codes. `'M'` balances scannability and density.

---

## Sanitizing Filenames

The `name` column from user CSV input can contain characters that are invalid in filenames:

```typescript
function sanitizeFilename(name: string): string {
  return name
    .replace(/[<>:"/\\|?*\x00-\x1f]/g, '_') // invalid chars → underscore
    .replace(/\.+$/, '')                       // no trailing dots
    .trim()
    .slice(0, 200)                             // cap at 200 chars
    || 'unnamed';                              // fallback for empty result
}
```

Windows rejects filenames containing `<>:"/\|?*` and control characters. macOS and Linux are more permissive but still choke on `/` and null bytes. The 200-character limit keeps filenames well within the 255-byte limit most filesystems enforce (leaving room for `.png`).

---

## Bundling Into a ZIP With JSZip

```typescript
import JSZip from 'jszip';

async function createZip(results: QrResult[]): Promise<Blob> {
  const zip = new JSZip();

  for (const result of results) {
    const filename = `${sanitizeFilename(result.name)}.png`;

    // dataUrl is "data:image/png;base64,<data>" — extract the base64 part
    const base64Data = result.dataUrl.split(',')[1];
    zip.file(filename, base64Data, { base64: true });
  }

  return zip.generateAsync({
    type: 'blob',
    compression: 'DEFLATE',
    compressionOptions: { level: 6 },
  });
}
```

`{ base64: true }` tells JSZip to decode the base64 string to binary before compressing. Without this flag, JSZip writes the raw base64 string as text, producing an invalid PNG file inside the ZIP.

PNG files are already compressed internally, so ZIP compression gives modest gains (5–15%) on PNG-heavy archives. `level: 6` is the default compression level — a good trade-off between speed and size.

---

## Triggering the Download

```typescript
function downloadZip(blob: Blob, filename: string): void {
  const url = URL.createObjectURL(blob);
  const a = document.createElement('a');
  a.href = url;
  a.download = filename;
  a.click();
  URL.revokeObjectURL(url);
}
```

---

## Putting It Together

```typescript
async function handleBulkGenerate(file: File, setProgress: (n: number) => void) {
  const rows = await parseCsv(file);

  const results = await generateQrCodes(rows, (done, total) => {
    setProgress(Math.round((done / total) * 100));
  });

  const zip = await createZip(results);
  downloadZip(zip, 'qr-codes.zip');
}
```

The full [Bulk QR Code Generator](https://ultimatetools.io/tools/misc-tools/bulk-qr-code-generator/) — CSV upload, live progress bar, 500 codes per batch, named PNG files in a ZIP — is live at Ultimate Tools. No server, no account required.