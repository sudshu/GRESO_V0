# CLAUDE.md — GRESO_V0 Codebase Guide

## Project Overview

**GRESO** (Global Rate of change of CO2 from Space-based Observations) is a scientific Python package that analyzes OCO-2 satellite CO2 measurements to compute annual global CO2 growth rates and validate them against NOAA Marine Boundary Layer (MBL) ground-based reference data.

**Author**: Sudhanshu Pandey (NASA JPL / Caltech)
**Publication**: Pandey et al. (2024), *AGU Advances*, 5, e2023AV001145
**Python requirement**: 3.7+

---

## Repository Structure

```
GRESO_V0/
├── greso_analysis.py       # Main analysis script — entry point
├── utils.py                # Scientific utility functions and core classes
├── README.md               # User-facing documentation
├── annual_growth_rate.png  # Output: generated comparison plot
└── CLAUDE.md               # This file
```

There is no build system, test suite, package configuration (`setup.py`, `pyproject.toml`), or CI/CD. The project is run directly as a Python script.

---

## Running the Analysis

```bash
# Default analysis (MIPV11 format, "all" data type)
python greso_analysis.py

# Analyze a specific OCO-2 data type
python greso_analysis.py --data-type LNLG
```

**Required dependencies** (install via pip if absent):
```bash
pip install numpy pandas xarray matplotlib scipy python-dateutil
```

**External data required at runtime:**
- OCO-2 MIPV11 NetCDF4 file (e.g., `OCO2_b11.2_10sec_GOOD_r3.nc4`) — path configured in `AnalysisConfig.data_file`
- NOAA growth rate reference — fetched automatically from FTP during analysis

---

## Architecture

### `greso_analysis.py` — Main Module

#### `AnalysisConfig` (dataclass)
Central configuration object. Modify this to change analysis parameters. Key fields:

| Field | Default | Description |
|---|---|---|
| `data_file` | hardcoded local path | Path to OCO-2 input file (must be updated) |
| `file_format` | `"mipv11"` | `"mipv11"` (NetCDF4) or `"standard"` (HDF5) |
| `lat_range` | `[-50, 50]` | Latitude bounds for filtering |
| `uncertainty_threshold` | `2.0` | Max xco2 uncertainty (ppm) |
| `bin_days` | `16` | Temporal bin width in days |
| `growth_rate_start_year` | `2015` | First year for growth rate output |
| `growth_rate_end_year` | `2025` | Last year for growth rate output |
| `output_filename` | `"annual_growth_rate.png"` | Output plot filename |

> **Important**: `data_file` defaults to an absolute path on the original developer's machine. Always set this to the actual file location before running.

#### `CO2GrowthRateAnalyzer`
Orchestrates the analysis pipeline:
- `process_oco_data(data_types)` — loads data, applies binning, returns area-weighted time series
- `calculate_growth_rates(time, co2)` — deseasonalizes, smooths, returns annual growth rates
- `load_noaa_reference_data()` — fetches NOAA MBL CSV via FTP

#### `VisualizationManager`
Creates comparison plots. Supports three data-type series with fixed styling:
- `all` → blue circles
- `LNLG` → red squares
- `OG` → green triangles

Also prints tabular growth rate summaries to stdout alongside the NOAA reference.

#### `create_standard_config(data_file_path)`
Helper to create an `AnalysisConfig` preset for legacy HDF5 standard format.

#### `main()`
Top-level workflow entry point: config → process OCO-2 → load NOAA → plot → save PNG.

---

### `utils.py` — Utility Library

#### Datetime / Time Conversion
- `datetime2year(dt_array)` — converts `datetime` objects to decimal years (e.g., `2015.5`)
- `year2datetime(year_float)` — inverse of above
- `giverMonthEdges(xxi)` / `give_month_edges(decimal_years)` — generate month boundary arrays in decimal years (functionally equivalent, both retained)

#### Spatial Analysis
- `give_grid_cell_area(lons_edges, lats_edges)` — returns km² area per cell using spherical geometry
- `area_weighted_mean(glon_centers, glat_centers, data_grid)` — area-weighted 2D mean
- `areaWeighted(glon, glat, aa)` — cosine-latitude weighted mean; handles both 1D and 2D inputs

