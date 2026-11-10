# HTML to Image: Why the Download Doesn't Match the Preview

You paste a card into a free HTML to image converter, the live preview looks exactly right, you hit download — and the file that lands has a different font, no logo, and the bottom row cut off.

That is not a flaky export. In a browser-based HTML to image converter like this one, **the preview and the downloaded file are produced by two different renderers, from two different documents**, so there is a short list of specific, predictable ways they disagree. This is a field guide to that list. I'm using the free [HTML to Image](https://ultimatetools.io/tools/image-tools/html-to-image/) converter on ultimatetools.io as the reference, because it runs entirely in the browser and I can point at exactly what happens at each step — but the same constraints apply to anything built on SVG `foreignObject`, html-to-image and dom-to-image included, which is why those libraries go to the trouble of inlining images and fonts before they serialise. (html2canvas takes a different route — it repaints the DOM onto a canvas itself — so it dodges the XML-parsing failure below and can fetch cross-origin images where CORS allows, at the cost of its own CSS-support gaps.)

## The 60-second version

1. Paste your markup into the **HTML Code** tab — or switch to **Upload HTML** and pick an `.html` or `.htm` file. It is read locally with `FileReader`, dropped into the HTML Code tab, and never sent anywhere.
2. In **Advanced Image Options** (open by default), set **Width** and **Height** to the pixel size you want, or click a Quick Size: Twitter Card 1200x628, OG Image 1200x630, Square 800x800, Story (9:16) 1080x1920. The defaults are 800x400.
3. In the same panel, choose a format — PNG, JPEG or WEBP — a scale (1x, 2x, 3x; the default 2x is real resolution, not upscaling) and a background (White, Transparent or a custom colour).
4. Hit **Generate & Download PNG** — the button is named after whichever format you picked. The file saves as `html-to-image.png` / `.jpeg` / `.webp`.

That is the whole happy path. The rest of this is what to do when the file doesn't match what you saw.

## How HTML to image conversion works in the browser (this explains every quirk)

There is no magic "render HTML to PNG" browser API. What actually happens is this:

```js
// what "convert HTML to image" means in a browser
const svg = `<svg xmlns="http://www.w3.org/2000/svg"
     width="${w * scale}" height="${h * scale}" viewBox="0 0 ${w} ${h}">
  <foreignObject width="100%" height="100%">
    <html xmlns="http://www.w3.org/1999/xhtml">
      <head><style>/* body rules, then your Custom CSS */</style></head>
      <body>${yourHtml}</body>
    </html>
  </foreignObject>
</svg>`;

const img = new Image();
img.src = "data:image/svg+xml;charset=utf-8," + encodeURIComponent(svg);
img.onload = () => {
  ctx.drawImage(img, 0, 0, w * scale, h * scale); // canvas is w*scale x h*scale
  a.href = canvas.toDataURL("image/png");         // or image/jpeg, image/webp
  a.download = "html-to-image.png";
  a.click();
};
```

Your markup goes into an SVG `<foreignObject>`, the SVG is serialised into a `data:` URL, that URL is loaded into an `HTMLImageElement`, the image is painted onto a canvas, and the canvas is exported with `toDataURL()`. Two consequences fall straight out of that:

1. Your HTML is now inside an XML document, and gets parsed as XML.
2. An SVG loaded through `<img>` renders in a static, isolated mode — no scripts, no network requests for subresources.

The live preview never touches that pipeline. It is an `<iframe srcDoc="…">` with `sandbox="allow-same-origin"`, built from its own separate document string. Two renderers, two documents, no shared code path. One thing they do share: none of it leaves the tab. There is no fetch and no API route in the tool at all.

## Eight ways the export differs from the preview — and the fix for each

**1. One unclosed tag kills the whole render.** `<br>`, `<img src="…">` and `<hr>` are fine in HTML and fatal in XML, as is a bare `&` in a query string, a stray `<` in text, or an HTML-only entity like `&nbsp;` or `&copy;` (use the numeric form, `&#160;` / `&#169;`). When the SVG won't parse, `img.onerror` fires and you get *"Couldn't render that HTML to an image — check for external images or fonts it can't reach."* — a slightly misleading message for this particular cause. Fix: self-close void elements (`<br />`, `<img … />`) and escape `&` as `&amp;`. There is no validator in the tool; this is simply what happens to your markup once it is embedded in an XML document, so check it first whenever *nothing at all* renders.

**2. Remote images and web fonts don't come through.** `<img src="https://cdn.example.com/logo.png" />` appears in the preview iframe (a normal document, it can fetch) and is missing from the download (an `<img>`-rendered SVG, it can't). Fix: inline the asset as a `data:` URI so nothing has to be fetched:

