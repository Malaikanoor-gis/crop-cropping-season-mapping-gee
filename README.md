# Cropland and Cropping-Season Mapping, Chakwal and Rawalpindi (crop year 2024-25)

A multi-sensor (Sentinel-2 optical + Sentinel-1 radar + terrain) Random Forest workflow in Google Earth Engine and Python. It maps land cover and cropland across two districts of northern Punjab, Pakistan (about 12,400 km²), estimates areas with uncertainty, and checks accuracy against points labelled by hand on high-resolution imagery.

**Main question:** how much cropland is there, how is it used across seasons, and how far can the numbers be trusted?

![Land-cover map of Chakwal and Rawalpindi, crop year 2024-25](figures/map_landcover_2024_25.png)

*Random Forest land-cover map (Sentinel-2 + Sentinel-1 + terrain), crop year 2024-25.*

## Key results

| Result | Value |
|---|---|
| Sample size | 3,598 random points (about 1,800 per district) |
| Best feature set | Optical + SAR + terrain (accuracy 0.679, cropland F1 0.763, spatial-block CV) |
| Mapped cropland | 46.4% of the area, about 5,757 km² (95% CI ± 202 km², sampling error only) |
| **Error-adjusted cropland** | **52.0% ± 7.7%, about 6,448 ± 954 km²** (corrected using the independent validation) |
| Independent validation (114 points, cropland vs not) | Overall accuracy 77.9%, user's accuracy 82.1%, producer's accuracy 73.3% |

### Sensor comparison (agreement with Dynamic World labels)

| Features | Random CV accuracy | Spatial-block CV accuracy | Cropland F1 (spatial) |
|---|---|---|---|
| Optical only | 0.651 | 0.645 | 0.728 |
| SAR only | 0.501 | 0.501 | 0.659 |
| Optical + SAR | 0.669 | 0.667 | 0.754 |
| Optical + SAR + terrain | 0.678 | 0.679 | 0.763 |

![Accuracy by feature set, random vs spatial-block cross-validation](figures/fig_benchmark.png)

What this shows:
- Random and spatial-block cross-validation give almost identical scores, so spatial autocorrelation between neighbouring points does not appear to inflate the results much.
- Radar alone is weak, but adding it to optical data improves accuracy, and terrain adds a little more.
- These scores measure agreement with automatic Dynamic World labels, not true accuracy. The manual validation below is the independent check.

![Confusion matrix against Dynamic World labels (spatial-block CV)](figures/fig_confusion.png)

*Confusion matrix (spatial-block CV) shows agreement with the automatic labels, not true accuracy.*

### Land cover (both districts, from the random sample)

| Class | Share | Area (km²) | 95% CI (km²) |
|---|---|---|---|
| Cropland | 46.4% | 5,757 | ± 202 |
| Trees | 21.9% | 2,722 | ± 161 |
| Rangeland/shrub | 16.0% | 1,979 | ± 147 |
| Built-up | 9.0% | 1,111 | ± 115 |
| Bare | 6.1% | 751 | ± 97 |
| Water | 0.7% | 86 | ± 33 |

These intervals reflect sampling error only. The error-adjusted cropland estimate above also accounts for classification error. District-level tables are in `landcover_areas.csv`.

### Independent validation (manual labels)

120 points (60 mapped as cropland, 60 as not cropland) were labelled blind on Google Maps satellite imagery. 114 were labelled and 6 were marked unsure and skipped.

Confusion matrix (rows = map, columns = my label):

| | crop | not_crop |
|---|---|---|
| **crop** | 46 | 10 |
| **not_crop** | 15 | 43 |

Overall accuracy is area-weighted (77.9%). Cropland user's accuracy is 82.1% (of mapped cropland, the share that is cropland). Cropland producer's accuracy is 73.3% (of true cropland, the share that was mapped).

### Cropping seasons (exploratory, NDVI-threshold based)

Share of mapped cropland, with a sensitivity test on the NDVI threshold (0.45 is the main result):

| Season class | NDVI 0.40 | NDVI 0.45 | NDVI 0.50 |
|---|---|---|---|
| Fallow / not sown | 15.9% | 23.6% | 33.1% |
| Rabi only (winter) | 13.9% | 14.0% | 13.0% |
| Kharif only (summer) | 37.7% | 38.9% | 38.2% |
| Double cropped | 32.5% | 23.5% | 15.7% |

Rabi-only and kharif-only shares are stable across thresholds. The split between double-cropped and fallow depends strongly on the threshold, so treat the season results as indicative, not validated.

