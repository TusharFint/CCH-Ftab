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
