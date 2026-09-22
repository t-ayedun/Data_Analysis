# Data Analysis — SBEE / ABMF

Analysis notebooks for SBEE site monitoring and ABMF diesel/energy work.

## The SBEE monthly report

The pipeline lives in **`Colab Notebooks/Monthly Report Grading + Report Visuals.ipynb`**.

It tunnels through the bastion into the SBEE Postgres database, pulls a month of
`smart_device_readings` for the 33 mapped gateways, scores every site, and emits
the report visuals.

**Sites online less than the equivalent of 10 full days that month are excluded
before scoring**, not graded — `INSUFFICIENT_DATA_SITES` catches any site with
fewer than 14,400 (`10 × 1440`) one-minute readings with `power_cut_flag == 0`,
and `NO_DATA_SITES` separately catches sites with zero rows at all. This
replaced an earlier, narrower rule that only excluded a site if it was offline
100% of the month: most of a site's score components are *means* over
whatever rows it has, so a site online for barely a day still got a full
month's grade from that one day alone — e.g. a site online ~1.3 days in July
still averaged out to a C, because the mean doesn't know the rest of the month
never happened for that site. The underlying mechanism is the same NaN
cascade either way: a dead/under-sampled row's `active_power_overall_total ==
0` makes `power_factor_cal` and `updated_transformer_load_percentage` both
`NaN`, which makes `is_underloaded` read as `0` (`NaN < 30` is `False`) — so
every `np.select()` score falls through to its `default=` branch, which for 4
of 5 components is the score for a *healthy* site. The site count each month
is therefore **33 minus however many were excluded**, printed by the
exclusion cell and by the online-sites figure below.

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

Right after `power_cut_flag` exists, a **diagnostic-only** cell reports each
site's reporting completeness for the month (actual rows vs. `days × 1440`
expected). It feeds no score and no export. A real month measured **86.7%**
completeness — real gaps exist, concentrated in a handful of long, multi-site,
near-identical-duration windows (one 82 hours, several ~22.7 hours across
different sites at once). That pattern matches network-wide outages the
report itself already documents, not scattered missing single minutes.

A direct SQL query against `smart_device_readings` (bypassing this notebook
entirely) confirmed those specific windows have **zero rows in the database**,
not rows this pipeline is failing to pull — ruling out an extraction bug. The
same query, run with real elapsed-time integration instead of a flat
1-minute-per-row assumption, moved the total by only ~0.1% for a real month —
because during a genuine multi-hour outage, true consumption really is near
zero, so there's nothing meaningful to recover by assuming a longer duration.
**Consumption/coupure-hours below stay on the simple flat-interval sum** for
that reason — the added complexity was tested and didn't earn its keep.

Two other `smart_device_readings` columns that looked like promising leads —
`net_import_active_energy_overall_total` and `dt_active` — turned out to be
`NULL` for every row sampled, and the two energy-looking columns that *were*
populated (`energy`, `distributed_electricity`) turned out to just be
`active_power_overall_total / 60` at different rounding precisions, not a
richer independent source. Dead end, ruled out with real data rather than
left open.

**The actual root cause: individual phases can read negative** (a reversed
CT/current-transformer on one leg, or genuine reverse power flow), and the
DB's `active_power_overall_total` is a *signed* sum of the three phases — so
a negative reading on even one phase partially cancels the other two,
understating true total power drawn on that row. This isn't just a
consumption bug: `active_power_overall_total` also feeds `power_factor_cal`
and `updated_transformer_load_percentage`, so it was exposed to grading too.

Fixed once, right after the pull, before anything reads the column: the
query now also selects `active_power_overall_phase_a/b/c`, and
`active_power_overall_total` is immediately recomputed as
`|phase A| + |phase B| + |phase C|` — overwriting the DB's signed total.
Everything downstream (consumption, `power_factor_cal`, grading) reads the
same column name and inherits the fix automatically; no other formula
needed to change. The original DB value survives as
`active_power_overall_total_raw` for comparison. A single `NULL` phase is
treated as `0`, not left to propagate as `NaN` — the same failure mode that
let fully-dead sites grade out at C/B before the whole-month exclusion was
added, here narrowed to a single missing phase on an otherwise-live row.
The local raw-data cache self-invalidates if it predates these columns,
rather than silently reusing stale, uncorrected data.

### Aperçu mensuel (monthly overview)

Right after grading, before the CSV export, five aggregate figures print:

| Indicateur | How it's computed |
| --- | --- |
| Postes en ligne | Sites with voltage present on any phase on the last day of data this month, out of the graded population — sites excluded below the 10-day online threshold are counted separately, not folded into this denominator |
| Score moyen par poste | `site_summary['total_score'].mean()` |
| Consommation totale (kWh) | `Σ active_power_overall_total × (1/60 h)` — `active_power_overall_total` is already in kW (`POWER_IS_WATTS = False`) and is the phase-absolute-value-corrected total (see above), not the DB's raw column |
| Revenu estimé (CFA) | Consommation × 125 CFA/kWh (`TARIFF_CFA_PER_KWH`) |
| Total des coupures (heures) | `Σ power_cut_flag × (1/60 h)` |

All five are computed on `df` *after* sites below the 10-day online threshold
are filtered out — the same population the pie chart, the CSV, and the map
use. Before the whole-month-only version of this rule, a fully-dead site's
~44,000 minutes of `power_cut_flag == 1` were counted as portfolio outage time
on a site that wasn't operating at all, substantially inflating this figure.
Consumption barely moves from the exclusion alone — an excluded site
contributes little real energy regardless. All five also feed the
month-over-month comparison table further down (not just this standalone
printout), each shown against last month's figure with a `%` variation —
revenue's variation is mathematically identical to consumption's, since it's
a constant multiple of it, but it's tracked as its own history column rather
than special-cased in the comparison logic.

`POWER_IS_WATTS` was originally left as an untested guess (`True`) and silently
undercounted a real month's consumption by 1000x. It's derived now, not
guessed: the grading pipeline's own `updated_transformer_load_percentage`
(`active_power / power_factor / capacity_kVA`) only comes out to a sane
0–150%-ish range if `active_power_overall_total` is already the same order of
magnitude as capacity (50–800 kVA across this portfolio) — i.e. kW. The cell
also checks this on every run and prints a loud warning if that month's
average power looks wildly out of scale with the portfolio's capacities,
rather than silently trusting the flag forever.

Notes on two of the figures: a site with zero voltage on one particular day (but
not the whole month — that case is excluded from grading entirely, above) still
reads as *both* offline that day *and* a day of power-cut time — the raw data
can't tell "no grid power" apart from "gateway stopped reporting" at that
granularity. And the online-site count also prints the specific offline site
IDs, not just the total, so it's checkable against sites you already know are
having issues.

Month-over-month variation needs last month's five figures, carried the same
way as the grade counts below: the cell prints `PREVIOUS_MONTH_OVERVIEW =
{...}` at the end of each run, ready to paste into next month's control
panel. `None` skips the variation columns for that run. A history row written
before `estimated_revenue_cfa` existed simply lacks that key — the comparison
falls back to `N/A` for that one figure rather than crashing, the same
graceful-degradation pattern used for every other Aperçu figure added since
this notebook started keeping history.

### CSV export

`<month>_grades.csv` uses the same French headers and column order as the
published report's spreadsheet — `Postes, Score cumulé (mensuel), Niveau
final, Code couleur, Puissance du transformateur, % de temps en sous-charge,
Score du taux de charge du transformateur, Score des coupures de courant,
Score du déséquilibre de courant, Score du facteur de puissance, Score du
courant de neutre, Score du DHT` — built straight from `site_summary`, which
is itself grouped from `df` *after* the exclusion cell ran. An excluded site
can't appear in this CSV; there's no separate filter to keep in sync with the
exclusion rule, and no manual row-deletion needed before handing it to the
report.

Outputs (all gitignored — reproduced by running the notebook) land under
`reports/<REPORT_MONTH>/` on the Colab VM's own disk, month-prefixed so
nothing overwrites a prior run: `<month>_grades.csv`,
`<month>_grade_distribution_pie.png`, `<month>_comparison_chart.png`,
`<month>_migration_sankey.html`, `<month>_migration_sankey.png`,
`<month>_graded_report_map.html`, `<month>_graded_report_map.png`,
`<month>_monthly_day_consumption.png` — plus a zipped copy of the whole
folder from the notebook's last cell, which also auto-downloads it. The two
PNGs are static exports of the interactive Sankey/map, for anyone who just
wants to open a file rather than a browser view: the Sankey PNG uses
`kaleido` (installed and Chrome-provisioned automatically on first use, via
`plotly.io.get_chrome()`, if not already present); the map PNG has no
plotly/folium equivalent, so it's a separate matplotlib scatter recreating
the same grade colors and score-scaled marker sizes, auto-scaled to the
plotted sites' own bounding box rather than a fixed city-wide view.

`reports/` is local to that one Colab session and does not survive to next
month — deliberately. `File → Open notebook → GitHub` loads only the
notebook's own content into a brand-new, disposable VM each time, often run
by whoever's turn it is, not necessarily the same person or Google account
as last month. Nothing tied to one person's Drive would reliably work for
someone else taking over, so this notebook doesn't rely on any shared
storage at all — see "Monthly control panel" below for how the one piece of
state that actually needs to carry over (last month's grade counts) is
handled instead.

### Monthly control panel

The notebook's first code cell is the only thing that should need editing each
month:

```python
REPORT_MONTH = "2026-08"          # this month's report
PREV_REPORT_MONTH = None          # None = auto (the month before); set
                                   # explicitly only for a non-adjacent comparison
