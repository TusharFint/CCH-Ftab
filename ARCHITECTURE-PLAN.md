# Finances Tab — Architecture Plan

## 1. File Structure (10 new files, ~700 lines)

```
features/finances/
  types/finance.types.ts              ~80L   — interfaces + type aliases
  lib/compute-derived.ts               ~60L   — pure fn: inputs -> computed P&L
  lib/finance.defaults.ts              ~20L   — default values constant
  services/financeService.ts           ~100L  -- API service layer (-> apiClient -> tRPC)
  hooks/useFinanceGrid.ts              ~30L   -- query hook (get grid data)
  hooks/useFinanceMutation.ts          ~50L   -- mutation hooks (save / update)
  components/FinancesModule.tsx        ~120L  -- main container (tabs, toolbar, Save All)
  components/FinanceGrid.tsx            ~200L  -- reusable spreadsheet grid (sticky cols)
  components/FinanceInputTab.tsx        ~40L   -- Input Fields tab content
  components/FinancePNLTab.tsx          ~30L   -- Site-wise P&L tab content
```

## 2. Data Flow

```
PAGE LOAD:
  FinancesModule -> useFinanceGrid({month})
    -> financeService.getFinanceGrid()
      -> apiClient.get() -> tRPC getFinanceGrid
        -> DB: SELECT finance_records JOIN sites + SQL computes
          -> returns FinanceGridRow[] (input + derived)
    -> setGridState(data) -> renders both tabs

CELL EDIT:
  onChange -> local gridState[site][field] = val
  -> computeDerived(site) -> recalc 13 values instantly
  -> re-render row (no API yet)
  -> debounce 500ms -> useUpdateRecord.mutate()
    -> financeService.updateRecord() -> apiClient.put() -> tRPC -> DB UPDATE
    -> toast.success/error

SAVE ALL:
  useSaveFinanceData().mutate({period, records[]})
    -> financeService.saveAll() -> apiClient.post() -> tRPC upsertBulk
    -> toast.success("Saved N sites") + invalidateQueries
```

## 3. Patterns to Follow (from codebase analysis)

| Concern | Pattern |
|---|---|
| Directive | `"use client"` at top of every `.tsx` |
| Icons | `lucide-react` (`DollarSign`, `Save`, `Search`) |
| Toast | `sonner` library (`toast.success()` / `toast.error()`) |
| API calls | Hook -> **Service** -> `apiClient` (3-layer, NOT direct tRPC) |
| Query keys | Simple array: `["finance-grid", periodStart, periodEnd]` |
| Mutation | `useMutation` + `queryClient.invalidateQueries` + toast |
| Error handling | `ErrorHandler.handleApiError()` from `lib/errorHandler.ts` |
| Types | `interface` for objects, `type` for unions, in `types/` subdir |
| CSS | Tailwind utility classes only |
| Table | Native `<table>` with sticky positioning (matches ProductionTable) |

## 4. Types (`types/finance.types.ts`)

```typescript
export interface FinanceInput {
  revenue_per_plate: number; packs_per_day: number;
  manpower_count: number; rate_per_manpower: number;
  manpower_days: number; num_sites: number;
  transport_cost_per_km: number; transport_kms: number;
  on_site_gas_cost: number; admin_percent: number;
  capex_amount: number; commission_percent: number;
  procurement_discount: number; aggregator_commission: number;
  consumables_percent: number; food_waste_percent: number;
  fuel_percent: number; electricity_percent: number;
  water_percent: number; food_cost: number;
}
export interface FinanceDerived { /* 13 computed fields */ }
export interface FinanceGridRow extends FinanceInput, FinanceDerived {
  site_id: string; site_name: string; period_start/end: string;
}
```

## 5. Service Layer (`services/financeService.ts`)

Follows `productionPlanService.ts` exactly:
- Add 3 endpoints to `repo/config.ts`: `finances.getFinanceGrid`, `finances.saveFinanceData`, `finances.updateFinanceRecord`
- Service object with 3 methods: `getFinanceGrid`, `saveFinanceData`, `updateFinanceRecord`
- Each method calls `apiClient.get/post/put()` which adds auth header, unpacks tRPC response

**Frontend-first:** Add mock fallback when backend isn't ready:
```typescript
if (!API_CONFIG.coreBaseUrl) return getMockFinanceGrid(params);
```

## 6. Hooks

**`useFinanceGrid.ts`** -- mirrors `useProductionPlan.ts` (~25 lines):
- Takes `{periodStart, periodEnd}`, returns `useQuery<FinanceGridResponse>`
- Query key: `["finance-grid", periodStart, periodEnd]`

**`useFinanceMutation.ts`** -- mirrors `useCreateClient.ts` (~45 lines):
- Key factory: `financeKeys = { all, grids, grid(period) }`
- `useSaveFinanceData()` -- mutation -> invalidate `financeKeys.grids()` -> toast
- `useUpdateRecord()` -- mutation -> invalidate -> toast
- Both use `ErrorHandler.handleApiError` on error

