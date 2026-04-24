# Store Health Diagnostic Report — Master Prompt & System Guide

> **Purpose:** This file documents everything needed to reproduce, extend, or re-run the Store Health Diagnostic Report system built for Lenskart Offline Operations. Any future user or Claude session can read this file and immediately understand what to build, how to connect to the data, and what output to generate.

---

## 1. What this system does

This is an AI-powered **Store Health Diagnostic Report** that:

1. Connects to the **Store Insights** semantic model in the **Offline Operations** Power BI Fabric workspace via the PowerBI MCP server
2. Pulls live KPI data for any store and time period the user selects
3. Validates the data in chat before generating anything
4. Computes an **AI Health Score (0–100)** and section-level scores
5. Generates a full interactive HTML report with charts, collapsible sections, AI narratives, and an actionable playbook — all rendered inside Claude

---

## 2. How to trigger this system

When a user pastes this prompt or asks to "generate a store health report", immediately show **3 selection inputs** using the `ask_user_input` tool:

**Input 1 — Store (single select)**
Show all available stores from the list below. Allow search by code, name, city, or region.

**Input 2 — Time period granularity (single select)**
Options: Monthly / Weekly / Quarterly / Daily / Yearly

**Input 3 — Benchmark (single select)**
Options: vs Last month (this year) / vs Last year same month / vs Target

After the user selects granularity, show a **second input** for the specific period (e.g., if Monthly → show Apr-26, Mar-26, Feb-26…).

---

## 3. Known stores (as of Apr 2026)

These are the stores with validated master data. The model has 1,572+ active/new stores total.

| Store Code | Store Name | Region | State | Town | Tier | Format | ZM | AOM | Size (sqft) | Clinics | Rent/mo |
|---|---|---|---|---|---|---|---|---|---|---|---|
| LKST1838 | Chennai Pallavaram | South | Tamil Nadu | Chennai | Tier-I | COCO | Kapilavai Poorna Shankar | Mohan Kumar | 1000 | 2 | ₹1,20,000 |
| LKST1781 | Delhi Chattarpur | North | Delhi | New Delhi | Tier-I | COCO | Atul Jindal | Bhawna Tiwari | 1050 | 2 | ₹1,15,000 |
| LKST1401 | Hyderabad Kondapur | South | Telangana | Hyderabad | Tier-I | COCO | Katikam Shravan Kumar | Devender | 880 | 4 | ₹1,30,000 |
| LKST1437 | Hyderabad Vanasthalipuram | South | Telangana | Hyderabad | Tier-I | COCO | Naresh Negi | Sudhakar Avula | 1700 | 4 | ₹1,45,000 |
| LKST1818 | Navi Mumbai Ulwe | West | Maharashtra | Mumbai | Tier-I | COCO | Vipul Shetty | Amisha Shetty | 1000 | 3 | ₹1,35,000 |
| LKST1621 | Vadodara Waghodia | West | Gujarat | Vadodara | Tier-II | COCO | Suraj Thakur | Deven Dhakan | 800 | 2 | ₹90,000 |
| LKST1764 | Nagpur Jaripatka | West | Maharashtra | Nagpur | Tier-II | COCO | TBD-ROMG | Salim | 800 | 1 | ₹85,000 |
| LKST1502 | Bhatinda Bhucho Mandi | North | Punjab | Bhatinda | Tier-III | COCO | TBD RON 1 | Deepak Gupta | 1560 | 1 | ₹70,000 |
| LKST3026 | Mumbai Girgaon Chowpatty | West | Maharashtra | Mumbai | Tier-I | COCO | Vipul Shetty | Chandrakant Wagh | 350 | — | ₹1,80,000 |
| LKST4342 | Nashik Dwarka | West | Maharashtra | Nashik | Tier-II | COCO | Mohsin | Tushar Shelar | 380 | — | ₹75,000 |
| LKST4389 | Bhagalpur Tilkamanjhi | East | Bihar | Bhagalpur | Tier-III | COCO | Subroto Bej | Manoj Kumar Jha | 320 | — | ₹55,000 |

---

## 4. Connection details

**MCP Server:** PowerBI MCP server (already connected)
**Operation:** `ConnectFabric`
**Workspace:** `Offline Operations`
**Semantic Model:** `Store Insights`
**Connection name format:** `Fabric-Offline%20Operations-Store Insights`