PREVIOUS_MONTH_COUNTS = [0, 6, 18, 5]  # from the end of LAST month's run
```

Change `REPORT_MONTH`, then `Runtime → Run all`. Every date, filename, and
chart title derives from `REPORT_MONTH`/`PREV_REPORT_MONTH`.
`PREVIOUS_MONTH_COUNTS` is the one field that has to be typed in by hand —
the last cell of every run prints the exact line to paste into next month's
control panel, e.g. `PREVIOUS_MONTH_COUNTS = [0, 6, 18, 5]`. That's a
deliberate choice, not a missing feature: it works identically no matter who
runs the notebook or which Google account they're signed into, because the
value travels inside the notebook file itself — the one thing every run
actually has — rather than in a shared file only some people can reach.
First time this notebook has ever graded sites? Use the real counts if
known, or `[0, 0, 0, 0]`.

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
- The Sankey's flow lines are a **real per-site join** against `history.parquet`
  (`Status`: `Graded`/`Offline`/`No Data`, one row per roster site every month) —
  not an aggregate estimate. A site offline last month and graded again this
  month flows `Offline → grade` directly; only a gateway serial genuinely never
  seen before lands in `Nouveaux Postes`. One transitional edge case: comparing
  against a month whose history predates the `Status` column routes that site to
  a dedicated `Statut inconnu` bucket rather than guessing — fires at most once,
  for the single month straddling this change.
- `Connect to SBEE Database.ipynb` and `Plots for Monthly Reports - No Access to
  DB.ipynb` still hand-edit dates/filenames/counts per month — not yet migrated to
  the control-panel pattern above.
- A site online *at least* 10 days but still down for part of the month (so not
  excluded) still gets its down-minutes scored via the same NaN-cascade default
  branches described above for 4 of 5 row-level components, inflating those
  components' means somewhat. `power_cut_score` does correctly punish the
  downtime (it's keyed off raw minute-count, not a row-mean), so `total_score`
  isn't blind to it — just understated relative to zeroing those specific
  components on `power_cut_flag==1` rows, which would be a bigger change to the
  scoring formulas than the 10-day exclusion threshold above. The threshold
  raise narrowed how much this can distort a grade (a site can no longer coast
  on a single good day), but doesn't eliminate it for a site online, say, 15 of
  31 days.
- Consommation totale sat ~7–10% below published/web-app figures for two
  checked months, even under the most generous possible accounting of the
  per-minute data. Confirmed not an extraction bug (the gaps genuinely don't
  exist in the database), not the per-minute interval assumption (tested,
  moved the total by ~0.1%), and not a richer unused column
  (`net_import_active_energy_overall_total`/`dt_active` are unpopulated;
  `energy`/`distributed_electricity` are just `power/60` restated). The actual
  cause — signed phase cancellation in the DB's `active_power_overall_total`
  — is now fixed (see the Aperçu mensuel section above). Whether it fully
  closes the gap to the published figures hasn't been confirmed against a
  live re-run yet.
- The SQL query (tunnel/query cell) hardcodes its own independent copy of the 33
  gateway serials in `WHERE gateway_serial IN (...)`, separate from
  `gateway_serial_mapping` (which runs *after* the query). Editing the mapping to
  add or remove a site has no effect on what's actually pulled from the DB unless
  the query's list is edited too.
