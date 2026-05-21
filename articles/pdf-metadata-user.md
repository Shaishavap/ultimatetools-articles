# How to View and Edit PDF Metadata Online for Free (No Upload, No Software)

Every PDF has a hidden layer most people never look at: metadata. Title, Author, Subject, Keywords, Creator, Producer, Creation Date, Modification Date. It's all embedded in the file — and it's often wrong, outdated, or filled in with software defaults you didn't intend to share.

The [PDF Metadata Viewer](https://ultimatetools.io/tools/pdf-tools/pdf-metadata-viewer/) lets you read and edit all of it in the browser. No software. No upload.

## What Is PDF Metadata?

PDF metadata is structured information embedded inside the file, separate from the visible page content. It's defined in the PDF specification and includes fields the document creator filled in (or left as defaults).

The standard fields:

- **Title** — the document's display name (separate from the filename)
- **Author** — who created it
- **Subject** — a short description or category
- **Keywords** — comma-separated terms for search indexing
- **Creator** — which application generated the original document (e.g., "Microsoft Word")
- **Producer** — which PDF engine converted it (e.g., "macOS PDF Engine")
- **Creation Date** — when the file was first created
- **Modification Date** — when it was last saved

## Why Metadata Often Needs Editing

**You exported from Word or Google Docs and the author is wrong.**

Corporate templates and shared documents often retain whoever originally created the template as the embedded author. If you export a proposal that was started by a colleague, their name is in the PDF metadata even if they didn't write the final version.

**Your software left a fingerprint you didn't intend.**

"Creator: Microsoft Word for Microsoft 365" or "Producer: GPL Ghostscript" leaks which tools you used. For professional deliverables, some clients or reviewers notice this. Clearing it or standardizing it is a minor cleanup step.

**You need keywords for discoverability.**

PDFs indexed by search engines or internal document management systems use keywords metadata for categorization. If you're publishing a PDF that should be discoverable, filling in the keywords field meaningfully matters.

**Legal and compliance contexts.**

Some document workflows require specific Author and Title fields to match what's visible in the document. Mismatched metadata fails automated validation checks.

**You're archiving or organizing a document library.**

Batch-editing metadata on a set of PDFs — setting consistent author names, adding subjects, fixing dates — makes them searchable without opening each one.

## How to Use the PDF Metadata Viewer

1. Open [PDF Metadata Viewer](https://ultimatetools.io/tools/pdf-tools/pdf-metadata-viewer/)
2. Upload your PDF — it loads into the browser, no server receives it
3. All metadata fields are read and displayed: Title, Author, Subject, Keywords, Creator, Producer, dates
4. Click any field to edit it directly
5. Download the updated PDF with the new metadata embedded

The changes are applied using [pdf-lib](https://pdf-lib.js.org/) entirely in your browser. The original file is not modified.

## What You Can and Can't Change

**You can edit:**
- Title
- Author
- Subject
- Keywords
- Creator
- Producer

**Date fields** (Creation Date, Modification Date) are displayed but typically managed by the PDF engine on save. When you download the updated PDF, the modification date updates automatically.

**You cannot change via metadata editing:**
- Page content, text, or images — those require a full PDF editor
- Encryption or password protection — use a [PDF Password Protect](https://ultimatetools.io/tools/pdf-tools/protect-pdf/) tool for that

## A Note on Privacy

PDF metadata can contain more than you intended to share. Some fields that appear blank when you open a PDF in a viewer are actually populated in the file. This tool lets you inspect what's actually embedded before you send a document externally.

Running the tool on a file before sharing it — even just to check what's there — is a reasonable step for sensitive documents.

## The Full Tool

The live tool is at [PDF Metadata Viewer → ultimatetools.io](https://ultimatetools.io/tools/pdf-tools/pdf-metadata-viewer/).

Paste any PDF, read what's embedded, edit what needs fixing, download the result. No account, no watermark, no upload to any server.

---

*Part of [Ultimate Tools](https://ultimatetools.io/) — a collection of free, privacy-first browser tools for developers and everyday users.*