// /api/audit.js
//
// Runs a real audit against a target URL using:
//   - Google PageSpeed Insights API (Lighthouse, synthetic)
//   - Chrome UX Report API (CrUX, real-user data, optional)
//   - A lightweight HTML fingerprint scan (platform detection)
//
// All API keys are read from process.env — never exposed to the client.
//
// REQUIRED ENV VARS (set in Vercel dashboard):
//   PSI_API_KEY        — Google API key with PageSpeed Insights API enabled
//   CRUX_API_KEY       — Google API key with Chrome UX Report API enabled
//                        (can be the SAME key as PSI_API_KEY if you enable both
//                         APIs on the same project)
//
// OPTIONAL:
//   ALLOWED_ORIGIN     — CORS allow-origin. Defaults to '*' for prototype.
//                        Set to your real domain (https://bm.scandiweb.com)
//                        before going public.

export default async function handler(req, res) {
  // ---- CORS ----
  const allowedOrigin = process.env.ALLOWED_ORIGIN || '*';
  res.setHeader('Access-Control-Allow-Origin', allowedOrigin);
  res.setHeader('Access-Control-Allow-Methods', 'POST, OPTIONS');
  res.setHeader('Access-Control-Allow-Headers', 'Content-Type');

  if (req.method === 'OPTIONS') {
    return res.status(204).end();
  }

  if (req.method !== 'POST') {
    return res.status(405).json({ error: 'Method not allowed. Use POST.' });
  }

  // ---- Parse + validate URL ----
  let { url } = req.body || {};
  if (!url || typeof url !== 'string') {
    return res.status(400).json({ error: 'URL required in request body.' });
  }

  url = normalizeUrl(url);
  if (!url) {
    return res.status(400).json({ error: 'URL appears invalid.' });
  }

  // ---- Validate env ----
  const psiKey = process.env.PSI_API_KEY;
  const cruxKey = process.env.CRUX_API_KEY || psiKey;

  if (!psiKey) {
    return res.status(500).json({
      error: 'Server misconfigured: PSI_API_KEY env var not set.'
    });
  }

  // ---- Run audits in parallel ----
  // Each one returns a result OR an error object — we never throw out of here,
  // because partial data is more useful than a blanket failure.
  const [mobile, desktop, crux, fingerprint] = await Promise.all([
    runLighthouse(url, 'mobile', psiKey),
    runLighthouse(url, 'desktop', psiKey),
    runCrux(url, cruxKey),
    runFingerprint(url)
  ]);

  // ---- Shape the response ----
  return res.status(200).json({
    audited_at: new Date().toISOString(),
    target_url: url,
    lighthouse: {
      mobile,
      desktop
    },
    crux,
    platform: fingerprint
  });
}

// ============================================================
// URL normalization
// ============================================================

function normalizeUrl(url) {
  url = url.trim();
  if (!url) return null;
  if (!/^https?:\/\//i.test(url)) {
    url = 'https://' + url;
  }
  url = url.replace(/\/+$/, '');
  try {
    const u = new URL(url);
    if (!u.hostname || !u.hostname.includes('.')) return null;
    return u.toString();
  } catch {
    return null;
  }
}

// ============================================================
// Lighthouse via PageSpeed Insights API
// ============================================================

async function runLighthouse(url, strategy, apiKey) {
  const endpoint = 'https://www.googleapis.com/pagespeedonline/v5/runPagespeed';
  const params = new URLSearchParams({
    url,
    strategy,                                  // 'mobile' or 'desktop'
    key: apiKey
  });
  // PSI takes 5 categories — we ask for the four that matter for the pitch
  ['performance', 'accessibility', 'best-practices', 'seo'].forEach(c =>
    params.append('category', c)
  );

  const fullUrl = `${endpoint}?${params.toString()}`;

  try {
    const r = await fetch(fullUrl);
    if (!r.ok) {
      const text = await r.text();
      return {
        ok: false,
        error: `PageSpeed API returned ${r.status}`,
        detail: text.slice(0, 400)
      };
    }
    const json = await r.json();
    return parseLighthouseResult(json);
  } catch (err) {
    return {
      ok: false,
      error: 'PageSpeed API request failed',
      detail: String(err.message || err)
    };
  }
}

