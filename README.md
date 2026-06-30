# Google Pixel Sales Program Dashboard

> An enterprise retail sell-out analytics portal for the Google Pixel program in India — built and managed by **Channelplay**.

![Status](https://img.shields.io/badge/status-active-1E8E3E)
![Build](https://img.shields.io/badge/build-single--file%20HTML-1B4DB1)
![Data](https://img.shields.io/badge/dataset-750K%20rows-19459E)
![Vanilla JS](https://img.shields.io/badge/JS-vanilla%20ES6-F7DF1E)
![No backend](https://img.shields.io/badge/backend-none%20(offline)-6B7686)

A self-contained, single-file web application that reproduces the Pixel program's Excel reporting workbook as an interactive, drillable dashboard. It ships as **one `.html` file** with the entire transaction dataset embedded and compressed — no server, no database, no runtime data fetching. Open it in a browser and it works.

---

## Table of contents

- [Overview](#overview)
- [Key features](#key-features)
- [Tech stack](#tech-stack)
- [How it works](#how-it-works)
- [Data inputs](#data-inputs)
- [Derivation rules](#derivation-rules)
- [Project structure](#project-structure)
- [Build from source](#build-from-source)
- [Usage](#usage)
- [Data assumptions & known gaps](#data-assumptions--known-gaps)
- [Deployment](#deployment)
- [Roadmap](#roadmap)
- [Branding](#branding)

---

## Overview

The dashboard is designed for **Google India leadership and regional/national sales managers** to track Pixel retail sell-out across the national footprint. It intentionally mirrors the existing Excel report — anyone familiar with the workbook will recognise the layout immediately — while adding navigation, persistent global filters, conditional formatting, transaction-level drill-down, search, and export.

The reference build covers:

| Dimension | Scale |
|---|---|
| Transaction rows | **750,000** (store × date × SKU grain) |
| Stores | **747** active POS |
| Partners | **38** retail partners |
| RTM channels | **3** (LFR, Wireless Chains, RD) |
| SKUs | **12** Pixel phones |
| Date window | 23 days (current MTD) |

Every reported number is **traceable** — clicking any value opens the Transaction Explorer filtered to the exact records that produced it. For example, a partner showing `3,615` QTD units resolves to a record set that sums to exactly `3,615`.

---

## Key features

- **Five sections** — Dashboard (executive overview + clickable KPIs), POS Performance, Zero Sales & Stock, Transaction Explorer, and Data Upload.
- **Faithful Excel reproduction** — QTD / YTD / MTD / Weekly sales, productivity and productivity-distribution tables, grouped by `RTM → Partner` and `Region → State`, with sticky headers, sticky first column, alternating row shading and Excel-style conditional formatting (heat scales).
- **Zero-sales & stock tracking** — weekly and daily zero-sales, zero-stock, and SIH (stock-in-hand) distribution tables.
- **Global filters** — FY, Quarter, Month, Week, RTM, Partner, Series, State, City, Store, ISD. Multi-select with search, select-all, clear, persistent state, and AND-combination across dimensions. Every filter updates every page.
- **Drill-down everywhere** — KPI cards and numeric cells route to the Transaction Explorer with the originating context applied (group + period + metric).
- **Virtualised Transaction Explorer** — renders the full 750K-row filtered set on scroll, with column sort, instant search, and CSV / Excel export.
- **Client-side upload** — drag-and-drop daily MTD sales files and the ISD attendance file, with structure validation, preview, and confirm-replace — all in-browser via SheetJS.
- **Fully offline** — the dataset is gzip-compressed and base64-embedded; decompression happens in the browser using the native `DecompressionStream` API.

---

## Tech stack

| Layer | Technology |
|---|---|
| UI | Vanilla **HTML5 / CSS3 / ES6** (no framework) |
| Data pipeline | **Python 3** + `openpyxl` |
| In-browser parsing | **SheetJS** (`xlsx` 0.18.5), embedded |
| Compression | `gzip` (Python) → `DecompressionStream` (browser), with `pako` fallback |
| Encoding | Dictionary-encoded, typed-array columnar fact table |
| Tests | **jsdom** headless render + reconciliation harness |
| Deployment | **GitHub Pages** (static) |

There are **no external runtime dependencies** for data. Web fonts (Inter + Material Symbols) load from Google Fonts when online; the application and its data work offline.

---

## How it works

### 1. Data pipeline (Python, build-time)

The raw 49 MB sales workbook is too large to embed as JSON. The pipeline collapses it into a compact payload:

1. **Join** each store to the RTM master mapping (`Google_RTM_mapping.xlsx`) to resolve RTM, Region, ISD identity and store metadata.
2. **Build a store dimension** — one record per store, carrying all repeating attributes (partner, state, city, RTM, region, ISD…), so each fact row only references a store index.
3. **Dictionary-encode** every categorical column (date, product, storage, color) to small integer indices.
4. **Pack** the fact table into typed arrays (`Uint8` / `Uint16`), concatenate, `gzip`, and base64-encode.

Result: **8.2 MB raw → 2.6 MB gzipped → 3.5 MB base64**, embedded directly in the HTML.

### 2. In-browser engine (runtime)

On load, the app base64-decodes and gunzips the payload, slices it back into typed arrays, and runs a **mask-based aggregation engine**:

- Filters compile to three boolean masks (`allowDate`, `allowProduct`, `allowStore`).
- A single pass over the fact table builds per-store / per-week / per-day rollups (units, SOH, presence).
- Report tables, KPIs and drill sets are all derived from these rollups, so a filter change re-aggregates the full 750K rows in a few milliseconds.

### 3. Assembly

A stitch step injects the stylesheet, SheetJS, the data payload, and the application script into the HTML shell to produce the final single-file build.

---

## Data inputs

### Sales file (`pixel_sample_data.xlsx`, sheet `in`) — daily MTD

| Column | Notes |
|---|---|
| Date | one row per store × date × SKU |
| Partner | retail partner name (join to RTM mapping) |
| State, City | |
| Store Code | **primary join key** to the mapping |
| GS Code, Store Name | |
| Product, Storage, Color | SKU attributes |
| Sales on selected date | daily units |
| Inventory added on selected date | daily inward |
| Total Sales as of selected date | cumulative units |
| Total Inv shipped to date | cumulative inward |

> The upload model is **daily MTD**: each file carries month-to-date data up to the previous day. When the calendar month rolls over, the prior month is preserved and the new month is appended.

### RTM mapping (`Google_RTM_mapping.xlsx`, sheet `Store List`)

Store master — Channel (RTM), Region, ISP/ISD identity, OneHub, TSM/RSM, store status, ISD deployment date, and the join keys (`Partner Store ID`, `Retailer DMS Code`).

### Attendance file (date-spread) — *uploaded later*

Activates the ISD-dependent metrics. Expected layout:

| Employee ID | Employee Name | Store Code | Store Name | City | State | 01 | 02 | … | 30 | Total Present | Total EL | Total SL | Total CL |
|---|---|---|---|---|---|---|---|---|---|---|---|---|---|

Daily cells use status codes `P / A / EL / SL / CL / WO`. `Store Code` must match the sales file so it joins.

---

## Derivation rules

These are applied deterministically from the source files — **no synthetic data is ever generated**.

| Derived field | Rule |
|---|---|
| **RTM** | Resolved per partner from the mapping (`LFR` / `WC` → Wireless Chains / `RD D2R` → RD). Partners not present in the mapping default to **RD D2R** until a future upload supplies the mapping. |
| **Series** | From Product — P10 / Pro / XL / Fold → `NPI (P10)`; 10a → `NPI-a`; P9 / Pro / XL → `N-1 (P9)`; 9a → `N-1a`; P8 family → `Others`. |
| **Region** | From the mapping's Region field (case-normalised); unmatched stores grouped as `Unmapped`. |
| **SOH / SIH / Zero-Stock** | `Total Inv shipped to date − Total Sales as of date`, per store-SKU, floored at 0. |
| **Productivity** | Units ÷ active POS (units per store, month-to-date). |
| **Week numbering** | ISO weeks within the data window; `CW` = latest week present. |

---

## Project structure

```
pixel-program-dashboard/
├── src/
│   ├── template.html        # HTML shell (page structure)
│   ├── styles.css           # Channelplay-branded design tokens + components
│   └── app.js               # data engine, filters, aggregations, renderers, upload
├── pipeline/
│   ├── build_data.py        # join + encode  → dims.json, fact payload
│   └── assemble.py          # stitch src + data + SheetJS → dist/index.html
├── data/                    # input workbooks (gitignored / Git LFS — large)
│   ├── pixel_sample_data.xlsx
│   ├── Google_RTM_mapping.xlsx
│   └── format.xlsx          # the reporting-spec workbook
├── dist/
│   └── pixel_program_dashboard.html   # final self-contained build (~4.7 MB)
├── tests/
│   └── render_harness.js    # jsdom smoke + reconciliation tests
└── README.md
```

> The raw sales workbook is ~49 MB — keep it out of Git (`.gitignore`) or track it with **Git LFS**. The built `dist/*.html` (~4.7 MB) is committable and is what GitHub Pages serves.

---

## Build from source

**Prerequisites:** Python 3.9+ (`pip install openpyxl`) and Node 18+ (for tests; provides `DecompressionStream`).

```bash
# 1. Encode the dataset (joins sales + RTM mapping, produces the compressed payload)
python pipeline/build_data.py \
    --sales data/pixel_sample_data.xlsx \
    --mapping data/Google_RTM_mapping.xlsx \
    --out build/

# 2. Assemble the single-file app
python pipeline/assemble.py --src src/ --build build/ --out dist/pixel_program_dashboard.html

# 3. (optional) Validate the build headlessly
npm install jsdom xlsx
node tests/render_harness.js
```

The test harness decodes the embedded data, renders every page, and asserts that country totals and drill-downs reconcile to ground truth (e.g. 15,086 total units, 747 stores, RTM split 9,850 / 3,029 / 2,207).

---

## Usage

1. **Open** `dist/pixel_program_dashboard.html` in Chrome or Edge (recent versions, for `DecompressionStream`).
2. **Filter** via the *Global Filters* drawer — selections persist across all pages.
3. **Drill** by clicking any KPI card or numeric cell to jump to the Transaction Explorer with that context applied.
4. **Explore** — sort columns, search instantly, and export the filtered set to CSV or Excel.
5. **Upload** new data on the *Data Upload* page — the daily MTD sales file replaces/extends the dataset and refreshes every page; the attendance file activates the ISD metrics.

---

## Data assumptions & known gaps

The dashboard surfaces gaps rather than papering over them. Current items, all designed to self-resolve as fuller data is uploaded:

- **275 of 747 stores (~33% of units)** don't join to the RTM mapping on any key. They're assigned **RD D2R** and grouped under `Unmapped` region until an updated mapping arrives. Their store-specific attributes (authoritative Region, ISD identity, tenure) are blank.
- **3 partners** (*Bhawar Life Style, Bulzer India, Lulu International Shopping Malls*) are absent from the mapping → **RD D2R** for now.
- **Attendance metrics** (Active ISDs, "active < 4 days", daily active-ISD, zero-sales attribution) show a `⧗` placeholder and activate when the attendance file is uploaded.
- **QTD / YTD** mirror current MTD data; **QoQ% / YoY% / LFL** columns render as `—` with the aggregation logic already wired to populate once multi-period data accumulates.
- **LOB** filter shows only *Phone* (the dataset contains only Pixel phones).

---

## Deployment

Static — deploy the `dist/` folder to **GitHub Pages**:

```bash
# Serve dist/ on the gh-pages branch (example)
git subtree push --prefix dist origin gh-pages
```

Then enable Pages on the `gh-pages` branch in repository **Settings → Pages**. Because everything is embedded, no build step or backend is required on the hosting side.

---

## Roadmap

- [ ] Ingest the ISD attendance file and light up the `⧗` metrics.
- [ ] Accumulate multi-month data to auto-populate QoQ% / YoY% / LFL.
- [ ] Refresh RTM/region/ISD for the 275 unmapped stores via an updated mapping.
- [ ] Expand LOB beyond Phone if Buds / Watch / Accessories enter the program.

---

## Branding

The header uses a **placeholder Channelplay wordmark**. Drop in the official logo asset (replace the `.logo-mark` / `.logo-wm` block in `template.html`) and set the exact brand hex in the `--brand` CSS variable in `styles.css`. The palette (Channelplay blue / white / light grey, with green/yellow/orange/red status colours) is centralised as CSS tokens for easy theming.

---

<sub>Internal operations dashboard built and managed by Channelplay. Not affiliated with Google's consumer branding.</sub>
