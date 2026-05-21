# Free Online Word Counter — Count Words, Characters, Sentences, and Reading Time Instantly

Word count is the number you always need and never quite know without stopping to check. Blog posts, essays, email newsletters, LinkedIn posts, cover letters — they all have limits or targets, and eyeballing it never works.

The [free online word counter](https://ultimatetools.io/tools/text-tools/word-counter/) shows six stats simultaneously as you type: words, characters, characters without spaces, sentences, paragraphs, reading time, and speaking time. No install, no login.

## What gets counted and how

**Words** — splits on whitespace, filters empty tokens. `"Hello   world"` counts as 2, not 3.

**Characters** — total length of the string including spaces, punctuation, and newlines.

**Characters (no spaces)** — same as above with all whitespace stripped. This is the number Twitter/X and LinkedIn character limits actually use.

**Sentences** — splits on `.`, `!`, and `?`. Each non-empty segment after splitting counts as one sentence.

**Paragraphs** — splits on line breaks. Each non-empty block of text counts as one paragraph.

**Reading time** — calculated at 200 words per minute, the average adult silent reading speed. `Math.ceil(words / 200)` rounded up to the nearest minute.

**Speaking time** — calculated at 130 words per minute, average conversational speech rate. Useful when writing a script, talk, or video voiceover.

All six stats update live as you type — no submit button, no refresh.

---

## How to use it

1. Open the [Word Counter](https://ultimatetools.io/tools/text-tools/word-counter/)
2. Paste or type your text into the input area
3. Read the six stat cards updating in real time above the text
4. Use the **Copy** button (bottom right of the input) to copy your text to clipboard
5. Use the **Clear** button to reset and start fresh

That's it. Nothing is sent to a server — the tool runs entirely in your browser.

---

## Common use cases

**Blog posts and articles**

Most platforms recommend 1,500–2,500 words for SEO. Medium's partner program has no strict minimum, but articles under 800 words rarely get curated. The [word count tool](https://ultimatetools.io/tools/text-tools/word-counter/) shows exactly where you stand while you're still drafting.

**LinkedIn and Twitter/X posts**

LinkedIn posts cap at 3,000 characters. Twitter/X caps at 280 (or 25,000 for premium). The **Characters** stat — not words — is what matters here. Paste your draft in, check the character count, trim as needed.

**Cover letters and academic submissions**

Most job applications ask for cover letters under 400 words. Most academic journals specify abstract word limits between 150 and 300 words. The word count updates as you write so you don't overshoot.

**Video scripts and presentations**

If you're recording a video or giving a talk and you know the target length, the **speaking time** stat maps your word count directly to minutes on screen. A 10-minute talk at 130 wpm = roughly 1,300 words.

**Email newsletters**

Most email marketers target 200–500 words for click-through newsletters, longer for digest formats. Reading time gives you a reader-facing estimate you can include in the subject line ("3 min read").

---

## Reading time vs speaking time — which one to use

Both are estimates based on average speeds. The right one depends on your format:

| Format | Use |
|---|---|
| Blog post, article, email | Reading time (200 wpm) |
| Video script, podcast script | Speaking time (130 wpm) |
| Presentation, talk | Speaking time (130 wpm) |
| Audiobook narration | Speaking time or slower (~120 wpm) |

If your audience is non-native English speakers, speaking time at a slower pace (110–120 wpm) is a safer estimate.

---

The full tool is live at [Word Counter](https://ultimatetools.io/tools/text-tools/word-counter/) — words, characters, sentences, paragraphs, reading time, and speaking time in one view. Paste your text and all six update instantly.