function parseLighthouseResult(json) {
  const lh = json.lighthouseResult;
  if (!lh) {
    return {
      ok: false,
      error: 'No lighthouseResult in PSI response'
    };
  }

  const cat = lh.categories || {};
  const audits = lh.audits || {};

  // Pull category scores (0–1, multiply to 0–100, round)
  const scores = {
    performance:    pctScore(cat.performance?.score),
    accessibility:  pctScore(cat.accessibility?.score),
    bestPractices:  pctScore(cat['best-practices']?.score),
    seo:            pctScore(cat.seo?.score)
  };

  // Pull Core Web Vitals + key timings
  const metrics = {
    lcp: numericValue(audits['largest-contentful-paint']),
    cls: numericValue(audits['cumulative-layout-shift']),
    inp: numericValue(audits['interaction-to-next-paint']),
    fcp: numericValue(audits['first-contentful-paint']),
    tbt: numericValue(audits['total-blocking-time']),
    ttfb: numericValue(audits['server-response-time']),
    si:  numericValue(audits['speed-index'])
  };

  // Pull notable diagnostic findings — these ARE real, from Lighthouse
  const findings = collectFindings(audits);

  return {
    ok: true,
    scores,
    metrics,
    findings,
    final_url: lh.finalUrl || lh.requestedUrl,
    lighthouse_version: lh.lighthouseVersion,
    fetch_time: lh.fetchTime
  };
}

function pctScore(s) {
  if (s === null || s === undefined) return null;
  return Math.round(s * 100);
}

function numericValue(audit) {
  if (!audit) return null;
  return {
    value: audit.numericValue ?? null,
    display: audit.displayValue ?? null,
    score: audit.score ?? null
  };
}

// Audits worth surfacing — the ones with concrete savings
const FINDING_AUDIT_IDS = [
  'unused-javascript',
  'unused-css-rules',
  'render-blocking-resources',
  'unminified-javascript',
  'unminified-css',
  'uses-text-compression',
  'uses-optimized-images',
  'uses-webp-images',
  'modern-image-formats',
  'efficient-animated-content',
  'duplicated-javascript',
  'legacy-javascript',
  'third-party-summary',
  'bootup-time',
  'mainthread-work-breakdown',
  'dom-size',
  'uses-long-cache-ttl',
  'uses-rel-preconnect'
];

function collectFindings(audits) {
  const found = [];
  for (const id of FINDING_AUDIT_IDS) {
    const a = audits[id];
    if (!a) continue;
    // Skip if the audit passed (score 1) or was not applicable
    if (a.score === 1) continue;
    if (a.scoreDisplayMode === 'notApplicable') continue;
    found.push({
      id,
      title: a.title,
      description: a.description,
      score: a.score,
      display: a.displayValue ?? null,
      // savings (bytes / ms) — when the audit reports it
      savingsBytes: a.details?.overallSavingsBytes ?? null,
      savingsMs:    a.details?.overallSavingsMs ?? null
    });
  }
  // Sort by impact — bigger savings first, then by score (worse = lower score first)
  found.sort((a, b) => {
    const aw = (a.savingsBytes || 0) + (a.savingsMs || 0) * 1000;
    const bw = (b.savingsBytes || 0) + (b.savingsMs || 0) * 1000;
    if (bw !== aw) return bw - aw;
    return (a.score ?? 1) - (b.score ?? 1);
  });
  return found;
}

// ============================================================
// CrUX (Chrome UX Report) — real-user data
// ============================================================

async function runCrux(url, apiKey) {
  const endpoint = `https://chromeuxreport.googleapis.com/v1/records:queryRecord?key=${apiKey}`;

  // CrUX prefers "origin" lookups for sites without enough page-level data.
  // We try URL-level first, fall back to origin-level if not found.
  const urlPayload = { url, formFactor: 'PHONE' };
  const originPayload = { origin: getOrigin(url), formFactor: 'PHONE' };

  try {
    let r = await postJSON(endpoint, urlPayload);
    let level = 'url';
    if (r.status === 404) {
      // Not enough data at URL level — try origin
      r = await postJSON(endpoint, originPayload);
      level = 'origin';
    }
    if (r.status === 404) {
      return {
        ok: true,
        has_data: false,
        reason: 'Insufficient real-user data — site does not have enough Chrome user traffic for CrUX to report on.'
      };
    }
    if (!r.ok) {
      const text = await r.text();
      return {
        ok: false,
        error: `CrUX API returned ${r.status}`,
        detail: text.slice(0, 400)
      };
    }
    const json = await r.json();
    return parseCrux(json, level);
  } catch (err) {
    return {
      ok: false,
      error: 'CrUX API request failed',
      detail: String(err.message || err)
    };
  }
}

