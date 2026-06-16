# Account Timeline — Replication Guide

How the MadCap "Account Timeline" interactive page was built, so it can be rebuilt for any client.

**Live example:** https://singlegrain-team.github.io/clients/madcap/account-timeline.html
**Built by:** Stephano (local, via Claude Code) — June 2026
**What it is:** a single self-contained HTML file. One horizontal Chart.js chart (one bar per day), a draggable scrubber/playhead, and a ribbon of dated event markers. Scrolling the chart moves a "NOW VIEWING" playhead; the active event card updates on the left. No backend, no build step — open the `.html` and it runs.

It answers the question Jessica asked on the 6/3 call: *show me, on a timeline, what changed in the account and what happened to traffic right after.*

---

## 1. Data — what feeds the chart

Three daily series, each one array, one value per day, aligned to the same date axis (`Apr 1 → Jun 4 2026`, 65 days). They live as plain JS constants at the top of the `<script>` block:

```js
const CLICKS = [251,236,206, ...];  // daily clicks, Google Ads account level
const CONV   = [24,18,17, ...];     // daily conversions, account level
const MQL    = [2,2,0, ...];        // daily paid-search MQLs from HubSpot ("Date entered MQL")
const START  = new Date(2026,3,1);  // axis anchor — month is 0-indexed (3 = April)
```