Always reconnect at the start of each session — tokens expire.

---

## 5. Exact execution steps (follow in order, never skip)

### Step 1 — Reconnect
```
Operation: ConnectFabric
Workspace: "Offline Operations"
Semantic Model: "Store Insights"
```

### Step 2 — Pull store master
```dax
EVALUATE FILTER(Dim_Store_Master, Dim_Store_Master[store code] = "<STORE_CODE>")
```

Extract: store name, store code, Format_detail, region, state, store_town, city_tier, size, aom, zm, clinic_count, store opening date, rent, circle_head, tango_status, Store_status, Store Maturity FYr_Group

### Step 3 — Determine benchmark period

| Benchmark selection | Logic |
|---|---|
| vs Last month (this year) | Immediately prior calendar month (Apr-26 → Mar-26) |
| vs Last year same month | Same month one year ago (Apr-26 → Apr-25) |
| vs Target | Use `proto` field from store master as revenue target |
| Weekly | Prior week (22-Apr-26 → 19-Apr-26) |
| Quarterly | Prior quarter (Q2-2026 → Q1-2026) |
| Daily | Same day prior week |

### Step 4 — Pull KPIs for BOTH periods (filtered by store AND date)

```dax
EVALUATE CALCULATETABLE(
  ROW(
    "Net Sales", [Net Sales],
    "Net Orders", [Net Orders],
    "Gross Orders", [Gross Orders],
    "Footfall", [Footfall],
    "Conversion", [Conversion],
    "Net ATV", [Net ATV],
    "Gross ATV", [Gross ATV],
    "Net Qty", [Net Qty],
    "Gross Qty", [Gross Qty],
    "PSPD", [PSPD (Net Sales)],
    "ET Wait Time", [ET Wait Time],
    "NOB per Staff", [NOB per Staff],
    "Prog Net Orders", [Prog Net Orders],
    "SV Net Orders", [SV Net Orders],
    "Eye Net Orders", [Eye Net Orders]
  ),
  dim_dynamic_date_master[Date-Parameter] = "<PERIOD>",
  Dim_Store_Master[store code] = "<STORE_CODE>"
)
```

Run this query **twice** — once for the selected period, once for the benchmark period.

### Step 5 — Compute derived KPIs

> **CRITICAL:** The model's `[Conversion]` measure uses a different internal footfall denominator. **Always discard `[Conversion]` from the model and compute manually:**

| Derived KPI | Formula |
|---|---|
| **Conversion %** | `Net Orders ÷ Footfall × 100` ← use this, not model's [Conversion] |
| UPT (Units per Transaction) | `Net Qty ÷ Net Orders` |
| Net ASP (Avg Selling Price per unit) | `Net Sales ÷ Net Qty` |
| Progressive share % | `Prog Net Orders ÷ (Prog Net Orders + SV Net Orders) × 100` |
| Return/cancel rate | `(Gross Orders − Net Orders) ÷ Gross Orders × 100` |
| Revenue per Staff (est.) | `NOB per Staff × Net ATV` |
| Conv loss orders | `(BM Conversion% − CUR Conversion%) ÷ 100 × CUR Footfall` |
| Conv loss revenue | `Conv loss orders × CUR Net ATV` |
| Revenue gap (abs) | `BM Net Sales − CUR Net Sales` |

### Step 6 — Show validation table in chat and wait for confirmation

Show a clean table with ALL raw and derived KPIs:

| KPI | DAX Measure / Source | Current Period | Benchmark | Var % | Abs Change | Status |
|---|---|---|---|---|---|---|

Status emoji logic:
- 🔴 Critical — >5% adverse
- 🟡 Watch — 2–5% adverse
- 🔵 On track — within ±2%
- 🟢 Healthy — >2% positive
- 🟢★ Improving — for inverted KPIs (ET wait time) where lower is better

Also show the store master fields.

**Ask the user:** "Do these numbers match what you see in Power BI? Confirm or flag any discrepancies before I build the report."

**Wait for user confirmation. Do not proceed to Step 7 without it.**

### Step 7 — Compute AI Health Scores

**Individual KPI scores (0–100):**