function parseCrux(json, level) {
  const record = json.record || {};
  const metrics = record.metrics || {};

  return {
    ok: true,
    has_data: true,
    data_level: level,                      // 'url' or 'origin' — origin is fallback
    period: record.collectionPeriod || null, // 28-day period
    metrics: {
      lcp:     percentilesAndHistogram(metrics.largest_contentful_paint),
      cls:     percentilesAndHistogram(metrics.cumulative_layout_shift),
      inp:     percentilesAndHistogram(metrics.interaction_to_next_paint),
      fcp:     percentilesAndHistogram(metrics.first_contentful_paint),
      ttfb:    percentilesAndHistogram(metrics.experimental_time_to_first_byte || metrics.server_response_time)
    }
  };
}

function percentilesAndHistogram(metric) {
  if (!metric) return null;
  return {
    p75: metric.percentiles?.p75 ?? null,
    histogram: (metric.histogram || []).map(b => ({
      start: b.start,
      end:   b.end,
      density: b.density
    }))
  };
}

function getOrigin(url) {
  try {
    const u = new URL(url);
    return `${u.protocol}//${u.hostname}`;
  } catch {
    return url;
  }
}

async function postJSON(url, body) {
  return fetch(url, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify(body)
  });
}

// ============================================================
// Platform fingerprinting — Magento, Hyva, etc.
// ============================================================
//
// We DO NOT claim certainty. We collect markers and report what we found.
// If we find Magento markers, we say "Likely Magento — N indicators."
// Never "DETECTED: Magento 2.4.5" unless we have a generator tag stating it.

async function runFingerprint(url) {
  try {
    const controller = new AbortController();
    const timeout = setTimeout(() => controller.abort(), 8000);

    const r = await fetch(url, {
      headers: {
        // Pretend to be a real browser so we get the real HTML
        'User-Agent': 'Mozilla/5.0 (compatible; ScandiwebAudit/0.1; +https://scandiweb.com/)'
      },
      redirect: 'follow',
      signal: controller.signal
    });
    clearTimeout(timeout);

    if (!r.ok) {
      return {
        ok: false,
        error: `Page fetch returned ${r.status}`
      };
    }

    const headers = Object.fromEntries(r.headers.entries());
    const html = await r.text();

    return analyzeHtml(html, headers);
  } catch (err) {
    return {
      ok: false,
      error: 'Page fetch failed',
      detail: String(err.message || err)
    };
  }
}