![Mean NDVI of mapped cropland by month, Nov 2024 to Oct 2025](figures/fig_ndvi_profile.png)

## Method

1. **Study area:** Chakwal and Rawalpindi districts (FAO GAUL boundaries, geoBoundaries as fallback).
2. **Features (Nov 2024 to Oct 2025):** monthly Sentinel-2 NDVI (Cloud Score+ masking, gaps filled with the annual median), annual Sentinel-2 bands and NDBI/MNDWI, Sentinel-1 VV/VH medians for rabi and kharif, SRTM elevation and slope.
3. **Labels:** Dynamic World yearly mode, merged to six classes (water, trees, rangeland/shrub, cropland, built-up, bare).
4. **Sampling:** simple random points in each district, features extracted at 20 m.
5. **Model:** Random Forest (300 trees, balanced class weights), compared across four feature sets with random 5-fold and spatial-block (about 5 km) 5-fold cross-validation. Area estimates use out-of-fold predictions.
6. **Area estimation:** proportions from the random sample scaled to district area, with 95% confidence intervals.
7. **Independent validation:** 120 blind points as described above. Accuracy is area-weighted and used to produce the error-adjusted cropland area.
8. **Cropping seasons:** rabi (Dec to Apr) and kharif (Jul to Oct) peak NDVI against a threshold of 0.45, with 0.40 and 0.50 as sensitivity tests.

## Limitations

- Training labels come from Dynamic World, an automatic product. Cross-validation scores measure agreement with those labels, not true accuracy. The manual validation is the independent check.
- The validation set is small (114 labelled points) and covers cropland vs not cropland only. It was labelled by one person, and Google Maps imagery dates may not match 2024-25.
- The map under-detects cropland (producer's accuracy 73.3%), which is why the error-adjusted estimate (52.0%) is higher than the mapped share (46.4%).
- Areas come from a random sample of points, not a pixel-by-pixel count.
- Crop types (wheat, maize, groundnut and so on) are not mapped, because no field labels were available.
- Cropping-season classes depend on NDVI thresholds that are assumptions and are not field-validated.
- One crop year only. Months without clear Sentinel-2 pixels were filled with the annual median NDVI.

## Possible next steps

- Collect field or high-resolution reference labels for the other land-cover classes and for cropping seasons.
- Extend to several crop years and test temporal transfer of the model.
- Test gap-filled or harmonic NDVI time series instead of monthly medians.

## Repository contents

| File | Description |
|---|---|
| `crop_cropping_season_mapping.ipynb` | Full Colab notebook |
| `random_points_features.csv` | Random sample points with all features and Dynamic World labels |
| `benchmark_results.csv` | Feature-set and cross-validation comparison |
| `landcover_areas.csv` | Land-cover areas with 95% confidence intervals |
| `cropping_season_areas.csv` | Cropping-season areas at three NDVI thresholds |
| `validation_accuracy.csv` | Accuracy from the manual validation |
| `cropland_area_adjusted.csv` | Error-adjusted cropland area |
| `map_weights.json` | Mapped cropland share used to weight the accuracy and area estimates |
| `validation_sheet_BLIND.csv` | Blind validation sheet (map class hidden) |
| `validation_key_DO_NOT_OPEN.csv` | Map class of each validation point, kept separate so labelling stayed blind |
| `figures/map_landcover_2024_25.png` | Land-cover map |
| `figures/fig_benchmark.png`, `figures/fig_confusion.png`, `figures/fig_ndvi_profile.png` | Figures shown in this README |

## How to run

1. Open the notebook in Google Colab.
2. In Cell 1, set `GEE_PROJECT` to your own Earth Engine project ID.
3. Run Cells 1 to 7 in order (Cell 4 is the slow one), then Cell 8 to create the blind validation sheet.
4. Download `validation_sheet_BLIND.csv`, label each point as `crop`, `not_crop` or `unsure` without opening the key file, and upload it in Cell 9.
5. Run the error-adjustment cell, then the last cell to zip the outputs.

**Data:** Sentinel-2 SR Harmonized, Cloud Score+, Sentinel-1 GRD, SRTM, Dynamic World, FAO GAUL (all via Google Earth Engine).
**Tools:** Google Earth Engine (Python API), scikit-learn, pandas, NumPy, matplotlib.

## Author

Malaika Noor, BS (Hons.) Geo-Informatics, PMAS Arid Agriculture University Rawalpindi.
