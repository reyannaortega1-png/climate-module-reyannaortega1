# Plan: Load Arctic Sea Ice Extent and Plot the Trend

**Scope:** Part 2, Tasks 2.1–2.3 of `climate.ipynb`
**Data source:** NSIDC Sea Ice Index G02135 — `https://noaadata.apps.nsidc.org/NOAA/G02135/north/monthly/data/`

## Context

The notebook asks us to load Arctic sea ice extent data and plot the trend. This is a critical-thinking exercise: the model may produce code that runs cleanly but loads the wrong file, the wrong column, or misrepresents the data. Each task requires verifying against source documentation, not trusting generated code.

## Steps

### Step 1 — Task 2.1: Record prompt (Task 2.1)

Record the prompt used to ask the model to load Arctic sea ice extent and plot the trend. This is documentation only — no code execution.

### Step 2 — Task 2.2: Inspect the data directory before loading (Task 2.2)

1. Fetch the directory listing at `https://noaadata.apps.nsidc.org/NOAA/G02135/north/monthly/data/` to identify all available files.
2. Read the NSIDC G02135 documentation at `https://nsidc.org/data/G02135` to understand file naming conventions, column definitions, units, and missing-value encodings.
3. Identify the specific file(s) containing monthly sea ice extent data (likely `north-monthly_sea_ice_extent.txt` or similar).
4. Record: what URL the model's code would have used, whether it matches a real file, and how much of the record it would cover.

### Step 3 — Load and inspect the data (Task 2.3, part A)

1. Download the identified file using `pandas.read_csv` with `sep='\s+'` (whitespace-delimited, consistent with Part 1 pattern).
2. Inspect the header/comment block of the raw file to determine actual column names and meanings.
3. Compare column names in the loader against the file's own header block — flag any mismatches (same verification pattern as Task 1.1).
4. Check for missing-value encodings (NSIDC typically uses `-9999` as a sentinel). Count affected rows per column.
5. Determine units from the documentation (million km² for sea ice extent).

### Step 4 — Task 2.3: Decide September minimum vs annual mean (Task 2.3, part B)

1. Compute both:
   - **Annual mean** sea ice extent for each year.
   - **September minimum** sea ice extent for each year (September is the Arctic sea ice minimum month).
2. Scientific decision: The September minimum is the standard index for Arctic sea ice loss because it captures the peak melt season's low point, which is the most climatically meaningful and most-studied metric. The annual mean dilutes the seasonal signal. For the question "Is Arctic sea ice declining?", the September minimum is the right choice. Document this reasoning.

### Step 5 — Plot the trend (Task 2.3, part C)

1. Use `plotnine` (consistent with Part 1 of the notebook).
2. Plot September minimum sea ice extent vs. year.
3. Label axes: x = "Year", y = "September Sea Ice Extent (million km²)".
4. Title: something descriptive like "September Arctic Sea Ice Minimum Extent".
5. No `print` statements in code cells. No comments. No error handling. Each cell does one thing.

### Step 6 — Verification block 2

Complete the verification block with substantive answers (not restated from the prompt):

1. **Extent:** What geographic region does this index cover? (Arctic Ocean, ~25°N and northward)
2. **Missing data:** How is missing data encoded? How many rows affected? How handled?
3. **Units:** Million km² — confirmed against documentation.
4. **Completeness:** What years does the record cover? Are there gaps?
5. **Cross-check:** Does the trend direction match published NSIDC results? Can you find one published figure to compare against?

## Key Decisions to Make

| Decision | Recommendation | Rationale |
|---|---|---|
| Which file to load | Monthly extent file from G02135 north directory | Not the "typical year" file — that is a normalized reference, not the raw record |
| pandas vs ibis | pandas | Dataset is small (~40 years × 12 months), fits comfortably in memory; ibis overhead is unnecessary |
| September minimum vs annual mean | September minimum | Standard climatological index for Arctic sea ice loss; annual mean dilutes seasonal signal |
| Missing value handling | Replace `-9999` with `NaN` before any computation | Sentinels must not enter statistics |

## Risks

- **Wrong file loaded:** The NSIDC directory contains multiple files (extent, concentration, typical year, etc.). Loading the wrong one produces a plausible-looking but incorrect plot — exactly the failure mode this exercise tests for.
- **Sentinel values treated as real:** `-9999` in raw data will produce extreme outliers in plots and statistics if not filtered.
- **Column name mismatch:** Generated code may assign column names that don't match the actual file header.

## Validation

1. Notebook cell runs without error (required by GitHub Actions `pytest --nbval-lax`).
2. Plot shows a declining trend consistent with published NSIDC Arctic sea ice decline (~13% per decade for September minimum).
3. Verification block answers are in own words, reference specific data properties.
4. No `print()` calls in code cells. No comments. No try/except.
