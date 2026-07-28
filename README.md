# KQLHistorian

Recreate Wonderware Historian retrieval semantics (Cyclic, TWA StairStep, TWA Linear) on top of a Microsoft Fabric Eventhouse / KQL database, complete with a parameterized real-time dashboard that lets users switch retrieval modes from a dropdown.

The goal is to give industrial-data teams a clean, reproducible blueprint for migrating Wonderware-style retrieval onto modern Kusto, *including the retroactive-drift edge cases that make historian reporting tricky*.

---

## What's in here

```
docs/
  End-to-End-Historian-KQL-Guide.md         ← read this first (concepts + walkthrough)
  Wonderware-Historian-to-KQL-Design.md     ← design notes
  kql/
    01_schema_and_functions.kql             ← tables, MV, fn_ww_* functions
    02_seed_sample_data.kql                 ← demo data (now()-relative)
    03_validation_tests.kql                 ← per-mode sanity checks
    04_twa_corrected_functions.kql          ← Wonderware-accurate fn_ww_twa_*_v2 functions
    05_twa_comparison_test.kql              ← original-vs-corrected-vs-Historian validation harness
  dashboard/
    WonderwareRealtimeDashboard.template.json  ← parameterized dashboard definition
scripts/
  _env.ps1                                  ← loads .env / env vars (dot-sourced by the others)
  deploy_dashboard.ps1                      ← push the dashboard to Fabric
  read_param.ps1                            ← read a dashboard parameter back from Fabric
  reseed.ps1 / reseed2.ps1                  ← refresh demo data against current clock
  test_trend.ps1                            ← verify fn_ww_view returns rows for each mode
  compare_modes.ps1                         ← side-by-side numerical comparison
.env.example                                ← copy to .env and fill in
```

---

## Prerequisites

| Tool | Why |
|---|---|
| Microsoft Fabric workspace with an Eventhouse + KQL DB | Where the schema, data, and dashboard live |
| An existing KQL Dashboard item in that workspace | `deploy_dashboard.ps1` updates an existing dashboard by id (create an empty one first via the Fabric UI) |
| Azure CLI (`az`) | Token acquisition for both Fabric REST and Kusto APIs |
| PowerShell 5.1+ or PowerShell 7+ | All scripts are `.ps1` |

The scripts use `az account get-access-token` against two resources:
- `https://api.fabric.microsoft.com` — Fabric control plane (dashboards)
- `https://kusto.kusto.windows.net` — Kusto data + management plane

Run `az login --allow-no-subscriptions` (with `--tenant <id>` if needed) before first use.

---

## Quick start

### 1. Clone and configure

```powershell
git clone https://github.com/matthieulefebvre-publiccollab/KQLHistorian.git
cd KQLHistorian
copy .env.example .env
notepad .env
```

Fill in:

| Variable | Where to find it |
|---|---|
| `KUSTO_CLUSTER_URI` | Eventhouse → System overview → *Query URI* |
| `KUSTO_DATABASE` | The KQL database name inside the Eventhouse |
| `FABRIC_WORKSPACE_ID` | URL of your Fabric workspace (`/groups/<guid>`) |
| `FABRIC_KQL_DB_ID` | KQL DB item id (item details pane → Copy item ID) |
| `FABRIC_DASHBOARD_ID` | KQL Dashboard item id (create an empty dashboard first, then copy its id) |

You can also set these as real environment variables instead of using `.env`. The loader checks both.

### 2. Create the schema and functions

Open the KQL DB in the Fabric UI and run, in order:

