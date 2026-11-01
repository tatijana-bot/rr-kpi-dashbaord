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

## Git
- Container is ephemeral; re-clone with
  `git clone https://github.com/tatijana-bot/rr-kpi-dashbaord.git`.
- Commit + push each change. End commit messages with the attribution footer the session
  provides.
