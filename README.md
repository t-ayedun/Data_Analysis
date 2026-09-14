# Data Analysis — SBEE / ABMF

Analysis notebooks for SBEE site monitoring and ABMF diesel/energy work.

## The SBEE monthly report

The pipeline lives in **`Colab Notebooks/Monthly Report Grading + Report Visuals.ipynb`**.

It tunnels through the bastion into the SBEE Postgres database, pulls a month of
`smart_device_readings` for ~30 gateways, scores every site, and emits the report
visuals.

Scoring components (100 points total):

| Component | Weight |
| --------------------------- | -----: |
| Transformer load percentage |     25 |
| Power cut                   |     25 |
| Current unbalance           |     15 |
| Neutral current             |     15 |
| THD                         |     10 |
| Power factor                |     10 |

Grades: **A** ≥ 90, **B** ≥ 75, **C** ≥ 60, **D** < 60 (🟢 🟡 🟠 🔴).

Outputs (all gitignored — reproduced by running the notebook) land under
`reports/<REPORT_MONTH>/`, month-prefixed so nothing overwrites a prior run:
`<month>_grades.csv`, `<month>_grade_distribution_pie.png`,
`<month>_comparison_chart.png`, `<month>_migration_sankey.html`,
`<month>_graded_report_map.html`, `<month>_monthly_day_consumption.png` — plus
a zipped copy of the whole folder from the notebook's last cell.

### Monthly control panel

The notebook's first code cell is the only thing that should need editing each
month:

```python
REPORT_MONTH = "2026-08"       # this month's report
PREV_REPORT_MONTH = None       # None = auto (the month before); set explicitly
                                # only for a non-adjacent comparison
```

Change `REPORT_MONTH`, then `Runtime → Run all`. Every date, filename, chart
title, and the previous month's grade counts (auto-loaded from that month's
own saved CSV, with a `MANUAL_PREVIOUS_MONTH_COUNTS` fallback for the first
run or a missing file) derive from those two values — no other cell should
need hand-editing.

### Related notebooks

| Notebook | Role |
| --- | --- |
| `Plots for Monthly Reports - No Access to DB.ipynb` | Same visuals, grade counts hardcoded — offline fallback. Same hand-edit-per-month problem as below; not yet migrated to the control-panel pattern — planned follow-up |
| `Connect to SBEE Database.ipynb` | Earlier version of the same pipeline (January, `jan_grades.csv`). Same hand-edit-per-month problem; not yet migrated — planned follow-up |
| `Weekly Report Plots.ipynb` | Same DB connection, 7-day window — the **weekly** report |
| `Reliability Data.ipynb`, `Reliability Metrics Q1 2026.ipynb`, `Combine CSVs & Final Reliability Metrics Visual.ipynb` | Multi-month power-cut / reliability series |
| `ABMF May 2024.ipynb`, `June 2024.ipynb` | ABMF project — grid/generator kWh from CSVs, unrelated to SBEE |

`Colab Notebooks/Archived Reports/` holds one-off monthly notebook clones from
before the control panel existed (July, August) — kept as a record of what those
reports actually looked like. That pattern is retired: the canonical notebook
above is reused every month now, nothing new gets cloned.

## Running the notebooks

Execution happens on a **Colab runtime**, not locally — that is where the secrets
live and where the bastion accepts connections.

1. In VS Code, sign in with **Google** (person icon → Sign in). The Colab extension
   needs a Google session; a GitHub session does nothing for it.
2. `Select Kernel` → `Colab` → connect.
3. Run the `paramiko` downgrade cell first (`paramiko<3.4.0` — the bastion key is an
   older format newer paramiko rejects). Accept the runtime restart.
4. Run the imports cell, then the secrets/tunnel cell.

If the kernel picker does nothing: check for a **Restricted Mode** banner and run
`Workspaces: Manage Workspace Trust` → Trust. The Colab extension declares no
untrusted-workspace capability, so it is silently disabled in an untrusted folder.

### Fallback — run via GitHub

If the extension will not cooperate: in Colab web, `File → Open notebook → GitHub`,
run there, then `File → Save a copy in GitHub` to send outputs back. Pull locally.

## Secrets

Never in the repo. Set them in the Colab web UI (🔑 key icon in the sidebar), with
**Notebook access** enabled for each:

`BASTION_KEY` (full multi-line PEM) · `BASTION_IP` · `BASTION_USER` ·
`DB_PRIVATE_IP` · `SBEE_DB_USER` · `SBEE_DB_PASSWORD` · `SBEE_DB_NAME`

`DB_PORT` is hardcoded to `5432` in the notebook.

Note the inconsistent prefixing: bastion vars are bare, DB credentials are `SBEE_`-prefixed,
but `DB_PRIVATE_IP` is not. Names are case-sensitive and must match exactly.

## Version control

Notebook outputs are **stripped on commit** by an `nbstripout` filter, so diffs show
the logic (thresholds, weights, queries) instead of base64 blobs. The working copy
keeps its outputs.

The filter is wired to `python -m nbstripout` because `nbstripout.exe` is not on PATH.
After a fresh clone, re-establish it:

```sh
pip install nbstripout
git config filter.nbstripout.clean "python -m nbstripout"
git config filter.nbstripout.smudge cat
git config filter.nbstripout.required true
git config diff.ipynb.textconv "python -m nbstripout -t"
```

Deliberately untracked: the large CSVs, `Archive/` (original Drive zip exports),
generated report artifacts, `Untitled*.ipynb` scratch notebooks, and the
`Data Analyst Capstone Project - Ibrahim Salman/` folder (a colleague's work).

## Known issues

- The secrets/tunnel cell catches its own exceptions and prints `❌ Database error`,
  so the cell "succeeds" while `df` is never assigned. Failures surface later as a
  confusing `NameError: name 'df' is not defined`.
- The Sankey's flow lines (who moved from which grade to which) are a best-fit
  *estimate* reconciling both months' totals, not a verified per-site migration —
  this notebook doesn't retain each site's grade across months to compute a real
  one. Marked in the chart's own title and in a code comment.
- `Connect to SBEE Database.ipynb` and `Plots for Monthly Reports - No Access to
  DB.ipynb` still hand-edit dates/filenames/counts per month — not yet migrated to
  the control-panel pattern above.
