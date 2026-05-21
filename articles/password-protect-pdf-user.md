# How to Password Protect a PDF Without Adobe Acrobat

Adobe Acrobat is the standard tool for password-protecting PDFs, but it costs $19.99/month. For occasional use — protecting a contract before emailing it, locking a financial document — that subscription doesn't make sense.

Here are the free options, what they actually do, and when to use each.

---

## Option 1: Browser-based PDF password protection (no upload)

The [PDF Password Protect tool](https://ultimatetools.io/tools/pdf-tools/pdf-password-protect/) at Ultimate Tools applies password encryption entirely in your browser. The PDF never leaves your device.

**How it works:**
1. Open the tool
2. Drop your PDF file
3. Enter a password (and confirm it)
4. Click "Protect PDF"
5. Download the encrypted file

The output PDF requires the password to open. Works in any PDF reader — Adobe Reader, Preview on Mac, PDF.js in browsers, mobile PDF apps.

**What kind of encryption?** The tool uses AES-128 encryption (standard for PDF password protection). This is the same algorithm Adobe Acrobat uses by default for its "Document Open Password" option.

**When to use this:** Any time you need to email a sensitive document. Contracts, NDAs, tax documents, financial statements. The file never touches a third-party server, which matters when the document contains personal or confidential data.

---

## Option 2: Google Chrome / Edge print-to-PDF trick

Chrome and Edge can't add passwords, but this workaround works for some use cases:

1. Open the PDF in Chrome
2. Press `Ctrl+P` (Windows) or `Cmd+P` (Mac)
3. Choose "Save as PDF"
4. In some operating systems, you can add a password in the print dialog

This is limited — macOS Preview is actually better for this use case.

---

## Option 3: macOS Preview (free, built-in)

If you're on a Mac:

1. Open the PDF in Preview
2. File → Export as PDF
3. Click "Show Details" in the export dialog
4. Check "Encrypt"
5. Enter a password
6. Save

Preview uses AES-128 encryption. The protected PDF opens normally in any PDF reader.

---

## Option 4: LibreOffice (free, cross-platform)

LibreOffice can open and re-export PDFs with password protection:

1. Open LibreOffice Draw
2. File → Open → select your PDF
3. File → Export as PDF
4. In the PDF Options dialog → Security tab
5. Set "Open Password"
6. Export

LibreOffice is free and open source. The export may slightly reformat the PDF depending on its original content.

---

## What about online tools that upload your file?

Many sites offer PDF password protection but require uploading your file. For a non-sensitive document (a public-facing brochure, a restaurant menu), that's fine. For financial records, legal documents, or anything with personal data, uploading to an unknown server creates unnecessary risk.

The [browser-based option](https://ultimatetools.io/tools/pdf-tools/pdf-password-protect/) processes everything locally, so this is a non-issue.

---

## Can the password be removed?

Yes, by the person who knows the password. Document Open Passwords (the kind that prevents opening the file) can also be removed by specialized tools if the password is weak or common. Use a strong, unique password for documents that genuinely need to stay private.

---

## Quick comparison

| Method | Cost | Upload required | Platform |
|---|---|---|---|
| Ultimate Tools (browser) | Free | ❌ Never | Any browser |
| macOS Preview | Free | ❌ Local | Mac only |
| LibreOffice | Free | ❌ Local | Windows, Mac, Linux |
| Adobe Acrobat | $19.99/mo | ❌ Local | Windows, Mac |
| Most online tools | Free/paid | ✅ Yes | Any browser |

For a free, cross-platform option that doesn't require a file upload, the [PDF Password Protect tool](https://ultimatetools.io/tools/pdf-tools/pdf-password-protect/) is the quickest path.