| Variance vs benchmark | Score |
|---|---|
| > +10% | 95 |
| +5 to +10% | 85 |
| 0 to +5% | 72 |
| -2 to 0% | 55 |
| -5 to -2% | 40 |
| -10 to -5% | 25 |
| < -10% | 10 |

**For inverted KPIs (ET Wait Time — lower is better):**

| Variance vs benchmark | Score |
|---|---|
| < -10% | 95 (great improvement) |
| -5 to -10% | 85 |
| 0 to -5% | 72 |
| 0 to +5% | 55 |
| +5 to +10% | 40 |
| > +10% | 25 |

**Section scores:**
```
Revenue score     = Net Sales×0.40 + Conversion×0.35 + ATV×0.25
Operations score  = ET Wait Time×0.60 + NOB per Staff×0.40
Merch score       = UPT×0.50 + ASP×0.30 + Conversion×0.20
Infra score       = 72 (contextual — adjusted by physical audit inputs)

Overall AI Health Score = Net Sales×0.30 + Conversion×0.25 + Footfall×0.15
                        + Net Orders×0.10 + ATV×0.10 + ET Wait Time×0.10
```

**Score labels:**
- 80–100 → Excellent
- 65–79 → Good
- 50–64 → Needs attention
- 35–49 → At risk
- 0–34 → Critical

### Step 8 — Generate AI narratives (3–4 sentences each, ZM-level briefing tone, specific numbers)

Write these 6 narrative sections from the data:

1. **Revenue & financial performance** — cover: primary revenue driver or gap, PSPD as the period-length-adjusted truth, correlation between footfall/conversion/ATV
2. **Correlation engine insight** — cover: which KPI pair is diverging, which funnel stage is leaking (footfall → conversion → ATV → revenue), what the data implies about in-store behaviour
3. **Operations & staffing** — cover: ET wait time direction and conversion impact, NOB per staff vs benchmark, estimated revenue impact of the staffing gap
4. **Operations section rating rationale** — justify the section score with exact numbers and weighted formula
5. **Merchandising & basket quality** — cover: UPT vs ASP pattern (upsell failure / cheap mix / healthy), Progressive share gap vs 25% target, what lens mix says about staff recommendation quality
6. **Merchandising section rating rationale** — justify with exact weighted formula

### Step 9 — Build the interactive HTML report (rendered inside Claude as a widget)

Use `visualize:show_widget` to render the full report. The widget must be fully self-contained with no external API calls at runtime.

---

## 6. Validated real data (use these exact numbers — confirmed against Power BI)

### LKST1838 — Chennai Pallavaram | Mar-26 vs Feb-26

**Mar-26 (current):**
```
Net Sales:    ₹9,87,657
Net Orders:   320.67
Footfall:     768
Conversion:   41.75%  ← derived: 320.67/768×100
Net ATV:      ₹3,080
PSPD:         ₹24,691/day
ET Wait Time: 4.185 min
NOB/Staff:    3.841
Net Qty:      595
Prog Orders:  45.584
SV Orders:    121.0
Eye Orders:   173.5
```

**Feb-26 (benchmark):**
```
Net Sales:    ₹7,55,468
Net Orders:   258.917
Footfall:     1,117
Conversion:   23.18%  ← derived: 258.917/1117×100
Net ATV:      ₹2,918
PSPD:         ₹20,985/day
ET Wait Time: 3.848 min
NOB/Staff:    3.383
Net Qty:      456
Prog Orders:  37.5
SV Orders:    94.233
Eye Orders:   148.517
```

**Derived KPIs:**
```
UPT (Mar):   1.856 vs 1.761 (+5.4%)
ASP (Mar):   ₹1,660 vs ₹1,657 (+0.2%)
Prog share:  27.36% vs 28.47% (-3.9%)
Return rate: 3.56% vs 3.75% (improving)
```

**Scores:** Overall 76 | Revenue 92 | Operations 62 | Merchandising 83 | Infrastructure 72

**The story:** Revenue up +30.7%, orders up +23.8% despite footfall down -31.2%. Conversion surged from 23.18% to 41.75% (+80.1%) — extraordinary but fragile. If footfall doesn't recover, revenue will collapse. ET wait worsening slightly (+8.8%) needs monitoring. Progressive share slipping from 28.47% to 27.36%.

---