function analyzeHtml(html, headers) {
  const lower = html.toLowerCase();
  const markers = [];

  // ---- Magento markers ----
  const magentoSignals = [];

  // Mage object reference — Magento 1 + 2 expose this
  if (/var\s+mage\b|window\.mage\b|\bmage\.cookies\b|\bmagento_/i.test(html)) {
    magentoSignals.push({ kind: 'js-mage-reference', strength: 'strong' });
  }
  // Static path — both M1 and M2 use /pub/static/version.../
  if (/\/pub\/static\/version\d+\//i.test(html)) {
    magentoSignals.push({ kind: 'pub-static-versioned-path', strength: 'strong' });
  }
  // /static/frontend/ paths — Magento 2 specific
  if (/\/static\/frontend\/[A-Za-z0-9_]+\/[A-Za-z0-9_]+\//.test(html)) {
    magentoSignals.push({ kind: 'm2-static-frontend-path', strength: 'strong' });
  }
  // RequireJS config from Magento
  if (/requirejs-config\.js/i.test(html)) {
    magentoSignals.push({ kind: 'requirejs-config', strength: 'medium' });
  }
  // Cookie / storage names Magento uses
  if (/X-Magento|mage-cache|mage-translation|mage-messages/i.test(html + JSON.stringify(headers))) {
    magentoSignals.push({ kind: 'magento-cookies-or-headers', strength: 'medium' });
  }
  // Generator meta tag — rare but conclusive when present
  const genMatch = html.match(/<meta[^>]+name=["']generator["'][^>]+content=["']([^"']+)["']/i);
  if (genMatch && /magento/i.test(genMatch[1])) {
    magentoSignals.push({ kind: 'generator-meta', strength: 'strong', value: genMatch[1] });
  }

  // ---- Hyva markers ----
  const hyvaSignals = [];

  // Hyva uses Alpine.js + Tailwind — together they're a strong signal
  const hasAlpine = /alpinejs|x-data=|x-show=|x-on:|@click=/i.test(html);
  const hasTailwind = /\btw-|tailwind|class="[^"]*\b(flex|grid|px-\d|py-\d|text-\w+-\d{3})/i.test(html);

  if (hasAlpine) hyvaSignals.push({ kind: 'alpinejs-present', strength: 'medium' });
  if (hasTailwind) hyvaSignals.push({ kind: 'tailwind-classes', strength: 'medium' });
  // Hyva-specific — checkout module, theme path, hyva-themes references
  if (/hyva[\-_]themes?|\/hyva\/|hyva-checkout/i.test(html)) {
    hyvaSignals.push({ kind: 'hyva-theme-reference', strength: 'strong' });
  }
  // Hyva does NOT ship requireJS or knockout — their absence + Alpine = stronger Hyva signal
  const hasKnockout = /knockoutjs|ko\.observable|data-bind=/i.test(html);
  const hasRequireJS = /require(\.min)?\.js|requirejs-config/i.test(html);
  if (hasAlpine && hasTailwind && !hasKnockout && !hasRequireJS) {
    hyvaSignals.push({
      kind: 'hyva-fingerprint-pattern',
      strength: 'strong',
      note: 'Alpine + Tailwind without RequireJS/Knockout'
    });
  }

  // ---- Magento 1 vs 2 hint ----
  let magentoVersionHint = null;
  if (/skin\/frontend\//i.test(html)) {
    magentoVersionHint = 'Magento 1 (skin/frontend path)';
  } else if (/\/static\/frontend\//i.test(html)) {
    magentoVersionHint = 'Magento 2 (static/frontend path)';
  }

  // ---- Frontend stack flags (independent of Hyva detection) ----
  const stack = {
    alpinejs: hasAlpine,
    tailwind: hasTailwind,
    knockout: hasKnockout,
    requirejs: hasRequireJS,
    jquery: /jquery(\.min)?\.js/i.test(html)
  };

  // ---- Verdict — confidence-graded, never overclaiming ----
  const magentoConfidence = strengthScore(magentoSignals);
  const hyvaConfidence = strengthScore(hyvaSignals);

  return {
    ok: true,
    is_magento: magentoConfidence.label,    // 'unlikely' | 'possible' | 'likely' | 'very likely'
    magento_signals: magentoSignals,
    magento_version_hint: magentoVersionHint,
    is_hyva: hyvaConfidence.label,
    hyva_signals: hyvaSignals,
    frontend_stack: stack,
    server: headers['server'] || null,
    cdn_hints: detectCdn(headers),
    response_headers_sample: pickHeaders(headers, [
      'server', 'x-powered-by', 'x-magento-cache-debug', 'x-magento-cache-control',
      'cf-ray', 'x-amz-cf-id', 'x-vercel-id', 'fastly-debug-digest'
    ])
  };
}

function strengthScore(signals) {
  // Count strength: strong=2, medium=1
  let score = 0;
  for (const s of signals) {
    score += s.strength === 'strong' ? 2 : 1;
  }
  let label = 'unlikely';
  if (score >= 4) label = 'very likely';
  else if (score >= 2) label = 'likely';
  else if (score >= 1) label = 'possible';
  return { score, label };
}

function detectCdn(headers) {
  const found = [];
  if (headers['cf-ray'] || /cloudflare/i.test(headers['server'] || '')) found.push('Cloudflare');
  if (headers['x-amz-cf-id'] || /cloudfront/i.test(headers['via'] || '')) found.push('CloudFront');
  if (headers['x-vercel-id']) found.push('Vercel');
  if (/fastly/i.test(headers['x-served-by'] || headers['fastly-debug-digest'] || '')) found.push('Fastly');
  if (/akamai/i.test(headers['x-akamai-transformed'] || headers['server'] || '')) found.push('Akamai');
  return found;
}

function pickHeaders(headers, keys) {
  const out = {};
  for (const k of keys) {
    if (headers[k]) out[k] = headers[k];
  }
  return out;
}
