# Building a Client-Side PDF Compressor with pdfjs-dist and pdf-lib

We needed a PDF compressor for [Ultimate Tools](https://ultimatetools.io/) that would run entirely in the browser — no server uploads, no file size limits, no privacy concerns. Here's how we built it using two complementary libraries: `pdfjs-dist` for rendering and `pdf-lib` for assembly.

---

## The Core Idea

The compression strategy is straightforward:

1. Render each PDF page to a Canvas at a controlled resolution
2. Export the Canvas as a JPEG with a quality parameter
3. Embed those JPEGs into a brand-new PDF

This approach replaces vector graphics, embedded fonts, metadata, and uncompressed images with compressed raster images. The trade-off is that text becomes non-selectable in the output — acceptable for most use cases like emailing, printing, or archiving.

---

## Setting Up pdfjs-dist

`pdfjs-dist` is Mozilla's PDF rendering library. It needs a web worker for parsing, which we load from a CDN:

```typescript
import type * as PdfjsLibType from "pdfjs-dist";

const pdfjsRef = useRef<typeof PdfjsLibType | null>(null);

useEffect(() => {
    import("pdfjs-dist").then((lib) => {
        lib.GlobalWorkerOptions.workerSrc =
            `https://unpkg.com/pdfjs-dist@${lib.version}/build/pdf.worker.min.mjs`;
        pdfjsRef.current = lib;
    });
}, []);
```

Two things to note:

- **Dynamic import**: We lazy-load `pdfjs-dist` because it's a large library (~400KB). No need to block initial page load.
- **Version-matched worker**: The worker URL includes `lib.version` so we never get a version mismatch between the main library and its worker.

If you're using pdfjs-dist v5.x with webpack (Next.js), you may hit a webpack 5 compatibility issue. We covered that fix in a [previous article](https://dev.to/shaishav_patel_271fdcd61a/how-we-moved-pdf-processing-from-server-to-browser-and-killed-oom-crashes-ekm).

---

## Loading the PDF

When the user drops a file, we read it into an ArrayBuffer and hand it to pdfjs:

```typescript
const onDrop = useCallback(async (acceptedFiles: File[]) => {
    const f = acceptedFiles[0];
    if (!f) return;

    setFile(f);
    const pdfjs = pdfjsRef.current;
    if (!pdfjs) { alert("PDF library not ready yet"); return; }

    const arrayBuffer = await f.arrayBuffer();
    const loadingTask = pdfjs.getDocument({ data: arrayBuffer });
    const pdf = await loadingTask.promise;

    setPdfDoc(pdf);
    setPageCount(pdf.numPages);
}, []);
```

The `PDFDocumentProxy` object (`pdf`) gives us `numPages` and a `getPage(i)` method. Pages are 1-indexed.

---

## The Compression Presets

We defined three presets that control two parameters — JPEG quality and rendering scale:

```typescript
const presets = [
    { id: "extreme",     quality: 0.3,  scale: 0.75 },
    { id: "recommended", quality: 0.6,  scale: 1.5  },
    { id: "light",       quality: 0.85, scale: 2.0  },
];
```

**Quality** (0.1 to 1.0): Controls JPEG compression. Lower = smaller files, fuzzier text. At 0.6 the text is still readable; at 0.3 it gets noticeably soft.

**Scale** (0.5x to 3.0x): Controls the Canvas rendering resolution. A higher scale means more pixels captured before JPEG compression, which preserves more detail. But it also means larger intermediate images and more processing time.

The interaction between these two parameters is what makes the presets work:

- **Extreme** (0.3 quality, 0.75x scale): Small canvas + aggressive JPEG compression = maximum shrinkage
- **Recommended** (0.6 quality, 1.5x scale): Larger canvas compensates for moderate JPEG compression = good balance
- **Light** (0.85 quality, 2.0x scale): Large canvas + gentle JPEG compression = minimal quality loss

Users can also go fully custom with range sliders.

---

## The Compression Loop

This is the core of the compressor — a sequential loop over every page:

```typescript
const compressPdf = async () => {
    if (!pdfDoc || !pdfjsRef.current) return;

    const newPdfDoc = await PDFDocument.create();

    for (let i = 1; i <= pdfDoc.numPages; i++) {
        setProgress(`Processing page ${i} of ${pdfDoc.numPages}...`);

        // 1. Get page and calculate dimensions at target scale
        const page = await pdfDoc.getPage(i);
        const viewport = page.getViewport({ scale });

        // 2. Create a canvas and render the page to it
        const canvas = document.createElement("canvas");
        const context = canvas.getContext("2d")!;
        canvas.width = viewport.width;
        canvas.height = viewport.height;

        await page.render({
            canvasContext: context,
            viewport: viewport,
        }).promise;

        // 3. Export canvas as JPEG with quality parameter
        const imgDataUrl = canvas.toDataURL("image/jpeg", quality);

        // 4. Embed JPEG into the new PDF
        const imgBytes = await fetch(imgDataUrl)
            .then((res) => res.arrayBuffer());
        const jpgImage = await newPdfDoc.embedJpg(imgBytes);

        // 5. Add a page with exact same dimensions and draw the image
        const newPage = newPdfDoc.addPage([viewport.width, viewport.height]);
        newPage.drawImage(jpgImage, {
            x: 0, y: 0,
            width: viewport.width,
            height: viewport.height,
        });
    }

    // 6. Save and create download URL
    const pdfBytes = await newPdfDoc.save();
    const blob = new Blob([pdfBytes], { type: "application/pdf" });
    const url = URL.createObjectURL(blob);

    setCompressedPdfUrl(url);
    setCompressedSize(blob.size);
};
```

Let me break down why each step matters:

### Step 1-2: Rendering to Canvas

`getViewport({ scale })` returns the page dimensions multiplied by the scale factor. A standard A4 page at 72 DPI is about 595x842 points. At scale 1.5x, the canvas becomes 893x1263 pixels.

We create a fresh canvas per page. This is important for memory — reusing one canvas and just resizing it works too, but creating fresh ones avoids stale pixel data and lets the browser GC the old one.

### Step 3: The JPEG trick

`canvas.toDataURL("image/jpeg", quality)` is where the magic happens. The quality parameter (0 to 1) controls the JPEG compression ratio. This single API call handles all the DCT compression, chroma subsampling, and Huffman encoding.

Why JPEG and not PNG? Because JPEG's lossy compression is dramatically more effective for raster content. A page rendered at 1.5x scale as PNG might be 3MB. The same page as JPEG at 60% quality might be 150KB.

### Step 4: The data URL to ArrayBuffer round-trip

`fetch(imgDataUrl).then(res => res.arrayBuffer())` — this looks odd, but it's the cleanest way to convert a data URL to an ArrayBuffer that `pdf-lib` can consume. The alternative is manual base64 decoding, which is more code for no benefit.

### Step 5: Preserving page dimensions

`newPdfDoc.addPage([viewport.width, viewport.height])` creates a page that exactly matches the rendered canvas. The image is drawn from corner to corner, so the output looks identical to the original — just compressed.

---

## Showing the Results

After compression, we calculate and display the size reduction:

```typescript
const formatBytes = (bytes: number, decimals = 2) => {
    if (bytes === 0) return '0 Bytes';
    const k = 1024;
    const sizes = ['Bytes', 'KB', 'MB', 'GB'];
    const i = Math.floor(Math.log(bytes) / Math.log(k));
    return parseFloat((bytes / Math.pow(k, i)).toFixed(decimals))
        + ' ' + sizes[i];
};