### LKST1781 — Delhi Chattarpur | Mar-26 vs Feb-26

**Mar-26 (current):**
```
Net Sales:    ₹24,12,285
Net Orders:   974.05
Footfall:     4,362
Conversion:   22.33%  ← derived: 974.05/4362×100
Net ATV:      ₹2,477
PSPD:         ₹60,307/day
ET Wait Time: 8.975 min
NOB/Staff:    5.040
Net Qty:      1,584
Prog Orders:  122.97
SV Orders:    515.38
Eye Orders:   677.13
```

**Feb-26 (benchmark):**
```
Net Sales:    ₹24,66,219
Net Orders:   939.29
Footfall:     4,278
Conversion:   21.96%  ← derived: 939.29/4278×100
Net ATV:      ₹2,626
PSPD:         ₹68,506/day
ET Wait Time: 10.539 min
NOB/Staff:    5.212
Net Qty:      1,609
Prog Orders:  123.60
SV Orders:    480.65
Eye Orders:   641.29
```

**Derived KPIs:**
```
UPT (Mar):   1.626 vs 1.713 (-5.1%)
ASP (Mar):   ₹1,523 vs ₹1,533 (-0.6%)
Prog share:  19.26% vs 20.46% (-5.8%)
Return rate: 7.01% vs 7.62% (improving)
```

**Scores:** Overall 66 | Revenue 57 | Operations 77 | Merchandising 51 | Infrastructure 72

**The story:** More footfall (+2%), more orders (+3.7%), better conversion — but revenue is down -2.2%. The entire gap is ATV: ₹2,477 vs ₹2,626 = ₹149 less per bill × 974 bills = ₹1,45,126 value erosion. ET improved dramatically from 10.54 → 8.97 min. UPT down -5.1%, Progressive share 19.26% (below 25% target) — pure upsell failure.

---

## 7. Report structure (all 7 sections)

### Section 0 — Store header card
- Store name (large), code, region, state, town
- Badge row: Format (COCO/FOFO), Tier, Size sq ft, Period, Benchmark
- Meta grid: ZM, AOM, Clinics, Opened, Rent, Circle Head, Tango status, Store maturity
- AI Health Score ring (SVG, colour-coded: green ≥75, amber 55–74, red <55)

### Section 1 — Raw data validation table
Every KPI with: Name | DAX measure/derivation | Current value | Benchmark value | Variance % | Abs change | Status pill

### Section 2 — Revenue & financial performance
- Section rating badge (score/100 + label)
- 6 mini metric cards: Net Sales, Net Orders, Footfall, Conversion, Net ATV, PSPD
- Full KPI table with variance and status pills
- Revenue gap and PSPD trend detail rows
- Charts: Sales & Orders bar chart, Conversion & ATV bar chart
- AI narrative text
- Correlation engine insight box (purple left border)
- Rating rationale with exact weighted calculation shown

### Section 3 — Operations & staffing
- Section rating badge
- 4 mini metric cards: ET Wait Time, NOB/Staff, Revenue per Staff, ET Status
- KPI table + ET bottleneck status row
- Chart: ET wait time & NOB bar chart
- Editable textarea: "Grooming & discipline — add from store visit"
- AI narrative + rating rationale

### Section 4 — Merchandising & basket quality
- Section rating badge
- 4 mini metric cards: UPT, Net ASP, Progressive share %, Net Qty
- KPI table with Prog/SV/Eye orders breakdown
- Basket diagnosis row (UPT↓+ASP↓ = Upsell failure; UPT↑+ASP↓ = Cheap mix; both healthy = Basket OK)
- Progressive gap to 25% target row
- Charts: UPT & ASP bar chart, SV vs Progressive doughnut chart
- Editable textarea: "Display & VM freshness — add from store visit"
- AI narrative + rating rationale

### Section 5 — Store infrastructure
- Section rating badge
- 4 mini metric cards: Store size, PSPD, Sales/sqft/day, Clinics
- KPI table with PSPD, format, tier info
- Editable textarea: "Maintenance issues — add from store visit"
- Rating rationale

### Section 6 — Actionable playbook
- AI-generated badge
- 6–8 action items with:
  - 🔴 Immediate / 🟡 Today / 🔵 This week priority
  - Short title (max 8 words)
  - Category tag (ET queue, Lens upsell, Daily ritual, ZM visit, etc.)
  - 2–3 sentence body with specific numbers and named staff

