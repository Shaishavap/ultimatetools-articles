# Convert HEIC to JPG Free — No Upload, Your iPhone Photos Stay on Your Device

Someone AirDrops you a photo, or you copy your iPhone's camera roll to a Windows PC, and the files won't open. They're **HEIC** — Apple's default photo format since iOS 11. Windows wants you to buy a codec from the Microsoft Store to view them, half of Android handles them inconsistently, and upload forms simply reject them. The fix is converting them to JPG — but there's a catch with most converters that's worth pausing on before you use one.

## The catch: most HEIC converters upload your photos

Type "heic to jpg" into a search engine and the top results all work the same way: you upload your photos to their server, they convert them, you download the result. For a screenshot of a spreadsheet, fine. But camera-roll photos are among the most personal files you own — kids, home, documents, whiteboards from work. Sending them to an anonymous server, where you have no idea how long they're retained, is a real trade you're making for a format change.

It's also unnecessary. Browsers can run the HEIC decoder themselves (as WebAssembly), which means the conversion can happen **entirely on your device** — the photo never leaves your machine.

## Convert HEIC to JPG in the browser — nothing uploaded

The [free HEIC to JPG converter](https://ultimatetools.io/tools/image-tools/heic-to-jpg/) does exactly that:

- **Drop in your .heic file** (or .heif — both work, including files that Windows reports with no file type)
- **The browser decodes it on your device** — you'll see a brief "Decoding iPhone photo…" moment the first time while the decoder loads, then it's instant for the rest
- **JPG is pre-selected** — download with one click. PNG and WebP are there too if you need lossless output or the smallest web format
- **Nothing is uploaded** — the decoding runs in your browser, so it works the same whether the photo is a sunset or a signed contract

No signup, no watermark, no file-count limit.

## Converting a whole camera roll at once

If you've just moved months of photos off an iPhone, converting one at a time is not a plan. Switch to the **Batch Convert** tab, drop in all the HEIC files together, and download every converted JPG in a single ZIP. The decoder loads once and chews through the queue file by file — a few hundred photos is a coffee-break job, and still nothing leaves your device.

## Will the quality drop?

Practically, no. The conversion exports at 92% JPEG quality, which is visually indistinguishable for sharing, printing, and editing. One honest caveat: newer iPhones capture HDR (10-bit) photos, and JPG is an 8-bit format — extremely bright highlights get tone-mapped down, which in rare cases shifts them very slightly. If you need a lossless copy, choose PNG output instead of JPG from the same tool.

Live Photos behave the way you'd want: the still image converts to JPG; the 3-second motion clip is a separate video stream and isn't part of the output.

## Why not just change the iPhone setting?

You can — Settings → Camera → Formats → "Most Compatible" makes the iPhone shoot JPG directly. Two reasons people don't: HEIC files are roughly **half the size** of JPG at the same quality, so the setting costs you storage on every photo you ever take. And it does nothing for the thousands of HEIC photos you already have. Keep the efficient format on the phone; convert on the rare occasion something else needs a JPG.

## Related Tools

- [compress a photo without losing visible quality](https://ultimatetools.io/tools/image-tools/image-compressor/) — shrink the converted JPGs further before emailing a batch or uploading to a size-capped form
- [resize an image to exact pixel dimensions online](https://ultimatetools.io/tools/image-tools/image-resizer/) — scale a converted photo to a platform's required size for profiles, listings, or printing
- [convert images between JPG, PNG, and WebP free](https://ultimatetools.io/tools/image-tools/image-converter/) — the full converter, for when the input isn't HEIC but the format still needs to change

Your photos shouldn't have to visit a stranger's server to change file extension. **[Convert HEIC to JPG free →](https://ultimatetools.io/tools/image-tools/heic-to-jpg/)** — drop in iPhone photos, single or batch, decoded on your device, nothing uploaded.