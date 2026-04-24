# Store-Performance-Diagnostic-Agent
An AI agent that identifies conversion change drivers per store, generates store-specific actionable playbooks for degrowing stores, and delivers automated health reports for Zonal Managers, built on the Store Insights Fabric semantic model. 

Store Performance Diagnostic Agent

An AI agent that identifies conversion change drivers per store, generates actionable playbooks for degrowing stores, and delivers automated store health reports for Zonal Managers — built on the Lenskart Store Insights Fabric semantic model.


Table of Contents

What is this?
How it works
Quick start
Data model & connection
KPIs & metrics
Diagnostic logic
AI Health Score
Report structure
Validated store data
Known stores
Sharing & deployment
Technical constraints & gotchas
Files in this repository


1. What is this?
The Store Performance Diagnostic Agent is a conversational AI agent built inside Claude that connects live to Lenskart's Store Insights Power BI Fabric semantic model and generates a complete store health diagnostic report for any of 1,572 active stores — in under 2 minutes, with no dashboard, no SQL, no manual analysis required.
What it does

Connects to the Store Insights model via the PowerBI MCP server and Fabric XMLA endpoint
Pulls 15+ KPIs per store per time period with a single structured DAX query
Applies a 3-level diagnostic logic (Funnel Check → Operational Friction → Basket Quality) to find the exact revenue leak
Computes an AI Health Score (0–100) across Revenue, Operations, Merchandising, and Infrastructure
Generates a store-specific actionable playbook with 🔴 Immediate / 🟡 Today / 🔵 This week actions named to specific staff
Renders a fully interactive HTML report with Chart.js charts, collapsible sections, and ZM audit fields — inside Claude

Who uses it
RoleHow they use itZonal Manager (ZM)Reviews health score before weekly calls. Decides which store to visit and why.Area Operations Manager (AOM)Receives named playbook actions. Adds grooming and VM observations.Circle HeadReviews cluster-level scores. Spots systemic patterns across zones.Analytics teamValidates DAX, extends diagnostic scenarios, calibrates score weights.

2. How it works
User selects: Store code + Period + Benchmark
       ↓
Agent connects to Store Insights via PowerBI MCP
       ↓
Pulls store master (ZM, AOM, size, clinics, rent, Tango)
       ↓
Pulls 15+ KPIs for current period AND benchmark period
(both filtered by store code AND date parameter)
       ↓
Computes derived KPIs (Conversion, UPT, ASP, Prog share %)
       ↓
Shows validation table in chat → waits for user confirmation
       ↓
Computes AI Health Score (Overall + 4 section scores)
       ↓
Writes AI narratives (Revenue, Ops, Merch, Correlation engine)
       ↓
Generates playbook from diagnostic rules
       ↓
Renders full interactive HTML report widget inside Claude

3. Quick start
To generate a report
Paste the following prompt into Claude (with the PowerBI MCP server connected):
Run the Store Performance Diagnostic Agent.
Show me selection inputs for store, period, and benchmark.
Or paste the full contents of PROMPT.md directly.
Input options
InputOptionsStoreSearch by code (LKST1838), name (Chennai), city (Mumbai), or region (South)GranularityMonthly / Weekly / Quarterly / Daily / YearlyPeriodE.g., Apr-26, Mar-26, Q2-2026, 19-Apr-26Benchmarkvs Last month (this year) · vs Last year same month · vs Target
Requirements