1. [`docs/kql/01_schema_and_functions.kql`](docs/kql/01_schema_and_functions.kql) — creates `ww_tag_config`, `ww_raw_analog`, `ww_query_audit`, the `ww_mv_latest_by_tag` materialized view, and the `fn_ww_*` retrieval functions.
2. [`docs/kql/02_seed_sample_data.kql`](docs/kql/02_seed_sample_data.kql) — populates 4 tags with `now()`-relative sample points covering the last 30 minutes, including one late-arriving point that demonstrates retroactive drift.
3. [`docs/kql/04_twa_corrected_functions.kql`](docs/kql/04_twa_corrected_functions.kql) — installs the Wonderware-accurate `fn_ww_twa_stair_v2`, `fn_ww_twa_linear_v2`, and `fn_ww_twa_auto` functions (drop-in `_v2` replacements; the originals stay for comparison). See [TWA accuracy corrections](#twa-accuracy-corrections) below.

To validate the corrected functions against real Historian output, run [`docs/kql/05_twa_comparison_test.kql`](docs/kql/05_twa_comparison_test.kql) — a four-part harness (Test A: linear original-vs-v2, Test B: stairstep original-vs-v2, Test C: per-tag auto dispatch, Test D: corrected-vs-pasted-Historian exact validation). Edit the `PARAMETERS` block, then run each `// ----`-delimited section on its own.

Alternatively re-seed from PowerShell any time the demo window goes stale:

```powershell
powershell -ExecutionPolicy Bypass -File .\scripts\reseed2.ps1
```

### 3. Verify the functions return data

```powershell
powershell -ExecutionPolicy Bypass -File .\scripts\test_trend.ps1
```

Expected output (counts vary slightly based on bucket alignment):

```
Cyclic: rows=120 oldest=... newest=...
TWA_Linear: rows=124
TWA_StairStep: rows=85
```

If any of the three returns 0, your seed data has fallen outside the 30-minute window — re-run `reseed2.ps1`.

### 4. Compare modes numerically

```powershell
powershell -ExecutionPolicy Bypass -File .\scripts\compare_modes.ps1
```

For a single tag, prints `timestamp | Cyclic | Stair | Linear` per minute. Cyclic and Stair hold flat between raw samples; Linear interpolates between them. This is the easiest way to *see* the retrieval-mode differences.

### 5. Deploy the dashboard

```powershell
powershell -ExecutionPolicy Bypass -File .\scripts\deploy_dashboard.ps1
```

This reads [`docs/dashboard/WonderwareRealtimeDashboard.template.json`](docs/dashboard/WonderwareRealtimeDashboard.template.json), substitutes `__CLUSTER_URI__` / `__KQL_DB_ID__` / `__KQL_DB_NAME__` / `__WORKSPACE_ID__` from your `.env`, and POSTs to the Fabric `updateDefinition` endpoint.

Open the dashboard in Fabric. You'll see five tiles:

| Tile | What it shows |
|---|---|
| **Active Tags** | Count of enabled tags |
| **Latest Values** | Most recent value per tag (via the MV) |
| **Tag Trend View** | Time-series chart driven by `fn_ww_view(_retrieval_mode, …)` — switch modes from the dropdown |
| **Retroactive Drift Detector** | Tags whose latest TWA bucket changed since the previous run (late-arrival evidence) |
| **Late Arrivals (30m)** | Points where `ingest_time - source_timestamp > 5m` |

### 6. Read back a parameter (debugging)

```powershell
powershell -ExecutionPolicy Bypass -File .\scripts\read_param.ps1
```

Prints the JSON definition of the `_retrieval_mode` parameter. Useful if you edit the template and want to confirm Fabric accepted your change — Fabric's parameter schema is picky (see [`.github/.skills/07-realtime-dashboard/SKILL.md`](.github/.skills/07-realtime-dashboard/SKILL.md) for the validated rules).

---

## How the three retrieval modes differ

| Mode | Formula | Behavior between raw samples | Use it for |
|---|---|---|---|
| **Cyclic** | last value at-or-before each boundary | Flat (sample-and-hold) | Faithful replay of "what did the operator see at time T?" |
| **TWA StairStep** | time-weighted avg, segments held flat | Flat within segment | Digital / setpoint tags, modes, switch states |
| **TWA Linear** | time-weighted avg of linear interpolation | Smooth ramp between points | Analog process variables (temperature, pressure, flow) |

Late-arriving points retroactively change **TWA Linear** the most, **TWA StairStep** moderately, and **Cyclic not at all** (Cyclic only cares about the last value at-or-before each boundary — once that boundary passes with no point present, a later arrival doesn't move it).

For full background and worked examples see [`docs/End-to-End-Historian-KQL-Guide.md`](docs/End-to-End-Historian-KQL-Guide.md).

---

## TWA accuracy corrections

Comparing the original `fn_ww_twa_*` output against real Wonderware Historian TWA exports surfaced four independent discrepancies. [`docs/kql/04_twa_corrected_functions.kql`](docs/kql/04_twa_corrected_functions.kql) fixes all four as drop-in `_v2` functions:

| # | Problem | Symptom | Fix in `_v2` |
|---|---|---|---|
| 1 | **Label offset** | Wonderware timestamps each bucket at the interval **end**; the originals used `bin()` = interval **start**, so `Historian[T] == KQL[T − interval]` | Label buckets at `(bucket + interval)` |
| 2 | **Wrong interpolation per tag** | A stair-step tag was averaged as linear, so every transition diverged | Drive interpolation from `ww_tag_config.interpolation_type` via `fn_ww_twa_auto` |
| 3 | **5-second densification loss** | `make-series` at 5s + `any(value)` collapsed sub-5s points and only approximated the integral | Exact per-segment trapezoidal integration — no densification, no lost points |
| 4 | **Fabrication across bad-quality gaps** | `series_fill_linear` bridged a multi-hour bad-quality dropout with a fake ramp instead of `NULL` | Quality-aware: only integrate Good→Good segments; any bucket overlapping a bad/NULL span → `NULL` |

Once validated with `05_twa_comparison_test.kql`, repoint `fn_ww_view` to the `_v2` functions (see the note at the bottom of `04_twa_corrected_functions.kql`).

---

## Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| `Missing required environment variables` from `_env.ps1` | `.env` not filled in or not at repo root | `copy .env.example .env` and edit |
| `test_trend.ps1` shows 0 rows | Seed data older than 30 minutes | Re-run `scripts\reseed2.ps1` |
| Trend tile says "No results found" | Same as above, or wrong tag list in the tile | Re-seed, refresh dashboard |
| Trend chart looks flat across modes | Mixed Y-axis scales (deviations 0-1 vs temp 70°C) | Filter to a single tag in the dashboard to see the per-mode differences |
| `updateDefinition` returns 400 with "must NOT have unevaluated properties" | Dashboard parameter JSON shape — discriminator wrong | See the verified schema in [`.github/.skills/07-realtime-dashboard/SKILL.md`](.github/.skills/07-realtime-dashboard/SKILL.md) |
| `az account get-access-token` errors | Not logged in or wrong tenant | `az login --tenant <tid> --allow-no-subscriptions` |

---

## License

No license file is included yet. Treat the code as "all rights reserved" until a license is added. Open an issue if you need permission to reuse.
