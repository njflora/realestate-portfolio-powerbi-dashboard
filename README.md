# Real Estate Development Portfolio Dashboard

A three-page Power BI dashboard built to demonstrate end-to-end BI development for a real estate investment/development business — from data modeling through DAX, interactive UX, and presentation-ready design. Built as a portfolio piece, with the specific use case modeled on land acquisition, development, and resale across multiple countries.

## Business questions answered

- **Portfolio overview**: What projects are active, where are they located, and what stage of the development pipeline are they in?
- **Investment & profitability**: What's the return profile of the portfolio — cost breakdown, margins, and which projects perform best?
- **Market positioning**: Is the portfolio priced competitively against realistic market benchmarks in each country, and where is there room to adjust?

## Tools used

Power BI Desktop, Power Query (M), DAX, Power BI bookmarks & Selection pane, custom Power BI theme (JSON), Python (pandas) for synthetic dataset generation.

## Data source and an important disclosure

This project uses a **synthetic dataset** generated to reflect realistic real estate development economics. The cost ranges and structure were inspired by a real, publicly shared project brochure (a residential villa development on Hiiumaa Island, Estonia), used only as a reference point for realistic land cost, build cost, and pricing figures — no real project data, real company figures, or real financial information is used or represented anywhere in this dashboard. All project names, countries beyond the general region, and numbers are fabricated for demonstration purposes.

## Data model

A star schema with one fact table and two dimension tables:

- **Dim_Project** — 24 projects across Estonia, Sweden, Finland (incl. Åland Islands), and Saudi Arabia. Fields: project name, country, region/city, project type (Residential/Commercial), status (Planning/Construction/Selling/Sold), start and target completion dates.
- **Fact_Plots** — 127 individual plots/units across those projects. Fields: plot size, building size, land cost per m², build cost per m², total investment, list price, market value estimate, sale status.
- **Dim_MarketBenchmark** — country/property-type/tier price benchmarks (Nationwide Avg, Premium, Ultra-Premium), used for market positioning comparisons.

**Design decision**: `Dim_MarketBenchmark` is deliberately **not** joined via a physical relationship. Because a single country + property type maps to multiple benchmark tiers, a standard one-to-many relationship would be ambiguous. Instead, all benchmark lookups are handled in DAX using `SELECTEDVALUE` + `FILTER` + `CALCULATE`.

## Dataset iteration history

The dataset went through several deliberate revisions as issues surfaced during the build:

1. **v1 (8 projects, 78 plots)** — initial build. Working, but too small: with only 8 projects spread evenly across 4 pipeline statuses, the funnel chart rendered as a flat rectangle rather than a meaningful funnel shape.
2. **v2 (24 projects, even 6/6/6/6 status split)** — expanded scale, but the even split still didn't produce a visually tapering funnel (a funnel only tapers when counts decrease at each stage).
3. **v3 (24 projects, tapered 9/7/5/3 status split)** — corrected the distribution to reflect a realistic pipeline (more opportunities at Planning than make it through to Sold), producing a proper tapering funnel and a more defensible business narrative.
4. **v4 (corrected market benchmark table)** — discovered that the benchmark price table had been modeled on land-price tiers (~€70–950/m²), while the "actual price" comparison measure summed full land + build cost (~€2,500–4,700/m²). These were never comparable. Rebuilt the benchmark table to reflect full property cost tiers so the market comparison chart is genuinely apples-to-apples.

## Page 1 — Project Portfolio

- KPI cards: Active Projects, Total Plots, Sell-Through Rate, Countries.
- Map (classic Map visual — see Limitations) showing projects by country, sized by active project count, colored by property type.
- Funnel chart of pipeline stage (Planning → Construction → Selling → Sold).
- Gantt-style project timeline. Built using the community Gantt Chart custom visual, with a duration measure (`DATEDIFF` wrapped in `MIN`/`MAX` to avoid single-value evaluation errors).
- **Full-screen toggle**: rather than relying on Power BI bookmarks to resize a single visual (bookmarks reliably capture visibility but did not reliably capture size/position changes in testing), the dashboard uses **two separate Gantt visuals** — a compact one and an enlarged one — toggled via the Selection pane and bookmarked visibility states. A pragmatic workaround once the standard approach proved unreliable.

## Page 2 — Investment & Profitability

- KPI cards: Total Investment, Total List Price, Profit Margin, Potential Upside.
- Donut chart: land vs. build cost split.
- Bar chart: Top 10 projects by profit margin, using a **Top N filter** rather than showing all 24 — titled explicitly as "Top 10" for transparency rather than silently hiding the rest.
- Plot-level detail table with conditional formatting: a red→green color scale on margin, data bars on investment size, styled header row.
- Slicers: Country and Sale Status (chosen deliberately over Project Name/Type, since these answer the two questions most relevant to a financials page).