**Playbook generation rules:**
- ET > 8 min → 🔴 Immediate: redeploy optometrist to peak hours, activate Teleoptom
- Conversion below benchmark → 🔴 Immediate: floor engagement coaching brief
- Progressive share < 22% → 🟡 Today: Progressive pull-through coaching
- Always → 🟡 Today: 9 AM morning huddle grooming check
- ATV below benchmark → 🟡 Today: premium planogram audit
- Always → 🔵 This week: 5-day coating attachment rate sprint
- Always → 🔵 This week: ZM store visit recommendation with health score

### Section 7 — AI Health Score weightage
- Methodology explanation
- Visual weight bars for 6 components
- Scoring band table
- Exact arithmetic shown: `55×0.40 + 72×0.35 + 40×0.25 = 57/100`

---

## 8. Key diagnostic logic

### Level 1 — Funnel check
- Footfall (opportunity) → Net Orders (success) → Conversion = Orders/Footfall×100
- If footfall ↑ but orders ↓ or flat → move to Level 2

### Level 2 — Why of conversion (two sub-paths)
**Sub-path A: Staffing/service**
- ET wait time > 8 min → Critical (billing/ET bottleneck)
- ET 6.5–8 min → Watch
- ET < 6.5 min → Normal

**Sub-path B: Merchandise**
- Serviceable fill rate below benchmark → Stock gap
- Progressive share < 22% → Lens mix issue

### Level 3 — Value check (basket quality)
- ATV = Net Sales / Net Orders
- UPT = Net Qty / Net Orders
- If UPT ↓ + ATV ↓ → Upsell failure (customers buying one cheap item)
- If UPT ↑ + ATV ↓ → Cheap mix (many items but all low-value/scheme)
- If UPT ↑ + ATV ↑ → Basket healthy

### Diagnostic scenarios

| Scenario | Pattern | Diagnosis |
|---|---|---|
| A — Conversion trap | Footfall ↑, Conversion ↓ | Billing/ET queues or stock-outs |
| B — Transaction size gap | Conversion ↑, ATV ↓ | Upselling failure or cheap lens mix |
| C — Demand collapse | All KPIs ↓ | Location/visibility/competition problem |
| D — Invisible store | Footfall ↓, Conversion ↑, ATV ↑ | Great team, weak local marketing |
| E — Volume without revenue | Orders ↑, Net Sales ↓ | Discounting/scheme erosion |
| F — Star store | All KPIs ↑ | Benchmark for cluster |

---

## 9. Key business context

- **KPIs that define store growth/degrowth:** Net Sales, Net Orders, Footfall, Conversion, ATV (Average Transaction Value)
- **Progressive lens share target:** 25% of eye orders (currently most stores 19–28%)
- **ET wait time thresholds:** Critical >8 min | Watch 6.5–8 min | Normal <6.5 min
- **PSPD** = Per Store Per Day net sales — use this for period-length-adjusted comparisons (partial months vs full months)
- **Conversion formula:** Always use `Net Orders ÷ Footfall × 100`. The model's `[Conversion]` measure uses a different denominator and does NOT match Power BI visuals.
- **UPT target:** ~2.0 units per transaction
- **Staff metric:** NOB per Staff = orders per staff per day (higher = more productive)
- **Revenue per staff est.:** NOB per Staff × Net ATV
- **Tango:** Store maturity/readiness classification system (1.0 → 2.0 → 3.0)

---

## 10. Available date parameters

**Monthly (newest first):** Apr-26, Mar-26, Feb-26, Jan-26, Dec-25, Nov-25, Oct-25, Sep-25, Aug-25, Jul-25, Jun-25, May-25, Apr-25, Mar-25, Feb-25, Jan-25, Dec-24...

**Weekly (newest first):** 22-Apr-26, 19-Apr-26, 12-Apr-26, 05-Apr-26, 29-Mar-26, 22-Mar-26, 15-Mar-26, 08-Mar-26, 01-Mar-26...

**Quarterly:** Q2-2026, Q1-2026, Q4-2025, Q3-2025, Q2-2025, Q1-2025, Q4-2024, Q3-2024...

