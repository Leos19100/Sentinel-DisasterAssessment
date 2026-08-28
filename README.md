# Flood Impact Assessment — Niger–Benue Floods, Nigeria (October 2022)

Google Earth Engine–based remote-sensing analysis of the October 2022 Niger–Benue floods in Nigeria, built as the code appendix to an Imperial College London MSc Transport Research Project. The analysis starts as a tightly-scoped, statistically validated case study around Lokoja (Kogi State) and is progressively extended — to road-network and population impact, to a multi-year temporal-anomaly analysis, and finally to all 37 states and territories of Nigeria.

## Contents

- [Overview](#overview)
- [Repository layout](#repository-layout)
- [Study area and event window](#study-area-and-event-window)
- [The notebooks](#the-notebooks)
- [Data sources](#data-sources)
- [Key results](#key-results)
- [Getting started](#getting-started)
- [Limitations and validation](#limitations-and-validation)
- [Outputs reference](#outputs-reference)
- [Methodology references](#methodology-references)

## Overview

The project uses Sentinel-1 (SAR), Sentinel-2 (optical), CHIRPS (precipitation), WorldPop (population) and OpenStreetMap (road network) data on Google Earth Engine to answer four progressively broader questions about the same flood event:

1. **Where did it flood, and how confident can we be about it?** (`Index_Analysis.ipynb`)
2. **What did that flooding do to roads and people?** (`Impact_Assessment.ipynb`)
3. **Was this actually unusual, and did the landscape recover?** (`Temporal_Evolution.ipynb`)
4. **Does any of this hold up outside Lokoja, across an entire country?** (`National_Flood_Assessment.ipynb`)

Each notebook is self-contained — it re-derives the flood masks it needs directly from Sentinel-1/Sentinel-2 rather than depending on another notebook's saved output — while staying numerically consistent with the others, since they are meant to be read together as one appendix.

## Repository layout

```
.
├── README.md
├── requirements.txt
├── notebooks/
│   ├── Index_Analysis.ipynb             # 1. flood detection + threshold validation
│   ├── Impact_Assessment.ipynb          # 2. road network + population exposure
│   ├── Temporal_Evolution.ipynb         # 3. multi-year anomalies + recovery
│   └── National_Flood_Assessment.ipynb  # 4. generalization to all of Nigeria
└── outputs/
    ├── *.png   # figures exported by all four notebooks
    ├── *.csv   # tables exported by all four notebooks
    └── *.tif   # binary flood-mask rasters (Lokoja ROI)
```

All four notebooks write their figures and tables into the same shared `outputs/` folder via a small `save_figure`/`save_table` helper redefined at the top of each one, so `outputs/` accumulates the combined evidence of all four runs. Filenames are prefixed where needed to avoid collisions (e.g. everything `National_Flood_Assessment.ipynb` exports is prefixed `National_`).

## Study area and event window

The reference region of interest (ROI) is a rectangle around Lokoja, at the confluence of the Niger and Benue rivers — one of the areas worst affected by the October 2022 floods:

- **ROI:** `ee.Geometry.Rectangle([6.3, 7.4, 7.1, 8.2])` — roughly 785,000 ha (0.8° × 0.8° at this latitude)
- **Center point:** `[6.73, 7.80]`
- **Pre-flood window:** 2022-07-01 to 2022-08-31
- **Flood window:** 2022-09-25 to 2022-10-31

These constants, and the two flood-detection thresholds they're used to calibrate — **Sentinel-2 MNDWI (B3, B11) > 0** and **Sentinel-1 VH < −19 dB** (after a 50 m focal-median speckle filter) — are derived and statistically validated once, in `Index_Analysis.ipynb`, then reused verbatim (never re-tuned) by every other notebook, including the national-scale one.

## The notebooks

Meant to be read/run in this order; each builds on the previous one's validated *methodology*, not its saved files.

### 1. `Index_Analysis.ipynb` — Index-based flood detection and threshold validation

Establishes the ROI and event windows, computes NDWI/MNDWI (Sentinel-2) and VH backscatter change (Sentinel-1) between pre-flood and flood composites, and statistically validates the two thresholds above. Also runs an empirical study of Sentinel-2 cloud observability around the event — monthly climatology, post-event observation probability, Sentinel-1 complementarity — to justify why SAR is needed alongside optical data at all, then closes with a cross-index correlation check.

> **Key result:** 53,650.96 ha of net-new flooded area by MNDWI vs. 38,411.59 ha by VH — only ~5–7% of the ~785,000 ha ROI.

### 2. `Impact_Assessment.ipynb` — Road network and population exposure

Takes the validated masks and asks what they mean on the ground: how much of the road network is impassable or uncertain, which localities risk becoming cut off, and how many people sit inside the flood extent. Road data is pulled live via `osmnx`/Overpass (not a pre-staged asset); population comes from WorldPop, aggregated to LGA level using FAO GAUL boundaries. Adds one refinement the index notebook doesn't need: fixing a single Sentinel-1 `orbitProperties_pass` between the two composites, to avoid viewing-geometry artifacts.

Flow: flood-mask recap → road acquisition, buffering, zonal stats, operational classification, cartography → network connectivity / isolated-node analysis → population exposure and zonal statistics → cross-cutting synthesis (population near the isolated road network) → validation and limitations.

> **Key result:** 94,375 people (8.71% of the ROI's population) inside the fused flood mask; 1,038.2 km of road newly isolated, cutting off 1,012 network nodes and ~29,423 people within 500 m of the isolated network.

### 3. `Temporal_Evolution.ipynb` — Multi-year time series, anomalies, and recovery

Moves from a single before/after snapshot to a 2019–2025 monthly time series (Sentinel-2 NDVI/MNDWI/NBR, Sentinel-1 VV/VH, CHIRPS v3 precipitation), asking whether the October 2022 signal is actually unusual against normal seasonal variation, when it broke from baseline, and whether the landscape recovered. Builds a per-calendar-month climatology, standardized (z-score) anomalies, CUSUM break detection, an STL trend/seasonal/residual decomposition, a multi-sensor fusion confidence score, a threshold-sensitivity check, and a persistence-based recovery test — then repeats the same lens on CHIRPS rainfall to check whether the flood was locally rainfall-triggered.

> **Key finding — the most important methodological result in the repository:** the ROI-mean anomaly signal at flood onset is *muted*, not a clean spike (small/negative z-scores, no sharp CUSUM or STL-residual rupture exactly at flood onset). Section 17.1 works through why: the flood covers only ~5–7% of the ROI (spatial dilution against ordinary month-to-month noise), compounded by a 2019-onward trend decline — with no inflection at flood onset — that leaks through a climatology which only removes seasonality, not trend. This is a property of ROI-wide spatial averaging, not a processing error — see [Limitations and validation](#limitations-and-validation).

NDVI and NBR (vegetation-linked) recover within 1 month; the water-linked signals recover far slower — MNDWI after 7 months, VH after 9. The 30-day rainfall total before flood onset was 106.7 mm *below* the 22-year climatological mean (z = −2.81): the flood was not locally rainfall-triggered in the window examined.

### 4. `National_Flood_Assessment.ipynb` — Does the methodology generalize nationwide?

The largest notebook in the repository (85 cells: 41 markdown, 44 code). Tests whether the same methodology — unchanged thresholds, unchanged date windows — holds up when the ROI becomes all 36 states plus the Federal Capital Territory, and whether the same flood looks the same everywhere in Nigeria. Section 4 deliberately runs the naive generalization first (one `reduceRegions` call over all 37 states at once) and lets it fail/time out, as empirical justification for the per-state pipeline architecture used from Section 5 onward. Three parts then follow the same structure as the three notebooks above, generalized: Part I re-runs index analysis per state; Part II re-runs the temporal-evolution machinery on a handful of priority states selected by impact; Part III re-runs road/population impact nationally (population) and for the priority states only (roads — OSM/Overpass queries don't scale to all 37 states in one run).

> **Key results:** ~3.26M ha of fused flood extent and ~5.45M people affected nationally. Kogi State's own total (173,295.7 ha) is consistent with the original Lokoja ROI figure once the ~3.8× area difference between the two is accounted for. Priority states selected for deep-dive: **Borno, Yobe, Sokoto, Taraba, Niger**. But the Sentinel-1/Sentinel-2 agreement ratio swings from 0.15 (Abia) to 86.62 (Zamfara) across states — evidence that the Lokoja-calibrated thresholds don't transfer uniformly everywhere (Sections 7, 12, 13, 24.1).

## Data sources

| Source | Earth Engine ID | Role | Resolution | Used in |
|---|---|---|---|---|
| Sentinel-2 SR Harmonized | `COPERNICUS/S2_SR_HARMONIZED` | Optical water index (MNDWI/NDWI), NDVI, NBR | 10–20 m | all four |
| Cloud Score+ | `GOOGLE/CLOUD_SCORE_PLUS/V1/S2_HARMONIZED` | Per-pixel clear-sky probability (`cs_cdf` ≥ 0.6) | 10 m | Index, Temporal, National |
| Sentinel-1 GRD | `COPERNICUS/S1_GRD` | All-weather SAR water detection (VH), IW mode | 10 m | all four |
| CHIRPS v3 Pentad | `UCSB-CHC/CHIRPS/V3/PENTAD` | Precipitation time series / climatology | ~5.5 km | Temporal, National |
| WorldPop | `WorldPop/GP/100m/pop` | Population exposure | ~100 m | Impact, National |
| FAO GAUL 2015 (level 1 / level 2) | `FAO/GAUL/2015/level2` (+ level 1) | State / LGA administrative boundaries | vector | Impact, National |
| OpenStreetMap road network | via `osmnx` / Overpass API | Infrastructure exposed to the flood | vector | Impact, National |

## Key results

**Lokoja ROI, bitemporal snapshot** (`Index_Analysis.ipynb`, `Impact_Assessment.ipynb`)

| Metric | Sentinel-2 (MNDWI) | Sentinel-1 (VH) | Fused (S1 OR S2) |
|---|---:|---:|---:|
| Pre-flood water (Aug 2022) | 6,438.57 ha | 21,346.38 ha | — |
| Flood-peak water (Oct 2022) | 60,085.71 ha | 58,841.53 ha | — |
| Net-new flooded area | 53,650.96 ha | 38,411.59 ha | — |
| Population affected | 87,586 (8.08%) | 61,934 (5.71%) | 94,375 (8.71%) |

Road network: 1,389.6 km still accessible, 31.6 km probably cut, 22.6 km needing manual verification (full breakdown by road class in `outputs/Road_Impact_By_Category.csv`). Connectivity: 1,012 road-network nodes / 1,038.2 km newly isolated, affecting ~29,423 people within 500 m.

**Temporal evolution** (`Temporal_Evolution.ipynb`)

| Index | Recovery month | Months since flood onset |
|---|---|---:|
| NDVI (S2) | 2022-10 | 1 |
| NBR (S2) | 2022-10 | 1 |
| MNDWI (S2) | 2023-04 | 7 |
| VH (S1) | 2023-06 | 9 |

30-day pre-flood rainfall: 148.2 mm vs. 254.9 mm climatological mean (z = −2.81). CUSUM break alert for both MNDWI and VH: 98 days after flood onset — part of the evidence behind the muted-signal finding above.

**National generalization** (`National_Flood_Assessment.ipynb`)

| Metric | Value |
|---|---:|
| States/territories analyzed | 37 (36 states + FCT) |
| National fused flood extent | ~3.26M ha |
| National population affected | ~5.45M |
| Priority states for deep-dive | Borno, Yobe, Sokoto, Taraba, Niger |
| S1/S2 agreement ratio, range across states | 0.15 (Abia) – 86.62 (Zamfara) |
| Reference run: 37-state pipeline (Part I only) | 26.1 minutes |

## Getting started

### Requirements

- Python 3.13 (the notebooks were authored/run against 3.13.5), with Jupyter/JupyterLab
- A Google Cloud project with the Earth Engine API enabled and registered for Earth Engine access
- `pip install -r requirements.txt`

### Earth Engine authentication

Every notebook starts with:

```python
import ee
ee.Authenticate()
ee.Initialize(project='festive-oxide-485716-b0')
```

`ee.Authenticate()` opens a browser login the first time and caches credentials locally (`~/.config/earthengine/credentials`) after that. **Replace `festive-oxide-485716-b0` with your own Earth Engine–enabled Cloud project ID** — the one in the notebooks is private to the original author's Google account and will not work for anyone else.

### Running the notebooks

Run top to bottom inside Jupyter/JupyterLab, in this order:

1. `Index_Analysis.ipynb`
2. `Impact_Assessment.ipynb`
3. `Temporal_Evolution.ipynb`
4. `National_Flood_Assessment.ipynb`

Runtimes differ a lot. `Index_Analysis.ipynb` and `Impact_Assessment.ipynb` run in a few minutes; `Temporal_Evolution.ipynb`'s multi-year monthly time series takes roughly 10–20 minutes; `National_Flood_Assessment.ipynb` is much longer (its 37-state Part I pipeline alone took 26.1 minutes in the reference run, before Parts II and III's additional per-state loops on top). For that last one specifically, consider executing cell-by-cell (e.g. with `nbclient`'s `NotebookClient.execute_cell`, saving the notebook after every cell) rather than a single `jupyter nbconvert --execute`, so a failure partway through a long run doesn't discard the output of every cell that already finished.

All figures and tables are written to `../outputs/` relative to `notebooks/` — i.e. the top-level `outputs/` folder shown above — created automatically if it doesn't exist.

## Limitations and validation

Each notebook ends with its own "Validation and Limitations" section; the two most substantive findings are:

- **The ROI-mean temporal anomaly signal is muted, not absent** (`Temporal_Evolution.ipynb` §17.1, revisited nationally in `National_Flood_Assessment.ipynb` §24.1). Averaging over a large, mostly-unflooded ROI dilutes a spatially concentrated event down to roughly the same order of magnitude as ordinary baseline noise, compounded by a multi-year trend that a seasonal-only climatology doesn't remove. A pixel- or zone-level statistic (instead of one ROI-wide mean), and/or detrending before z-scoring, would be the architectural fix — not different CUSUM/anomaly thresholds on the same series.
- **Threshold transfer across Nigeria is an open, state-dependent question** (`National_Flood_Assessment.ipynb` §7, §12, §13, §24). The Lokoja-calibrated thresholds produce a plausible national ranking and a Kogi figure consistent with the original ROI result, but the Sentinel-1/Sentinel-2 agreement ratio varies by two orders of magnitude across states, and regional cloud-observability differences mean some states are inherently harder to observe optically than Lokoja was.

Further caveats carried across notebooks: monthly dB averaging on Sentinel-1 rather than per-scene analysis, cloud gaps linearly interpolated before STL decomposition, CHIRPS's coarse ~5.5 km footprint relative to the flood extent, and CUSUM/anomaly/recovery parameters that were not individually re-tuned per region.

## Outputs reference

`outputs/` (55 files as of the reference run: 34 CSV tables, 19 PNG figures, 2 GeoTIFF flood-mask rasters) accumulates every figure and table exported by all four notebooks, prefixed by source where relevant (`National_*` = `National_Flood_Assessment.ipynb`). Highlights:

- `Sentinel1_FloodMask_Lokoja.tif` / `Sentinel2_FloodMask_Lokoja.tif` — binary flood-mask rasters for the Lokoja ROI, ready to load in QGIS/ArcGIS
- `Flood_Comparison_Summary_Lokoja.png` — the headline before/after cartography
- `National_Flood_Extent_By_State.csv`, `National_Flood_And_Population_By_LGA.csv` — the full state- and LGA-level national tables behind the [Key results](#key-results) numbers above
- Every notebook's own closing "Summary of Exported Outputs" cell lists exactly what that run wrote, with file sizes

---

*No license has been added yet — until one is, standard copyright applies and the code is not licensed for reuse. Add a `LICENSE` file if you want to grant reuse rights explicitly.*
