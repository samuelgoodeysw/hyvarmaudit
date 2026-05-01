# Scandiweb BM Upgrade Audit — Real Data Setup

This is the upgrade audit wizard wired to **real data** — Google PageSpeed Insights (Lighthouse), Chrome UX Report (CrUX, real-user data), and a lightweight HTML platform fingerprinter.

No API keys live in the browser. They're held server-side in a Vercel function.

## What's in this folder

```
audit-real/
├── api/
│   └── audit.js          ← Vercel serverless function — calls Google APIs, returns JSON
├── public/
│   └── index.html        ← The wizard frontend (calls /api/audit)
├── package.json
├── vercel.json           ← Sets function timeout to 60s (Lighthouse can be slow)
└── README.md             ← This file
```

## Setup — start to finish (~15 minutes)

### 1. Get a Google API key

1. Go to https://console.cloud.google.com/
2. Create a new project (or pick an existing one) — call it something like `scandiweb-audit`
3. Open the API Library and enable **both**:
   - **PageSpeed Insights API**
   - **Chrome UX Report API**
4. Go to **Credentials → Create credentials → API key**
5. Copy the key. (Optional but recommended: restrict the key to those two APIs and to your Vercel deployment domain.)

You can use the same key for both APIs. Free tier covers way more than you'll need for prototyping (PSI = 25,000 requests/day).

### 2. Push to GitHub

If this folder isn't already a Git repo:

```bash
cd audit-real
git init
git add .
git commit -m "Initial real-data audit wizard"
gh repo create scandiweb-bm-upgrade-audit --public --source=. --push
```

(Or use the GitHub web UI to create a repo and push.)

### 3. Deploy to Vercel

1. Sign in to https://vercel.com
2. **Add New → Project**
3. Import your GitHub repo
4. Framework preset: **Other** (Vercel will auto-detect the `api/` folder and `public/` folder — no build step needed)
5. Don't deploy yet — first add the env var:
6. Open **Settings → Environment Variables** and add:
   - `PSI_API_KEY` = your Google API key
   - `CRUX_API_KEY` = same key (or a separate one if you've split them)
7. (Optional, recommended for production) `ALLOWED_ORIGIN` = `https://your-domain.com`
8. Click **Deploy**

Within ~30 seconds you'll have a URL like `https://scandiweb-bm-upgrade-audit.vercel.app`.

### 4. Test it

1. Open your Vercel URL
2. Enter a URL — try a real Magento merchant site (e.g. one of your prospects)
3. Wait 20–40 seconds (Lighthouse takes time on cold sites)
4. Real scores, real findings, real platform detection

If the audit fails, the browser console will show the error. Most common causes:
- API key not set, or APIs not enabled — check Vercel env vars
- Site blocks bots / has aggressive WAF — try a different URL to confirm
- Quota exceeded — unlikely on free tier, but check the Cloud Console

## What's real, what's illustrative

**Real (from APIs):**
- Lighthouse mobile + desktop scores (Performance, Accessibility, Best Practices, SEO)
- Synthetic Core Web Vitals (LCP, CLS, INP, FCP, TBT, TTFB) — single Lighthouse run
- Real-user Core Web Vitals from CrUX (28-day median p75 from real Chrome users)
- Diagnostic findings — actual Lighthouse audit IDs with real byte/ms savings figures
- Platform fingerprint — markers found in the HTML (we report what we found, never claim certainty)
- Server / CDN detection — from real response headers

**Illustrative (clearly labelled):**
- "Hyva typical 85–95" benchmarks — pulled from Hyva's published case studies. We never claim a guarantee.
- The "if you moved" comparison rows — the Hyva column is a typical, not a promise.

This distinction is built into the UI copy. The footnote says: *"Live audit · Real scores from Google PageSpeed Insights + Chrome UX Report. Hyva comparisons are illustrative — based on Hyva's published case studies, not guarantees."*

## Things to do before this goes public

1. **Lock down CORS** — set `ALLOWED_ORIGIN` to your real domain so other sites can't burn your quota
2. **Rate-limit the endpoint** — easy add: cap requests per IP, e.g. via Upstash Redis on Vercel free tier. Not strictly needed for internal use, but worth doing if you embed the wizard publicly
3. **Cache results** — same URL audited within 24 hours should return the cached result rather than burning another API call. Vercel KV or Upstash Redis, both free tier
4. **Wire submit to HubSpot** — currently the JSON payload only `console.log`s. ~30 minutes to wire to a HubSpot form embed so submissions land in CRM
5. **Custom domain** — `scandiweb-bm-upgrade-audit.vercel.app` is fine for testing; `bm-audit.scandiweb.com` for prospects
6. **Run on yourselves first** — audit some Scandiweb client sites + check the output lines up with what you know. Reality-check the platform detector before showing prospects

## Quotas + costs

- **PageSpeed Insights**: free, 25,000 requests/day — you'll never hit this organically
- **CrUX**: free, generous quota — same Google API key
- **Vercel functions**: free tier covers ~100 GB-hours/month execution. Each audit is ~30s of function time → you can run 12,000 audits/month on free tier
- **Vercel hosting**: static + functions on Hobby plan = $0

So the whole thing is free to run for prototype-and-internal-use volumes.

## Local development

```bash
npm install -g vercel
vercel dev
```

Vercel CLI runs the static site + the function locally. Set the env vars in `.env.local`:

```
PSI_API_KEY=AIzaSy...
CRUX_API_KEY=AIzaSy...
```

(`.gitignore` already excludes `.env.local`.)

## Architecture notes

The function in `api/audit.js`:
- Validates and normalizes the URL
- Runs four operations in parallel:
  - Lighthouse mobile via PSI API
  - Lighthouse desktop via PSI API
  - CrUX query (URL-level, fallback to origin-level if no data)
  - Direct fetch of the HTML for platform fingerprinting
- Aggregates the results
- Returns `{ lighthouse: {mobile, desktop}, crux, platform, audited_at, target_url }`

Each operation is wrapped in its own try/catch — partial data is more useful than blanket failure. If Lighthouse fails but CrUX succeeds (or vice versa), you still get a useful report.

The platform fingerprinter is deliberately conservative. It collects "signals" (regex matches in the HTML, response headers) and returns a confidence label — `unlikely / possible / likely / very likely` — never an absolute "DETECTED."

## When you outgrow this

Real next steps when this proves useful:
- **Schema validation** (free, Google Rich Results Test API) — adds "structured data found" to the report
- **Mobile-Friendly Test** (free) — adds the official Google mobile-friendly verdict
- **Screenshot service** (urlbox.io / ScreenshotAPI / self-hosted Chromium) — adds visual reference to the report
- **BuiltWith API** — replaces our DIY fingerprinter with industry-standard platform detection (paid, ~$295/mo)
- **Ahrefs / SEMrush API** — adds organic SEO context (paid)
- **PDF export** — generate a downloadable report sales can email out, separate from the live UI