**A note on correct aggregation**: the Profit Margin measure uses `DIVIDE(SUM(...) - SUM(...), SUM(...))` rather than averaging pre-calculated row-level percentages. This ensures the portfolio total reflects a true dollar-weighted margin, not a naive "average of averages" — a common DAX mistake that would let a small high-margin project distort the picture.

## Page 3 — Market Benchmark

- A combo "Line and Stacked Column Chart" showing each country's realistic market price range (as a floating band) against the portfolio's actual average price per m².
- **Floating band technique**: since Power BI has no native range/band visual, the band is built using a hidden-base stacking trick — `Benchmark Low` is added as an invisible (100% transparent) base series, with `Benchmark Range` (`High − Low`) stacked on top, so it visually floats between the true low and high values rather than starting at zero.
- **Why a combo visual, not two overlaid visuals**: an earlier version used two separate stacked visuals (an Area chart and a Line chart) manually overlaid at matching size/position. This worked visually but broke interactivity — only the top visual could be hovered or clicked, and keeping two independent Y-axis scales manually synchronized was fragile. Switching to a single native combo visual (columns + line in one object) fixed both problems: one shared axis, full tooltip and cross-filter interactivity.
- KPI cards: Portfolio Avg Price per m², Markets Within Benchmark, Markets Below Benchmark.
- Headroom chart: euro gap between actual price and the market floor per country.
- Position % chart: where each country's price falls as a percentage between the market floor (0%) and ceiling (100%).
- Raw benchmark reference table, with Benchmark Tier and Property Type slicers.

## Key technical issues encountered and fixed

This project surfaced several real, instructive bugs — documented here rather than hidden, since diagnosing and fixing them was a meaningful part of the work:

- **`SELECTEDVALUE` returning blank in ambiguous context**: benchmark lookup measures used `SELECTEDVALUE(Fact_Plots[project_type])` to match a specific tier. When a visual only sliced by country (not type), `project_type` had multiple values in context, `SELECTEDVALUE` returned blank, and the whole measure returned blank. Fixed by adding an `ISBLANK()` fallback so the measure gracefully averages across types when type isn't pinned to a single value.
- **Wrong table source for a same-named field**: several tables in the model have a `country` column (`Dim_Project`, `Fact_Plots`, and — critically — the unrelated `Dim_MarketBenchmark`). Accidentally using the `Dim_MarketBenchmark` version on a chart axis caused a silent, hard-to-spot bug: since that table isn't related to anything else, no filter propagated to the measures, and every value returned blank or defaulted incorrectly (in one case, causing `BLANK − x` to silently evaluate as `−x`, producing a chart that looked plausible but was wrong). Diagnosed by cross-checking against a manually-verified diagnostic table.
- **Duplicate field cross-join**: adding `country` from more than one table onto the same table visual produced a full cross-join (16 rows instead of 4), since the two copies weren't recognized as equivalent for grouping purposes.
- **Y-axis scale mismatch on overlaid visuals**: Power BI auto-scales each visual's axis independently. Two overlaid charts sharing the same pixel space but different auto-scaled ranges meant a data point's visual position didn't accurately reflect its value. Fixed by manually forcing identical Min/Max on both axes — and ultimately resolved more robustly by switching to a single combo visual instead.
- **Conditional formatting decimal vs. percentage confusion**: Power BI's conditional formatting rules operate on the raw decimal value (0.15) even when a field displays as a formatted percentage (15%). Typing "15" into a rule bound intending "15%" silently breaks the logic.

## Design system

A custom Power BI theme (JSON) was built to match the reference brochure's navy/blue palette, with a single pink accent color reserved specifically for "the metric that matters most" (the actual price line on the benchmark chart) — never used as a generic decorative color. Blue is used for all neutral/structural data; red/green is reserved specifically for performance judgments (margin, benchmark position), keeping the color language consistent and meaningful across all three pages.

## Limitations and honest notes

- All financial figures are synthetic and illustrative, calibrated against publicly available reference figures — not real data from any company.
- The classic Map visual (Page 1) is on Microsoft's deprecation path in favor of Azure Maps; used here since it remains functional, with awareness that a production report would migrate to the newer visual.
- Market benchmark tiers (Nationwide/Premium/Ultra-Premium) are blended via averaging when a chart doesn't filter to one specific tier — a simplification made explicit rather than silently hidden.

## Skills demonstrated

Star schema data modeling (including recognizing when *not* to force a physical relationship), Power Query transformations (merges, custom columns, type handling), DAX (`SELECTEDVALUE`/`FILTER`/`CALCULATE` patterns, correct weighted aggregation, Top N filtering), advanced visual construction (hidden-base stacking, combo visuals, bookmark-driven interactivity), conditional formatting, custom theme design, and iterative debugging grounded in checking actual numbers against expected values rather than assuming a chart is correct because it renders.

