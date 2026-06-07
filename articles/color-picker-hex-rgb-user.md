# HEX to RGB, HSL & CMYK in One Click — Free Color Picker & Converter

You have a `#2563EB` in the design file and the canvas API wants `rgb(37, 99, 235)`. Or the brand guide is in CMYK and your CSS needs HEX. Converting a color between formats is a tiny, constant tax on front-end and design work — and most of the time you just want to **convert HEX to RGB** (or the other way) without opening a heavyweight design app. Here's how the formats relate, and the fastest way to move between them.

## Why one color has so many formats

A single color can be written several ways, each suited to a different job:

- **HEX** — `#2563EB`. Six hexadecimal digits, two each for red, green, blue. The default in CSS and HTML.
- **RGB** — `rgb(37, 99, 235)`. The same three channels as 0–255 values. What canvas, image data, and many APIs expect.
- **HSL** — `hsl(221, 83%, 53%)`. Hue, saturation, lightness. The easiest format for *adjusting* a color — nudge lightness up for a tint, down for a shade — and it's native in CSS.
- **HSV / HSB** — hue, saturation, value. The model most design-tool color pickers actually use internally.
- **CMYK** — cyan, magenta, yellow, key (black). The subtractive model for print.

They all describe the same point in color space; they're just different coordinate systems. The friction is only in the translation.

## The fastest way: pick and convert in the browser

For a one-off conversion — or honestly most of them — a browser tool beats doing the hex math by hand or firing up Photoshop. The [free color picker and converter](https://ultimatetools.io/tools/coding-tools/color-picker/) shows every format at once:

- **Pick a color** with the native picker, drag the RGB sliders, or paste a HEX code
- **Read HEX, RGB, HSL, HSV, and CMYK** updating live as you go
- **Copy any format** with one click, ready to paste into CSS or a design ticket
- **Runs entirely in your browser** — nothing is uploaded

Paste `#2563EB`, read `rgb(37, 99, 235)` and `hsl(221, 83%, 53%)`, copy the one you need. That's the whole loop.

## HEX to RGB (and back), quickly

If you ever need to do it by hand: a HEX code is just three base-16 numbers. `#2563EB` splits into `25`, `63`, `EB` → convert each from hexadecimal to decimal → `37`, `99`, `235`. Going back, take each 0–255 channel, convert to a two-digit hex value, and join them. It's not hard, but it's exactly the kind of thing you don't want to do five times a day — which is why the converter shows both directions instantly.

## Don't skip the contrast check

Picking a nice color is half the job; making sure text on it is *readable* is the other half. The [WCAG contrast ratio](https://ultimatetools.io/tools/coding-tools/color-picker/) measures how distinct two colors are in luminance. The thresholds:

- **AA** needs **4.5:1** for normal text (3:1 for large or bold text ≥ 18.66px)
- **AAA** needs **7:1** for normal text (4.5:1 for large)

The picker shows your color's ratio against white and black with pass/fail badges, so you can catch a low-contrast button or link before it ships — not in an accessibility audit later.

## Building a palette from one color

Need a set of shades for hover states, borders, and backgrounds? Start from your base color and generate **tints** (mixing toward white) and **shades** (mixing toward black). The tool's tints-and-shades strip does this for you — click any swatch to make it the active color and read its exact values. It's a quick way to get a consistent ramp without guessing hex codes.

## One honest caveat about CMYK

The CMYK value from any RGB-based tool is a *mathematical* conversion — a solid reference, but not a color-managed one. Real print output depends on your printer's ICC profile, the paper, and the press. So use the CMYK number to communicate intent, but always proof critical print work with the actual print shop. For screen work (HEX, RGB, HSL, HSV) what you see is what you get.

## Related Tools

- [beautify and format messy CSS code online free](https://ultimatetools.io/tools/coding-tools/code-beautifier/) — tidy the stylesheet you just pasted your new color values into
- [convert HTML and CSS into a PNG or JPEG image](https://ultimatetools.io/tools/image-tools/html-to-image/) — export a styled color swatch or component as an image for a doc or slide
- [create a custom-colored QR code free with no signup](https://ultimatetools.io/tools/misc-tools/qr-code-generator/) — reuse your brand HEX values in a QR code that matches the rest of your design

Stop hand-converting hex codes. **[Open the free color picker & converter →](https://ultimatetools.io/tools/coding-tools/color-picker/)** — pick a color, read HEX, RGB, HSL, HSV and CMYK at once, check contrast, copy what you need. Nothing uploaded.