```html
<img src="data:image/png;base64,iVBORw0KGgoAAA…" width="64" height="64" alt="" />
```

The [Base64 Encoder & Decoder](https://ultimatetools.io/tools/coding-tools/base64-encoder-decoder/) on the same site has a **File to Base64** button for this. One detail: it hands back the bare base64 string, not the full URI, so you add the `data:image/png;base64,` prefix yourself — match the MIME type to your file.

Same story for `@import`ed stylesheets and Google Fonts links: use a system font stack (`font-family: Arial, Helvetica, sans-serif`) or accept the fallback. The tool says as much in-panel. You will also see this described elsewhere as canvas "tainting"; with a same-origin `data:` URL the honest description is simpler — the asset never loads in the first place. Either way the fix is the same.

**3. JavaScript never runs — in either path.** The export is a static SVG. The preview iframe's sandbox grants `allow-same-origin` and *not* `allow-scripts`. So a chart library, a component that mounts on load, or anything that builds its DOM at runtime renders as an empty box in both places. Fix: hand it the rendered output instead of the source. Open the real page, inspect the node, **Copy → Copy outerHTML**, paste that.

**4. The font changes between preview and export.** This one is specific to how the two documents are built: the exported SVG's body rule ends with `font-family: Arial, sans-serif`, and the preview document's does not. Text with no explicit font renders in the iframe's default — usually a serif — in the preview, and in Arial in the file. Fix: never leave it to defaults. Declare it once in the Custom CSS tab, which is injected into *both* documents:

```css
body { font-family: "Segoe UI", Roboto, Arial, sans-serif; }
```

**5. Anything past the Height value is cropped — and the preview won't warn you.** The SVG's `viewBox` is `0 0 width height` and the foreignObject is 100% of it, while `body` only gets a `min-height`. Content that grows past your Height falls outside the viewBox and is never painted. The preview, being a real scrollable document, shows it happily.

The trap is the obvious-looking diagnostic. The preview pane is a fixed window — under 500px wide on a desktop layout — onto a document whose body is hard-set to your full Width. At the default 800px, and at every one of the four presets, that document is wider than the pane, so **the preview scrolls horizontally almost always, and that tells you nothing about cropping.** Chase it and you'll keep raising Height forever.

Fix: measure, don't eyeball. The preview has no marker for where Height ends — `body` only gets a `min-height`, so it quietly grows to fit whatever you paste. Cheapest check: set Height to something far larger than you need, export once, open the file and look at where the content actually stops, then set Height to that and export again. Or draw the boundary yourself: put `body { height: 630px; outline: 2px dashed red; }` (your Height value) in the Custom CSS tab — the body stops growing, the dashed line sits exactly where the export will crop, and anything hanging below it is what you'd lose. Delete the rule before the real export.

**6. A 16px gutter you never asked for.** Both documents apply `padding: 16px` to the body, so an OG Image export at 1200x630 gives you 1168x598 of usable area — enough to leave a visible 16px gutter around a gradient you meant to run edge-to-edge. Fix: the Custom CSS is interpolated *after* the built-in body rule, so a plain override wins on source order:

```css
body { padding: 0; }
```

**7. Transparent plus JPEG comes out white.** JPEG has no alpha channel, so the background resolves to `#ffffff` and the canvas is pre-filled with white before the draw. The tool flags this with an amber warning rather than letting you discover it in the file. Fix: PNG or WEBP when you need the alpha.

**8. A very large canvas fails — sometimes loudly, sometimes silently.** Scale multiplies the canvas, not the layout, and the result is handed over as a `data:` URL on an anchor. Browsers cap both canvas area and string length, and the two failure shapes are worth separating:

- Where `toDataURL()` genuinely throws — Firefox raises `NS_ERROR_FAILURE` on an oversized canvas — the tool catches it and shows *"Couldn't build the image — try a smaller size, or switch format to PNG."*
- Where the canvas is simply blanked past the cap, `toDataURL()` returns the string `"data:,"`, nothing throws, and the anchor click produces a broken or empty file with **no error banner at all**. That is the behaviour to expect on iOS Safari, which has historically had the lowest ceiling of the majors — around 16.7 megapixels, roughly a 4096x4096 canvas. A Story preset at 1080x1920 with 3x scale is 3240x5760, or 18.7 megapixels: comfortably fine on desktop Chrome, past that iOS limit.

Worth knowing either way: the 2400px maximum on the width and height inputs is an HTML attribute only — the JS clamps the *minimum* (`Math.max(100, …)`), so a typed 5000 goes straight into state and gets multiplied by your scale. Fix: 2x instead of 3x, or smaller dimensions. If a download does nothing at all and no banner appears, assume this rather than a bug in your markup.

## HTML to JPEG, JPG, PNG or WEBP — choose by what's inside the card

Format matters more here than it does for photos, because HTML-to-image output is usually text on flat colour, which is the worst possible input for a lossy codec. The one number to hold in your head: an OG card at 1200x630 with the default 2x scale rasterises to 2400x1260 — roughly 3 megapixels of mostly flat fills and sharp edges, which is exactly the content PNG compresses well and JPEG mangles visibly.

- **PNG** — the default, lossless, keeps the alpha channel. Sharp text edges, hairline borders and flat fills stay exactly as rendered. The quality field is disabled for PNG, and correctly so: `toDataURL("image/png")` is called with no quality argument, so any number you set there would do nothing.
- **JPEG** — lossy DCT plus chroma subsampling, which is precisely what puts ringing and mosquito noise around crisp letterforms and hard colour boundaries. Quality is passed through as `quality / 100`, default 90. Keep it there or higher: the usual photo advice about dropping to 70-75 does not transfer, because artefacts appear on letterform edges long before they'd be noticeable on a photograph. JPEG earns its place when the card is *dominated* by a photo or a big multi-stop gradient, or when whatever you're uploading to refuses PNG.
- **WEBP** — usually the smallest file at comparable quality, and it keeps transparency, so it's the sane default for an image headed straight into a web page. Two caveats: some CMS uploaders still reject the extension, and the tool does no feature detection, so on a browser that can't encode WEBP you would quietly get PNG bytes inside a `.webp` file.

Don't take anyone's file-size table on faith, mine included — export the same card as PNG and as WEBP at 90 and compare the two files. It's two clicks and nothing uploads.

One naming detail if you came here searching for **HTML to JPG**: the download is always `html-to-image.jpeg`, never `.jpg`, and the filename is never derived from the HTML file you uploaded. `.jpg` and `.jpeg` are the same bytes and the same codec, so rename it if a picky uploader insists on the three-letter form.

## The parts that behave better than you'd expect

- **Scale is real resolution, not upscaling.** The `viewBox` stays at your CSS dimensions while the canvas is `width x scale` by `height x scale`, so the layout is composed at 1200x630 and rasterised at 2400x1260. That is genuinely sharper output, not a resampled blur. Default is 2x; the choices are 1x, 2x and 3x.
- **Custom CSS lands in both documents**, which is why it is the right place for anything the preview and the export must agree on — fonts, padding, resets.
- **Four size presets**: Twitter Card 1200x628, OG Image 1200x630, Square 800x800, Story (9:16) 1080x1920. A preset highlights only when both width and height match it, so you can tell at a glance whether you're still on-spec after typing a custom number.

## What it doesn't do (honestly)

- **No JavaScript execution.** If your artwork is built at runtime, this approach cannot help you. Headless Chrome — Puppeteer or Playwright — is the right tool: a real browser that runs scripts, fetches fonts and images over the network, and screenshots a live URL. That's what you want in a pipeline; this is what you want for a one-off with nothing to install.
- **It isn't a URL screenshotter.** It renders markup you paste, not a page you point it at. If the thing you want is already on screen in Chrome, DevTools → right-click the node → **Capture node screenshot** is faster, and it does run JS.
- **Tailwind and other utility classes don't resolve.** There's no build step and no stylesheet, so `class="p-4 rounded-xl"` renders as unstyled markup. Write the equivalent real CSS in the Custom CSS tab, or use inline `style` attributes.
- **No preview of the rasterised result.** The file goes straight to download; you find out by opening it.
- **Scale is fixed at 1x/2x/3x** — no free-form multiplier.
- **One card at a time.** No templating, no batch, no data merge. A hundred OG images with different titles needs a script and a headless browser, not a form.

## Related Tools

- [convert an image file to base64 online free](https://ultimatetools.io/tools/coding-tools/base64-encoder-decoder/) — the fix for failure mode 2: it hands you the base64 payload, you prepend `data:image/png;base64,`, and the renderer never has to fetch anything
- [convert HTML to PDF free with selectable text](https://ultimatetools.io/tools/pdf-tools/html-to-pdf/) — when the output needs real text and page sizes instead of pixels on a canvas
- [compress PNG and WEBP images free in your browser](https://ultimatetools.io/tools/image-tools/image-compressor/) — a 3x export is a large file; shrink it before it goes on a page

Paste your markup, set Width and Height to the size you actually want, and pull the file — free, nothing uploads: [HTML to Image Converter](https://ultimatetools.io/tools/image-tools/html-to-image/)