const calculateReduction = () => {
    if (!file || compressedSize === 0) return 0;
    return ((file.size - compressedSize) / file.size) * 100;
};
```

Users see the original size, compressed size, and percentage reduction. This immediate feedback helps them decide whether to adjust the presets and compress again.

---

## Why Not Use a Server?

We deliberately avoided server-side compression. Here's why:

1. **Privacy**: PDFs contain sensitive data — contracts, medical records, financials. Users shouldn't have to trust a third party.
2. **No file size limit**: The limit is the user's device memory, not a server upload cap.
3. **Zero latency**: No upload/download time. A 20MB PDF on a slow connection would take 30+ seconds just to upload.
4. **No infrastructure cost**: No storage, no compute, no cleanup jobs for temp files.

The trade-off is that very large PDFs (100+ pages) can be slow on older devices. But for the 95% use case — compressing a 5-30 page document — browser performance is fine.

---

## Limitations and Honest Trade-offs

This approach isn't perfect. Be upfront with users about what they lose:

- **Text becomes non-selectable**: The output is essentially a JPEG-per-page PDF. Text search, copy-paste, and accessibility are gone.
- **Vector graphics are rasterized**: If the original PDF has crisp vector diagrams, they'll become pixel-based after compression.
- **Processing is sequential**: Each page is rendered one at a time. A 50-page document at 2.0x scale takes noticeable time.
- **Memory usage**: At high scale values, the Canvas can get large. A single A4 page at 3.0x scale creates a ~5000x7000 pixel canvas.

For use cases where selectable text matters, consider server-side tools like `ghostscript` or `qpdf` that can compress without rasterizing. But for a browser-only, zero-upload tool — the canvas-to-JPEG approach is hard to beat.

---

## Try It

The tool is live at [Compress PDF](https://ultimatetools.io/tools/pdf-tools/compress-pdf/).

It's part of [Ultimate Tools](https://ultimatetools.io/) — a free, privacy-first collection of 24+ browser-based utilities for PDFs, images, QR codes, and developer tools. All processing happens client-side.

Source architecture: Next.js App Router, `pdfjs-dist` v5.x for rendering, `pdf-lib` for PDF assembly, `react-dropzone` for file handling. No server APIs involved in the compression flow.