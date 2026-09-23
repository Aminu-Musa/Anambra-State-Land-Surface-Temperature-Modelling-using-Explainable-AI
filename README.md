# Anambra State Land Surface Temperature Modelling using Explainable AI

Spatially explicit, interpretable modelling of land surface temperature (LST) across three
cities of Anambra State, Nigeria — **Awka**, **Onitsha** and **Nnewi** — from Landsat 9
Collection 2 Level-2 imagery, with driver attribution by Shapley additive explanations (SHAP).

Two things distinguish this repository from a conventional LST modelling pipeline.

**The validation is spatially honest.** Land surface temperature is strongly autocorrelated
(Moran's I of 0.98–0.99 in all three cities), so neighbouring pixels are near-duplicates. A
random train–test split leaves those near-duplicates on both sides of the divide and measures
interpolation rather than prediction. Here the autocorrelation range is estimated from an
empirical variogram, and that range fixes a block size and an equal buffer distance which
govern partitioning, predictor screening, hyperparameter tuning and evaluation alike. The cost
is visible: buffering removes between 15% and 56% of otherwise-available training pixels per
fold. The benefit is an accuracy figure that means what it says.

**The dominant driver differs in every city.** Three cities, one climate, 40 km apart, the same
predictor set and the same modelling protocol — and three different answers. That result is the
argument for treating urban heat drivers as contingent on urban form rather than general.

---

## Headline results

| | Awka | Onitsha | Nnewi |
|---|---|---|---|
| Scene | LC09 188/056, 2024-12-19 | LC09 189/056, 2024-12-26 | LC09 189/056, 2024-12-26 |
| Pixels (30 m) | 248,532 | 248,043 | 248,584 |
| Moran's I (LST) | 0.9840 | 0.9931 | 0.9780 |
| Variogram range | 3,156 m | 2,251 m | 2,139 m |
| Block / buffer | 3,500 m | 2,500 m | 2,500 m |
| Best model | LightGBM | Random forest | LightGBM |
| Blocked R² | 0.550 | 0.889 | 0.323 |
| RMSE | 1.45 °C | 1.87 °C | 1.87 °C |
| Random-split R² | 0.781 | 0.924 | 0.669 |
| **Inflation** | **+0.231** | **+0.035** | **+0.346** |
| Leave-one-block-out | 0.557 ± 0.133 | 0.859 ± 0.102 | 0.453 ± 0.202 |
| **Dominant driver** | **NDVI** (1.26 °C) | **elevation** (1.77 °C) | **NDBI** (1.38 °C) |

The *Inflation* row is the point of the exercise: it is what a random split would have added to
the reported score. In Nnewi it would have more than doubled the apparent skill.

---

## Method

1. **Scene selection.** Landsat 9 only, no Landsat 8, so thermal characterisation stays
   consistent across cities. Dry-season window (Nov 2024 – Mar 2025); a scene qualifies at ≥98%
   clear over the region of interest, and the hottest qualifying scene is retained.
2. **LST retrieval.** Taken directly from the Collection 2 Level-2 surface temperature band
   (`ST_B10`), rescaled to physical units at ingestion. No user-side emissivity inversion —
   emissivity is already embedded in the USGS production chain via ASTER GED.
3. **Predictors.** Nine spectral indices, terrain from SRTM 90 m, four sub-pixel land-cover
   fractions from linear spectral mixture analysis, and contextual layers (population, building
   height, distance to permanent water, longitude), all on a common 30 m grid.
4. **Land cover.** LSMA extends Ridd's V–I–S model to four endmembers (V–I–S–W): omitting water
   under a sum-to-one constraint would load river pixels onto the impervious and soil fractions,
   both dark in the infrared, biasing the decomposition exactly where LST is lowest.
5. **Spatial diagnosis.** Global Moran's I (KNN, k = 8, row-standardised, 999 permutations),
   then an empirical variogram on 6,000 sampled pixels per city with a second-order trend
   surface removed, fitted with a spherical model by weighted least squares.
6. **Partitioning.** Blocks assembled into six contiguous groups by a serpentine sweep cut into
   segments of equal pixel count; the median-sized group is withheld entirely, the remaining
   five become buffered cross-validation folds.
7. **Screening.** Candidates compete *within* variable groups on a composite of
   0.4 × |ρ| + 0.6 × mutual information, admitted only while every VIF stays below 10. Run on
   training pixels only.
8. **Models.** Random forest, XGBoost and LightGBM, tuned by randomised search scored on the
   buffered folds.
9. **Interpretation.** TreeSHAP on the selected model, plus Moran's I on out-of-fold residuals
   as a diagnostic — not as a criterion the model must satisfy.

Everything server-side runs in Google Earth Engine; modelling runs in a hosted Python runtime.
No data is processed on local hardware.

---

## Repository structure

```
notebooks/          One Colab notebook per city (Awka, Onitsha, Nnewi)
tables/             Per-city CSV outputs: scene selection, variogram
                    diagnostics, full predictor screening trail, block
                    design, model comparison, leave-one-block-out, SHAP
figures/            Workflow diagram, study area, block partitions,
                    variograms, correlations, scatter, SHAP, LST maps
manuscript/         Manuscript and supporting material
```

---

## Data sources

All inputs are public and accessible through Google Earth Engine:

| Dataset | Use |
|---|---|
| Landsat 9 Collection 2 Level-2 (USGS) | LST and surface reflectance |
| CGIAR-CSI SRTM 90 m v4 | Elevation, slope, aspect |
| ESRI Global Land Cover (Karra et al., 2021) | Endmember sampling, categorical reference |
| GHS-POP R2023A | Population |
| GHS-BUILT-H R2023A | Building height |
| JRC Global Surface Water | Distance to permanent water |

---

## Reproducing

Open a city notebook in Colab, authenticate Earth Engine, and run the cells in order. Each
notebook writes its tables and figures to a shared results directory so the three cities can be
compared. Block size and buffer distance are derived from that city's own variogram rather than
set by hand, so changing the scene or the region of interest changes them automatically.

Required: `earthengine-api`, `xee`, `geemap`, `rioxarray`, `libpysal`, `esda`, `scikit-learn`,
`xgboost`, `lightgbm`, `shap`, `statsmodels`.

---

## Citation

> Eneche, P. S. U., & Musa, A. (in preparation). *Unveiling urban heat drivers in Anambra State:
> Explainable AI insights into land surface temperature.*

---

## Authors

**Patrick Samson Udama Eneche** — Faculty of Geo-Information Science and Earth Observation (ITC),
University of Twente, Enschede, The Netherlands

**Aminu Musa** — Department of Geography, Kogi State University, Anyigba, Nigeria
<aminumusa669@gmail.com>

---

## Licence

Code is released under the MIT Licence — see [LICENSE](LICENSE).

Derived tables, figures and modelling datasets are released under
[CC BY 4.0](https://creativecommons.org/licenses/by/4.0/) — see [LICENSE-DATA](LICENSE-DATA).

The input datasets (Landsat 9, SRTM, ESRI Global Land Cover, GHS-POP, GHS-BUILT-H and the JRC
Global Surface Water layer) remain under the terms of their respective providers.

If you use this code or the derived data in academic work, please cite the paper above.
