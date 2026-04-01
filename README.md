# Impacteers — Weekly KPI Dashboard

[![Dashboard](https://img.shields.io/badge/Live%20Dashboard-GitHub%20Pages-185FA5?style=for-the-badge&logo=github)](https://YOUR_USERNAME.github.io/impacteers-kpi-dashboard/)
[![GA4](https://img.shields.io/badge/Data-Google%20Analytics%204-E37400?style=for-the-badge&logo=google-analytics)](https://analytics.google.com)
[![GSC](https://img.shields.io/badge/Data-Search%20Console-4285F4?style=for-the-badge&logo=google)](https://search.google.com/search-console)
[![Version](https://img.shields.io/badge/Version-4.0-0F3460?style=for-the-badge)](/)

> **Self-hosted weekly KPI dashboard for Impacteers built directly on Google Analytics 4 and Google Search Console APIs — zero third-party tools, zero cost.**

---

## Live Dashboard

> Sample GitHub Pages is enabled:
> **https://impacteers2026-sketch.github.io/marketing-kpi-dashboard/**

> Logic Reference GitHub Page Section is enabled:
> **https://impacteers2026-sketch.github.io/marketing-kpi-dashboard/#:~:text=COMPUTED%20FIELD%20LOGIC%20REFERENCE%20%E2%80%94%20DEV%20IMPLEMENTATION%20GUIDE**

---

## What This Dashboard Does

Tracks platform health across all 4 Impacteers properties in one view:

| Property | Purpose |
|---|---|
| `impacteers.com` | Brand hub, blog, talent platform |
| `futurejobs.impacteers.com` | Programmatic job listings |
| `recruitment.impacteers.com` | B2B recruiter portal |
| `examprep.impacteers.com` | Govt exam prep content |

---

## Dashboard Sections

| # | Section | Visual | Question it answers |
|---|---|---|---|
| 3.1 | Weekly Snapshot | 4 KPI cards | Are we growing this week? |
| 3.2 | Traffic by Domain | Multi-line chart | Which property spiked or dropped? |
| 3.3 | Channel Mix | Donut chart | Where is our traffic coming from? |
| 3.4 | Top Pages | Horizontal bar | What content is doing the real work? |
| 3.5 | Engagement vs Bounce | Scatter plot | Where is content sticky vs where do people leave? |
| 3.6 | CTR Decay Curve | Line chart + table | Which pages underperform vs expected CTR at their rank? |
| 3.7 | Opportunity Filter | Interactive table | What opportunities do we have to improve rankings? |
| 3.8 | Branded vs Non-Branded | Grouped bar + table | Are people finding us without knowing us? |
| 3.9 | LLM Referral Traffic | Bar + quality table | Are AI assistants recommending us? |
| 3.10 | Engagement Health | 3-panel grid | Are visitors actually engaging or just bouncing? |

---

## Filter Controls

| Filter | Type | GA4 Param |
|---|---|---|
| Date range | Dropdown + custom date popup | `dateRanges[startDate, endDate]` |
| Sub-domain | Dropdown | `hostname` dimensionFilter (EXACT) |
| Channel | Dropdown (includes LLM Citation) | `sessionDefaultChannelGroup` |
| Organic only | Checkbox toggle | `sessionMedium = organic OR sessionSource in llmDomains` |

> All filter changes trigger a fresh GA4 API call. Debounce at **300ms** to avoid quota waste.

---

## Metrics Reference

### GA4 Metrics

| Metric | API Field | Section |
|---|---|---|
| Total Users | `totalUsers` | Weekly Snapshot |
| New Users | `newUsers` | Weekly Snapshot |
| Sessions | `sessions` | Weekly Snapshot |
| Engagement Rate | `engagementRate` | Engagement Health |
| Avg. Engagement Time | `averageSessionDuration` | Engagement Health |
| Bounce Rate | `bounceRate` | Engagement vs Bounce |
| Session Channel Group | `sessionDefaultChannelGroup` | Channel Mix |
| Google Jobs Apply | `sessionSourceMedium` | Jobs Tracker |

### GSC Metrics

| Metric | API Field | Section |
|---|---|---|
| Clicks | `clicks` | Search Summary |
| Impressions | `impressions` | Search Summary |
| CTR | `ctr` | CTR Decay Curve |
| Average Position | `position` | Opportunity Table |
| Query | `query` | Branded vs Non-Branded |

> **Excluded (Future Phase):** Key Events / Conversions, Revenue — not yet instrumented.

---

## Computed Fields — Dev Reference

These fields are **not returned by any API**. They must be computed client-side or in your backend before render. All threshold values must live in `config.json` — never hardcode in render logic.

<details>
<summary><strong>Week-on-week (WoW) % delta</strong></summary>

```js
// Request both ranges in ONE GA4 API call
dateRanges: [
  { startDate: "7daysAgo", endDate: "today" },
  { startDate: "14daysAgo", endDate: "8daysAgo" }
]

// GA4 returns dateRangeName per row — map accordingly
const delta = ((thisWeek - lastWeek) / lastWeek) * 100;
const display = (delta > 0 ? "▲ " : "▼ ") + Math.abs(delta).toFixed(1) + "% WoW";

// NOTE: Position card inverts colour — lower position = better
```

</details>

<details>
<summary><strong>KPI status tag logic</strong></summary>

```js
function getTag(current, target, type) {
  if (type === "position") {
    if (current <= target)       return "Healthy";
    if (current <= target * 1.5) return "Monitor";
    return "At risk";
  }
  const r = current / target;
  if (r >= 1.0) return "On track";
  if (r >= 0.8) return "Growing";
  if (r >= 0.6) return "Monitor";
  if (r >= 0.4) return "Early";
  return "Below";
}

// Badge colour map:
// "On track" / "Healthy"            → green
// "Growing" / "Monitor" / "Early"   → amber
// "Below" / "At risk"               → red
```

> Store all `target` values in `config.json`. Re-run this function on every data refresh — never cache tag strings.

</details>

<details>
<summary><strong>GSC position badge colour</strong></summary>

```js
function posBadge(pos) {
  if (pos <= 5)  return "badge-up";    // green  — Top rank
  if (pos <= 10) return "badge-warn";  // amber  — Opportunity zone
  return "badge-down";                 // red    — Quick-win target
}
```

</details>

<details>
<summary><strong>Branded vs non-branded query classification</strong></summary>

```js
// Store in config.json — update when new brands launch
const brandTerms = ["impacteers", "futurejobs", "xooper", "genrichers"];

function isBranded(query) {
  return brandTerms.some(t => query.toLowerCase().includes(t));
}

// Aggregate after classification
rows.forEach(r => {
  const bucket = isBranded(r.query) ? branded : nonBranded;
  bucket.impressions += r.impressions;
  bucket.clicks      += r.clicks;
});
```

> No native branded/non-branded field exists in GSC. This split must run entirely client-side.

</details>

<details>
<summary><strong>LLM Citation channel filter</strong></summary>

```js
// Store in config.json — update quarterly as new LLM tools emerge
const llmDomains = [
  "chat.openai.com",
  "perplexity.ai",
  "gemini.google.com",
  "claude.ai",
  "you.com",
  "copilot.microsoft.com",
  "bing.com/chat",
  "kagi.com"
];

// GA4 API dimensionFilter for parallel LLM call:
{
  andGroup: [
    { fieldName: "sessionMedium", stringFilter: { value: "referral" } },
    { fieldName: "sessionSource", inListFilter: { values: llmDomains } }
  ]
}
```

> Run as a **separate parallel GA4 call** — do not mix into the main channel query.

</details>

<details>
<summary><strong>Organic only toggle filter scope</strong></summary>

```js
// When organic toggle = ON, GA4 dimensionFilter becomes:
{
  orGroup: [
    { fieldName: "sessionMedium", stringFilter: { value: "organic" } },
    { fieldName: "sessionSource", inListFilter: { values: llmDomains } }
  ]
}

// Includes:  sessionMedium = organic  OR  sessionSource in llmDomains
// Excludes:  direct, cpc, paid, all other referral (unless LLM source)
```

> LLM referrals are included because they represent intent-driven organic discovery.

</details>

<details>
<summary><strong>CTR opportunity score</strong></summary>

```js
const BENCHMARK_CTR  = 0.05;  // 5% — configurable in config.json
const MIN_IMPRESSIONS = 500;  // configurable in config.json

function opportunityScore(row) {
  return row.impressions * Math.max(0, BENCHMARK_CTR - row.ctr);
}

// Apply filter + score + sort
const quickWins = rows
  .filter(r =>
    r.position >= posMin &&
    r.position <= posMax &&
    r.impressions > MIN_IMPRESSIONS
  )
  .map(r => ({ ...r, score: Math.round(opportunityScore(r)) }))
  .sort((a, b) => b.score - a.score);

// Higher score = more clicks left on the table at 5% CTR benchmark
// Rows with negative score (already above benchmark) are filtered out
```

</details>

<details>
<summary><strong>GJE segment isolation</strong></summary>

```js
// GA4 buckets Google Jobs Apply into Organic Search by default.
// Must be extracted BEFORE channel grouping or Organic numbers are inflated.

// Step 1: Run a parallel GA4 query filtered to:
//   sessionSourceMedium = "google_jobs_apply / organic"

// Step 2: Subtract GJE sessions from Organic Search total

// Step 3: Add GJE as its own separate segment in the channel mix donut
```

</details>

<details>
<summary><strong>Domain filter re-query</strong></summary>

```js
const domainMap = {
  all:         null,  // no filter applied
  main:        "www.impacteers.com",
  futurejobs:  "futurejobs.impacteers.com",
  recruitment: "recruitment.impacteers.com",
  examprep:    "examprep.impacteers.com"
};

// Add to GA4 dimensionFilter on dropdown change:
{
  fieldName:    "hostname",
  stringFilter: { matchType: "EXACT", value: domainMap[selectedDomain] }
}

// Debounce all filter changes at 300ms to avoid quota waste
```

> All 4 domains must share **one GA4 property** with cross-domain tracking enabled.

</details>

---

## config.json Structure

```json
{
  "targets": {
    "sessions":        20000,
    "engagementRate":  0.50,
    "avgPosition":     5.0,
    "gjeSessions":     1800,
    "llmSessions":     500,
    "newUserPct":      0.88
  },
  "alertThresholds": {
    "sessionsDropPct":    -0.20,
    "engagementRateMin":   0.25,
    "avgPositionMax":     12.0,
    "gjeSessionsDropPct": -0.15
  },
  "brandTerms": ["impacteers", "futurejobs", "xooper", "genrichers"],
  "llmDomains": [
    "chat.openai.com", "perplexity.ai", "gemini.google.com",
    "claude.ai", "you.com", "copilot.microsoft.com"
  ],
  "opportunityFilter": {
    "benchmarkCTR":    0.05,
    "minImpressions":  500
  }
}
```

> Non-devs can update targets and thresholds here without touching any JS code.

---

## API Setup

### Step 1 — Google Cloud Project

```bash
# 1. Go to https://console.cloud.google.com
# 2. Create project: "Impacteers-KPI-Dashboard"
# 3. Enable APIs:
#    - Google Analytics Data API
#    - Google Search Console API
```

### Step 2 — Service Account

```bash
# IAM & Admin → Service Accounts → Create
# Name: impacteers-kpi-sa
# Download JSON key → store securely (never commit to git)
export GOOGLE_APPLICATION_CREDENTIALS="/path/to/impacteers-kpi-sa.json"
```

### Step 3 — Grant Access

```
GA4:  Admin → Property Access Management → Add service account email → Viewer
GSC:  Settings → Users and Permissions → Add service account email → Full User
      (Repeat for all 4 properties)
```

### Step 4 — Core GA4 Request

```json
POST https://analyticsdata.googleapis.com/v1beta/properties/{PROPERTY_ID}:runReport

{
  "dateRanges": [
    { "startDate": "7daysAgo", "endDate": "today" },
    { "startDate": "14daysAgo", "endDate": "8daysAgo" }
  ],
  "dimensions": ["date","hostname","sessionDefaultChannelGroup","pagePath","city"],
  "metrics": ["sessions","totalUsers","newUsers","engagedSessions",
               "engagementRate","bounceRate","averageSessionDuration"],
  "dimensionFilter": {
    "filter": {
      "fieldName": "hostname",
      "inListFilter": { "values": ["www.impacteers.com","futurejobs.impacteers.com",
                                    "recruitment.impacteers.com","examprep.impacteers.com"] }
    }
  },
  "limit": 1000
}
```

### Step 5 — Core GSC Request

```json
POST https://www.googleapis.com/webmasters/v3/sites/{SITE_URL}/searchAnalytics/query

{
  "startDate": "2026-03-25",
  "endDate":   "2026-04-01",
  "dimensions": ["query","page","device","country"],
  "searchType": "web",
  "rowLimit":   500
}
```

> Run for each of the 4 verified URL-prefix properties separately.

---

## KPI Benchmarks (Mar 2026 Baseline)

| KPI | Current | Target | Alert if | Status |
|---|---|---|---|---|
| Sessions (weekly) | 17,650 | 20,000 | < 14,000 | 🟡 Growing |
| Engagement Rate | 37.7% | 50% | < 25% | 🟡 Below target |
| GSC Avg Position | 7.4 | < 5.0 | > 12.0 | 🟡 Monitor |
| GJE Sessions | 1,265 | 1,800 | < 900 | 🟢 Healthy |
| LLM Referral | 280 | 500 | < 100 | 🟡 Early stage |
| New User % | 95.2% | > 88% | < 80% | 🟢 On track |
| Non-Branded CTR | ~3% | > 5% | < 1.5% | 🟡 Optimise |

---

## Implementation Checklist

### Phase 1 — Setup (Week 1)
- [ ] Create GCP project, enable GA4 Data API + Search Console API
- [ ] Create Service Account, download JSON key
- [ ] Add service account to GA4 as Viewer
- [ ] Add service account to all 4 GSC properties as Full User
- [ ] Verify all 4 sub-domains as URL-prefix properties in GSC
- [ ] Test GA4 API call — confirm sessions return for all 4 hostnames
- [ ] Test GSC API call — confirm clicks/impressions/position per site
- [ ] Push to GitHub + enable GitHub Pages — confirm live URL

### Phase 2 — Data Layer (Week 1–2)
- [ ] Build GA4 fetch: weekly traffic, channel mix, top pages, engagement
- [ ] Build GSC fetch: weekly summary, query table, opportunity filter
- [ ] Implement WoW delta with two-dateRange GA4 call
- [ ] Build LLM Citation parallel GA4 call
- [ ] Build GJE segment extraction before channel grouping
- [ ] Build branded/non-branded client-side classification
- [ ] Implement JSON cache + Monday 6 AM IST n8n cron

### Phase 3 — UI Wiring (Week 2–3)
- [ ] Wire all 4 filter controls to GA4 params with 300ms debounce
- [ ] Replace all Chart.js dummy data arrays with live API responses
- [ ] Wire position range filter to `filterOpportunities()` with live GSC data
- [ ] Compute all status tags, WoW deltas, opportunity scores from live data

### Phase 4 — Alerts + Distribution (Week 3–4)
- [ ] Define alert thresholds in `config.json`
- [ ] Connect alerts to n8n Telegram bot
- [ ] End-to-end test with live data
- [ ] Update live GitHub Pages URL in dashboard topbar

### Update the dashboard with the live URL

Once you have the URL, open `index.html` and find this line in the topbar:

```html
<span class="badge-live">● Dummy Data</span>
```

Update it to include your live link, then push again:

```bash
git add index.html
git commit -m "chore: add live GitHub Pages URL to topbar"
git push
```

### Updating the dashboard later

Every time you make changes to `index.html`, just run:

```bash
git add index.html
git commit -m "update: describe what changed"
git push
```

The live URL updates automatically within 60 seconds.

---

## Security Notes

| Scenario | Recommendation |
|---|---|
| Dummy data version (current) | Safe to push to public repo |
| Live API version | Use a **private repo** or backend proxy |
| API credentials | **Never** commit the JSON key file to git |
| `.gitignore` | Add `*.json` or specifically `impacteers-kpi-sa.json` |

```bash
# Add this to .gitignore before pushing live API version
echo "impacteers-kpi-sa.json" >> .gitignore
echo "config.json" >> .gitignore  # if config contains sensitive values
git add .gitignore
git commit -m "chore: add gitignore for credentials"
```

---

## Rate Limits

| API | Limit | Strategy |
|---|---|---|
| GA4 Data API | 50,000 tokens/day, 10 concurrent requests/property | Cache responses, refresh weekly |
| GSC Search Analytics | 200 requests/day per site, 25,000 rows/response | Paginate with `startRow` if needed |

---

## Tech Stack

| Layer | Technology |
|---|---|
| Dashboard UI | HTML + CSS + Poppins (Google Fonts) |
| Charts | Chart.js 4.4.1 (CDN) |
| Data Source 1 | Google Analytics 4 Data API v1beta |
| Data Source 2 | Google Search Console API v3 |
| Auth | Google Cloud Service Account |
| Hosting | GitHub Pages (free) |
| Automation | n8n (scheduled cron + Telegram alerts) |
| Cost | INR 0 |

---

## File Structure

```
impacteers-kpi-dashboard/
├── index.html        # Full dashboard — all sections, charts, filter logic
├── README.md         # This file — dev reference and setup guide
├── config.json       # Thresholds, targets, brandTerms, llmDomains (DO NOT commit live values)
└── .gitignore        # Exclude credentials and sensitive config
```

---

## Excluded (Future Phase)

These metrics are reserved for the next instrumentation phase:

- **Key Events / Conversions** — activate when sign-up and apply events are instrumented in GA4
- **Revenue / eCommerce** — activate when course purchase tracking is live

---

*Impacteers / Genrichers Innovations Pvt Ltd — Chennai*
*Dashboard v4.0 — April 2026 — GA4 + GSC Direct API*
*Crafted by — Girish Kumar G | Programmatic SEO Manager*
