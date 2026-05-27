# Tell Bing to Re-Crawl Your URLs Instantly with IndexNow — A 5-Minute Tutorial

Most developers know about Google Search Console's URL Inspection → Request Indexing flow. It is slow (one URL at a time), rate-limited (~10/day), and only affects Google. Bing has its own indexing API called IndexNow, and almost nobody uses it because the documentation is buried and the example code is split across multiple Microsoft KB pages.

This tutorial gets you from zero to "submit 100 URLs to Bing in one HTTP request" in 5 minutes. The same protocol is also used by Yandex, Seznam, and other smaller search engines. Knowing this one API gives you reverse-indexing on every search engine outside Google in a single call.

---

## What IndexNow Actually Does

IndexNow is a push-based indexing protocol. You POST a JSON payload listing the URLs you want re-crawled. The receiving search engine acknowledges the submission and queues those URLs for fresh crawling — typically within 24-48 hours.

Compare this to the pull model (sitemaps, crawl budgets) where you wait for the search engine to discover changes on its own schedule. IndexNow flips the relationship: you tell the engine when to look, instead of waiting to be looked at.

This matters most when:

- You ship metadata changes (title, description, schema) and want them indexed before the next natural crawl cycle
- You add new pages and do not want to wait for sitemap parsing
- You delete or restructure pages and need the old URLs purged

For sites under daily-deploy cadence (most production content sites in 2026), IndexNow turns "I shipped this Tuesday, Bing sees it Friday" into "I shipped this Tuesday, Bing sees it Wednesday".

---

## The Setup (One-Time)

You need three things before you can submit URLs:

**A key** — any random 8-128 character hex string. I use 32 hex chars (effectively a UUID without dashes). Generate one once and use it forever.

**A key file** at the root of your domain — `https://yourdomain.com/your-key.txt` — containing just the key as plain text. This is how IndexNow verifies you actually control the domain.

**The URL of the key file** — `keyLocation` in your submissions.

Concretely:

```bash
# Generate a key (one time)
$ openssl rand -hex 16
f3a8b2c1d4e5f6a7b8c9d0e1f2a3b4c5

# Create the key file
$ echo "f3a8b2c1d4e5f6a7b8c9d0e1f2a3b4c5" > public/f3a8b2c1d4e5f6a7b8c9d0e1f2a3b4c5.txt

# Deploy. Verify the file is accessible:
$ curl https://yourdomain.com/f3a8b2c1d4e5f6a7b8c9d0e1f2a3b4c5.txt
f3a8b2c1d4e5f6a7b8c9d0e1f2a3b4c5
```

That is the entire setup. No account creation, no OAuth, no API key request.

---

## The Submission (curl)

Submit one URL:

```bash
curl -X POST "https://api.indexnow.org/IndexNow" \
  -H "Content-Type: application/json; charset=utf-8" \
  -d '{
    "host": "yourdomain.com",
    "key": "f3a8b2c1d4e5f6a7b8c9d0e1f2a3b4c5",
    "keyLocation": "https://yourdomain.com/f3a8b2c1d4e5f6a7b8c9d0e1f2a3b4c5.txt",
    "urlList": ["https://yourdomain.com/page-1/"]
  }'
```

Submit a batch of URLs in one call (up to 10,000 per submission):

```bash
curl -X POST "https://api.indexnow.org/IndexNow" \
  -H "Content-Type: application/json; charset=utf-8" \
  -d '{
    "host": "yourdomain.com",
    "key": "f3a8b2c1d4e5f6a7b8c9d0e1f2a3b4c5",
    "keyLocation": "https://yourdomain.com/f3a8b2c1d4e5f6a7b8c9d0e1f2a3b4c5.txt",
    "urlList": [
      "https://yourdomain.com/page-1/",
      "https://yourdomain.com/page-2/",
      "https://yourdomain.com/page-3/"
    ]
  }'
```

Successful response: **HTTP 200** with empty body. Failure responses:

- **HTTP 400** — malformed JSON or missing required field
- **HTTP 403** — key verification failed (the key file at `keyLocation` does not contain the key in the request, or is not accessible)
- **HTTP 422** — invalid host or invalid URL format
- **HTTP 429** — rate limit exceeded (rare; the per-day quota is in the thousands)

---

## The PowerShell Version

If you are on Windows and want a Windows-native script:

```powershell
$body = @{
  host        = "yourdomain.com"
  key         = "f3a8b2c1d4e5f6a7b8c9d0e1f2a3b4c5"
  keyLocation = "https://yourdomain.com/f3a8b2c1d4e5f6a7b8c9d0e1f2a3b4c5.txt"
  urlList     = @(
    "https://yourdomain.com/page-1/",
    "https://yourdomain.com/page-2/"
  )
} | ConvertTo-Json -Compress

try {
  $r = Invoke-RestMethod -Uri "https://api.indexnow.org/IndexNow" `
    -Method Post `
    -Body $body `
    -ContentType "application/json; charset=utf-8" `
    -UseBasicParsing
  Write-Output "IndexNow: OK"
} catch {
  $code = $_.Exception.Response.StatusCode.value__
  $desc = $_.Exception.Response.StatusDescription
  Write-Output "IndexNow: $code $desc"
}
```

Use single-quoted strings or here-strings for the JSON body if you have any apostrophes or backticks in URLs — PowerShell interpolation is greedy.

---

## A Node.js Variant for CI

If you want this in your deploy pipeline:

```ts
import { writeFileSync } from "fs";

const submitToIndexNow = async (urls: string[]) => {
  const body = {
    host: "yourdomain.com",
    key: process.env.INDEXNOW_KEY,
    keyLocation: `https://yourdomain.com/${process.env.INDEXNOW_KEY}.txt`,
    urlList: urls,
  };

  const res = await fetch("https://api.indexnow.org/IndexNow", {
    method: "POST",
    headers: { "Content-Type": "application/json; charset=utf-8" },
    body: JSON.stringify(body),
  });

  if (!res.ok) {
    console.error(`IndexNow failed: ${res.status} ${res.statusText}`);
    process.exit(1);
  }

  console.log(`IndexNow: submitted ${urls.length} URLs (HTTP ${res.status})`);
};

// Example: submit URLs from a deploy manifest
const urls = JSON.parse(process.env.DEPLOY_MANIFEST_URLS || "[]");
if (urls.length > 0) {
  await submitToIndexNow(urls);
}
```

Wire this into your post-deploy step. Every push that ships new/modified URLs auto-pings Bing.

---

## What IndexNow Will Not Do

A few caveats worth knowing:

**Not Google.** Google does not participate in IndexNow. As of mid-2026, Google's official stance is "we use other signals to determine when to crawl". For Google submissions, you still need GSC URL Inspection → Request Indexing, manually, one URL at a time.

**Not a ranking signal.** Submitting URLs to IndexNow does not affect their ranking. It only affects how quickly the search engine sees changes. A page that is not ranking will not start ranking just because you submitted it.

**Not a crawl guarantee.** Bing may choose to deprioritize submissions if you submit too frequently or if the submitted URLs have low value. The protocol is "we will look at this soon"; not "we will index this within X hours".

**Subject to spam filtering.** Submitting 10,000 URLs from a brand-new domain triggers anti-abuse heuristics. Bing tends to be more responsive to sites with established crawl history.

---

## Where to Get Bing's Confirmation

After submitting, you can verify Bing received it via:

**Bing Webmaster Tools** → IndexNow Submitted URLs report. Shows every URL submission, timestamp, and status. (Note: this report shows SUBMISSION dates, not CRAWL dates. To see when Bing actually re-crawled, use the Site Explorer report which has a `Last crawled` column per page.)

The lag between submission and crawl is typically 24-48 hours. Some submissions get crawled within hours; others take a few days. There is no SLA.

---

## Where Else IndexNow Works

IndexNow is a shared protocol. The same submission goes to:

- **Bing** (Microsoft)
- **Yandex** (Russia)
- **Seznam** (Czech Republic)
- **Naver** (South Korea, partial)
- **DuckDuckGo** (indirectly via Bing — submissions to Bing flow through to DuckDuckGo's index)

One POST to `api.indexnow.org` reaches all of them. If your audience includes Russia, the Czech Republic, or South Korea — or if you care about DuckDuckGo's privacy-conscious search audience — this is the easiest cross-engine indexing call available.

---

## Related Tools

- [free SEO Word Counter with Flesch-Kincaid readability and AI grade-level rewrite](https://ultimatetools.io/tools/text-tools/word-counter/) — useful for measuring whether your meta descriptions match your audience's reading level
- [free URL Explainer with UTM tag decoder and phishing flags](https://ultimatetools.io/tools/security-tools/url-explainer/) — for the analytics side of SEO audits
- [free JSON Formatter with AI Fix Broken JSON](https://ultimatetools.io/tools/text-tools/json-formatter/) — useful when wiring IndexNow into a CI pipeline with structured data submissions

---

The reason IndexNow is underused is not technical — it is documentation. The protocol is straightforward (one JSON POST, no auth flow, no rate limits worth worrying about), but the official Microsoft Learn page buries it under "advanced Bing Webmaster" content. Most developers never find the page. Now you have the curl command, the PowerShell script, and the Node.js variant in one place. Wire it into your next deploy.

**[Try the free utility tools that use IndexNow for daily metadata updates](https://ultimatetools.io/tools/)**