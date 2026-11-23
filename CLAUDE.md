# Reputation Resolutions — KPI Dashboard

Single static `index.html` (dark theme) that reports KPIs for Reputation Resolutions, an
online reputation-management company. Data is pulled live from **HubSpot CRM (portal
21116307)** and **monday.com**, then hand-written into the HTML. Deployed via GitHub Pages
from `main`. There is no build step — edit `index.html` directly.

## Operating rules (HubSpot) — do not break these
- **Never bulk-enroll existing records into a workflow without asking first.**
- **Turn workflows OFF, never delete them.**
- **Relabel property options, never delete them.**
- When closing a deal, a **Closed Lost Reason is required** (conditional-logic rule on the
  Closed Lost stage in the Sales, Outreach, and Cold Leads pipelines).

## Closed Lost → Cold Leads system (set up 1 Nov 2026)
**Auto-close of unreached inbound leads** — three contact workflows, same logic: wait 14
days, then a "reached, booked or won" check (meeting booked known, OR Recent Deal Close Date
known, OR an associated call outcome = Connected / Meeting booked). If nothing matched, set
the deal to Closed Lost with a reason.
- `1885443849` Hot & Warm – close if never reached → **Unresponsive**
- `1880158781` Stalled – task then email cadence → **Unresponsive**
- `1882115139` Cold – email then close:
  - Lead Status = Unqualified (set by website wizard) → closes immediately, reason **"Not a
    service we offer"** (internal value `Missing key feature`), no email.
  - Other Cold → sequence or nothing, then 14 days → reason **Low Quality Lead**.
- "Price too high" is **no longer** set automatically.

**Recycling** — `1914938041` "Closed Lost 90 days → Cold Leads (Identified)" (deal-based, ON):
- Trigger: stage = Closed Lost (Sales or Outreach) AND reason in {Unresponsive, Not the right
  time, Price too high, Lost to competitor}.
- Waits 90 days; skips if reopened or contact Lead Tier = Do Not Book; else moves deal to
  **Cold Leads Pipeline → Identified (stage 62397017)**.
- Never recycles: Low Quality Lead, Not a service we offer, Other, or no reason.
- 99 existing matching deals were hand-enrolled 1 Nov 2026; they move ~30 Jan 2027.
- Stale Outreach deals (90 days no activity) also land in Cold Leads → Identified via
  `1712878950`.

**Inside Cold Leads**
- `1914938496` "meeting booked → Call Scheduled" (deal-based, ON): a deal in Cold Leads
  Identified/Outreach/Nurturing with an associated meeting starting after yesterday moves
  (that deal only) to **Cold Leads → Call Scheduled (62397018)**. Works with any meeting link.
- Old `369521483` is OFF (fired on two retired links, moved every deal on the contact).
- Cold Leads deals that reach Closed Lost do **not** recycle again — final.
- Rep path: Identified → Outreach (when work starts) → Nurturing / Call Scheduled (auto) /
  Closed Lost with a reason.

## HubSpot query notes (`query_crm_data`)
- The tool parameter is **`sql`** (not `query`). Missing it returns "[sql] Missing required field".
- Unsupported: `DISTINCT`, `AS` aliases, `CASE WHEN`, string funcs, `COALESCE`, `HAVING`,
  `JOIN`/subqueries, `associations.OBJECT` in SELECT. One object type per query.
- Use `COUNT(*)` (not `COUNT(hs_object_id)` — that sums the IDs). For money use
  `amount_in_home_currency`.
- Only one `DATE_TRUNC` per query; for multiple date filters use range comparisons
  (`>= '2026-10-01' AND < '2026-11-01'`).
- Stage-entry dates: `hs_v2_date_entered_<stageid>`.

### "Calls Booked" metric — use this, nothing else
- **Definition:** a deal with the **`call_outcome`** property set (Showed / No Show / Cancelled /
  Rescheduled), bucketed by **`createdate`** month. This is the team-maintained source and it
  reproduces the historical dashboard exactly (Mar = 166, Apr = 128).
  `SELECT DATE_TRUNC(createdate,'MONTH'), call_outcome, COUNT(*) FROM DEAL WHERE call_outcome IS NOT NULL AND createdate >= '<m-start>' AND createdate < '<m-end>' GROUP BY DATE_TRUNC(createdate,'MONTH'), call_outcome`
- Show rate = Showed &divide; (Showed + No Show). Report Calls Booked + Showed/No-Show/Canceled.
- **Do NOT** count "deals entered the Call Scheduled stage" (`hs_v2_date_entered_9550124` etc.) —
  that undercounts ~6&times; (it gave the bogus "8" for Nov, "48" for Oct). And do NOT use the raw
  MEETING object (counts every meeting incl. service/internal; ~2&times; too high).
- `call_outcome` is logged thoroughly on Sales-pipeline deals but **sparsely on Outreach**, so a
  per-pipeline split of booked calls is unreliable — keep headline counts monthly, not by pipeline.
