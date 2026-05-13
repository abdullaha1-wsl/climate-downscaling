# GEE + xclim QDM Bias Correction Pipeline

A scientifically rigorous Python pipeline for bias-correcting CMIP6 climate projections using **Quantile Delta Mapping (QDM)**, with observational reference data fetched via Google Earth Engine (GEE) and processing powered by `xclim`/`xsdba`.

Supports both **precipitation** (`pr`) and **near-surface temperature** (`tas`) for any geographic domain, across multiple models, scenarios, and future time intervals.

---

## Scientific Background

This pipeline implements:

- **Quantile Delta Mapping (QDM)** — Cannon et al. (2015), *J. Climate*. A transfer function is trained once on the historical baseline period and applied to all future scenarios. This is the scientifically correct approach: the correction is scenario-independent.
- **Wet-day frequency adjustment** — Themeßl et al. (2012), *Int. J. Climatol.*. Applied to precipitation only. Uses observed reference wet-day statistics (not corrected future data) as the target frequency.

---

## Features

- Fetches observational reference data **once** (CHIRPS for precipitation, ERA5-Land for temperature) and reuses it across all iterations
- Fetches historical CMIP6 **once per model** (not once per scenario × interval)
- Trains the QDM transfer function **once per model**, reused for every future period and scenario
- Streams large GEE collections in 5-year chunks to avoid memory errors
- Outputs per-model corrected NetCDF files plus ensemble statistics (mean, median, P10, P90)
- CF-compliant NetCDF output with proper CRS metadata for correct rendering in QGIS and other GIS tools

---

## Requirements

### Python packages

```bash
pip install earthengine-api xarray xclim xsdba xee netCDF4 dask
```

### Accounts & access

- A [Google Earth Engine](https://earthengine.google.com/) account with an active cloud project
- A [Google Colab](https://colab.research.google.com/) environment (or local equivalent with GEE credentials configured)
- Google Drive mounted at `/content/drive` for output storage

---

## Data Sources

| Variable | Reference Dataset | Collection ID | Native Resolution |
|----------|------------------|---------------|-------------------|
| `pr` (precipitation) | CHIRPS Daily | `UCSB-CHG/CHIRPS/DAILY` | ~5.5 km (0.05°) |
| `tas` (temperature) | ERA5-Land Daily | `ECMWF/ERA5_LAND/DAILY_AGGR` | ~11 km (0.1°) |

CMIP6 historical and scenario data are sourced from `NASA/GDDP-CMIP6` via GEE.

---

## Configuration

All user settings are in **Section 1** of the script. Key parameters:

```python
# Choose variable
VARIABLE = 'tas'   # 'pr' for precipitation, 'tas' for temperature

# Spatial domain [lon_min, lat_min, lon_max, lat_max]
EXTENT = [72.346, 33.011, 78.165, 38.259]

# CMIP6 models and scenarios
MODELS    = ['EC-Earth3', 'CNRM-CM6-1', 'GFDL-ESM4', 'MPI-ESM1-2-LR', 'GISS-E2-1-G']
SCENARIOS = ['ssp245', 'ssp585']

# Calibration period
BASELINE_START, BASELINE_END = '1990-01-01', '2014-12-31'

# Future projection windows
FUTURE_INTERVALS = [
    ('2026-01-01', '2050-12-31', '2026-2050'),
    ('2051-01-01', '2075-12-31', '2051-2075'),
    ('2076-01-01', '2100-12-31', '2076-2100'),
]

# QDM settings
NQUANTILES = 50          # number of quantiles
QDM_GROUP  = 'time.month'  # monthly grouping

# Output directory (Google Drive)
OUTPUT_DIR = '/content/drive/MyDrive/test'
```

---

## Pipeline Overview

```
Step 1  →  Fetch observational reference (CHIRPS / ERA5-Land) over baseline period
Step 2  →  Fetch & preprocess historical CMIP6 for each model over baseline period
Step 3  →  Train QDM transfer function for each model (hist vs. ref)
Step 4  →  For each scenario × future interval × model:
               • Fetch future CMIP6
               • Regrid to reference grid
               • Apply pre-trained QDM
               • (pr only) Wet-day frequency adjustment
               • Save corrected NetCDF
           →  Build ensemble statistics (mean / median / P10 / P90)
```

---

## Output Files

All outputs are written to `OUTPUT_DIR`. The naming convention is:

| File | Description |
|------|-------------|
| `qdm_{var}_{model}_{scenario}_{interval}.nc` | Bias-corrected output for one model |
| `qdm_ensemble_stats_{var}_{scenario}_{interval}.nc` | Ensemble mean, median, P10, P90 |
| `qdm_ensemble_mean_{var}_{scenario}_{interval}.nc` | Ensemble mean (convenience file) |
| `qdm_ensemble_median_{var}_{scenario}_{interval}.nc` | Ensemble median (convenience file) |

NetCDF files include a `crs` variable and `grid_mapping` attributes following CF conventions, so they open correctly as georeferenced rasters in QGIS, ArcGIS, and similar tools.

---

## Usage

1. Open the script in Google Colab.
2. Set `VARIABLE`, `EXTENT`, `MODELS`, `SCENARIOS`, and `OUTPUT_DIR` in Section 1.
3. Run all cells. GEE authentication will be triggered automatically if needed.
4. Outputs appear in your Google Drive folder as processing completes.

To run locally instead of Colab, remove the `drive.mount()` call and set `OUTPUT_DIR` to a local path. Ensure GEE credentials are configured via `earthengine authenticate`.

---

## Known Issues & Notes

- **QGIS rendering as a strip** — ensure your NetCDF includes the `crs` variable and `grid_mapping` attribute (see `add_spatial_metadata()` in the script). Without these, QGIS ignores the lat/lon coordinates.
- **Calendar conflicts** — CMIP6 models use non-standard calendars (360-day, no-leap). The pipeline converts all data to a standard calendar before any time-based grouping.
- **Memory on long runs** — GEE data is fetched in 5-year chunks. If you hit memory limits, reduce chunk size in `_fetch_chunked()`.
- **Model availability** — not all CMIP6 models are available for all scenarios in the NASA GDDP collection. Models that fail to fetch are skipped with a warning; the ensemble is built from whichever models succeeded.

---

## References

Cannon, A. J., Sobie, S. R., & Murdock, T. Q. (2015). Bias correction of GCM precipitation by quantile mapping: How well do methods preserve changes in quantiles and extremes? *Journal of Climate*, 28(17), 6938–6959. https://doi.org/10.1175/JCLI-D-14-00754.1

Themeßl, M. J., Gobiet, A., & Heinrich, G. (2012). Empirical-statistical downscaling and error correction of regional climate models and its impact on the climate change signal. *Climatic Change*, 112(2), 449–468. https://doi.org/10.1007/s10584-011-0224-4

---

## License

MIT