## 7. Components

### `FinancesModule.tsx` -- Container
- Own data fetcher via `useFinanceGrid`
- State: `activeTab ("inputs"|"pnl)`, `searchTerm`, `period`
- Renders toolbar (title, breadcrumb, search, Save All, tabs) + active tab

### `FinanceGrid.tsx` -- Shared Spreadsheet
- Props: `rows[], sites[], data{}, onCellChange`
- Handles: sticky header/top/left/right, section headers, input cells, calc cells, AVG column
- **Used by BOTH tabs** with different row definitions

### `FinanceInputTab.tsx` / `FinancePNLTab.tsx`
- Thin wrappers that pass their respective `INPUT_ROWS` / `PNL_ROWS` definitions to `<FinanceGrid>`

## 8. Navigation Wiring (edit 3 existing files)

| File | Change |
|---|---|
| `layout/lib/constants.ts` | Add `{ title:'Finances', icon:DollarSign, viewType:'finances' }` to NAVIGATION_ITEMS |
| `layout/types/layout.types.ts` | Add `'finances'` to ViewType union |
| `layout/components/DashboardLayout.tsx` | Add `case 'finances': return <FinancesModule />` |

## 9. Implementation Order

| Step | What | Deps |
|---|---|---|
| 1 | Types + compute-derived lib + defaults | Nothing |
| 2 | Service layer (+ mock fallback) + config edits | Step 1 |
| 3 | Hooks | Step 2 |
| 4 | FinanceGrid component | Step 1 |
| 5 | Tab components | Steps 3+4 |
| 6 | FinancesModule container | Steps 3+5 |
| 7 | Navigation wiring (3 file edits) | Step 6 |
| 8 | Manual test + `pnpm test -- --run` | Step 7 |
| 9 | Backend (DB migration, schema, repo, router) | Independent |

Steps 1-8 are pure frontend, developable immediately with mock data. Step 9 (backend) runs in parallel.

## 10. Known Risk: Sticky Scroll Overlap

Native `<table>` `position: sticky` has overlap issues during scroll (same as existing ProductionTable). Options:
- Ship as-is (matches codebase convention)
- Fix later if users report issues
- CSS Grid replacement (proven fix but diverges from convention)

**Recommendation:** Ship with `<table>`, address later.

---

## 11. Export Site-wise P&L to Excel

### Overview

An **Export P&L button** in the Site-wise P&L toolbar that downloads an `.xlsx` file matching the **"Site PL format" sheet** from `Chefcraft __ Fint.xlsx`.

### Target Excel Structure

Matches the "Site PL format" sheet from the client Excel exactly — 4 columns + 25 site columns + AVG column:

| Col A | Col B | Col C | Col D | Col E...Z (25 sites) | Last Col |
|---|---|---|---|---|---|---|
| **Expense / Line Item** | **Allocation / Computation Basis** | **Narration** | **System** | **W-F 09TH** ... **IKEA** | **AVG** |
| REVENUE | | Food Sales | Revenue per plate × packs/day | ops platform | ₹44,700 | ₹66,420 | ₹17,400 | ... | ₹xxx |
| DIRECT COSTS (A) | Wages / Manpower Cost | manpower × rate × days / sites | Input screen | ₹37,700 | ₹2,46,600 | ₹65,160 | ... | ₹xxx |
| | Consumables & Chemicals | 0.10% of sales | comes from food sales | ₹44.70 | ₹56.24 | ₹21.88 | ... | ₹xxx |
| | Food Waste | 1% of sales | comes from food sales | ₹447.00 | ₹562.42 | ₹218.79 | ... | ₹xxx |
| | Fuel Cost | 4% of sales | comes from food sales | ₹1,788 | ₹2,249 | ₹875.16 | ... | ₹xxx |
| | Electricity | .34% of sales | comes from food sales | ₹151.98 | ₹191.22 | ₹74.39 | ... | ₹xxx |
| | Water | .34% of sales | comes from food sales | ₹151.98 | ₹191.22 | ₹74.39 | ... | ₹xxx |
| | Food Cost | As per MIS | ops platform under menu planning | ₹18,450 | ₹14,119 | ₹6,130 | ... | ₹xxx |
| | Transportation Cost – Vehicle | cost/km × kilometers | input variable screen | ₹403.00 | ₹962.00 | ₹273.00 | ... | ₹xxx |
| | On-Site Gas | Enter cost | input variable screen | ₹2,320 | ₹4,106 | ₹1,100 | ... | ₹xxx |
| INDIRECT COSTS (B) | Administrative | .3% of sales | comes from food sales | ₹xxx | ₹xxx | ₹xxx | ... | ₹xxx |
| | Commission | Contractual % of gross sales | Calculation part | ₹xxx | ₹xxx | ₹xxx | ... | ₹xxx |
| | Capex – Pots, Pans, Carriers | capex / num sites | input variable screen | ₹xxx | ₹xxx | ₹xxx | ... | ₹xxx |
| SUMMARY & DERIVED METRICS | Total Cost (A+B+Capex) | Calculation part | ₹xxx | ₹xxx | ₹xxx | ... | ₹xxx |
| | Procurement Discount | Calculation part | Calculation part | ₹xxx | ₹xxx | ₹xxx | ... | ₹xxx |
| | Net Profit / (Loss) | Calculation part | Calculation part | ₹xxx | ₹xxx | ₹xxx | ... | ₹xxx |
| | Net Profit / Loss (%) | Calculation part | Calculation part | xxx% | xxx% | xxx% | ... | xxx% |
| | Aggregator commission if any | Calculation part | Calculation part | ₹xxx | ₹xxx | ₹xxx | ... | ₹xxx |
| | Final Profit / Loss | Calculation part | Calculation part | ₹xxx | ₹xxx | ₹xxx | ... | ₹xxx |
| | Final Profit / Loss % | Calculation part | Calculation part | xxx% | xxx% | xxx% | ... | xxx% |