- 2026 monthly series (Calls Booked): Jan 145, Feb 142, Mar 166, Apr 128, May 135, Jun 88,
  Jul 68, Aug 163, Sep 126, Oct 157, Nov(MTD d14) 54. YTD 1,372 · 82.9% show rate.

### Pipelines & key stage IDs
- Sales (default): Call Scheduled `9550124`, Contract Sent `decisionmakerboughtin`,
  Qualified `appointmentscheduled`, closed won `closedwon`, closed lost `closedlost`.
- Outreach (`91135766`): Call Scheduled `168535628`, Closed Won `168535631`.
- Cold Leads (`27377755`): Identified `62397017`, Call Scheduled `62397018`.
- Contact lead-source property: `outreach_lead_source` (label "RR Lead Source").
  Buckets: Inbound = Inbound Website + Phone Call - Inbound Website; Outreach = Trustpilot /
  Google Review / Glassdoor / Referral; Sub-Brand = RemoveNews.ai + Courtrecordremoval.com.
  "Unassigned" = untagged, exclude from lead mix.

## monday.com key boards
- **02 Invoices** `18421436500` (ChargeOver ledger): Amount `numeric_mm547fr4`, Paid-to-Date
  `numeric_mm7ejn7v`, Status `color_mm54y56c`, Paid Date `date_mm548kcs`, Invoice Date
  `date_mm546txw`, Billing Type `color_mm57y0vd`, HubSpot Deal ID `text_mm54yyeb`,
  Payment History `text_mm7e6p54`. Cash rule: Voided/Refunded = 0; else use Oct Paid Date +
  Paid status → Paid-to-Date (if filled) else Amount.
- **Reviews / removals** `9996609606`: Overall Status `color_mkwjdnxc`, Platform
  `color_mkwj79a7`, Final Decision `date_mkxw79tz`. Success% = Successful ÷ (Successful+Denied).
- Pulls are large; use `items_page(limit:500)` + `next_items_page(cursor)`; aggregate in a
  subagent and return compact numbers.

## Dashboard structure
- Overview scorecard (default landing) + a Month dropdown + quarter tabs + deep-dive tabs
  (Retention, Service Delivery, Recurring Revenue, Sales Funnel, Collections/AR, Pipeline).
- Components: `.kpi-row`/`.kpi-card`, `.card`/`.card-title`/`.dot`, `.split-bar`/
  `.split-segment`/`.split-legend`, inline SVG charts, `.ov-card` (clickable).
- **Before committing**, validate div balance:
  `python3 -c "s=open('index.html').read(); print(s.count('<div')-s.count('</div>'))"` → 0.
- Revenue headline uses **revenue-producing deals** (amount > 0), split by pipeline; $0 and
  test ("ZZ TEST") deals are excluded from the headline and noted separately.
- **Always label which revenue a figure is, using these two terms consistently:**
  **Deal Amount** = closed-won contract value (`amount_in_home_currency`, by close date) vs
  **Realized Amount** = cash actually received (`realized_value_via_monday`, which lags).
  Never show a bare "revenue" number without saying which. Most dashboard headlines are
  Deal Amount; Realized Amount always lags and must be labelled as such.

## Paid Ads (Google Ads)
- There is a SEPARATE "Google Ads Tracker" artifact (owned by Tatijana, shared with org):
  `https://claude.ai/artifact/8WTwkKMjQESrxVGYmPqCfi`. It owns ALL ad detail — campaigns,
  clicks, impressions/CTR, CPC, conversions, and a "Trend" tab (monthly + week-1-vs-week-4
  click rate and its correlation with sales). That artifact has Google Ads access; THIS
  session does not (no Google Ads connector) — get ad-platform numbers from the user or that
  artifact, don't try to query Google Ads here.
- The KPI board carries ONLY a one-line **"Paid Ads (2026)"** Overview card (exec rollup)
  that links to the ads dashboard's **canonical URL (no `?sk=` share key — this page is
  public)**. Do NOT duplicate ad detail (weekly click rate, correlation, campaigns) on the
  KPI board — that stays on the ads dashboard. Split: exec rollup here, detail there.
- Attribution: ad-sourced = CONTACT `hs_analytics_source = 'PAID_SEARCH'`. The RR Lead Source
  option "Facebook Ad - Calendly" (`outreach_lead_source`) is EMPTY/unused — don't use it.
  Ad-sourced closed-won: `DEAL WHERE CONTACT.hs_analytics_source = 'PAID_SEARCH' AND
  hs_is_closed_won = true`, by `closedate`.
- 2026 card figures: **$14,336 spent -> $116,960 signed (contract value), 10 deals = 8.2x**.
  Lifetime 2021-2026: $86,526 spent -> $507,576 signed (8.5x). "Signed" = contract value by
  close date (THIS YEAR = 2026 closedate); cash actually collected lags (~$19,070 on 2026 ad
  deals via the `realized_value_via_monday` property, filled on only 5 of 10).

## Git
- Container is ephemeral; re-clone with
  `git clone https://github.com/tatijana-bot/rr-kpi-dashbaord.git`.
- Commit + push each change. End commit messages with the attribution footer the session
  provides.