Active Claude session at claude.ai
PowerBI MCP server connected (available in Claude's tool settings)
Access to the Offline Operations workspace in Fabric


4. Data model & connection
Connection details
Operation:       ConnectFabric
Workspace:       Offline Operations
Semantic Model:  Store Insights
Tenant:          myorg
XMLA Endpoint:   powerbi://api.powerbi.com/v1.0/myorg/Offline%20Operations

Note: Connection tokens expire between sessions. The agent always reconnects at the start of each run via ConnectFabric.

Key tables
TablePurposeDim_Store_MasterAll store metadata — code, name, ZM, AOM, tier, format, size, clinics, rent, Tango statusdim_dynamic_date_masterDate dimension — use [Date-Parameter] for ALL period filtersDim_PID_Brand_CategoryProduct dimension — brand, category, lens typeDim_Lens_PackageLens package dimension
Standard KPI pull query
daxEVALUATE CALCULATETABLE(
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

Always run this query twice — once for the current period and once for the benchmark period.


5. KPIs & metrics
The five growth/degrowth KPIs
KPIFormulaWhat it measuresNet SalesFrom modelRevenue after returnsNet OrdersFrom modelUnique bills after cancellationsFootfallFrom modelWalk-ins (Tango counter system)Conversion %Net Orders ÷ Footfall × 100 ⚠️% of walk-ins who buyNet ATVFrom modelRevenue per transaction

⚠️ Critical: The model's [Conversion] measure uses a different internal footfall denominator and does NOT match Power BI visuals. Always discard [Conversion] from the model. Always compute manually as Net Orders ÷ Footfall × 100.
Confirmed discrepancy: LKST1838 Mar-26 — model says 43.88%, manual = 41.75%. User verified 41.75% matches Power BI.

Derived KPIs (all computed after data pull)
KPIFormulaUPT (Units per Transaction)Net Qty ÷ Net OrdersNet ASP (Avg Selling Price)Net Sales ÷ Net QtyProgressive share %Prog Orders ÷ (Prog + SV Orders) × 100Return / cancel rate(Gross Orders − Net Orders) ÷ Gross Orders × 100Revenue per StaffNOB per Staff × Net ATVConv loss orders(BM Conv% − CUR Conv%) ÷ 100 × CUR FootfallConv loss revenueConv loss orders × Net ATVRevenue gapBM Net Sales − CUR Net Sales
Operational thresholds
KPINormalWatchCriticalET Wait Time< 6.5 min6.5 – 8 min> 8 minProgressive share≥ 25%22 – 25%< 22%Return / cancel rate< 7%7 – 10%> 10%Conversion≥ 22%18 – 22%< 18%UPT≥ 1.81.6 – 1.8< 1.6
Number formatting
TypeFormatExampleStore-level revenue₹X,XX,XXX₹9,87,657Network revenue₹X.XX Cr₹371.6 CrOrders / FootfallX.XX L12.5 LPercentagesXX.XX%22.33%ET Wait TimeX.XX min6.15 minUPTX.XXX1.626PSPD₹XX,XXX/day₹24,691/day

6. Diagnostic logic
Level 1 — Funnel check
Footfall → (× Conversion) → Net Orders → (× ATV) → Net Sales

IF Footfall ↑ but Orders ↓ or flat
→ Customers entering but not buying → proceed to Level 2
Level 2 — Why of conversion (two sub-paths)
Sub-path A: Staffing / Service
ET Wait Time > 8 min   → CRITICAL: billing/ET queue bottleneck
ET Wait Time 6.5–8 min → WATCH: friction building
ET Wait Time < 6.5 min → Normal
NOB per Staff below bm → understaffing or low productivity
Sub-path B: Merchandise
Serviceable fill rate ↓ → stock gap on high-demand SKUs
Progressive share < 22% → assortment or recommendation gap
Zero-stock high-velocity → Out-of-stock diagnosis
Level 3 — Basket quality
UPTATVDiagnosis↓ below benchmark↓ below benchmarkUpsell failure — single item, no coatings or add-ons↑ above benchmark↓ below benchmarkCheap mix — many items, all low-value or clearance≈ flat↓ below benchmarkPrice drag — discount or markdown erosion↑ above benchmark↑ above benchmarkBasket healthy — focus on footfall/conversion
Diagnostic scenarios (6 patterns)
ScenarioKPI patternDiagnosisPrimary actionA — Conversion trapFootfall ↑, Conversion ↓Queue / stock-outET audit, peg count, shift rosterB — Transaction gapConversion ↑, ATV ↓Upsell failureCoating attach, Progressive coachingC — Demand collapseAll KPIs ↓No demand / visibilityFrontage, Google Maps, local campaignD — Invisible storeFootfall ↓, Conv ↑, ATV ↑Great team, low awarenessWhatsApp CRM, referrals, hyperlocalE — Volume w/o valueOrders ↑, Sales ↓Discount / scheme erosionDiscount log, scheme ratio auditF — Star storeAll KPIs ↑Benchmark storeDocument & share cluster playbook

7. AI Health Score
Overall score formula
Overall = sc(Net Sales)×0.30 + sc(Conversion)×0.25 + sc(Footfall)×0.15
        + sc(Net Orders)×0.10 + sc(ATV)×0.10 + sc(ET Wait, inverted)×0.10
Section formulas
Revenue      = sc(Net Sales)×0.40 + sc(Conversion)×0.35 + sc(ATV)×0.25
Operations   = sc(ET Wait, inv)×0.60 + sc(NOB per Staff)×0.40
Merchandising = sc(UPT)×0.50 + sc(ASP)×0.30 + sc(Conversion)×0.20
Infrastructure = 72 (contextual — adjusted by physical audit inputs)
KPI scoring table
Variance vs benchmarkScore (normal KPIs)Score (inverted: ET wait)> +10%9510+5 to +10%85250 to +5%7240−2 to 0%5555−5 to −2%4072−10 to −5%2585< −10%1095
Score labels & colours
ScoreLabelColour80–100Excellent🟢 Green65–79Good🟢 Green50–64Needs attention🟡 Amber35–49At risk🔴 Red0–34Critical🔴 Red

8. Report structure
The full report renders as an interactive HTML widget inside Claude with 7 collapsible sections:
#SectionKey contents0Store headerName, ZM, AOM, clinics, rent, Tango status, AI Health Score ring1Raw data validationEvery KPI: DAX source, current value, benchmark, variance, status pill2Revenue & financial performance6 metric cards, KPI table, charts, AI narrative, Correlation engine insight3Operations & staffingET wait time, NOB/staff, ET bottleneck status, editable grooming textarea4Merchandising & basket qualityUPT, ASP, Progressive share, lens mix doughnut chart, basket diagnosis5Store infrastructurePSPD, sales/sqft/day, editable maintenance textarea6Actionable playbook🔴 Immediate / 🟡 Today / 🔵 This week — named to ZM, AOM, optometrist7AI Health Score weightageWeight bars, scoring bands, exact arithmetic shown
Playbook generation rules
ConditionPriorityAction generatedET Wait Time > 8 min🔴 ImmediateRedeploy optometrist, activate Teleoptom for peak hoursConversion below benchmark🔴 ImmediateFloor engagement coaching brief at morning huddleProgressive share < 22%🟡 TodayProgressive pull-through coaching — every 35+ customerAlways🟡 Today9 AM morning huddle grooming checkATV below benchmark🟡 TodayPremium SKU planogram auditAlways🔵 This week5-day coating attachment rate sprintAlways🔵 This weekZM store visit with health score

9. Validated store data
These values are confirmed against live Power BI report outputs and are the ground-truth reference.
LKST1838 — Chennai Pallavaram · Mar-26 vs Feb-26 ✅
KPIMar-26Feb-26VarianceStatusNet Sales₹9,87,657₹7,55,468+30.7%🟢 HealthyNet Orders320.7258.9+23.8%🟢 HealthyFootfall7681,117−31.2%🔴 CriticalConversion %41.75%23.18%+80.1%🟢 HealthyNet ATV₹3,080₹2,918+5.6%🟢 HealthyPSPD₹24,691/day₹20,985/day+17.7%🟢 HealthyET Wait Time4.18 min3.85 min+8.8%🟡 WatchNOB per Staff3.843.38+13.5%🟢 HealthyUPT1.8561.761+5.4%🟢 HealthyNet ASP₹1,660₹1,657+0.2%🔵 On trackProgressive share27.36%28.47%−3.9%🟡 Watch
AI Health Score: 76/100 (Good) · Revenue 92 · Operations 62 · Merchandising 83 · Infrastructure 72

The story: Revenue up 30.7% despite footfall down 31.2%. 768 walk-ins generated more revenue than 1,117 in February — pure conversion surge (41.75% vs 23.18%). Fragile: if conversion normalises while footfall stays depressed, revenue will fall sharply. Footfall recovery is the #1 priority.


LKST1781 — Delhi Chattarpur · Mar-26 vs Feb-26 ✅
KPIMar-26Feb-26VarianceStatusNet Sales₹24,12,285₹24,66,219−2.2%🟡 WatchNet Orders974.1939.3+3.7%🟢 HealthyFootfall4,3624,278+2.0%🔵 On trackConversion %22.33%21.96%+1.7%🔵 On trackNet ATV₹2,477₹2,626−5.7%🔴 CriticalPSPD₹60,307/day₹68,506/day−12.0%🔴 CriticalET Wait Time8.97 min10.54 min−14.8%🟢★ ImprovingNOB per Staff5.045.21−3.3%🟡 WatchUPT1.6261.713−5.1%🔴 CriticalNet ASP₹1,523₹1,533−0.6%🔵 On trackProgressive share19.26%20.46%−5.8%🔴 Critical
AI Health Score: 66/100 (Good) · Revenue 57 · Operations 77 · Merchandising 51 · Infrastructure 72

The story: More footfall, more orders, better conversion — yet revenue is down. The entire gap is ATV: ₹149 less per bill × 974 bills = ₹1,45,126 value erosion. ET improved dramatically (10.54 → 8.97 min). UPT fell 5.1% and Progressive share is 19.26% — well below the 25% target. Pure upsell/lens-mix failure, not a demand problem.


10. Known stores
CodeNameRegionTierFormatZMAOMLKST1838Chennai PallavaramSouthTier-ICOCOKapilavai Poorna ShankarMohan KumarLKST1781Delhi ChattarpurNorthTier-ICOCOAtul JindalBhawna TiwariLKST1401Hyderabad KondapurSouthTier-ICOCOKatikam Shravan KumarDevenderLKST1437Hyderabad VanasthalipuramSouthTier-ICOCONaresh NegiSudhakar AvulaLKST1818Navi Mumbai UlweWestTier-ICOCOVipul ShettyAmisha ShettyLKST1621Vadodara WaghodiaWestTier-IICOCOSuraj ThakurDeven DhakanLKST1764Nagpur JaripatkaWestTier-IICOCOTBD-ROMGSalimLKST1502Bhatinda Bhucho MandiNorthTier-IIICOCOTBD RON 1Deepak GuptaLKST3026Mumbai Girgaon ChowpattyWestTier-ICOCOVipul ShettyChandrakant WaghLKST4342Nashik DwarkaWestTier-IICOCOMohsinTushar ShelarLKST4389Bhagalpur TilkamanjhiEastTier-IIICOCOSubroto BejManoj Kumar JhaST608Vasundhara OpticalsWestTier-IIIFOFOChetanRameez Bashir Shaikh

Total active/new stores in model: 1,572 as of April 2026.


11. Sharing & deployment
The report renders inside Claude by default. To share externally:
MethodHowBest forNetlify Dropnetlify.com/drop → drag HTML file → instant URLQuick stakeholder sharingGitHub PagesUpload as index.html → enable Pages → permanent URLRecurring reportsEmail attachmentDownload HTML → attach to email → recipient double-clicksOffline viewingPrint / PDFOpen in Chrome → Ctrl+P → Save as PDFFormal reportsNotion / ConfluenceEmbed Netlify URL via embed blockInternal wikis

The HTML report is fully self-contained — no internet connection needed to open it. All data and charts are embedded.


12. Technical constraints & gotchas
Must-know rules
1. NEVER use [Conversion] from the model
   Always: Conversion = Net Orders / Footfall × 100

2. ALWAYS filter by BOTH store code AND date parameter
   CALCULATETABLE(..., dim_dynamic_date_master[Date-Parameter] = "Mar-26",
                        Dim_Store_Master[store code] = "LKST1838")

3. ALWAYS validate data in chat before building any report
   Show the full KPI table. Wait for user confirmation.

4. Use PSPD for partial-month comparisons, not raw revenue
   Apr-26 raw looks −29.7% vs March. PSPD is +4.4%. PSPD is the truth.
Known model issues
IssueRoot causeFix[Conversion] mismatchDifferent footfall denominator internallyDerive manually always[Store Stock] DAX errorReferences [In-Transit Store Stock] without store contextExclude from ROW() queriesDim_Date_Time_Line[Filter] errorBuggy calculated columnUse dim_dynamic_date_master onlyToken expiry between sessionsXMLA tokens are session-scopedAlways call ConnectFabric firstINFO.MEASURES() returns 1,137 rowsToo large for inline contextPipe output to file and grep
Known UI / widget issues
IssueRoot causeFixCORS error on file:// openWidget called Anthropic API at runtimeHardcode all data as JS constantsCharts not renderingScript runs before canvas in DOMsetTimeout(() => drawCharts(), 100)Old charts flickeringCanvas reuse error on rebuildcharts.forEach(c => c.destroy()) firstDropdown not closingonblur fires before onmousedownsetTimeout(() => closeDD(), 200) on blurposition: fixed collapses widgetiframe viewport constraintNever use fixed positioning

13. Files in this repository
FileDescriptionREADME.mdThis file — complete agent documentationPROMPT.md9-step execution guide for re-running the agent in any Claude sessionmemory.mdFull knowledge base — model structure, KPIs, diagnostic logic, UI patterns, gotchastranscript.txt1,000-word plain text conversation transcriptStore_Diagnostic_Agent_Transcript.docxFull formatted DOCX transcript with tables and validated datastore_health_LKST1838_Mar26.htmlValidated interactive report — Chennai Pallavaram, Mar-26 vs Feb-26

Tech stack
ComponentTechnologyAI agent runtimeClaude Sonnet 4.5 via claude.aiAI APIAnthropic Messages APIData connectionPowerBI MCP Server → Fabric XMLAQuery languageDAXReport renderingvisualize:show_widget (inline HTML in Claude)ChartsChart.js 4.4.1FrontendHTML / CSS / JavaScript (vanilla)FontsGoogle Fonts (Sora, DM Mono, DM Serif Display)DistributionNetlify Drop / GitHub Pages / email attachment

Business context

Company: Lenskart — optical retail chain (eyeglasses, contact lenses, accessories)
Formats: COCO (Company Operated Company Owned) and FOFO (Franchise Owned Franchise Operated)
Scale: 1,572+ active/new stores across India
Eye test system: In-store optometrists + Teleoptom (remote eye testing)
Footfall system: Tango counters (physical sensors at store entrance)
Financial year: April–March (FY2025-26 = Apr 2025 – Mar 2026)
Progressive lens target: 25% of eye orders (most stores range 19–28%)
Tango maturity: 1.0 → 2.0 → 3.0 (store operational readiness levels)