#### Statistical / Scientific Processing
- `harmonics(params, x, numpoly, numharm)` — evaluates harmonic components
- `fitFunc(params, x, numpoly, numharm)` — polynomial + harmonics fit function
- `errfunc(p, x, y, numpoly, numharm)` — residual for least-squares fitting
- `deseasonalize(obs_time, obs, numpoly=4, numharm=4)` — removes seasonal cycle via harmonic fitting; returns deseasonalized (time, co2) pair
- `boxcar_smooth(obs, jx=1)` — moving average smoother with half-width `jx`
- `monthlyGrowthRate(obs_time, mgrarea)` — interpolates deseasonalized series and computes monthly differences
- `giveAnnual(y)` — sums 12-month windows to produce annual values
- `giveGrowthRates(oxc, oyc, st_tim, ed_tim)` — full pipeline: deseasonalize → smooth → monthly growth → annual growth

#### Data Loading
- `get_10Sec_data_baker(file_path_10s, data_type, lat_range)` — loads MIPV11 NetCDF4 OCO-2 file; applies uncertainty + data type + latitude filters; reads `xco2_2019_scale` variable; decodes `sounding_id` to `datetime` objects
- `getNOAAgrowthRates()` — **deprecated**: fetches NOAA data and plots directly (uses `from numpy import *` side effects); replaced by `CO2GrowthRateAnalyzer.load_noaa_reference_data()`

#### `CO2GR` Class
Core spatial-temporal binning engine:
- `binOCOdata(deldays, st_time, aggmethod)` — bins OCO-2 observations into a 3D grid (time × lat × lon) with configurable resolution (`binres`: 5°, 10°, or 20°)
- `getRegionOCOSeris()` — extracts area-weighted time series for a named region

Supported regions: `globe`, `NET`, `TRO`, `SET`, `NH`, `SH`

#### `OCODataProcessor` Class
Clean OOP wrapper for data loading:
- `_apply_quality_filters(dataset, data_type)` — applies uncertainty and data-type masks
- `_load_mipv11_data(data_type)` — delegates to `get_10Sec_data_baker()`
- `_load_standard_data(data_type)` — loads legacy HDF5 format using `xarray`
- `_extract_datetime_components(sounding_ids)` — fast integer arithmetic to decode sounding IDs
- `_convert_to_decimal_years(datetime_components)` — vectorized decimal year calculation
- `load_oco_data(data_type)` — main entry; dispatches to correct format loader

---

## Data Processing Pipeline

```
OCO-2 File (NetCDF4 or HDF5)
        ↓
Quality Filtering
  - xco2_uncertainty < 2.0 ppm
  - data_type: OG=6, LNLG<3, all≠3
  - latitude within [-50°, 50°]
        ↓
Temporal Binning (16-day windows)
+ Spatial Gridding (5° lat/lon grid)
        ↓
Area-Weighted Global Average (cos(lat) weights)
  → time series: obs_time, obs_co2
        ↓
Deseasonalization (4-harmonic fit removal)
        ↓
Boxcar Smoothing (window=3)
        ↓
Monthly Growth Rates (interpolated differences)
        ↓
Annual Aggregation (sum 12 monthly values)
        ↓
Compare with NOAA MBL Reference
+ Generate PNG Plot + Console Table
```

---

## Data Formats

### MIPV11 (Default, Recommended)
- Format: NetCDF4 (`.nc4`)
- Key variable: `xco2_2019_scale` (2019-calibrated CO2)
- Download: https://gml.noaa.gov/ccgg/OCO2_v11mip/download.php
- Direct file: https://gml.noaa.gov/aftp/user/andy/OCO-2/OCO2_b11.2_10sec_GOOD_r2.nc4

### Standard (Legacy)
- Format: HDF5 (`.h5`)
- Key variable: `xco2`
- Use `create_standard_config()` to set up

### Data Type Classifications (OCO-2 `data_type` field)
| Value | Type | Meaning |
|---|---|---|
| 6 | OG | Ocean glint |
| <3 | LNLG | Land nadir + land glint |
| ≠3 | all | All valid types |

