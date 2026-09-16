# Plan: Total Industrial Output by Origin Region, EXIOBASE 3 (2022)

**Task:** climate.ipynb, Task 3.1–3.3 (Part 3: Emissions by industry)
**Data:** `https://data.source.coop/youssef-harby/exiobase-3/4588235/parquet/year=2022/format=ixi/matrix=Z/data.parquet`
**User decision (confirmed):** compute **Z row sums only** — `SUM(value) GROUP BY origin_region` on the 2022, ixi, matrix=Z file. This is what the prompt "from that file" literally allows.

## Definition and caveat (state in the notebook)

- Z is the **inter-industry transaction matrix**: rows = producing (origin) sector/region, columns = using (dest) sector/region.
- Row sums = **intermediate sales**, **not total output**. Gross output requires adding final demand `Y` (`x = Z·1 + Y·1`). Caveat: recompute Z-only against Z+Y optionally to quantify the gap; record "Z-only" label honestly in answers — this is exactly the recurring-answer trap the module targets.
- Values are in **million EUR** (EXIOBASE 3.8.1 convention; Z/Y have no `unit` column — confirm from README/metadata before quoting units).
- 2022 rows are **nowcasts** (1995–2020 observed, 2021–22 nowcast) per source.coop product description — note in verification block.

## Environment

- `requirements.txt` already has `duckdb==1.3.2`, `ibis-framework[duckdb]`, `pandas`, `pyarrow`.
- Use `s3://us-west-2.opendata.source.coop/youssef-harby/exiobase-3/...` or the HTTPS URL; `SET enable_object_cache = true;` once for speed. DuckDB's httpfs is preinstalled in the course env (do not `INSTALL/LOAD httpfs` if autoload works; install only if the query fails).

## Steps (implementation agent)

1. **Record the prompt** used (the exact user message) and the library choice (expected: pandas, the default in training data; record actual choice) in the Task 3.1 markdown cell.

2. **Schema discovery (one cell, one query each):**
   ```sql
   DESCRIBE SELECT * FROM read_parquet('<z-url>') LIMIT 1;
   SELECT count(*) AS nrows, count(DISTINCT origin_region) AS n_origin_region,
          count(DISTINCT origin_sector) AS n_origin_sector
   FROM read_parquet('<z-url>');
   ```
   Expected: columns `origin_region`, `dest_region`, `origin_sector`, `dest_sector_or_category`, `value`; 49 origin regions; ~49×163=~7987 origin sectors; ~64M rows (dense) or ~2–5M (sparse) — do not hardcode counts, report what DESCRIBE returns. If `year` exists as a column, constrain `year = 2022` (else trust the path partition).

3. **Compute (DuckDB via ibis, streamed aggregation):**
   ```sql
   SELECT origin_region, SUM(value) AS total_output_meur
   FROM read_parquet('<z-url>')
   GROUP BY origin_region
   ORDER BY total_output_meur DESC;
   ```
   This is the answer: 49 rows, region + total (million EUR). Include the 5 "Rest of World" regions whose codes start with `W` (e.g., WA/WM/WL — list exact codes from the query); do **not** silently exclude them as the README's country examples do, but label them as RoW aggregates.

4. **Naive-pandas attempt (Task 3.2):** `pd.read_parquet('<z-url>')` + `groupby('origin_region')['value'].sum()` inside the notebook's `measure()` harness (resource.setrlimit-style peak RSS). Record peak memory (GB), wall time (s), and whether it finished — expected: OOM/very slow on ~64M rows in a 4 GB session; if it succeeds, record numbers and compare answers.

5. **Constrained attempt (Task 3.3):** ibis with DuckDB backend, same aggregation; `ibis.read_parquet(...).aggregate(total_meur=ibis._['value'].sum(), by='origin_region')` or raw `con.sql(...)`. Measure with the same harness; fill the comparison table and verify **Z-only answer matches step 3 exactly** (bitwise-equal or within float tolerance — report which).

6. **Verification (in the notebook answer/verification block):**
   - `n_origin_region == 49`; region codes match EXIOBASE 3.8.1 list (44 countries + 5 RoW).
   - Global check: `SUM(value)` over Z (all origins × all destinations) must equal sum of the 49 region totals; recompute independently via `GROUP BY dest_region` sum (column check) — totals must be equal.
   - No missing `value`; check negatives (Z should be non-negative; small negatives possible) and report `min(value)`.
   - Sanity: total intermediate sales should be on the order of tens of trillions EUR; top regions expected US, CN, etc. Compare the sum of the 49 Z-row totals with the same year's 2020 observed value (same partition `year=2020`) to confirm scale/plausibility of the 2022 nowcast.
   - State caveat explicitly: reported number is **intermediate (interindustry) sales by origin region**, not gross output; a Z+Y variant exists and the gap quantifies final-demand share (compute optionally, one SQL query, for the notebook narrative).
   - Units: confirm million EUR from README/zenodo record (4588235).

7. **Fill notebook:** paste computed table (or first 10–20 rows + note "49 total"), answer "Which library did it choose, unprompted?" with the actual observation, and fill Tasks 3.2/3.3 measurement table. Existing conventions: no code comments, no print, one action per cell.

## Risks

- **Wrong matrix:** Z (not Y/F_satellite); ixi (not pxp).
- **Wrong direction:** group by `origin_region` (producing side), not `dest_region` (using side) — mistaking this reverses the answer.
- **Memory blowup:** full-frame pandas read of a dense ~64M-row file in a 4 GB session is expected to OOM (that is the Task 3.2 finding — do not "fix" it by installing more RAM; record the failure).
- **Mislabeling Z row sums as "total output":** keep the caveat; Z+Y is the true gross output.
- **Excluding RoW regions** to make the table look like a country ranking.

## Deliverable

A 49-row table: `origin_region` → `total (million EUR)` (Z row sums, 2022, ixi), plus recorded tool choice/measurements and the definition caveat in the notebook cells for Tasks 3.1–3.3.