### File Naming

`Finance_P&L_{YYYY-MM-DD}.xlsx` (e.g., `Finance_P&L_2026-06-23.xlsx`)

### Button Placement

In `FinancesModule.tsx` toolbar, next to Save All:

```
[🔍 Search...] [💾 Save All] [📥 Export P&L]
                              ↑ NEW
```

- Icon: `FileSpreadsheet` from lucide-react
- Styling: `bg-emerald-50 text-emerald-700 hover:bg-emerald-600 rounded-lg`

### Files to Create/Modify

| # | File | Action | Lines |
|---|---|---|---|
| 1 | `features/finances/lib/export/financeExport.ts` | **NEW** (~200 lines) | Export function |
| 2 | `features/finances/components/FinancesModule.tsx` | **EDIT** (+5 lines) | Add Export button + handler |

### Library & Dependencies

```typescript
import ExcelJS from "exceljs";        // Already used in production export
import { saveAs } from "file-saver";   // Already used in production export
```
**No new packages needed.**

### Implementation: `lib/export/financeExport.ts`

#### Function Signature
```typescript
export async function exportFinancePNLToExcel(
  gridData: FinanceGridRow[],
  periodStart: string,
  periodEnd: string,
): Promise<void>
```

#### Sheet Layout
```
paperSize: 'landscape'
fitToPage: true; fitToWidth: 1; fitToHeight: 2
margins: { left: 0.5, right: 0.5, top: 0.5, bottom: 0.5 }
```

#### Column Definitions

| Col | Header | Width | Style |
|---|---|---|---|
| A | Expense / Line Item | 28 | bold, size 12, bg-slate-50, border thin dark |
| B | Allocation / Computation Basis | 45 | italic, size 10, text-gray-500, wrapText |
| C | Narration | 20 | italic, size 9, text-gray-400, wrapText |
| D | System | 18 | italic, size 9, text-gray-400, wrapText |
| E..last-1 (sites) | 12 each | bold, size 10, center-aligned |
| Last col | AVG | 12 | bold, size 10, center, bg-slate-100 |

#### Row Rendering Order

```
Row 1: Title header (merged A:D)
  → "CHEFCRAFT HOSPITALITY LLP — SITE-WISE P&L"
  → bold, size 18, center, fill: #2E8F45 (green), font-color: white, height: 35pt

Row 2: Period info (merged A:D)
  → "Period: {periodStart} to {periodEnd}"
  → bold, size 12, center, fill: #F0FDF4, text-gray-600, height: 22pt

For each SECTION in PNL_ROWS definition:
  Section Header Row (merged A:lastCol):
    → UPPERCASE section name (REVENUE, DIRECT COSTS(A), INDIRECT COSTS(B), SUMMARY)
    → bold, size 11
    → Fill colors:
        REVENUE → bg-green-50, border-green-200, text-green-900
        DIRECT COSTS(A) → bg-amber-50, border-amber-200, text-amber-900
        INDIRECT COSTS(B) → bg-amber-50, border-amber-200, text-amber-900
        SUMMARY → bg-slate-100, border-slate-300, text-slate-800
    → height: 22pt

  Data Row:
    Col A: Line item label (bold, size 10, left-align, white bg)
    Col B: Formula description (italic, size 9, text-gray-500, wrapText)
    Col C: System source ("ops platform", "Input screen", or "Calculation part")
    Col D: blank
    Site cells:
      Money values → ₹XX,XXX.XX (#,##0.00), size 10, center
      Percentages → XX.X% (+/- prefix), bold semibold
      Positive profit → text-emerald-700 (money) / text-emerald-600 (%)
      Negative loss → text-red-700 (money) / text-red-600 (%)
      Total Cost → text-amber-900, bold
      Zero/empty → "-" (em-dash)
      Borders: thin light gray (#E5E7EB), bg white
    AVG cell: same formatting as data cells + bg-slate-100

Footer Row (merged A:lastCol):
  → "PREPARED BY : CHEFCRAFT SYSTEM"
  → bold, size 11, center, gray-500, height: 22pt
```

