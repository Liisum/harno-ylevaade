# HARNO Analytics Dashboard

Single-file analytics dashboard for HARNO project progress. The application is delivered as a static `index.html` file and renders multiple analysis views from in-file data structures.

## Latest update (April 2026)

Reporting period: **aprill 2026**. Source: `28.04.2026 Andmed HARNO aruande jaoks.xlsx`.

Three view tabs:

- **Lõpetamise analüüs** (Completion analysis) — compares lõpetajad vs katkestajad on aggregate KPIs.
- **Osalejate ülevaade** (Participants overview) — full demographic breakdown of everyone who started the program.
- **Kaardivaade** (Municipality map view) — geographic distribution of participants on Estonian municipalities.

### Audience definitions (important)

The dashboard distinguishes between two distinct populations, and the same source workbook is sliced differently for the two analysis stories:

| View | Population | Source sheets | Count (April 2026) |
|---|---|---|---|
| Lõpetamise analüüs — Lõpetajad | Completed the program | `Registreeritud` | 158 |
| Lõpetamise analüüs — Katkestajad | Did not complete (started + dropped, or canceled before start) | `Katkestatud` + `Tühistatud` | 128 |
| Osalejate ülevaade / Kaardivaade | Everyone who **started** the program | `Registreeritud` + `Katkestatud` | 223 |

`Tühistatud` registrations are people who canceled before the program started. They are counted as katkestajad in the completion-rate story (because they did not complete), but they are excluded from the participants overview and the map (because they never participated).

## What the dashboard includes

### 1) Completion analysis

- Core KPIs: lõpetajad, katkestajad, dropout rate, and gender split.
- Education comparison chart (lõpetajad vs katkestajad).
- Location comparison chart and doughnut charts for lõpetajad/katkestajad location shares.
- "Uued andmed" metrics (average age and employment share comparison).
- Automatically generated key insights based on the data block.

### 2) Participants overview

Built from the embedded participant dataset (`reportData.participantsOverview.sourceRows`, which contains `Registreeritud + Katkestatud` rows) and transformed at runtime.

- KPI cards: participant count, municipalities, average age, age range, gender split, unique organizations.
- Regional distribution chart.
- Demographics: gender distribution, age groups, age by gender.
- Education breakdown (total + by gender).
- Occupation distribution.
- Employer/sector distribution with unique organization count.
- Data-driven narrative insights.

### 3) Map view

- Stylized Estonia map with **county boundaries** as the background layer.
- **Municipality bubbles** plotted on county centroids, scaled by participant count.
- **Bubble color gradient** (saturated blue → dark navy) reinforces the size signal so smaller bubbles stay visible against the land fill.
- The map reads directly from `reportData.participantsOverview.geography.municipalPoints` — the same structure already produced by the participant aggregation pipeline. No separate data file is required.
- Legend shows sample bubble sizes, a continuous color gradient bar, and a dynamic note ("X osalejat Y omavalitsusest · Z kirje elukoht puudu").
- Hover/focus tooltip displays exact participant count per municipality.

#### Map data sources (loaded at runtime)

