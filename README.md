# Hospet Cost Studio

One app for the cost of one tonne of hot metal (Rs/tHM): the **sinter model**, the **MBF (furnace) model** and a
**combined model** that feeds the sinter plant's result into the furnace. The two original dashboards are kept whole.

    pip install -r requirements.txt
    streamlit run app.py

Python 3.10+ recommended. Google Fonts and the Material icon font need internet; every font has a fallback.

## What is in it

| Workspace | Pages |
|---|---|
| **Combined cost model** (landing) | Hot metal cost, Uploads & shared settings, Basicity scan, Price drivers, Saved scenarios, Export, Glossary |
| **Sinter model** (original dashboard, 12 pages) | Upload & Settings, Inputs, RM Stock & Materials, Dashboard, Recipe & Composition, Inventory Usage, Manual Burden Control, Scenario Analysis, Plant Run Validation, Productivity, Wet Specific Consumption, Reports |
| **MBF model** (original dashboard, 10 pages) | Upload & settings, Inputs, Materials & stock, Dashboard, Burden & cost, Slag oxides, Trends, Scenario analysis, Reports & export, Heat audit |

## Navigation

One sidebar, no model switch: a search box, then three collapsible sections (Combined, Sinter, MBF) whose pages are rounded pills with
an icon. The arrow at the top right folds the sidebar into an icon rail (the three models, then the pages of the current one).
Colours and fonts are the shared theme in `shared/theme.py`.

## How the combined model works

1. The sinter model runs on its own Excel file. Its result (price = dry raw-material cost + sinter O&M, plus Fe, CaO, SiO2,
   Al2O3, MgO) replaces the furnace table's one switched-on **Sinter** row. Moisture, S and Mn stay from the MBF Excel.
2. The furnace model runs and says how much sinter it uses (kg/tHM). Times the plan (tHM) that is the sinter tonnage
   needed, and the sinter model runs again at that tonnage. This repeats until the tonnage settles.
3. Rules: the run reports whether it converged; a failed sinter run stops the loop with the sinter model's message;
   sinter is capped at what the plant can make within tolerance (Optimal or Relaxed), and the furnace then takes more
   ore for the rest; sinter and furnace materials, stock and stock checks are never mixed, even when names match.
4. **Furnace O&M** (Rs per tonne of hot metal, default 0) is added to the raw-material cost, so the hot metal cost is
   raw materials (sinter row included, with its own O&M) + furnace O&M. Set it under *Uploads & shared settings* or *MBF > Inputs*; it is one setting.

The **Hot metal cost** page shows both burdens: the sinter burden (dry kg and Rs per tonne of sinter, BF returns at zero cost) and the
furnace burden (dry kg and Rs per tHM, with O&M as its own row). Each table's total ties to the model's own cost.

### One answer on every page

The MBF workspace has a switch, **Sinter row in the furnace table**: *From sinter model* (default) or *Typed row* (the original MBF behaviour).
In the default mode the Sinter row takes the **combined run's** sinter result whenever a combined run is current (same sinter tonnage, same
sinter cap, same plan), so the MBF page and the combined page give the same sinter share and cost. Before any combined run it takes the sinter
dashboard's own last run (its planning tonnage) and says so; that run can differ from the combined result because the combined loop re-runs the
sinter model at the tonnage the furnace needs. Changing any input marks the combined run stale and the MBF page falls back to the sinter dashboard's run until you run both again.
Plan (tHM) sets the furnace's stock caps and the sinter tonnage; the MBF page follows the combined plan while it uses a combined run.

## Editing what you upload

*Uploads & shared settings* (both files) and *Sinter model > Upload & Settings* show an editable table under the uploader: on/off, stock, price,
chemistry, Tech min/max (sinter), moisture, group, and add or delete rows. **Nothing changes until you press Confirm changes**; *Discard edits*
throws them away. Bad values (negative, above 100 %, duplicate names, ...) are refused with the reason. The furnace table is the MBF dashboard's own,
so *MBF > Materials & stock* sees the same edit.

## Known behaviour to read before trusting a number

* **Knife-edge near the sinter ceiling.** With the sample files the furnace's sinter demand can flip between two burdens
  within a tonne or so of sinter, because several slag limits bind together. The loop narrows it to about half a tonne
  and shows the result from the side where the plant makes at least what the furnace uses. Differences of about
  Rs 50/tHM or less are noise. The page says so when it happens.
* **Relaxed sinter.** The sample sinter misses SiO2 (6.07 against 5.8) and Al2O3/SiO2 but is inside the approved
  tolerance, so the hot metal cost is an estimate.
* A combined run takes a few to 25 s; price drivers about a minute; scans about 20 s per few points.

## Layout

    app.py                 router, sidebar (search, sections, icon rail), MBF sinter-row banner
    combined/loop.py       the loop (no Streamlit code): convergence, cap, failure stop, knife-edge, burden compositions
    combined/handoff.py    sinter -> furnace row (combined run first), stale detection, input diffs, the two stock checks
    combined/page.py       the combined pages
    shared/editor.py       editable material tables with a Confirm step (sinter file, furnace file)
    shared/nsrun.py        runs the original dashboards (see below)
    shared/theme.py        palette, fonts, one stylesheet for all three workspaces, page grid
    sinter/                original sinter dashboard + engine (optimizer.py)       -- unchanged
    mbf/                   MBF dashboard + engine (optimiser.py, analytics.py) + its tests -- V7 plus the O&M setting
    sample_inputs/         SInter_Input.xlsx, MBF_Input.xlsx
    tests/                 combined tests; ENGINE_HASHES.txt records the SHA-256 of the engine and dashboard files

### How the originals are reused

`shared/nsrun.py` reads each original `app.py`, applies a few mechanical rewrites **in memory**, and runs it: it drops
`set_page_config`, the dashboard's own CSS and sidebar, gives each dashboard a prefixed view of `st.session_state`
and prefixed widget keys (`sinter__`, `mbf__`) so they never collide, and maps the old colours and fonts to the shared theme.
Every page function, table, chart and export is the original code. Both still run alone: `streamlit run sinter/app.py`, `streamlit run mbf/app.py`.

## Tests

    pip install -r requirements-dev.txt
    pytest -q tests                  # combined tests
    PYTHONPATH=. pytest -q mbf/tests # the MBF tests (O&M defaults to 0, so every V7 number is unchanged)

`tests/test_engines_unchanged.py` fails if an engine or dashboard file differs from `ENGINE_HASHES.txt`. The sinter files are untouched; the three MBF
lines were regenerated when the O&M setting was added.
