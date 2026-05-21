# How to Compress a PDF Without Losing Quality — 4 Free Methods

PDF files get large fast — especially scanned documents, PDFs with embedded images, or multi-page reports exported from design tools. Email attachment limits, upload portals with size caps, and slow file transfers all become problems.

Here are four free methods to compress a PDF, what each one actually does, and when to use which.

---

## Method 1: Browser-based compression (no upload)

The [PDF Compressor](https://ultimatetools.io/tools/pdf-tools/compress-pdf/) at Ultimate Tools compresses your PDF entirely in your browser. The file never leaves your device.

**How it works:**
1. Open the tool
2. Drop your PDF file
3. Choose compression level (light, medium, strong)
4. Click Compress
5. Download the compressed file

The browser-side processing uses pdf-lib to re-encode the document. Image-heavy PDFs see the largest reductions (50–80% file size reduction is common). Text-only PDFs see smaller reductions since text data compresses well even in the original.

**When to use this:** Any PDF with sensitive content (contracts, financial documents, medical records). The file never touches a server.

---

## Method 2: Adobe Acrobat (free web version)

Adobe offers a free PDF compressor at the Adobe Acrobat web tool. It requires signing in with an Adobe ID (or Google/Apple account), but the compression itself is free for occasional use.

The Adobe tool uses server-side processing — your file is uploaded, compressed, and downloaded back. Quality tends to be good because Adobe's compression engine is mature.

**When to use this:** When you don't have privacy concerns and want a well-known brand's compression engine.

---

## Method 3: macOS Preview (built-in, free)

On a Mac, you can reduce PDF file size using Preview's "Quartz Filter":

1. Open the PDF in Preview
2. File → Export
3. Quartz Filter → Reduce File Size
4. Export

The Reduce File Size filter aggressively compresses images, sometimes too aggressively. The result can look pixelated for image-heavy PDFs. Best for text-heavy documents where image quality is less critical.

**When to use this:** Quick compression on a Mac without any additional tools. Acceptable for internal documents where exact image quality doesn't matter.

---

## Method 4: Ghostscript (command line, free)

For developers or power users who want maximum control:

```bash
gs -sDEVICE=pdfwrite -dCompatibilityLevel=1.4 \
   -dPDFSETTINGS=/screen \
   -dNOPAUSE -dQUIET -dBATCH \
   -sOutputFile=compressed.pdf input.pdf
```

`-dPDFSETTINGS` controls quality:
- `/screen` — lowest quality, smallest size (72 DPI images)
- `/ebook` — medium quality (150 DPI images)
- `/printer` — high quality (300 DPI images)
- `/prepress` — highest quality (300+ DPI, color profiles preserved)

Ghostscript is the most powerful option — it rebuilds the PDF from scratch and can achieve much higher compression ratios than web tools.

**When to use this:** Batch processing multiple PDFs, scripted automation, or when you need fine-grained control over output quality.

---

## Why some PDFs compress more than others

**Image-heavy PDFs** — Large JPEG or PNG images embedded in a PDF compress dramatically. A scanned document that's 20MB often compresses to 4–5MB.

**Text-only PDFs** — Text in PDFs is stored as vector data or fonts, which already compresses well. A 2MB text PDF might only reduce to 1.7MB.

**Already-compressed PDFs** — PDFs that were previously exported at screen resolution or already optimized won't compress further. You may see 0% or even slight size increase from recompression.

**Encrypted PDFs** — Password-protected PDFs can't be recompressed without first removing the password.

---

## Quick comparison

| Method | Cost | Upload required | Platform |
|---|---|---|---|
| Ultimate Tools (browser) | Free | ❌ Never | Any browser |
| Adobe Acrobat web | Free (login req.) | ✅ Yes | Any browser |
| macOS Preview | Free | ❌ Local | Mac only |
| Ghostscript | Free | ❌ Local | Any OS |

For a free, no-upload option that works in any browser, the [PDF Compressor](https://ultimatetools.io/tools/pdf-tools/compress-pdf/) is the quickest path — drop the file, download, done.