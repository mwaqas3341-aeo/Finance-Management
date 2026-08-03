# Finance-Management — Performance Audit & Optimization Report

## Scope note (please read first)

The GitHub repo (`mwaqas3341-aeo/Finance-Management`) contains **only the
front-end**: a single 1,277-line `index.html`. All data logic — Sheets
reads/writes, auth, aggregation — lives in a Google Apps Script Web App
(`script.google.com/macros/s/.../exec`) that is **not in this repo** and
isn't reachable from here (it's tied to your Google account). That means:

- I audited and optimized everything on the client side directly.
- Everything about "Google Apps Script API calls" and "Sheets read/write
  efficiency" (Phase 1) is diagnosed *indirectly*, from call patterns and
  known Apps Script behavior — I couldn't read `Code.gs`. **This is very
  likely where most of your load-time actually goes.**
- I can't push commits to your GitHub repo (no write access from here) or
  make GitHub Actions run on a real schedule myself. What I've delivered
  is the code + workflow file — you (or a "paste this in" pass with me)
  still need to add it to the repo and to the Apps Script project.

## Phase 1 — What's actually slow

**1. Apps Script cold starts + per-request Sheets overhead (biggest cost, unverified — not in repo).**
Every `api()` call is a fresh HTTPS round trip to a `script.google.com/.../exec`
endpoint. Apps Script Web Apps routinely add 1–4s of cold-start latency, and if
the server code calls `SpreadsheetApp.openById()` fresh each request, or reads
cell-by-cell instead of one `getDataRange().getValues()`, each call gets much
slower again. I can't confirm this without the server code, but it's the
single most common cause of "Apps Script app feels slow," and it fits your
symptoms (long loads, worse than the amount of data would suggest).

**2. No client-side caching at all.**
Every tab switch re-fetches from scratch: opening Settings always calls
`getCategories` again, opening Dashboard always calls `getDashboardData`,
`getMilkDashboardData`, `getMilkRecords` again — even if you were just there
30 seconds ago and nothing changed.

**3. Page-load sequence was partly serialized.**
`initApp()` awaited `getUserSettings`, *then* `getCategories`, *then*
(dashboard + milk calls nested inside `refreshDashboard`, one after another).
Several of these calls don't depend on each other and can run concurrently.

**4. The red-screen bug.**
`api()`'s error handler did:
```js
if (!resp.ok) { const text = await resp.text(); throw new Error(`HTTP ${resp.status}: ${text}`); }
```
When a request 404s (e.g. against a not-yet-created GitHub-hosted JSON file,
or a misconfigured/expired Apps Script deployment), `text` can be an entire
HTML error page. That whole page then gets stuffed into a `toast(...)` and
rendered — which is exactly the giant unreadable red block in your
screenshot. This is fixed in the delivered file (see below).

## What was changed in `index.html` (delivered)

1. **Response cache + in-flight de-dupe** for all read-only actions
   (`getUserSettings`, `getCategories`, `getRecurringExpenses`,
   `getDashboardData`, `getTransactions`, `getMilkDashboardData`,
   `getMilkRecords`), with short TTLs (20s for anything date-range/live-ish,
   5 min for settings/categories). Any write action clears the cache
   automatically, so you never see stale data after saving/editing/deleting.
2. **Parallelized page load**: `getUserSettings` and `getCategories` now
   fire together instead of sequentially; inside `refreshDashboard()`, the
   dashboard, milk-dashboard, and milk-records calls now fire together
   instead of dashboard-then-milk.
3. **Fixed the error handler** so a failed request shows a short, sanitized
   message (`Server error (HTTP 404). Please try again.`) instead of
   dumping the raw response body to the screen.

These are conservative, behavior-preserving changes — no feature was
removed or altered, only request timing and error display.

## Phase 2–4 — GitHub-hosted data + nightly sync

This is feasible for read-mostly reference data, with a caveat: it requires
changes on the Apps Script side that I can't make for you (no access), so
I've delivered the pieces to add yourself:

- `github-sync/apps-script-export-snippet.gs` — a new `exportForGithub`
  action to add to your Apps Script `doPost` dispatcher. Protected by a
  separate secret (Script Property), not your per-user auth token.
- `github-sync/nightly-sync.yml` — a GitHub Actions workflow (cron, roughly
  11:30 PM UTC — **adjust for your timezone**, GitHub cron is UTC-only and
  "best effort," can slip by several minutes) that calls that action and
  commits `data/reference.json` to the repo.
- `github-sync/client-fallback-snippet.js` — how `index.html` would read
  that JSON, with a **silent fallback** to the live Apps Script call if the
  file is missing or the fetch fails. This directly prevents a repeat of
  the red-screen bug for this new code path.

**What stays live (Phase 4), unchanged:** dashboard summaries, settings,
categories-as-edited, and anything the user just changed — those keep
calling Apps Script directly. Only slow-changing reference data (categories
list, recurring expenses list, historical monthly summaries) is a good fit
for nightly static export; per-user live transactions and today's numbers
should not be baked into a nightly file.

**Honest limitation:** I did not enable this end-to-end — that needs you to
(a) paste the `.gs` snippet into your Apps Script project and fill in your
actual sheet-reading logic, (b) add a Script Property secret, (c) add the
matching repo secret, (d) commit the workflow file, (e) enable Actions if
not already on. Happy to walk through any of those steps with you.

## Before/after benchmarks

I don't have a way to hit your live Apps Script endpoint or measure real
network timing from here, so I can't give you real before/after numbers —
anything I typed here would be made up. If useful, I can write you a tiny
benchmarking snippet (times each `api()` call in the console) to run
yourself before/after deploying, and you can paste the numbers back to me.

## Files delivered

- `index.html` — optimized front-end (drop-in replacement)
- `PERFORMANCE_REPORT.md` — this report
- `github-sync/apps-script-export-snippet.gs`
- `github-sync/nightly-sync.yml`
- `github-sync/client-fallback-snippet.js`
