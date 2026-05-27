# The 13 SEO Title-Length Bugs I Shipped — And the 5-Line Audit Script That Catches Them

Bing Site Scan flagged 13 of my Next.js tool pages this week with the same warning: "Title too long — over 70 characters". Google was not flagging them directly, but Google had been silently SERP-truncating the same titles for weeks. The tail of each title — the differentiator, the keyword, the reason someone would click my result over the others — was getting cut off mid-word.

This is the kind of SEO bug that does not produce a crisis but produces a slow leak. Every search result with a truncated title earns fewer clicks than it should. Multiply by the page count and the time window, and the cumulative cost is real traffic that never showed up.

Here is the audit script I added to my weekly SEO session so this cannot recur, plus the rewrite pattern for fixing the 13 pages.

---

## The Title-Length Rules

Three thresholds matter for any content site:

- **Google SERP**: titles truncate at around 580 pixels wide, which corresponds to roughly 60 characters for typical title content. Past that, the title shows in results as `Your Great Tool — All The Best Features And Even ...` with the tail eaten by ellipsis.
- **Bing**: more lenient on display but flags titles over 70 characters in its Site Scan tool as a "Title too long" warning. Bing publishes the rule because longer titles correlate with lower CTR in their data.
- **LinkedIn and social previews**: most platforms truncate under 70 characters. A title cut mid-word in social previews looks unprofessional and reduces click-through.

The target: **≤ 60 characters**. The hard limit: **never over 70**.

---

## How I Got Into the Mess

The bugs were created by people doing reasonable things in good faith.

**Adding differentiators incrementally.** "PDF Metadata Viewer & Editor Online Free" is 41 characters. Adding "— View PDF Properties, No Upload" makes it 75 characters. Each addition felt like it added information. None of the additions individually crossed a threshold that triggered a warning.

**Concatenating features over time.** A new feature ships, the title gets `+ AI Line-Item Writer` appended to reflect it. Same story — each addition is small, no one rewrites the whole title.

**No automated check.** The title length lives in `metadata.title` in a Next.js page file. There is no compile-time check that flags lengths over a threshold. CI does not catch it. Build succeeds. Page deploys. The truncation happens silently in production.

The accumulation pattern is the diagnostic clue. If you find one over-70 title on your site, you probably have ten more.

---

## The Audit Script

Here is the bash one-liner that catches the bugs. Run it from the project root before every SEO session:

```bash
for f in app/tools/**/page.tsx; do
  title=$(grep -A0 'title:' "$f" 2>/dev/null | head -1 | sed -E 's/.*title: "(.*)".*/\1/')
  len=$(echo -n "$title" | wc -c)
  if [ "$len" -gt 70 ]; then
    echo "❌ $len chars: $f"
    echo "    $title"
  elif [ "$len" -gt 60 ]; then
    echo "⚠️  $len chars: $f"
    echo "    $title"
  fi
done
```

What it does:

- Iterates every `page.tsx` file under `app/tools/`
- Extracts the string literal from `metadata.title` using grep + sed
- Measures the byte length with `wc -c` (close enough to character count for ASCII titles; if you have multi-byte characters in titles, this slightly over-counts but errs on the safe side)
- Prints anything over 70 in red (❌) and anything over 60 in yellow (⚠️)

Output looks like:

```
❌ 75 chars: app/tools/pdf-tools/pdf-metadata-viewer/page.tsx
    PDF Metadata Viewer & Editor Online Free — View PDF Properties, No Upload
❌ 82 chars: app/tools/security-tools/url-explainer/page.tsx
    URL Explainer Free — Decode UTM Tags, Tracking Params & Suspicious Links with AI
⚠️ 65 chars: app/tools/misc-tools/some-tool/page.tsx
    Some Tool Online Free — Feature A, Feature B, Feature C
```

The ❌ results are the fix list. The ⚠️ results are watch-list items.

---

## CI Variant

If you want this as a hard gate in CI, here is a version that exits non-zero on any ❌:

```bash
#!/usr/bin/env bash
set -e
fail=0
for f in app/tools/**/page.tsx; do
  title=$(grep -A0 'title:' "$f" 2>/dev/null | head -1 | sed -E 's/.*title: "(.*)".*/\1/')
  len=$(echo -n "$title" | wc -c)
  if [ "$len" -gt 70 ]; then
    echo "::error file=$f::Title too long ($len chars, max 70): $title"
    fail=1
  fi
done
exit $fail
```