**Benchmark mapping (LM-TY):**
```
Apr-26 → Mar-26
Mar-26 → Feb-26
Feb-26 → Jan-26
Q2-2026 → Q1-2026
Q1-2026 → Q4-2025
```

**Benchmark mapping (LY):**
```
Apr-26 → Apr-25
Mar-26 → Mar-25
Feb-26 → Feb-25
Q2-2026 → Q2-2025
```

---

## 11. Report widget technical spec

- **Render method:** `visualize:show_widget` (renders inline in Claude)
- **No external API calls** at runtime (no CORS errors)
- **All data hardcoded** as JS constants from model pull in Step 4
- **All AI narratives hardcoded** as strings from Step 8
- **Charts:** Chart.js 4.4.1 from cdnjs.cloudflare.com
- **Dark mode:** Full support via CSS variables
- **Sections:** All collapsible via click on header
- **Editable fields:** Grooming, VM freshness, Maintenance — for ZM audit notes
- **Score ring:** SVG circle with stroke-dashoffset animation
- **Playbook drill-down buttons:** Use `sendPrompt()` to continue in chat

---

## 12. Number formatting rules

| Data type | Format |
|---|---|
| Store-level revenue | `₹X,XX,XXX` (e.g., ₹9,87,657) |
| Network-level revenue | `₹X.XX Cr` (e.g., ₹371.6 Cr) |
| Orders / Footfall (network) | `X.XX L` (e.g., 12.5 L) |
| Percentages | `XX.XX%` (e.g., 22.33%) |
| ET Wait Time | `X.XX min` (e.g., 6.15 min) |
| UPT | `X.XXX` (3 decimal places) |
| NOB per Staff | `X.XX` (2 decimal places) |
| PSPD | `₹XX,XXX/day` |

---

## 13. What NOT to do

1. **Never use `[Conversion]` from the model** — always derive it as `Net Orders / Footfall × 100`
2. **Never build the report without first validating numbers in chat** — always show the validation table and wait for user confirmation
3. **Never skip the store master pull** — ZM, AOM, rent, clinics, Tango status are all needed for the playbook
4. **Never use API calls inside the widget HTML** — causes CORS errors when opened as a file
5. **Never skip the benchmark period pull** — variances are meaningless without the comparison
6. **Never confuse period-length effects** — always mention PSPD when comparing partial months to full months

---

## 14. Example full run (reference)

```
Store:     LKST1838 — Chennai Pallavaram
Period:    Mar-26 (Monthly)
Benchmark: vs Last month (this year) → Feb-26
```

**Steps executed:**
1. Connected to Store Insights / Offline Operations ✓
2. Pulled store master — COCO, South, Tier-I, 1000 sqft, 2 clinics, ZM: Kapilavai Poorna Shankar ✓
3. Benchmark = Feb-26 ✓
4. Pulled Mar-26 KPIs + Feb-26 KPIs (both filtered by store code LKST1838) ✓
5. Computed Conversion = 320.67/768×100 = 41.75% (model shows 43.88% — discarded) ✓
6. Validated all numbers in chat — user confirmed ✓
7. Scores: Overall 76, Revenue 92, Ops 62, Merch 83, Infra 72 ✓
8. AI narratives written ✓
9. Interactive report widget rendered in Claude ✓

**Key insight surfaced:**
> Revenue up 30.7% despite footfall down 31.2% — entirely driven by conversion surge from 23.18% to 41.75%. The store is converting brilliantly but has a severe footfall vulnerability. If conversion normalises, revenue will fall sharply. Footfall recovery is the #1 priority.

---

## 15. Sharing the report

The report runs inside Claude. For sharing with stakeholders:

| Method | Steps |
|---|---|
| **Email attachment** | Screenshot the report in Claude, or ask Claude to generate the HTML file for download |
| **Netlify Drop (live URL)** | Go to netlify.com/drop → drag the HTML file → get a shareable URL instantly |
| **GitHub Pages** | Upload index.html to a public repo → enable Pages → permanent URL |
| **PDF** | Open in browser → Print → Save as PDF (print stylesheet hides config/buttons) |
| **Notion/Confluence** | Embed the Netlify URL via embed block |

---

*Last updated: Apr 2026 · Built for Lenskart Offline Operations · Validated against Store Insights Fabric XMLA*
