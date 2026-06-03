# Smart Node Reroute Dashboard — AI Handoff Document

> **For:** Any AI agent or developer picking up this project  
> **Maintained by:** Mariana (hi.designres@gmail.com)  
> **Repo:** https://github.com/hidesignres-maker/smart-node-dashboard  
> **Last updated:** June 2026  
> **Version:** B (current working prototype)

---

## 1. Project Context

This is a **clickable HTML prototype** for a Smart Node Reroute Dashboard built for a PepsiCo internal operations tool. The dashboard monitors product reroute events across distribution centers (DCs).

- **Stack:** Single-file HTML — vanilla JS, Tailwind-free, Chart.js 4.4.1 (CDN), inline CSS.
- **No build system.** Everything lives in `smart-node-dashboard-version-b.html` (also deployed as `index.html` on Vercel).
- **Purpose:** Stakeholder validation prototype. Not production code. Data is 100% static/hardcoded.
- **Audience:** Operations teams managing product rerouting across DCs.

### Key terminology
| Term | Meaning |
|---|---|
| Source DC | The distribution center that originated the reroute |
| Destination | Where the rerouted product ends up (a DC name or "Canceled") |
| Canceled | Treated as a destination/final bucket for now — meaning TBD, pending validation with Sarah |
| Reroute | A product being redirected from its original DC to another location |
| Top Product | The highest-volume SKU rerouted from a given source DC |

---

## 2. File Structure

```
smart-node-dashboard/
├── index.html                          ← Version B (Vercel entry point)
├── smart-node-dashboard-version-b.html ← Primary working file
├── claude_sns_omo_dashboard_final.html ← Earlier iteration (reference only)
├── html-claude.html                    ← Earlier iteration (reference only)
└── HANDOFF.md                          ← This document
```

**Only edit `smart-node-dashboard-version-b.html`.** After changes, copy it over `index.html` and push to GitHub.

---

## 3. Dashboard Layout Architecture

The layout is approved. **Do not redesign or reorder sections** without stakeholder sign-off.

```
┌─────────────────────────────────────────────────────────┐
│ HEADER: Title + Date Range picker + Source DC filter    │
├──────────────────────────┬──────────────────────────────┤
│ KPI Cards (4 columns)                                    │
├──────────────────────────┬──────────────────────────────┤
│ Reroutes by Source DC    │ Reroute Reasons               │
│ chart (70%)              │ donut + legend (30%)          │
├─────────────────────────────────────────────────────────┤
│ Source DC Performance (expandable table, full width)     │
├─────────────────────────────────────────────────────────┤
│ ── Exploratory Section ────────────────────────────────  │
│ Destination Distribution │ Product × Destination Heatmap │
│ (30%)                    │ (70%)                         │
└─────────────────────────────────────────────────────────┘
```

### Grid classes
```css
.row   { display: grid; gap: 12px; margin-bottom: 12px; }
.col-7 { grid-template-columns: 7fr 3fr; }   /* chart + donut */
/* Destination row uses inline style: grid-template-columns: 3fr 7fr */
```

---

## 4. KPI Cards

Four static cards — values update partially when Source DC filter changes.

| Card | ID | Default value | Behavior when DC selected |
|---|---|---|---|
| Total Reroutes | `kpi-reroutes` | 69 | Updates to DC-specific reroute count |
| Avg Handling Time | — | 3.8 min | Static |
| Est. Net Time Saved | `kpi-saved` | 41.6 hrs | Recalculates: `reroutes × 36.2 / 60` |
| First-Pass Success | — | 93% | Static |

---

## 5. Reroute Reasons Card

- Donut chart using Chart.js with `cutout: '68%'`
- Center label hardcoded: **69 reroutes**
- **Three confirmed reasons only:** Stale Items · Inventory Discrepancy · Damaged Item

| Reason | % | Count |
|---|---|---|
| Stale Items | 45% | 31 reroutes |
| Inv. Discrepancy | 35% | 24 reroutes |
| Damaged Item | 20% | 14 reroutes |

**Constraint:** Do not add, rename, or recolor reasons without team approval. Legend shows name + `%% · N reroutes` on two lines.

---

## 6. Chart: Reroutes by Source DC / Trend

The chart **switches type** based on the Source DC filter:

- **All DCs selected** → horizontal bar chart (`indexAxis: 'y'`), shows all 5 DCs
- **Specific DC selected** → line chart, shows Mon–Sun daily trend (sample data: `[3,2,4,3,4,1,1]`)

Key functions:
```js
switchChart(dcKey)       // called from globalFilter()
buildSourceDCChart()     // renders the horizontal bar chart
buildTrendChart(dcName)  // renders the daily line chart
```

Chart instance stored in `let dcChart = null`. Always destroyed before rebuilding to avoid Chart.js canvas reuse error.

---