- **County boundaries**: [`buildig/EHAK`](https://github.com/buildig/EHAK) — monthly snapshot of Estonian Land and Spatial Development Board (Maa- ja Ruumiamet) administrative classification, fetched via jsDelivr (`cdn.jsdelivr.net/gh/buildig/EHAK@master/topojson/maakond.json`).
- **Fallback country outline**: `world-atlas@2/countries-50m.json` via jsDelivr — used only if the EHAK county source fails to load. In fallback mode internal county borders are not drawn but the country outline and bubbles still render.
- **Municipality centroids**: hardcoded table inside `index.html` (`municipalityCentroids`). Covers all current Estonian municipalities (post-2017 administrative division, ~77 entries) so the map keeps working as the participant dataset grows.

## Chart value labels (for static export)

All charts render their numeric values directly on the chart surface (in addition to the on-hover tooltip), so the dashboard remains readable when exported as a static image (e.g. embedded in a slide deck).

The behavior is provided by a small in-file Chart.js plugin (`valueLabelsPlugin`, registered globally). It supports:

- vertical bars → label drawn above each bar
- horizontal bars → label drawn just past the bar tip
- doughnut/pie segments → label drawn in the segment center, in white; segments below a configurable threshold (~4–6% of total) are skipped to avoid crowding

Per-chart format is configured via `options.plugins.valueLabels.format` (e.g. `v => v.toFixed(1) + '%'` for percentage charts, `v => String(v)` for count charts). Vertical-bar charts have an extra `padding.top` to leave headroom for the label, and horizontal-bar charts have a `padding.right` for the same reason.

The plugin is self-contained (no external dependency on `chartjs-plugin-datalabels`), so the dashboard still ships with only the existing Chart.js / d3 / topojson CDN scripts.

## Tech stack

- Plain HTML/CSS/JavaScript
- [Chart.js 4.4.1](https://cdnjs.cloudflare.com/ajax/libs/Chart.js/4.4.1/chart.umd.js)
- [d3 7.8.5](https://cdnjs.cloudflare.com/ajax/libs/d3/7.8.5/d3.min.js) — used by the map view for projection and SVG rendering
- [topojson 3.0.2](https://cdnjs.cloudflare.com/ajax/libs/topojson/3.0.2/topojson.min.js) — TopoJSON → GeoJSON conversion for county boundaries
- Google Font: DM Sans

No build tools or package manager are required.

## Run locally

Open the project root and launch `index.html` in any modern browser. Note that the map view requires network access (CDN fetch of d3, topojson, and the county boundaries TopoJSON).

```bash
# Option 1: open directly
open index.html

# Option 2: serve on a local web server (Python)
python3 -m http.server 8080
# then open http://localhost:8080
```

## Data update workflow

The dashboard is configured so that monthly updates happen in a single place:

1. Update `reportMeta` (`period`, `source`).
2. Update completion-section blocks in `reportData` (these are pre-aggregated KPIs comparing lõpetajad vs katkestajad):
   - `overview` — counts and gender split
   - `education` — % distribution by education level
   - `location` — % distribution by region (3 buckets for the bar chart, 4 for the doughnut)
   - `newData` — average age and employed share
3. Replace participant source rows in `reportData.participantsOverview.sourceRows`. **This must contain `Registreeritud + Katkestatud` rows only — do not include `Tühistatud`.** The Osalejate ülevaade and Kaardivaade aggregates rebuild automatically from this list.
4. Verify the auto-generated insights still match the expected story for the period.
5. Check responsive behavior (`@media (max-width: 600px)`) and chart auto-scaling.

The map view auto-aggregates and re-renders when the participant dataset changes — no separate map regeneration step is needed. If a new municipality appears in the data that is not yet in the `municipalityCentroids` table, it will be logged to the browser console as "Kaardiga liitmata omavalitsused" and counted under "kirje elukoht puudu" in the legend; add the missing entry to the centroid table to surface it on the map.

### Building the participant rows from the workbook

When updating from the source XLSX:

1. Read the `Registreeritud` sheet → these rows are the lõpetajad and they belong in the participants list.
2. Read the `Katkestatud` sheet → these rows are programmis katkestanud (started but did not complete) and they also belong in the participants list.
3. **Skip the `Tühistatud` sheet for the participants list** — these people canceled before starting and are not counted as participants. They are still counted as katkestajad in the Lõpetamise analüüs section, but that section uses pre-aggregated counts in `reportData.overview/education/location/newData`, not the row-level `sourceRows`.
4. From each kept row, retain only these seven fields (the rest are PII and not used by the dashboard): `Elukoha omavalitsuse väli`, `Sugu`, `Sünniaasta`, `Vanus`, `Osaleja haridustase`, `Amet`, `Tööandja`. `Sugu`, `Sünniaasta` and `Vanus` are pre-computed from `Isikukood` during the data update step so that the personal identification code itself never appears in the published HTML — see `UPDATE_INSTRUCTIONS.md` section 6 for the rationale and the exact derivation rules. The pipeline derives region, education bucket, occupation group, and employer sector from the verbatim fields and reads `Sugu`, `Sünniaasta`, `Vanus` directly.

## QA checklist (manual acceptance)

Run this checklist before release in **Chrome and Firefox**:

1. **Headline counts are consistent**
   Open Lõpetamise analüüs and confirm `Lõpetajad + Katkestajad = registreeritute koguarv` for the period. Open Osalejate ülevaade and confirm `Osalejaid kokku = Lõpetajad + programmis katkestanud` (i.e. excludes Tühistatud). For April 2026: 158 + 65 = 223.
2. **Value labels render on every chart**
   Each bar and doughnut shows its value directly on the chart, not only on hover. Bars have headroom above and to the right. Doughnut center labels are white and small slices are correctly suppressed.
3. **Map renders with data**
   Open the `Kaardivaade` tab and verify county boundaries + bubbles render together.
4. **Map handles network failure gracefully**
   Block network access (or block requests to `cdn.jsdelivr.net/gh/buildig/EHAK`) and reload — verify that the fallback country outline (`world-atlas`) renders and bubbles still appear without JS crashes. Check the browser console for the expected warning.
5. **Legend values are coherent**
   Confirm the size legend samples (1, 5, 20, max) visually match the bubble sizes on the map, and the gradient bar endpoints match the bubble colors at min/max counts.
6. **"Asukohata kirjed" logic is unchanged**
   Verify the legend note count equals the sum of `Teadmata` plus any municipalities missing from the centroid table; participant totals still match overall KPIs.
7. **Major municipalities are visually correct**
   Confirm highest-count municipalities (e.g. Tallinn, Tartu linn) have dominant bubble sizes and correct approximate placement (Tallinn at the north coast, Tartu in southeast).
8. **Tooltip readability**
   Verify the tooltip text remains readable on mouse hover and keyboard focus.
9. **Mobile / narrow viewport behavior**
   In responsive mode (~420px width), confirm the map remains legible (it auto-scales to container width via `viewBox`), bubbles stay dominant, and the legend wraps cleanly across rows.
10. **Static-export check**
    Take a screenshot of each chart and confirm the value labels make the chart self-explanatory without the surrounding HTML (this is the use case the value-labels plugin is for).

### Optional lightweight browser check (future tooling)

If Playwright or similar is introduced later, add a smoke test that validates:

- map SVG exists and contains at least one `<path>` (county/country outline) and one `<circle>` (bubble),
- legend includes the dynamic "kirje elukoht puudu" note (when applicable),
- network mock for the EHAK URL returning a 404 still results in a rendered map (fallback path),
- after `Chart.register(valueLabelsPlugin)`, every rendered chart canvas contains drawn text in its 2D context (the plugin output).

## Repository structure

```text
.
├── index.html   # Entire dashboard app (UI + data + transformation + charts + map)
└── README.md    # Project documentation
```

## Notes

- This is intentionally a single-file analytics prototype for quick iteration and data handoff.
- All labels and most UI text are in Estonian to match reporting usage.
- The map view requires network access at runtime. If offline operation becomes a requirement, the EHAK TopoJSON file (~50 KB) can be downloaded once and bundled locally; update the URL constant `COUNTY_TOPOJSON_URL` in `index.html` accordingly.
