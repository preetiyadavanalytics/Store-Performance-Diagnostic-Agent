# Store-Performance-Diagnostic-Agent
An AI agent that identifies conversion change drivers per store, generates store-specific actionable playbooks for degrowing stores, and delivers automated health reports for Zonal Managers, built on the Store Insights Fabric semantic model. 

================================================================================
STORE PERFORMANCE DIAGNOSTIC AGENT
Lenskart Offline Operations  |  Internal Hackathon  |  April 2026
================================================================================

Built with Claude Sonnet  |  PowerBI MCP Server  |  Microsoft Fabric XMLA
1,572 stores  |  < 2 min per report  |  3 diagnostic levels  |  15+ KPIs

An AI agent that identifies conversion change drivers per store, generates
actionable playbooks for degrowing stores, and delivers automated store
health reports for Zonal Managers — built live on the Store Insights
Fabric semantic model.

--------------------------------------------------------------------------------
TABLE OF CONTENTS
--------------------------------------------------------------------------------

  1.  What is this?
  2.  How it works
  3.  Quick start
  4.  Data model & connection
  5.  KPIs & metrics
  6.  Diagnostic logic
  7.  AI Health Score
  8.  Report structure
  9.  Validated store data
  10. Known stores
  11. Sharing & deployment
  12. Technical constraints & gotchas
  13. Files in this repository
  14. Tech stack
  15. Business context

--------------------------------------------------------------------------------
1.  WHAT IS THIS?
--------------------------------------------------------------------------------

The Store Performance Diagnostic Agent is a conversational AI agent built
inside claude.ai that connects live to Lenskart's Store Insights Power BI
Fabric semantic model and generates a complete store health diagnostic report
for any of 1,572 active stores — in under 2 minutes, with no dashboard,
no SQL, and no manual analysis required.

WHAT IT DOES

  - Connects to Store Insights via the PowerBI MCP server and Fabric XMLA
  - Pulls 15+ KPIs per store per period with a single structured DAX query
  - Applies 3-level diagnostic logic: Funnel Check > Friction > Basket Quality
  - Computes an AI Health Score (0-100) across Revenue, Ops, Merch, Infra
  - Generates a store-specific playbook: Immediate / Today / This week actions
    named to specific staff (ZM, AOM, optometrist)
  - Renders a fully interactive HTML report with charts, collapsible sections,
    and editable ZM audit fields — inside Claude

WHO USES IT

  Zonal Manager (ZM)       Reviews health score before weekly calls.
                           Decides which store to visit and why.

  Area Ops Manager (AOM)   Receives named playbook actions. Adds grooming
                           and VM observations to the report.

  Circle Head              Reviews cluster-level scores. Spots systemic
                           patterns across zones.

  Analytics team           Validates DAX, extends scenarios, calibrates weights.

--------------------------------------------------------------------------------
2.  HOW IT WORKS
--------------------------------------------------------------------------------

  Step 1   User selects: Store code + Period + Benchmark
  Step 2   Agent connects to Store Insights via ConnectFabric (PowerBI MCP)
  Step 3   Pulls store master: ZM, AOM, size, clinics, rent, Tango status
  Step 4   Pulls 15+ KPIs for current period AND benchmark period
           (filtered by BOTH store code AND date parameter)
  Step 5   Computes derived KPIs: Conversion*, UPT, ASP, Progressive share %
  Step 6   Shows validation table in chat — waits for user confirmation
  Step 7   Computes AI Health Score (Overall + 4 section scores)
  Step 8   Writes AI narratives: Revenue, Ops, Merch, Correlation engine
  Step 9   Generates playbook from diagnostic rules
  Step 10  Renders full interactive HTML report widget inside Claude

  * Conversion is ALWAYS computed manually as Net Orders / Footfall x 100.
    The model's [Conversion] measure uses a different internal denominator
    and does NOT match what users see in Power BI visuals.

--------------------------------------------------------------------------------
3.  QUICK START
--------------------------------------------------------------------------------