### Sounding ID Format
14-digit integer: `YYYYMMDDHHMMSSf` — encodes observation timestamp. Parsed via integer arithmetic in `OCODataProcessor._extract_datetime_components()`.

---

## Key Conventions

### Naming
- Python standard: `snake_case` for functions/variables, `PascalCase` for classes
- Scientific abbreviations are common: `xco2` (CO2 concentration), `dxco2` (uncertainty), `glon`/`glat` (grid lon/lat), `vd` (valid data boolean mask), `mgrarea` (monthly growth rate, area-weighted)
- Some legacy function names remain inconsistent with modern conventions (e.g., `CO2GR`, `giverMonthEdges` vs `give_month_edges`)

### Type Hints
- Used throughout `greso_analysis.py` and `OCODataProcessor`
- Partially absent in older utility functions in `utils.py`

### Logging
- `greso_analysis.py` uses the standard `logging` module at `INFO` level
- Log format: `%(asctime)s - %(levelname)s - %(message)s`
- Legacy code in `utils.py` uses `print()` directly

### `from numpy import *`
`utils.py` contains a wildcard import (`from numpy import *`) at the top level. This is a legacy pattern. The `getNOAAgrowthRates()` function relies on this for bare `mean()`, `errorbar()` calls. Do not remove this import without auditing those legacy functions.

### Configuration Coupling
`AnalysisConfig` is passed directly into `OCODataProcessor` and `CO2GrowthRateAnalyzer`. All parameters flow from this single config object.

---

## Known Issues / Technical Debt

1. **Hardcoded data path**: `AnalysisConfig.data_file` defaults to a developer-local path (`/Users/pandeysu/Desktop/...`). Users must override this before running.

2. **Duplicate functions**: `give_month_edges()` and `giverMonthEdges()` are functionally equivalent. `getNOAAgrowthRates()` is deprecated but retained.

3. **Wildcard import**: `from numpy import *` in `utils.py` pollutes the namespace. It is needed by legacy code.

4. **No automated tests**: The project has no test suite. Validation is performed by comparing output against NOAA reference values manually.

5. **Network dependency**: NOAA data is fetched via FTP at runtime. Analysis will fail without network access (no local cache fallback).

6. **`miss_month` hardcoded**: `CO2GR.getRegionOCOSeris()` hardcodes `self.miss_month = 2017.6`. This appears to mark a known data gap.

7. **`VisualizationManager` instantiates analyzer internally**: `create_growth_rate_plot()` creates a new `CO2GrowthRateAnalyzer` instance per data type for growth rate calculation, re-using the config. This is fine functionally but creates tight coupling.

---

## Extending the Analysis

### Add a new data type
1. Add the type string to the `data_types` list passed to `process_oco_data()`
2. Add a filter branch in `OCODataProcessor._apply_quality_filters()` and `get_10Sec_data_baker()`
3. Register a marker/color in `VisualizationManager.__init__()`

### Change spatial resolution
Set `cgr.binres` to `5`, `10`, or `20` before calling `binOCOdata()`. Grid lat/lon arrays in `binOCOdata()` adjust automatically.

### Change analysis region
Set `cgr.region` to one of: `globe`, `NET`, `TRO`, `SET`, `NH`, `SH` before calling `getRegionOCOSeris()`.

### Use a custom date range
Update `AnalysisConfig.growth_rate_start_year` / `growth_rate_end_year` and `start_time`.

---

## Output

- **PNG plot** (`annual_growth_rate.png`): Annual CO2 growth rates (OCO-2 vs NOAA MBL), saved at 300 DPI in the script directory
- **Console table**: Year-by-year growth rates with NOAA comparison and mean values
- **Log output**: Processing steps with timestamps at INFO level

---

## Git History

| Commit | Message |
|---|---|
| `611683f` | Update README.md |
| `af7640a` | docs: add CO2 growth rate comparison plot to README header |
| `2da2394` | feat: initialize GRESO CO2 growth rate analysis package with core functionality |

Active development branch: `claude/add-claude-documentation-n2Bo5`