The arrays must all be the same length. `START` + index → the date label. If a series is unavailable, leave it `null` per day (`spanGaps:true` is set on the MQL line so gaps don't break it).

### Where each series comes from

| Series | Source | How to pull |
|---|---|---|
| **CLICKS, CONV** (account-level daily) | Gateway: `googleads_account_performance` | Account `6333716782`, daily granularity, the window you want. For per-campaign breakdowns also use `googleads_campaign_performance`. |
| **MQL** (paid-search, daily) | **Client's HubSpot** (not the SG one) | See section 2. The gateway HubSpot tools only reach SingleGrain's portal, so this is a manual export. |
| **Brand auction metrics** (impr. share, top-of-page, CPC) used in event copy | Google Ads UI / the Brand slide from the 6/3 call | Not all of these are in the reporting API; pull from the platform. |

> **Change History is NOT in the gateway.** `change_event` is not exposed (it was tried 4× and failed). To get exact dates for structural changes (budget scaling, negative-keyword lists, conversion-goal swaps), export from the UI: **Google Ads → account 6333716782 → "Changes" → filter date range → export CSV.** Those rows are flagged in the event copy with a `⚑` until the CSV is pulled.

---

## 2. HubSpot MQL pull (the part that trips people up)

This is documented in full in the companion file **`MadCap - Account Timeline + HubSpot Pull Kit.html`** (sections 1 & 2). The short version:

1. **Never filter by current `Lifecycle Stage = MQL`** — that only shows leads *currently* sitting at MQL. A lead that became MQL then advanced to SAL disappears from the count. Always count by the **date the lead ENTERED the stage**.
2. HubSpot → **Reports → Create report → Single object → Contacts**.
3. Date filter on the entry-date property:
   - MQL → `Became a Marketing Qualified Lead Date`
   - SAL → `Became a Sales Accepted Lead Date`
   - SQL → `Became a Sales Qualified Lead Date`
   - Opportunity → `Became an Opportunity Date`
4. **Break down by `Original Source Drill-Down 2`** (`hs_analytics_source_data_2`) — this field carries the `utm_campaign`, so it's how you split by campaign.
5. Isolate channel with `Original Source Drill-Down 1`: `google` / `bing` / `linkedin`.

The **UTM → campaign decoder** (which `utm_campaign` string maps to which Google Ads campaign) is the table in section 2 of the Pull Kit, built from `campaign.tracking_url_template` on the account. Watch-outs captured there: Brand is split across two UTMs (sum both), and Consolidated-EU's final-URL-suffix overrides the template with the campaign ID.

---

## 3. The page — structure of the HTML

One file, three external CDN links only (Archivo font + Chart.js 4.4.1). Everything else is inline `<style>` and `<script>`.

```
<head>   Archivo font, Chart.js 4.4.1 CDN, all CSS inline (design tokens in :root)
<body>
  #prog        scroll progress bar
  .topbar      sticky client/watermark header + "fresh" pill
  .stagewrap   the interactive zone:
    .playhead    fixed "NOW VIEWING" dashed line at 24% of viewport
    .chartlane   <canvas id="chart"> — width = N days × 46px, so it scrolls horizontally
    .ribbon      one .mk marker per event, positioned by chart pixel coords
  .detail      the event card that re-renders as you scroll (left side)
  <script>     DATA arrays → Chart config → EVENTS array → scrubber wiring
```

### Chart config (Chart.js)
- `type:'bar'` for CLICKS (one bar per day, `dayW = 46px`, total width `N*dayW` → horizontal scroll).
- Two `type:'line'` overlays on a second y-axis (`y1`): CONV (solid black) and MQL (blue dashed).
- Bars after the "crash" index (`CRASH = 36`, = May 7) are colored red instead of tan — a hard-coded visual cutover marking where traffic stepped down.

### Events model
`EVENTS` is an array of factual annotations. Each:

```js
{ n:14, di:43, when:'May 14+', type:'crit', t:'Brand auction turns: lost to rank, CPC spikes',
  b:'From the week of May 11 the brand auction turned against us...',   // body, factual prose
  c:['Lost to rank 23%→62%','CPC ~$10→$65', ...],                       // chips
  bad:[true,true,true,false],                                           // which chips render red
  flag:'⚑ Auction detail in the impression-share table below.' }        // optional ⚑ for unconfirmed
```

- `di` = **chart day index** — places the marker on the right day. This is the only thing tying an event to the timeline; get it right.
- `type` drives color: `launch / crit / change / ext / plan / read` (palette in `T{}` / labels in `TL{}`).
- `bad` / `g` arrays color individual chips red/green.
- `flag` text appears when the data point still needs platform confirmation (Change History export).

### Scrubber wiring (bottom of script)
- `place()` computes each event's pixel x from `chart.scales.x.getPixelForValue(e.di)`, positions markers, sets a half-viewport pad on both ends so the scroll runs end-to-end.
- `onScroll()` finds the event whose x is nearest `scrollLeft` and sets it active → re-renders the card.
- Arrow keys + prev/next buttons call `goTo(idx)`; vertical wheel is translated to horizontal scroll inside the stage.
- Recomputes on `resize`.

To adapt: swap the three data arrays, set `START` and `CRASH`, rewrite `EVENTS`. The chart/scrubber code needs no changes.

---

## 4. Client-facing framing rules (do not skip)

These are MadCap-specific and were enforced on the live version:

- **Facts only, neutral.** Event copy states what the data shows ("clicks moved from ~250 to 71") and never editorializes blame. Anything not confirmed by platform data carries a `⚑` and points to where to confirm it.
- **MadCap funnel vocabulary:** MQL / SAL / SQL / Opportunity / DQ only. Never present "handraisers" as a separate metric — Jessica complained about that on a call.
- **Restructure timing:** the rebuild went live in **May 2026**. Object *creation* dates (Feb/Mar) and rebuild *go-live* (May) are deliberately separated — copy must not imply the structure was established or "held through" the auction shift.
- **Name spellings:** Paligo (not Polygo), Syndicate (not Xyleme), Laurie (not Lori).
- **Positive/shared framing** where the account had problems — report the data, let it speak.

---

## 5. Publishing (how it goes live)

The page is served by GitHub Pages from the **`SingleGrain-Team/clients`** repo (this is the same repo that hosts the TLC pacing pages and the Hyperdrive funnel page):

```
repo:  SingleGrain-Team/clients   (default branch: main)
path:  madcap/account-timeline.html
URL:   https://singlegrain-team.github.io/clients/<client>/<file>.html
```

So `madcap/account-timeline.html` in that repo → `singlegrain-team.github.io/clients/madcap/account-timeline.html`. To publish a new client timeline, drop `clients/<client>/<name>.html` into that repo and push. (Filenames are referenced in call scripts — pick a stable name and don't rename it.)

---

## 6. What's still open on the MadCap one

1. Export the Google Ads **Change History** (1/Feb → today) to put exact dates on the 4 red-flagged structural events.
2. Pull HubSpot MQL/SAL/SQL/Opp/DQ for May **by campaign** using sections 1–2 of the Pull Kit.
3. Optional: add Microsoft + LinkedIn UTMs/performance for a multi-channel timeline.
4. Tracking fix for Luiz: Consolidated-EU sends the campaign ID instead of the UTM name.

---

### Files
- `account-timeline.html` — the page (this guide describes how it's built)
- `MadCap - Account Timeline + HubSpot Pull Kit.html` — the data-pull companion (HubSpot paths + UTM decoder + change log table)