## 7. Source DC Performance Table

### All DCs mode (default)
Expandable table. Each row = one source DC. Click a row to expand its product detail panel below it.

**Columns:** `[chevron] | Source DC | Reroutes | Top Reason | Top Product | Top Destination`

- Only one row can be expanded at a time (`expandedKey` variable)
- Chevrons are inline SVG (Lucide-style): `SVG_CHEVRON_RIGHT` / `SVG_CHEVRON_DOWN`
- Expanded panel has a left indigo border (`border-left: 3px solid #6366f1`)

**Expanded inner table columns:**
`CA Order ID | Site Order ID | SKU | Description | Qty Rerouted | Destination | Reason`

Column widths (via colgroup, `table-layout: fixed`):
`12% | 12% | 13% | 34% | 10% | 10% | 9%`

### Single DC mode (when Source DC filter = specific DC)
The expandable parent table is **hidden** (`#perfTableWrap` → `display:none`).
A standalone table (`#perfSingleDCView`) is shown instead, with:
- Title: `"Top affected products from [DC Name]"`
- Subtitle: `"Product/order lines rerouted from [DC Name] during the selected period."`
- Same columns as the expanded inner table, using `.dc-detail-tbl` class
- Footer shows total qty rerouted

Functions: `buildSingleDCView(dcKey)` · `buildPerfTable()` (rebuilds the expandable table in background)

---

## 8. Static Data

All data is hardcoded in JS. No API calls.

### DC_ROWS
```js
{ key, name, reroutes, reason, topSku, product, target }
```
5 source DCs: Grand Prairie · Meadville · Indy · Phoenix · Joliet

### EXPANDED_DATA
Keyed by DC key. 5 order lines per DC.
```js
{ caOrder, siteOrder, sku, desc, qty, target, reason }
```

### HEATMAP_ROWS
```js
{ sku, name, vals: [Meadville, Phoenix, Indy, Joliet, Canceled] }
```

### PRODUCT_META
Lookup table: SKU → thumbnail class + initials.

| SKU | Product | Class | Init |
|---|---|---|---|
| SKU-1045 | Doritos Tortilla Chips Ultimate Garlic Parm Flavored, 9 1/4 Oz | `prod-dr` | DR |
| SKU-2031 | STARBUCKS COLD BREW 11 OZ SLEEK CAN 12 PACK VANILLA SWEET CREAM ECOM | `prod-sb` | SB |
| SKU-8842 | Cheetos Crunchy Flamin' Hot Limon, 8.5oz | `prod-ch` | CH |
| SKU-4421 | LIPTON PURELEAF ICED TEA UNSWEET WITH LEMON 18.5Z PLASTIC BOT 12PK BOX | `prod-pl` | PL |
| SKU-7708 | Lay's Cheddar Jalapeno Party Size, 12.5 Ounce | `prod-ly` | LY |

### Destination values
Target field can be any DC name or `"Canceled"`. Canceled is visually neutral — same weight as DC names. No red/warning styling.

---

## 9. Product Thumbnails

Rendered via two helper functions:

```js
prodCell(sku, name)       // wrapping, max 2 lines — used in wide columns
prodCellNarrow(sku, name) // nowrap + ellipsis — used in narrow "Top Product" column
```

Each renders:
```html
<div class="product-cell">
  <span class="prod-thumb prod-dr">DR</span>
  <span class="product-name">Doritos Tortilla Chips...</span>
</div>
```

Thumbnail colors (soft pastels, enterprise-neutral):
```css
.prod-dr { background: #fde68a; color: #92400e; } /* amber   */
.prod-sb { background: #bbf7d0; color: #14532d; } /* green   */
.prod-ch { background: #fecaca; color: #7f1d1d; } /* red     */
.prod-pl { background: #bae6fd; color: #0c4a6e; } /* sky     */
.prod-ly { background: #e9d5ff; color: #4c1d95; } /* violet  */
```

---

## 10. Exploratory Section

Marked with an "Exploratory" pill. This section is under validation.

### Destination Distribution card
- Shows aggregate breakdown of where rerouted products go (DC + Canceled bucket)
- ID `destSub` updates dynamically when a DC is selected
- Validation note (small italic): *"Canceled is shown as a destination bucket for validation. Confirm with Sarah whether this should be treated as a status, destination, or separate outcome."*

### Product × Destination Heatmap
- Blue density scale: `rgb(239,246,255)` → `rgb(29,78,216)` (low → high)
- Columns: Meadville · Phoenix · Indy · Joliet · Canceled · Total
- Canceled column uses the same blue scale — no red
- `border-collapse: collapse` with `border: 2px solid white` between cells

---

## 11. Date Picker

Custom period-week picker (PepsiCo fiscal calendar style).

