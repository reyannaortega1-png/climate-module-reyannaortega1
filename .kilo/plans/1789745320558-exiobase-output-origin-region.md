# Plan: Total Industrial Output by Origin Region, EXIOBASE 3 (2022)

**Task:** climate.ipynb, Task 3.1–3.3 (Part 3: Emissions by industry)
**Data:** `https://data.source.coop/youssef-harby/exiobase-3/4588235/parquet/year=2022/format=ixi/matrix=Z/data.parquet`
**User decision (confirmed in earlier planning session, carried forward):** compute **Z row sums only** — `SUM(value) GROUP BY origin_region` on the 2022, `ixi`, `matrix=Z` file. "From that file" is satisfied by this; do not silently switch to Z+Y.

## Definition and caveat (state in the notebook, don't silently decide)

- Z = inter-industry transaction matrix: rows (origin) are producers, columns (dest) are users. Row sums = **intermediate sales by origin region**, NOT gross output. True gross output is `x = Z·1 + Y·1` (add final demand from `matrix=Y`). Compute Z+Y later only as an optional, explicitly-labeled sanity gap.
- Units: **million EUR** (EXIOBASE 3.8.1 convention; Z/Y carry no `unit` column — confirm from the source.coop README / Zenodo record 4588235 before quoting).
- 2022 rows are **nowcasts** (1995–2020 observed, 2021–22 nowcast) per the source.coop product description — note this in the verification answer.

## Environment / execution notes

- `requirements.txt` already has `pandas`, `duckdb==1.3.2`, `ibis-framework[duckdb]`, `pyarrow`. Do not `pip install` anything.
- DuckDB `httpfs` is autoloadable in the course env; if the remote read fails with a missing-extension error, `INSTALL httpfs; LOAD httpfs;` — otherwise leave it alone. Optionally `SET enable_object_cache = true;` once for speed.
- Use the HTTPS URL above or `s3://us-west-2.opendata.source.coop/youssef-harby/exiobase-3/4588235/parquet/year=2022/format=ixi/matrix=Z/data.parquet`.
- This planning session has a restricted shell (no code execution). The implementation agent must run everything as notebook cells in `/home/jovyan/climate-module-reyannaortega1/climate.ipynb` (notebook-first agent, `.kilo/agents/data.md`), appending cells, not deleting existing ones.

## Steps (implementation agent)

1. **Record the prompt** (exact user message, and note "model chose pandas, unprompted" once observed — expected default from training data; record the actual library used) in the Task 3.1 markdown cell.

2. **Schema discovery** (one cell, one query):
   ```sql
   DESCRIBE SELECT * FROM read_parquet('<z-url>') LIMIT 1;
   SELECT count(*) AS nrows,
          count(DISTINCT origin_region) AS n_origin_region,
          count(DISTINCT origin_sector) AS n_origin_sector
   FROM read_parquet('<z-url>');
   ```
   Expected columns: `origin_region`, `dest_region`, `origin_sector`, `dest_sector_or_category`, `value`. Do not hardcode row counts — report what the query returns (~49 roots; ~163 sectors ⇒ 49×163≈7987 origin-sector combos; dense ≈64M rows or sparse 2–5M rows). If a `year` column exists, add `WHERE year = 2022`; otherwise trust the path partition.

3. **Compute the answer** (DuckDB via ibis, streamed aggregation — this is the recommended primary path):
   ```sql
   SELECT origin_region, SUM(value) AS total_output_meur
   FROM read_parquet('<z-url>')
   GROUP BY origin_region
   ORDER BY total_output_meur DESC;
   ```
   Result: 49 rows. Include the 5 Rest-of-World aggregates whose codes start with `W` (list exact codes found); label them as RoW, do not silently exclude them.

4. **Naive-pandas attempt (Task 3.2):** in a fresh cell, `pd.read_parquet('<z-url>')` then `groupby('origin_region')['value'].sum()`, wrapped in the notebook's existing `measure()` harness (records peak RSS + wall time). Record: peak memory (GB), elapsed (s), finished yes/no. Expected on a 4 GB session with ~64M rows: OOM or extreme slowness — that is the finding; do not "fix" by changing resources.

5. **Constrained attempt (Task 3.3):** ibis DuckDB backend doing the same aggregation:
   ```python
   import ibis
   con = ibis.duckdb.connect()
   t = con.read_parquet('<z-url>')
   t.group_by('origin_region').aggregate(total=ibis._['value'].sum())
   ```
   or raw `con.sql(...)` (equivalent to step 3). Measure with the same `measure()` harness; fill the comparison table:
   | approach | peak memory | elapsed | did it finish? |
   |---|---|---|---|
   | unguided | | | |
   | ibis / duckdb | | | |
   Verify the two approaches give the **same** answer (bitwise-equal or within float tolerance — report which; if different, that is a finding).

6. **Verification (in the notebook answer / Verification-block style):**
   - `n_origin_region == 49` (44 countries + 5 RoW); codes match EXIOBASE 3.8.1.
   - Grand total check: `SUM(value)` over the whole file equals the sum of the 49 region totals; also recompute `GROUP BY dest_region` and confirm equal grand sum.
   - Missing/null `value` count = 0; report `min(value)` (small negative values possible — say so).
   - Plausibility: total intermediate sales on the order of tens of trillions EUR; top origins expected US, CN, DE, JP, etc. Compare the 2022 result against the `year=2020` partition (`observed`, same query) to confirm scale and that the nowcast gap is plausible.
   - State caveat explicitly: reported number is intermediate sales by origin region, not gross output.

7. **Fill the notebook:** paste the 49-row table (or top 20 + note "49 total"), answer "Which library did it choose, unprompted?" (observed), Tasks 3.2/3.3 measurement table, and keep the Z-only caveat. Leave Task 3.4 reflection as evidence + short draft (pandas dominates training data / is the default `read_*`-aware answer) — final prose is the student's.

## Risks

- **Wrong matrix/file:** Z (not Y/F_satellite), `ixi` (not `pxp`), partition `year=2022`.
- **Wrong grouping:** `origin_region` (producer), NOT `dest_region` (user) — reversing flips the ordering/reverses the answer.
- **Memory blowup on naive pandas** (Task 3.2's actual purpose) — record, don't avoid.
- **Mislabeling Z row sums as "total output"** — keep the caveat.
- **Dropping RoW regions** to fake a clean country ranking.
- **Hardcoding counts** from prior expectations instead of reading them from the schema query.

## Deliverable

A 49-row `origin_region → total (million EUR)` table (2022, ixi, Z row sums) in climate.ipynb, plus the recorded tool choice, peak-memory/elapsed measurements for both approaches, equality check, and the Z-only vs gross-output caveat for Tasks 3.1–3.3.
