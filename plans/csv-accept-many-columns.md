# Plan: Accept multi-column CSV files with NPI header

## Context

Customers provide prescriber CSV files with many columns (e.g., 16 columns: FileType, ClientCd, PrescriberFname, PrescriberLname, **NPI**, StrDate, ...). The current CSV parser (`parseCsvNpis.ts`) rejects any file with >2 columns. However, these files have a properly named `NPI` header column — the parser should find it by name and extract NPIs from it, ignoring all other columns.

## Scope

**Single file change + tests:** `apps/web/utils/csv/parseCsvNpis.ts` and `apps/web/utils/csv/parseCsvNpis.test.ts`

No API, UI, or component changes needed — the parser already feeds into `validateNpiList()` which handles the rest.

## Current behavior (line 27)

```
columnCount > 2  →  reject with error
```

## Proposed behavior

```
columnCount ≤ 2  →  current logic (unchanged)
columnCount > 2  →  search header row for "npi" column (case-insensitive)
  ├── found     →  extract NPIs from that column index, skip header row
  └── not found →  reject with error
```

### Implementation steps

1. **RED** — Write failing tests:
   - Multi-column CSV with `NPI` header → extracts correct column
   - Multi-column CSV with `npi` (lowercase) header → works
   - Multi-column CSV with no NPI header → rejects with error
   - Multi-column CSV where NPI column has empty cells → skips empties
   - Use `from-client.csv` data (a few rows) as a realistic test case
   - Header-only multi-column file → "no data rows" error

2. **GREEN** — Modify `parseCsvNpis.ts`:
   - When `columnCount > 2`: scan `firstRowCells` for a cell matching `npi` (case-insensitive)
   - If found: record its index, set `hasHeader = true`, use that index for extraction
   - If not found: return current error
   - Update error message to hint that multi-column files need an NPI header
   - Extract NPI using the found column index (instead of hardcoded `[0]`)

3. **REFACTOR** — Clean up:
   - Extract `findNpiColumnIndex(cells: string[]): number` helper for clarity
   - Ensure all existing tests still pass (1-col and 2-col formats unchanged)

### Error message update

```
Current:  "Unsupported CSV format. File must have 1 column (NPI) or 2 columns (NPI, Name)."
New:      "Unsupported CSV format. File must have 1 column (NPI), 2 columns (NPI, Name), or include an NPI header column."
```

### Trimming

Already handled — `splitRow()` applies `.trim()` to every cell. Trailing spaces like `1659653723 ` are cleaned up. No changes needed.

## Files to modify

| File | Action |
|------|--------|
| `apps/web/utils/csv/parseCsvNpis.ts` | Add multi-column NPI header detection |
| `apps/web/utils/csv/parseCsvNpis.test.ts` | Add tests for multi-column scenarios |

## Verification

1. Run `npx jest parseCsvNpis --coverage` from `apps/web/` — all tests pass, 100% coverage
2. Run `npm run types:check` from `apps/web/`
3. Manual: drop `from-client.csv` in the FileDropzone — should parse and show validation results