- Two modes: **Period** (P1–P12) and **Period Week** (P1W1–P12W4)
- Year selector: 2019–2024
- Range selection: click start → click end
- Apply/Cancel buttons

State managed in `const DP = { mode, year, startP, startW, endP, endW, pendingStart, pendingEnd, pendingYear, selecting }`.

The picker **does not currently filter data** — it updates the label in the header only. Data filtering is a future implementation.

---

## 12. Reason Chips / Pills

All reason chips use a single neutral `.pill` class:
```css
.pill {
  background: #f3f4f6; border: 1px solid #e5e7eb;
  color: #374151; border-radius: 999px;
  padding: 4px 8px; font-size: 12px; font-weight: 500;
}
```

**Do not use blue/yellow/red** for reason chips. They are not severity indicators.

Resolved via `PILL` lookup:
```js
const PILL = {
  'Stale Items':           '<span class="pill">Stale Items</span>',
  'Inventory Discrepancy': '<span class="pill">Inv. Discrepancy</span>',
  'Damaged Item':          '<span class="pill">Damaged Item</span>'
};
```

---

## 13. Filter Behavior (`globalFilter`)

Called by `<select id="sourceDC" onchange="globalFilter()">`.

**All DCs:**
- Show `#perfTableWrap`, hide `#perfSingleDCView`
- Card title: "Source DC Performance"
- Chart: horizontal bar (all DCs)
- Reset KPIs to global values

**Specific DC:**
- Hide `#perfTableWrap`, show `#perfSingleDCView`
- Card title: "Top affected products from [DC]"
- Chart: daily line trend
- KPIs update to DC-specific values
- `expandedKey = null` (no expandable rows in single-DC mode)

---

## 14. Concepts Permanently Excluded

Never add these — they were explicitly removed after stakeholder review:

- ❌ Purchased / Partial purchase
- ❌ Product Resolution Summary
- ❌ Purchase DC / Purchase warning
- ❌ Red/green semantic colors on chips
- ❌ Warning icons for Canceled
- ❌ Outcomes section
- ❌ Reroute Type Breakdown
- ❌ PepFlex / Non-PepFlex segmentation
- ❌ External product image URLs

---

## 15. Dependencies

| Library | Version | CDN URL |
|---|---|---|
| Chart.js | 4.4.1 | `https://cdnjs.cloudflare.com/ajax/libs/Chart.js/4.4.1/chart.umd.js` |

No other external dependencies. No npm, no build step.

---

## 16. How to Make Changes

1. Edit `smart-node-dashboard-version-b.html` directly
2. Test by opening in browser (no server needed — pure HTML)
3. Copy file over `index.html`: `cp smart-node-dashboard-version-b.html index.html`
4. Push to GitHub:
   ```bash
   git add .
   git commit -m "describe your change"
   git push
   ```
5. Vercel auto-deploys on push to `main`

---

## 17. Pending Design Changes

> This section is updated as new changes are agreed with stakeholders.  
> Mark items as ✅ when completed.

### 🔴 Validation (Blocker)

- [ ] **Validate prototype with Sarah** — Review the current Version B prototype with Sarah to confirm:
  - Whether "Canceled" should remain a destination bucket, become a status, or be treated as a separate outcome
  - Whether the Exploratory section (Destination Distribution + Heatmap) is useful and should be promoted to the main view
  - Whether the selected-DC drilldown behavior meets operational needs

### 🟡 Product Imagery

- [ ] **Replace placeholder thumbnails with real product images** — Current thumbnails are colored initials (`DR`, `SB`, `CH`, etc.). Replace with actual product photography once assets are available. Images should be 28×28 or 32×32, border-radius 6px. Use `<img>` tags with local assets or approved CDN. Do not use external retail URLs.
- [ ] **Scale up product images** — Consider larger thumbnails (48×48) in the expanded product detail table for better prototype realism. Evaluate whether this makes the table too tall.

### 🟡 Color Palette

- [ ] **Revisit brand/UI palette** — The current palette uses system defaults (indigo `#6366f1` as accent, gray neutrals). Review with Mariana whether this should align to a specific brand system (e.g. PepsiCo Blue, internal design tokens).
- [ ] **Thumbnail palette** — Placeholder thumbnail colors (amber, green, red, sky, violet) are functional but not brand-aligned. Replace once official product color references are available.

### 🟢 Nice to Have (Future)

- [ ] **Date picker wires to data** — Currently the date range picker only updates the label. Connect it to filter the static data by period/week.
- [ ] **Destination row click → product list** — Clicking a destination in Destination Distribution could highlight the heatmap row. Discuss with team before implementing.
- [ ] **Real data connection** — Replace static JS arrays with API calls when backend is ready. All data lives in `DC_ROWS`, `EXPANDED_DATA`, `HEATMAP_ROWS`.

---

*Document generated June 2026. Update this file whenever architecture, data definitions, or design decisions change.*
