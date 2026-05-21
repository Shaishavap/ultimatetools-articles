# How to Split a PDF Online for Free (No Software, No Upload, No Signup)

You have a 40-page PDF. You need pages 1–10 as one file and pages 20–30 as another. Or you just want to pull out a single page.

Most ways to do this involve installing desktop software, paying for Adobe Acrobat, or uploading the file to a cloud service. None of those are necessary. Here's how to split a PDF for free, directly in your browser.

## Option 1: Extract a Range of Pages

The [Split PDF](https://ultimatetools.io/tools/pdf-tools/split-pdf/) tool lets you extract any range of pages:

1. Open the tool and upload your PDF
2. In the **Pages** tab, enter the page range you want (e.g., `1-10`, `5`, `12-20`)
3. Click **Extract** — a new PDF containing only those pages downloads instantly

The original file is unchanged. Nothing is uploaded to a server.

## Option 2: Split Into Fixed-Size Chunks

If you need to break a large PDF into equal parts — say, every 10 pages — use the chunk split:

1. Upload your PDF
2. Set the chunk size (e.g., 10 pages per file)
3. Download a ZIP containing all the resulting PDFs

Useful for distributing chapters, splitting reports into sections, or meeting file size limits.

## Option 3: Extract Every Page as a Separate File

Need each page as its own PDF?

1. Upload your PDF
2. Choose "Extract all pages"
3. Download a ZIP with one PDF per page

Common use case: splitting a scanned multi-page document into individual files for separate archiving.

## Why Not Just Use Adobe Acrobat?

Adobe Acrobat can split PDFs. But it costs $20–$25/month for the subscription that includes this feature. For something as simple as extracting a page range from a file you already have, that's a lot.

Free alternatives at a glance:

| Tool | Account required | File uploaded | Daily limit |
|---|---|---|---|
| Adobe Acrobat (free) | Yes | Yes | Yes |
| Smallpdf / ILovePDF | Yes | Yes | Yes |
| [Split PDF → ultimatetools.io](https://ultimatetools.io/tools/pdf-tools/split-pdf/) | No | No | None |

## Does Splitting Affect PDF Quality?

No. Splitting extracts the original PDF pages as-is. Text sharpness, image resolution, vector graphics, and embedded fonts are all preserved exactly as they were in the source file. There's no re-rendering or compression step.

The only exception: if you extract a page that contains cross-page elements (like a table that spans two pages), each page is still extracted independently — the cross-page relationship doesn't carry over. But the content on each extracted page is identical to the original.

## What About Password-Protected PDFs?

Password-protected PDFs cannot be split without removing the password first. If you have the owner password, you can unlock the PDF and then split it. The [PDF Password Protect](https://ultimatetools.io/tools/pdf-tools/protect-pdf/) tool in Ultimate Tools also handles unlocking.

## How It Works Under the Hood

The splitting is done entirely client-side using [pdf-lib](https://pdf-lib.js.org/) — a JavaScript library that reads and writes PDFs in the browser without any server involvement.

For a page range extraction:

```javascript
const srcDoc = await PDFDocument.load(pdfBytes)
const newDoc = await PDFDocument.create()
const [copiedPage] = await newDoc.copyPages(srcDoc, [pageIndex])
newDoc.addPage(copiedPage)
const newBytes = await newDoc.save()
```

The original file stays in your browser's memory throughout. Nothing is transmitted. The downloaded file is generated locally and saved directly to your device.

## Privacy

This matters especially for sensitive documents — legal filings, medical records, financial statements. You don't need to send those to a third-party server just to extract a few pages. The file never leaves your device at any point during the split.

## Try It

The full tool is live at [Split PDF → ultimatetools.io](https://ultimatetools.io/tools/pdf-tools/split-pdf/).

Upload your PDF, pick your pages, and download the result in seconds. No account, no upload, no limit.

---

*Part of [Ultimate Tools](https://ultimatetools.io/) — a collection of free, privacy-first browser tools for developers and everyday users.*