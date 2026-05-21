# How to Compress Images for Email — Reduce File Size Without Losing Quality

Email clients have attachment size limits. Gmail caps at 25 MB. Outlook at 20 MB. Most corporate mail servers at 10 MB. A batch of uncompressed photos from a modern phone — each 4–8 MB — hits those limits fast.

More importantly: large images slow down email loading on mobile, get flagged as spam more often, and fill recipients' inboxes faster than necessary. Compressing images before attaching them is good practice for any image going over email.

Compress any image free: [Image Compressor — Free, No Upload](https://ultimatetools.io/tools/image-tools/image-compressor/)

---

## Target File Sizes for Email

| Image type | Target size |
|---|---|
| Single photo attachment | Under 500 KB |
| Multiple photos (batch) | Under 300 KB each |
| Inline image in email body | Under 100 KB |
| Product / listing photo | Under 200 KB |
| Profile or headshot | Under 100 KB |

These targets keep individual emails under 5 MB even with multiple attachments, which avoids delivery failures and keeps loading fast on mobile connections.

---

## Step 1 — Resize First

The biggest file size gains come from resizing, not compression. A 4000×3000px photo at 300 DPI contains 12 million pixels. For an email attachment, 1200px wide is more than sufficient — screens rarely display email attachments larger than that.

**Quick resize targets for email:**
- General photo: 1200px wide max
- Profile photo / headshot: 600px wide
- Inline body image: 600–800px wide

Resize before compressing. A smaller image compresses better and the combined reduction is significantly larger than either step alone.

→ [Image Resizer — Free, No Upload](https://ultimatetools.io/tools/image-tools/image-resizer/)

---

## Step 2 — Choose the Right Format

**JPG** — use for photos. Lossy compression keeps file sizes small. The right format for nearly all email photo attachments.

**PNG** — use for screenshots, logos, and graphics with text or sharp edges. Lossless but larger than JPG for photographic content. Don't convert photos to PNG — it makes them bigger.

**WebP** — 25–35% smaller than JPG at equivalent quality. Supported in modern email clients and browsers, but some older email clients may not display WebP inline. Safe for attachments, use with caution for inline images.

**Avoid GIF for photos** — low quality and large file size. Only use GIF for short animations where no better option exists.

---

## Step 3 — Compress

After resizing to the right dimensions:

**For JPG:** Quality 75–80% hits the sweet spot for email. At 75%, the file is typically 60–70% smaller than the original with no visible quality loss on a screen. Below 70%, compression artefacts become visible on close inspection.

**For PNG:** Lossless compression removes metadata and optimises file structure without affecting pixel quality. Use maximum compression — there's no quality tradeoff.

→ [Image Compressor — Free, No Upload](https://ultimatetools.io/tools/image-tools/image-compressor/)

---

## Step 4 — Strip Metadata

Photos from phones and cameras embed EXIF metadata — GPS location, camera model, shooting settings, timestamps. This adds 20–50 KB per image and is invisible to the recipient.

For email, you almost always want to strip metadata:
- It removes location data (privacy)
- It reduces file size
- It removes information the recipient doesn't need

The image compressor strips EXIF metadata automatically during compression.

---

## The Full Workflow

1. **Resize** to 1200px wide (or smaller for headshots/inline images)
2. **Convert to JPG** if the image is a photo in PNG format
3. **Compress** at 75–80% quality
4. **Verify size** — right-click the file → Properties → check it's under your target

Total time: under 60 seconds per image. For a batch of 10 photos, the three tools handle each image in sequence.

---

## Quick Wins If You're in a Hurry

**Just compress — skip the resize:** If you're sending one photo and it's already a reasonable size, a single compression step at 75% quality will cut 50–60% of the file size without any resizing.

**Use the compressor's ZIP output:** The image compressor outputs multiple compressed images as a ZIP file. Attach the ZIP directly — recipients get all images in one attachment.

---

## All Three Tools, Free, No Upload

- [Image Resizer](https://ultimatetools.io/tools/image-tools/image-resizer/) — resize to exact pixel dimensions
- [Image Compressor](https://ultimatetools.io/tools/image-tools/image-compressor/) — compress JPG/PNG/WebP, strip metadata
- [Image Converter](https://ultimatetools.io/tools/image-tools/image-converter/) — convert PNG photos to JPG before compressing

All run in your browser. Your images are never uploaded to any server.