#### Number Formatting Rules

| Data Type | Format | Example |
|---|---|---|
| Currency (₹) | `#,##0.00` | `₹44,700.00` |
| Percentage | `0.0"+"%"` with +/- prefix | `+12.3%` or `-5.2%` |
| Zero/empty | `"-"` (em-dash) | `-` |
| AVG column | Same as data cells | Same as data cells |

#### Value Color Coding

| Condition | Font Color | Weight |
|---|---|---|
| Positive profit (money) | `#059669` (emerald-700) | Bold |
| Negative loss (money) | `#DC2626` (red-700) | Bold |
| Positive profit (%) | `#059669` (emerald-600) | Semibold |
| Negative loss (%) | `#DC2626` (red-600) | Semibold |
| Total Cost | `#D97706` (amber-900) | Bold |
| Zero/empty | `#94A3B8` (gray-400) | Normal |

### Integration in FinancesModule.tsx

```typescript
import { FileSpreadsheet } from "lucide-react";
import { exportFinancePNLToExcel } from "../lib/export/financeExport";

// Inside component:
const handleExport = async () => {
  if (!gridData || gridData.length === 0) {
    toast.error("No data to export");
    return;
  }
  await exportFinancePNLToExcel(gridData, periodStart, periodEnd);
  toast.success("P&L exported successfully");
};

// In toolbar JSX:
<button onClick={handleExport}
  className="flex items-center gap-2 px-4 py-2 bg-emerald-50 hover:bg-emerald-600 text-emerald-700 rounded-lg text-sm font-medium shadow-sm transition-colors">
  <FileSpreadsheet className="w-4 h-4" /> Export P&L
</button>
```

### Data Flow

```
User clicks "Export P&L"
  → handleExport() called
  → Validates gridData exists (from useFinanceGrid hook)
  → Calls exportFinancePNLToExcel(gridData, period)
    → Creates ExcelJS Workbook
    → Adds worksheet "Site PL format"
    → Writes title header, period header
    → Loops through PNL_ROWS definition:
        → Writes section headers (styled per spec above)
        → Writes data rows (formatted per type)
        → Writes AVG column
    → Writes footer
    → Generates Blob via writeBuffer()
    → saveAs(Blob, filename)
  → Browser triggers download
```

### Error Handling

| Scenario | Behavior |
|---|---|
| No data loaded yet | toast.error("No data to export"), no download |
| Grid has only zeros | Still exports (valid state) |
| ExcelJS error | toast.error("Export failed: {message}") |
| Browser blocks popup | file-saver handles natively |

### Updated File Structure

```
features/finances/
  types/
    finance.types.ts                 ← (existing)
  lib/
    compute-derived.ts                   ← (existing)
    finance.defaults.ts                    ← (existing)
    services/
      financeService.ts                     ← (existing)
    export/
      financeExport.ts                      ← NEW (~200 lines)
  hooks/
    useFinanceGrid.ts                       ← (existing)
    useFinanceMutation.ts                   ← (existing)
  components/
    FinancesModule.tsx                    ← EDIT (+5 lines for Export btn)
    FinanceGrid.tsx                        ← (existing)
    FinanceInputTab.tsx                      ← (existing)
    FinancePNLTab.tsx                        ← (existing)
```

**Total change:** 1 new file (~200 lines), 1 edited file (+5 lines)

### No Backend Changes Needed

Export is purely frontend — ExcelJS generates the `.xlsx` client-side. No new API endpoints, no DB changes, no migration required.

---

## 12. Implementation Phases (Updated)

| Phase | What | Deps | Status |
|---|---|---|---|
| 1 | Types + compute-derived lib + defaults | Nothing | Pending |
| 2 | Service layer (+ mock fallback) + config edits | Step 1 | Pending |
| 3 | Hooks | Step 2 | Pending |
| 4 | FinanceGrid component | Step 1 | Pending |
| 5 | Tab components | Steps 3+4 | Pending |
| 6 | FinancesModule container + **Export** | Steps 3+5 | Pending |
| 7 | Navigation wiring (3 file edits) | Step 6 | Pending |
| 8 | Manual test + `pnpm test -- --run` | Step 7 | Pending |
| 9 | Backend (DB migration, schema, repo, router) | Independent | Pending |