PREREQUISITES

  - Active session at claude.ai
  - PowerBI MCP server connected (enable in Claude's tool settings)
  - Access to the Offline Operations workspace in Microsoft Fabric

RUN THE AGENT

  Paste the contents of PROMPT.md into Claude, or use this shortcut:

    "Run the Store Performance Diagnostic Agent.
     Show me selection inputs for store, period, and benchmark."

INPUT OPTIONS

  Store         Search by code (LKST1838), name (Chennai), city (Mumbai),
                or region (South)

  Granularity   Monthly / Weekly / Quarterly / Daily / Yearly

  Period        Apr-26, Mar-26, Q2-2026, 19-Apr-26, etc.

  Benchmark     vs Last month (this year)
                vs Last year same month
                vs Target / Budget

--------------------------------------------------------------------------------
4.  DATA MODEL & CONNECTION
--------------------------------------------------------------------------------

CONNECTION DETAILS

  Operation:       ConnectFabric
  Workspace:       Offline Operations
  Semantic Model:  Store Insights
  Tenant:          myorg
  XMLA Endpoint:   powerbi://api.powerbi.com/v1.0/myorg/Offline%20Operations

  Note: Connection tokens expire between sessions. The agent always
  reconnects at the start of each run via ConnectFabric before any DAX query.

KEY TABLES

  Dim_Store_Master          All store metadata: code, name, ZM, AOM, tier,
                            format, size, clinics, rent, Tango status

  dim_dynamic_date_master   Date dimension. ALWAYS use [Date-Parameter] for
                            all period filters — no other date table.

  Dim_PID_Brand_Category    Product dimension: brand, category, lens type

  Dim_Lens_Package          Lens package dimension

STANDARD KPI PULL QUERY

  EVALUATE CALCULATETABLE(
    ROW(
      "Net Sales",       [Net Sales],
      "Net Orders",      [Net Orders],
      "Gross Orders",    [Gross Orders],
      "Footfall",        [Footfall],
      "Net ATV",         [Net ATV],
      "Gross ATV",       [Gross ATV],
      "Net Qty",         [Net Qty],
      "Gross Qty",       [Gross Qty],
      "PSPD",            [PSPD (Net Sales)],
      "ET Wait Time",    [ET Wait Time],
      "NOB per Staff",   [NOB per Staff],
      "Prog Net Orders", [Prog Net Orders],
      "SV Net Orders",   [SV Net Orders],
      "Eye Net Orders",  [Eye Net Orders]
    ),
    dim_dynamic_date_master[Date-Parameter] = "<PERIOD>",
    Dim_Store_Master[store code] = "<STORE_CODE>"
  )

  Run this query TWICE: once for current period, once for benchmark period.

--------------------------------------------------------------------------------
5.  KPIs & METRICS
--------------------------------------------------------------------------------

THE FIVE CORE GROWTH/DEGROWTH KPIs

  Net Sales       From model      Revenue after returns
  Net Orders      From model      Unique bills after cancellations
  Footfall        From model      Walk-ins via Tango counter system
  Conversion %    DERIVED (*)     % of walk-ins who buy
  Net ATV         From model      Revenue per transaction

  (*) CRITICAL WARNING — THE CONVERSION BUG
  ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
  The model's [Conversion] measure uses a different internal footfall
  denominator and does NOT match Power BI visuals.

  NEVER use:   [Conversion] from the model
  ALWAYS use:  Conversion % = Net Orders / Footfall x 100

  Confirmed by user against live Power BI:
    LKST1838 Mar-26:  Model = 43.88%  |  Manual = 41.75%  (correct)
    LKST1781 Mar-26:  Model = 23.11%  |  Manual = 22.33%  (correct)

DERIVED KPIs (computed after data pull, never from model)

  UPT (Units per Transaction)    Net Qty / Net Orders
  Net ASP (Avg Selling Price)    Net Sales / Net Qty
  Progressive share %            Prog / (Prog + SV) x 100
  Return / cancel rate           (Gross - Net Orders) / Gross x 100
  Revenue per Staff              NOB per Staff x Net ATV
  Conv loss orders               (BM Conv% - CUR Conv%) / 100 x CUR Footfall
  Conv loss revenue              Conv loss orders x Net ATV
  Revenue gap                    BM Net Sales - CUR Net Sales

OPERATIONAL THRESHOLDS

  KPI                  Normal          Watch           Critical
  -----------------    ---------       ---------       ---------
  ET Wait Time         < 6.5 min       6.5 - 8 min     > 8 min
  Progressive share    >= 25%          22 - 25%        < 22%
  Return/cancel rate   < 7%            7 - 10%         > 10%
  Conversion           >= 22%          18 - 22%        < 18%
  UPT                  >= 1.8          1.6 - 1.8       < 1.6

NUMBER FORMATTING

  Store-level revenue    Rs X,XX,XXX      e.g. Rs 9,87,657
  Network revenue        Rs X.XX Cr       e.g. Rs 371.6 Cr
  Orders / Footfall      X.XX L           e.g. 12.5 L
  Percentages            XX.XX%           e.g. 22.33%
  ET Wait Time           X.XX min         e.g. 6.15 min
  UPT                    X.XXX (3 dp)     e.g. 1.626
  NOB per Staff          X.XX (2 dp)      e.g. 5.04
  PSPD                   Rs XX,XXX/day    e.g. Rs 24,691/day

--------------------------------------------------------------------------------
6.  DIAGNOSTIC LOGIC
--------------------------------------------------------------------------------

The agent applies a 3-level structured leakage pathway. Each level is only
triggered if the previous level finds a gap.

LEVEL 1 — FUNNEL CHECK

  Footfall --> (x Conversion) --> Net Orders --> (x ATV) --> Net Sales

  IF footfall is UP but orders are flat or DOWN
  --> Customers are entering but not buying
  --> Proceed to Level 2

LEVEL 2 — WHY OF CONVERSION (OPERATIONAL FRICTION)

  Sub-path A: Staffing / Service
    ET Wait Time > 8 min     --> CRITICAL: billing / ET queue bottleneck
    ET Wait Time 6.5-8 min   --> WATCH: friction building, pre-empt now
    ET Wait Time < 6.5 min   --> Normal
    NOB/Staff below bm       --> Understaffing or low productivity

  Sub-path B: Merchandise
    Serviceable fill rate low --> Stock gap on high-demand SKUs
    Progressive share < 22%  --> Assortment or recommendation gap
    Zero-stock high-velocity  --> Out-of-stock diagnosis

LEVEL 3 — BASKET QUALITY (VALUE CHECK)

  UPT     ATV     Diagnosis
  ------  ------  --------------------------------------------------------
  Down    Down    Upsell failure — single item, no coatings or add-ons
  Up      Down    Cheap mix — many items, all low-value or clearance
  Flat    Down    Price drag — discounting or markdown erosion
  Up      Up      Basket healthy — focus on footfall / conversion instead

DIAGNOSTIC SCENARIOS (6 PATTERNS)

  A. Conversion Trap     Footfall up, Conversion down
                         Diagnosis: Queue or stock-out
                         Action: ET audit, peg count, shift roster review

  B. Transaction Gap     Conversion up, ATV down
                         Diagnosis: Upsell failure
                         Action: Coating attach rate, Progressive coaching

  C. Demand Collapse     All KPIs down
                         Diagnosis: No demand or visibility problem
                         Action: Frontage audit, Google Maps, local campaign

  D. Invisible Store     Footfall down, Conversion up, ATV up
                         Diagnosis: Great team, low local awareness
                         Action: WhatsApp CRM, referrals, hyperlocal ads

  E. Volume w/o Value    Orders up, Net Sales down
                         Diagnosis: Discount or scheme erosion
                         Action: Discount log, scheme ratio audit

  F. Star Store          All KPIs up
                         Diagnosis: Benchmark store for cluster
                         Action: Document playbook, share with peers

--------------------------------------------------------------------------------
7.  AI HEALTH SCORE
--------------------------------------------------------------------------------

OVERALL FORMULA

  Score = sc(Net Sales)   x 0.30
        + sc(Conversion)  x 0.25
        + sc(Footfall)    x 0.15
        + sc(Net Orders)  x 0.10
        + sc(ATV)         x 0.10
        + sc(ET Wait)inv  x 0.10

  "inv" = inverted scoring (lower ET wait = better = higher score)

SECTION FORMULAS

  Revenue        = sc(Net Sales) x 0.40  +  sc(Conversion) x 0.35  +  sc(ATV) x 0.25
  Operations     = sc(ET Wait)inv x 0.60  +  sc(NOB/Staff) x 0.40
  Merchandising  = sc(UPT) x 0.50  +  sc(ASP) x 0.30  +  sc(Conversion) x 0.20
  Infrastructure = 72 (contextual — adjusted by physical ZM audit inputs)

KPI SCORING TABLE

  Variance vs benchmark    Normal KPI score    Inverted (ET wait)
  ---------------------    ----------------    ------------------
  > +10%                   95                  10
  +5 to +10%               85                  25
   0 to +5%                72                  40
  -2 to  0%                55                  55
  -5 to -2%                40                  72
  -10 to -5%               25                  85
  < -10%                   10                  95

SCORE LABELS

  80 - 100    Excellent         (green)
  65 -  79    Good              (green)
  50 -  64    Needs attention   (amber)
  35 -  49    At risk           (red)
   0 -  34    Critical          (red)

--------------------------------------------------------------------------------
8.  REPORT STRUCTURE
--------------------------------------------------------------------------------

The report renders as an interactive HTML widget inside Claude with 7
collapsible sections, live Chart.js charts, and editable ZM audit fields.

  Section 0   Store header
              Store name, ZM, AOM, clinics, rent, Tango status,
              AI Health Score ring (SVG)

  Section 1   Raw data validation table
              Every KPI with DAX source, current value, benchmark,
              variance %, and status pill — confirm before trusting report

  Section 2   Revenue & financial performance
              6 metric cards, KPI table, bar charts, AI narrative,
              Correlation engine insight (purple left border)

  Section 3   Operations & staffing
              ET wait time, NOB/staff, ET bottleneck status,
              editable grooming textarea, rating rationale

  Section 4   Merchandising & basket quality
              UPT, ASP, Progressive share, SV vs Progressive doughnut
              chart, basket diagnosis row

  Section 5   Store infrastructure
              PSPD, sales/sqft/day, format, tier,
              editable maintenance textarea

  Section 6   Actionable playbook
              Priority-tiered actions named to ZM, AOM, and optometrist

  Section 7   AI Health Score weightage
              Weight bars, scoring band table, exact arithmetic per section

PLAYBOOK PRIORITY RULES

  Condition                         Priority    Action
  --------------------------------  ----------  --------------------------------
  ET Wait Time > 8 min              IMMEDIATE   Redeploy optometrist; Teleoptom
  Conversion below benchmark        IMMEDIATE   Floor engagement coaching brief
  Progressive share < 22%           TODAY       Progressive pull-through coaching
  Always                            TODAY       9 AM huddle grooming check
  ATV below benchmark               TODAY       Premium SKU planogram audit
  Always                            THIS WEEK   5-day coating attachment sprint
  Always                            THIS WEEK   ZM store visit with health score

--------------------------------------------------------------------------------
9.  VALIDATED STORE DATA
--------------------------------------------------------------------------------

All values below are confirmed against live Power BI outputs by the user.
These are the ground-truth reference numbers for both stores.

LKST1838 — Chennai Pallavaram  |  Mar-26 vs Feb-26
---------------------------------------------------

  KPI                   Mar-26          Feb-26 (BM)     Variance    Status
  --------------------  --------------  --------------  ----------  ----------
  Net Sales             Rs 9,87,657     Rs 7,55,468     +30.7%      Healthy
  Net Orders            320.7           258.9           +23.8%      Healthy
  Footfall              768             1,117           -31.2%      CRITICAL
  Conversion % (*)      41.75%          23.18%          +80.1%      Healthy
  Net ATV               Rs 3,080        Rs 2,918        +5.6%       Healthy
  PSPD                  Rs 24,691/day   Rs 20,985/day   +17.7%      Healthy
  ET Wait Time          4.18 min        3.85 min        +8.8%       Watch
  NOB per Staff         3.84            3.38            +13.5%      Healthy
  UPT                   1.856           1.761           +5.4%       Healthy
  Net ASP               Rs 1,660        Rs 1,657        +0.2%       On track
  Progressive share     27.36%          28.47%          -3.9%       Watch

  AI Health Score: 76/100 (Good)
  Revenue 92  |  Operations 62  |  Merchandising 83  |  Infrastructure 72

  The story: Revenue up 30.7% despite footfall down 31.2%. The store
  generated 62 more bills from 349 fewer walk-ins — a pure conversion
  surge from 23.18% to 41.75%. This is structurally fragile: if
  conversion normalises while footfall stays depressed, revenue will fall
  sharply. Footfall recovery is the #1 priority.

LKST1781 — Delhi Chattarpur  |  Mar-26 vs Feb-26
-------------------------------------------------

  KPI                   Mar-26          Feb-26 (BM)     Variance    Status
  --------------------  --------------  --------------  ----------  ----------
  Net Sales             Rs 24,12,285    Rs 24,66,219    -2.2%       Watch
  Net Orders            974.1           939.3           +3.7%       Healthy
  Footfall              4,362           4,278           +2.0%       On track
  Conversion % (*)      22.33%          21.96%          +1.7%       On track
  Net ATV               Rs 2,477        Rs 2,626        -5.7%       CRITICAL
  PSPD                  Rs 60,307/day   Rs 68,506/day   -12.0%      CRITICAL
  ET Wait Time          8.97 min        10.54 min       -14.8%      Improving
  NOB per Staff         5.04            5.21            -3.3%       Watch
  UPT                   1.626           1.713           -5.1%       CRITICAL
  Net ASP               Rs 1,523        Rs 1,533        -0.6%       On track
  Progressive share     19.26%          20.46%          -5.8%       CRITICAL

  AI Health Score: 66/100 (Good)
  Revenue 57  |  Operations 77  |  Merchandising 51  |  Infrastructure 72

  The story: More footfall, more orders, better conversion — yet revenue
  is still down. The entire gap is ATV: Rs 149 less per bill x 974 bills
  = Rs 1,45,126 value erosion. ET improved dramatically (10.54 to 8.97
  min). Progressive share at 19.26% is 5.74 pp below the 25% target.
  Pure upsell / lens-mix failure — not a demand problem.

--------------------------------------------------------------------------------
10. KNOWN STORES
--------------------------------------------------------------------------------

  Code        Name                        Region   Tier      Format  ZM
  ----------  --------------------------  -------  --------  ------  ------------------
  LKST1838    Chennai Pallavaram          South    Tier-I    COCO    Kapilavai P. Shankar
  LKST1781    Delhi Chattarpur            North    Tier-I    COCO    Atul Jindal
  LKST1401    Hyderabad Kondapur          South    Tier-I    COCO    Katikam Shravan Kumar
  LKST1437    Hyderabad Vanasthalipuram   South    Tier-I    COCO    Naresh Negi
  LKST1818    Navi Mumbai Ulwe            West     Tier-I    COCO    Vipul Shetty
  LKST1621    Vadodara Waghodia           West     Tier-II   COCO    Suraj Thakur
  LKST1764    Nagpur Jaripatka            West     Tier-II   COCO    TBD-ROMG
  LKST1502    Bhatinda Bhucho Mandi       North    Tier-III  COCO    TBD RON 1
  LKST3026    Mumbai Girgaon Chowpatty    West     Tier-I    COCO    Vipul Shetty
  LKST4342    Nashik Dwarka               West     Tier-II   COCO    Mohsin
  LKST4389    Bhagalpur Tilkamanjhi       East     Tier-III  COCO    Subroto Bej
  ST608       Vasundhara Opticals         West     Tier-III  FOFO    Chetan

  Total active / new stores in model: 1,572 as of April 2026

--------------------------------------------------------------------------------
11. SHARING & DEPLOYMENT
--------------------------------------------------------------------------------

The report renders inside Claude by default. Options to share externally:

  Netlify Drop       netlify.com/drop -- drag HTML file -- get instant URL
                     Best for: quick stakeholder sharing

  GitHub Pages       Upload as index.html -- Settings -- Pages -- enable
                     Best for: recurring / permanent URL

  Email attachment   Download HTML -- attach to email -- recipient opens
                     Best for: offline viewing

  Print / PDF        Open in Chrome -- Ctrl+P -- Save as PDF
                     Best for: formal reports, presentations

  Notion/Confluence  Embed Netlify URL via /embed block
                     Best for: internal wikis and runbooks

  Note: The HTML report is fully self-contained. No internet connection
  is required to open it. All data and charts are embedded as static
  assets -- no CORS issues, no external dependencies.

--------------------------------------------------------------------------------
12. TECHNICAL CONSTRAINTS & GOTCHAS
--------------------------------------------------------------------------------

NON-NEGOTIABLE RULES

  Rule 1:  NEVER use [Conversion] from the model.
           Always: Conversion = Net Orders / Footfall x 100

  Rule 2:  ALWAYS filter by BOTH store code AND date parameter.
           CALCULATETABLE(...,
             dim_dynamic_date_master[Date-Parameter] = "Mar-26",
             Dim_Store_Master[store code] = "LKST1838")

  Rule 3:  ALWAYS validate data in chat before building the report.
           Show the full KPI table. Wait for user confirmation. Then build.

  Rule 4:  Use PSPD for partial-month comparisons, not raw revenue.
           Apr-26 raw looks -29.7% vs March. PSPD is +4.4%. PSPD is truth.

KNOWN MODEL ISSUES

  Issue                            Root cause                   Fix
  -------------------------------  ---------------------------  -----------------------
  [Conversion] mismatch            Different footfall denom.    Always derive manually
  [Store Stock] DAX error          In-Transit ref without ctx   Exclude from ROW()
  Dim_Date_Time_Line[Filter] err   Buggy calculated column      Use dim_dynamic_date_master
  Token expiry between sessions    XMLA tokens are session-     Always call ConnectFabric
                                   scoped                       at session start
  INFO.MEASURES() 1,137 rows       Too large for context        Pipe to file, use grep

KNOWN UI / WIDGET ISSUES

  Issue                       Root cause                       Fix
  --------------------------  -------------------------------  -----------------------
  CORS error on file:// open  Widget called Anthropic API      Hardcode all data as
                              at runtime                       JS constants
  Charts not rendering        Script runs before canvas        setTimeout 100ms delay
                              exists in DOM
  Old charts flickering       Canvas reuse on rebuild          charts.forEach destroy()
  Dropdown not closing        onblur fires before onmousedown  setTimeout 200ms on blur
  position:fixed collapses    iframe viewport in Claude        Never use position:fixed

--------------------------------------------------------------------------------
13. FILES IN THIS REPOSITORY
--------------------------------------------------------------------------------

  README.md                              This file -- complete documentation
  PROMPT.md                              9-step execution guide for any Claude session
  memory.md                              Full knowledge base: model, KPIs, logic, gotchas
  transcript.txt                         1,000-word plain text conversation transcript
  approach_note.txt                      Hackathon approach note
  Store_Diagnostic_Agent_Transcript.docx Formatted DOCX with tables and validated data
  store_health_LKST1838_Mar26.html       Interactive report: Chennai Pallavaram, Mar-26

  File descriptions:

  PROMPT.md        9-step execution guide covering DAX queries, scoring
                   formulas, validated data, report spec, and what-not-to-do.
                   Any Claude session can read this and reproduce the agent.

  memory.md        Complete knowledge base: model structure, all KPI
                   definitions, diagnostic logic, validated numbers, UI
                   patterns, all known gotchas, and business context.

  transcript.txt   1,000-word plain text record of the full build session
                   across all 5 phases: model connection, diagnostic logic,
                   correlation engine, data validation, and report generation.

  approach_note.txt
                   Hackathon approach note covering the problem, our approach,
                   what we built, what makes it different, business impact,
                   and next steps. Written as a clean narrative.

  Store_Diagnostic_Agent_Transcript.docx
                   Full formatted Word document transcript with styled tables,
                   KPI validation data, diagnostic logic reference, and
                   section scores for both validated stores.

  store_health_LKST1838_Mar26.html
                   Standalone interactive HTML report for Chennai Pallavaram,
                   Mar-26 vs Feb-26. Open in any browser. No internet needed.

--------------------------------------------------------------------------------
14. TECH STACK
--------------------------------------------------------------------------------

  AI Agent Runtime    Claude Sonnet 4.5 via claude.ai
  AI API              Anthropic Messages API
  Data Connection     PowerBI MCP Server --> Microsoft Fabric XMLA
  Query Language      DAX
  Report Rendering    visualize:show_widget (inline HTML widget in Claude)
  Charts              Chart.js 4.4.1 (cdnjs.cloudflare.com)
  Frontend            HTML / CSS / JavaScript (vanilla -- no frameworks)
  Fonts               Google Fonts: Sora, DM Mono, DM Serif Display
  Distribution        Netlify Drop / GitHub Pages / Email attachment

--------------------------------------------------------------------------------
15. BUSINESS CONTEXT
--------------------------------------------------------------------------------

  Company            Lenskart -- optical retail (eyeglasses, contact lenses,
                     accessories)

  Store formats      COCO: Company Operated Company Owned
                     FOFO: Franchise Owned Franchise Operated

  Scale              1,572+ active / new stores across India

  Eye test system    In-store optometrists + Teleoptom (remote eye testing)

  Footfall system    Tango counters (physical sensors at store entrance)

  Financial year     April to March
                     FY2025-26 = April 2025 to March 2026

  Progressive target >= 25% of eye orders
                     Most stores range 19 to 28%

  Tango maturity     1.0 --> 2.0 --> 3.0
                     Indicates store operational readiness level

  Key roles          ZM: Zonal Manager (manages 15-30 stores per zone)
                     AOM: Area Operations Manager (direct store supervisor)
                     Circle Head: Regional head above ZM
                     Cluster Optometrist: Shared optometrist across stores

================================================================================
Built at the Lenskart Internal Hackathon  |  April 2026
Store Insights  /  Offline Operations  /  Microsoft Fabric XMLA  /  1,572 stores
================================================================================
