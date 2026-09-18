# Task 1.3 — What the numeric columns cannot tell you

## Context

- Notebook: `climate.ipynb`, Part 1 loads NOAA Mauna Loa CO2 (`co2_mm_mlo.txt`) with
  `pd.read_csv(..., comment='#')`, so the entire header block (all prose notes) is discarded.
- The existing DataFrame `df` has hand-labeled columns from Tasks 1.1/1.2:
  `['year','month','decimal_date','average','smooth','std_days','uncertainty','empty']`.
  Per the raw file header the true meaning of columns 5–8 is:
  `deseasonalized`, `days` (# days in monthly mean), `st_dev` (of daily means), `unc` (of monthly mean).
- Verified against the live file (fetched 2026-09-18): 822 rows, Mar 1958 – Aug 2026.
  - Mar 1958 – Apr 1974 (SIO/Keeling data): `days=-1`, `st_dev=-9.99`, `unc=-0.99` on all 194 rows.
  - Two post-1974 flagged rows: Dec 1975 (`days=-1, st_dev=-9.99`) and Apr 1984 (`st_dev=-9.99`),
    but both keep plausible-looking `average` values → interpolated months, no visible gap in dates.
  - Dec 2022 – Jul 2023 rows (Maunakea backup site during the Nov 2022 eruption) have entirely
    normal `days`/`st_dev`/`unc` values → the site change is invisible in the numeric columns.

## Prose notes in the header to work with (verified by reading the raw file)

1. Pre-May 1974 data are from C. David Keeling / Scripps (SIO), not NOAA; for SIO rows the
   Ndays/std/unc columns carry no information (marked with negatives).
2. Monthly means are constructed from daily means and corrected to month center using the
   average seasonal cycle (asymmetric missing days would bias the mean).
3. Missing months were **interpolated**, flagged only by negative `st_dev`/`unc` — the series
   has no visible gaps.
4. NOTE (eruption): Mauna Loa measurements suspended Nov 29, 2022; Dec 2022 – Jul 4, 2023
   observations are from Maunakea Observatories (~21 miles away). No numeric flag marks this.

## Deliverables (in the assignment's own convention: code cells + markdown answer)

Insert into `climate.ipynb` between the Task 1.3 markdown cell and the "**Your answer:**" cell:

### Step 1 — Read the header block the DataFrame threw away

One markdown label cell + one code cell:

```python
import urllib.request

lines = urllib.request.urlopen(
    "https://gml.noaa.gov/webdata/ccgg/trends/co2/co2_mm_mlo.txt"
).read().decode().splitlines()

prose = [l for l in lines if l.startswith("#")]
"\n".join(prose[16:])
```

(Adjust the slice so the cell displays the SIO/interpolation/eruption notes, not the license block.)

### Step 2 — Show each note's footprint in the data

Rename to the true column names first (this doubles as fixing Task 1.1):

```python
co2 = df.rename(columns={"smooth": "deseasonalized", "std_days": "days",
                         "uncertainty": "st_dev", "empty": "unc"})
```

Then three small evidence cells, each ending in an displayed expression (no `print`):

- SIO block: `co2[co2.year < 1974][["days", "st_dev", "unc"]].agg(["min", "max", "count"])`
  → proves 194 rows of placeholders and where the NOAA record actually starts (May 1974).
- Interpolated months: `co2[(co2.year > 1974) & (co2.st_dev < 0)]`
  → shows Dec 1975 and Apr 1984 look like real observations in `average`.
- Eruption window: `co2[(co2.decimal_date >= 2022.95) & (co2.decimal_date <= 2023.55)]`
  → shows normal flags; the discontinuity is a fact from the header, not recoverable from the frame.

### Step 3 — Markdown answer, in student's own words

Cover:
- What I learned from the notes and how it would change the plot/interpretation:
  error bars or `std`-based statistics are meaningless for 1958–Apr 1974; interpolated months
  must be treated as missing for variability/trend work even though the series looks gap-free
  (822 rows = every month since Mar 1958); the Dec 2022 – Jul 2023 segment is a different site
  and should be annotated or handled carefully in any trend spanning it.
- Direct answer to "would a model reading only the numeric columns have any way to know?":
  the sentinel conventions (2) can be *suspected* from weird round negative values but not
  explained (only the header says who/why: SIO vs NOAA, interpolation policy); the eruption
  site swap is completely invisible — those rows are statistically ordinary. `comment='#'`
  discards exactly the information that makes the numbers interpretable.

## Constraints

- Rubric: no `print` in cells, no comments unless essential, no `try`, one task per cell,
  markdown label cell above each code cell, display via last-expression auto-print.
- Use only `pandas` / stdlib (`urllib`) — already consistent with the notebook.
- Live values above were verified against the current file; do not hardcode numbers in the
  answer that the evidence cells don't display.

## Validation

- Run the new cells top-to-bottom; each displays the expected rows/summary.
- Cross-check the eruption-window rows against the raw text file (already done in this plan).
- CI runs via GitHub Actions (`nbval` reproducibility check) — notebook must execute cleanly.

## Open items (non-blocking)

- Task 1.1's written answer could be strengthened later with the true column meanings found
  here; only the rename in Step 2 is strictly needed for 1.3.