Wire it into your GitHub Actions or pre-commit hook and any future PR that introduces an over-70 title fails before merge.

---

## The Rewrite Pattern

For each flagged page, the fix follows a fixed pattern:

**Identify the primary keyword.** The query users actually search for. That word stays at the start.

**Cut one differentiator.** Usually the third or fourth comma-separated phrase. "GST, HST, VAT, Logo, Discount, Cloud Sync" becomes "GST/HST/VAT + AI Line-Items".

**Re-count.** If still over 60, cut another differentiator.

**Keep "Free" if the page is free.** It is a critical CTR signal.

**Drop `| Brand Name` from titles.** Wastes ~15 characters. Branding lives in the favicon, breadcrumbs, and URL itself.

Example before/after from my own sweep:

| Page | Before (chars) | After (chars) |
|---|---|---|
| Invoice Generator | "Free Invoice Generator with AI Line-Item Descriptions — GST, HST, VAT, Logo" (77) | "Free Invoice Generator — GST/HST/VAT + AI Line-Items" (54) |
| URL Explainer | "URL Explainer Free — Decode UTM Tags, Tracking Params & Suspicious Links with AI" (82) | "AI URL Explainer Free — Decode UTM Tags & Tracking Params" (59) |
| Blur Image | "Blur Image Online Free — Blur Face, Blur Part of Image, Full or Selective Blur" (80) | "Blur Image Online Free — Blur Face or Selective Area" (54) |

Total fix time per title: about 90 seconds. The 13-title sweep took 20 minutes including the audit run.

---

## After Fixing — Resubmit to GSC and Bing

A title change is invisible to search engines until they recrawl. Two ways to speed that up:

**Bing IndexNow.** Submit the modified URLs as a single JSON POST to `https://api.indexnow.org/IndexNow`. Bing typically recrawls within 24-48 hours of submission. The 13 URL submission in one call:

```bash
curl -X POST "https://api.indexnow.org/IndexNow" \
  -H "Content-Type: application/json; charset=utf-8" \
  -d '{
    "host": "yourdomain.com",
    "key": "your-indexnow-key",
    "keyLocation": "https://yourdomain.com/your-indexnow-key.txt",
    "urlList": ["https://yourdomain.com/page-1/", "https://yourdomain.com/page-2/"]
  }'
```

**Google Search Console.** Manual URL Inspection → Request Indexing for each URL. Daily quota is ~10. For larger batches, prioritize the highest-traffic pages.

---

## The Two SEO Lessons

**Audit the boring stuff at session start.** Title length, meta description length, missing FAQ schema, broken internal links — these are boring checks that humans skip and accumulators reward. A 30-second audit script at the start of every SEO session catches more issues than any other intervention.

**Use Bing Webmaster Tools.** Most people only check Google Search Console. Bing's Site Scan is independently useful — it catches issues Google's reports do not flag. The 70-character title warning is a Bing-specific report; Google has no equivalent dashboard alert. If you only watch GSC, you miss the diagnostic.

---

## Related Tools

- [free Word Counter with Flesch-Kincaid readability and AI text rewriter for grade-level simplification](https://ultimatetools.io/tools/text-tools/word-counter/) — useful for measuring whether your meta descriptions match your audience's reading level
- [free JSON Formatter with AI Fix Broken JSON tab](https://ultimatetools.io/tools/text-tools/json-formatter/) — for the JSON-LD structured data audits that pair with title-length checks
- [free URL Explainer that decodes UTM tags and tracking parameters](https://ultimatetools.io/tools/security-tools/url-explainer/) — for the analytics side of SEO audits

---

This is not a clever SEO trick. It is the SEO equivalent of automated lint rules — boring discipline that prevents stupid bugs. Once the audit is in your weekly workflow as a 30-second step at session start, the title-length issue cannot recur. Run the script this week and you might find 5 to 15 titles that need cleanup. Then it never bites you again.

**[Try the free utility tools at Ultimate Tools — all built on the title-length audit workflow](https://ultimatetools.io/tools/)**