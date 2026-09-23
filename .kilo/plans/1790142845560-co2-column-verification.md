# CO2 monthly file: column-name audit and fix (Part 1, climate.ipynb)

## Context
`climate.ipynb` Part 1 hand-writes the `columns` list for
`https://gml.noaa.gov/webdata/ccgg/trends/co2/co2_mm_mlo.txt`.
Target: verify names against the file's own header block so the columns mean what
the source says they mean. Source of truth = in-file header block (also confirmed by the
`co2_mm_mlo.csv` header: `year,month,decimal date,average,deseasonalized,ndays,sdev,unc`).

## Findings (verified against raw header)
The file has **8** columns. Header labels:
decimal date | monthly average | de-seasonalized | #days | st.dev | unc. of mon mean
(year and month columns are not labeled in the header block; they are implied.)

| # | Hand-written name | What the file actually holds | Verdict |
|---|---|---|---|
| 1 | `year` | Year (yyyy) | correct |
| 2 | `month` | Month (mm) | correct |
| 3 | `decimal_date` | Decimal date (yyyy.mm) | correct |
| 4 | `average` | Monthly average CO2 (ppm) | correct (units not in name) |
| 5 | `smooth` | De-seasonalized CO2 (ppm) | MISSTATES - not smoothed; seasonal cycle *removed* |
| 6 | `std_days` | # days with measurements (count, 10-31; `-1` placeholder) | MISSTATES - a count, not a standard deviation |
| 7 | `uncertainty` | st.dev of the monthly mean (0.1-1.2; `-9.99` placeholder) | MISSTATES - it is a standard deviation, not the uncertainty |
| 8 | `empty` | unc. of monthly mean (0.05-0.6; `-0.99`/`0.00` placeholder) | MISSTATES - column is real, NOT empty |

## Which misstatement changes a published figure
`empty` is the dangerous one (paired with `uncertainty`): a coder building the
standard CO2 figure with error bars reaches for `uncertainty` (which actually holds
`stdev` of the monthly mean) and never uses the true uncertainty column because its
name says it is empty. Result:
- Error bars ~2-3x too wide (e.g. Aug 1980: stdev 1.05 vs unc 0.50; May 2005: 1.16 vs 0.44).
- On the 196 flagged/missing months the value is `-9.99`, so bars plunge far below zero
  and the y-axis is stretched.
- The real uncertainty column is silently discarded (not empty - holds 0.06-0.58).

Secondary: taking `smooth` literally and plotting it as the "smoothed" trend line gives
a jittery series (residual wiggles ~±0.5-1 ppm), not the smooth curve in NOAA's published
figure; note the actual trend line is not in this file at all. Taking `std_days` as a
standard deviation for error bars gives ±10-31 ppm bars, dwarfing the ~±3 ppm seasonal
cycle and inverting for 1958-1974 (`-1`).

The notebook's own plot (cell 48, `decimal_date` vs `average`) is unaffected - it uses
only correctly named columns.

## Recommended change
In the `columns` list:
```python
columns = ['year', 'month', 'decimal_date', 'average', 'deseasonalized',
           'ndays', 'stdev', 'unc']
```
Task 1.2 handling (missing values): treat `ndays == -1`, `stdev == -9.99`,
`unc <= 0` as sentinels; replace with NaN (196 rows total: 194 SIO rows Mar 1958-Apr 1974
+ Dec 1975 + Apr 1984).

Downstream cells that must be updated when renaming: the `df.rename(...)` cell
(no longer needed once the list itself is fixed) and the cells referencing
`co2.st_dev` / `co2.days` / `co2.unc` keep working if the rename stays; otherwise
update to the new names.

## Validation
- Re-run `df.head()` - values line up with the header labels (315.71 average,
  314.44 deseasonalized, -1 ndays, -9.99 stdev, -0.99 unc for 1958-03).
- Cross-check against `co2_mm_mlo.csv`: identical values and flag counts
  (`sdev < 0` -> 196 rows).
- Confirm no column holds the wrong content (e.g. `unc` < `stdev` and
  `unc` ≈ `stdev / sqrt(ndays)`).
