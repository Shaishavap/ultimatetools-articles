# How to Optimize Images for Web — Reduce File Size, Pick the Right Format, Set the Right Dimensions

Images are the biggest contributor to page weight on most websites. A single unoptimized hero image can add 3–5 MB to a page load. On a slow connection, that's the difference between a page that loads in 2 seconds and one that takes 12.

Here's the full checklist — format, dimensions, compression — with free tools to do each step in your browser.

---

## Step 1 — Resize to the Display Dimensions

The most common mistake: uploading a 4000×3000px photo when the maximum display width is 1200px. The browser downloads the full 4000px image and scales it down with CSS. You're sending 3× more data than needed.

**Rule:** Image pixel dimensions should match (or be close to) the largest size they'll appear on screen.

Common target dimensions:
| Use case | Target width |
|---|---|
| Full-width hero image | 1440–1920px |
| Blog post body image | 800–1200px |
| Open Graph / social preview | 1200×628px |
| Product thumbnail | 400–600px |
| Avatar / profile photo | 200–400px |

Resize before compressing — a smaller image compresses better.

→ [Image Resizer — Free, No Upload](https://ultimatetools.io/tools/image-tools/image-resizer/)

---

## Step 2 — Choose the Right Format

**JPG** — use for photographs and images with gradients. Lossy compression produces small files with acceptable quality. Bad for images with text, sharp edges, or transparency.

**PNG** — use for screenshots, logos, icons, and images that need transparency. Lossless compression means larger files than JPG for the same image. Use for UI assets where sharpness matters.

**WebP** — use for everything when browser support isn't a concern (it's now supported by all modern browsers). WebP is 25–35% smaller than JPG at equivalent quality, and supports transparency like PNG. The best default for web.

**GIF** — avoid. Use WebP or video for animations. GIFs are large and low quality.

**Practical rule:**
- Photos → WebP (fallback: JPG)
- UI assets with transparency → WebP (fallback: PNG)
- Legacy browser support needed → JPG or PNG

→ [Image Converter — JPG, PNG, WebP](https://ultimatetools.io/tools/image-tools/image-converter/)

---

## Step 3 — Compress

After resizing and choosing the format, compress to remove unnecessary data.

For JPG: quality 75–85% is the sweet spot. Below 70%, visible artefacts appear. Above 85%, file size increases without visible quality improvement.

For PNG: lossless compression removes metadata and optimizes the file structure without affecting pixel data.

For WebP: quality 80% gives excellent results with files 30–40% smaller than equivalent JPG.

→ [Image Compressor — Free, No Upload](https://ultimatetools.io/tools/image-tools/image-compressor/)

---

## The Correct Order

1. **Resize first** — get to target dimensions before compressing
2. **Convert format** — switch to WebP if you haven't already
3. **Compress last** — apply compression to the already-resized, correctly-formatted image

Compressing first and then resizing is wasteful — you compress data you'll throw away when you resize.

---

## Target File Sizes

| Image type | Target size |
|---|---|
| Hero / banner | Under 200 KB |
| Blog post image | Under 100 KB |
| Thumbnail | Under 30 KB |
| Icon / avatar | Under 10 KB |
| Open Graph image | Under 200 KB |

These aren't hard rules, but pages that keep images in these ranges load fast on both fast and slow connections.

---

## Quick Wins for Existing Sites

**Audit your largest images** — in Chrome DevTools, open the Network tab, filter by Img, sort by size. Your worst offenders will be immediately visible.

**Strip metadata** — photos from cameras and phones embed EXIF data (GPS location, camera model, shooting settings). This adds 10–50 KB per image and is invisible to users. The image compressor strips it automatically.

**Serve scaled images** — use the `srcset` attribute in HTML to serve different sizes to different screen widths. Mobile users get a 600px image, desktop users get 1200px.

```html
<img
  src="hero-1200.webp"
  srcset="hero-600.webp 600w, hero-1200.webp 1200w, hero-1920.webp 1920w"
  sizes="100vw"
  alt="Hero image"
/>
```

---

## All Three Tools, Free, No Upload

- [Image Resizer](https://ultimatetools.io/tools/image-tools/image-resizer/) — resize to exact pixel dimensions
- [Image Converter](https://ultimatetools.io/tools/image-tools/image-converter/) — convert to WebP, JPG, or PNG
- [Image Compressor](https://ultimatetools.io/tools/image-tools/image-compressor/) — compress with quality control, strip metadata

All run in your browser. No file is uploaded to any server.