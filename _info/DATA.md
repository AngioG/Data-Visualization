# GRINS AQCLIM — data reference

## Provenance and files

GRINS AQCLIM **v.2.0.3**, [Zenodo 17605148](https://doi.org/10.5281/zenodo.17605148),
CC BY 4.0; [paper](https://doi.org/10.1038/s41597-026-07712-3).
Distribution: **16 original files, 4,425,847,873 bytes**.

| Distributed file(s) | Content |
| --- | --- |
| `GRINS_AQCLIM_points_Italy_y*.csv` | Six daily partitions: 20132014, 20152016, 20172018, 20192020, 20212022, 2023 |
| `GRINS_AQCLIM_points_Italy.rda` | R representation; never append to CSV observations |
| `Station_registry_information.CSV`, `.rda` | Static station registry |
| `GRINS_AQCLIM_imputation_uncertainty.csv`, `.rda` | Sparse daily imputation SDs |
| `GRINS_AQ_NO2_LAUs_Italy.rds` | Municipality NO₂ predictions |
| `GRINS_CL_LAUs_Italy.rds` | Municipality climate |
| `metadata_GRINS_LAUs.csv`, `.rds` | Municipality attributes; RDS includes geometry |
| `example_LAUs_merge_and_visualization.R` | Municipality join/map example using `PRO_COM`, `mean_t2m` |

## Station data

Nominal grid: **744 stations × 4,017 dates (2013-01-01–2023-12-31) = 2,988,648 rows**;
not that many valid measurements. All six CSV headers match: **59 columns = 2 keys
+ 48 AQ + 9 CL**. Wind scan confirms the row count; full key/date validation is pending.

**Keys:** `AirQualityStation` = text station ID (e.g. `STA.IT0063A`);
`time` = calendar date, not hourly timestamp. Not every site measures every
pollutant or operated throughout. No original hourly timestamps, sensor IDs,
EEA validity flags or complete hourly imputation mask are supplied.

### Pollutants

`AQ_<stat>_<pollutant>`; all 6 × 8 combinations exist.
`stat`: `min` minimum, `q1` P25, `mean` arithmetic mean, `med` P50, `q3` P75,
`max` maximum, all **within-day** summaries in concentration units, not independent
measurements. Percentiles across daily means are different quantities.

| Pollutant suffix | Meaning | Unit |
| --- | --- | --- |
| `CO` | Carbon monoxide | mg/m³¹ |
| `NH3` | Ammonia | µg/m³ |
| `NO` | Nitrogen monoxide | µg/m³ |
| `NO2` | Nitrogen dioxide | µg/m³ |
| `O3` | Ozone | µg/m³ |
| `PM10` | Particulate matter, nominal aerodynamic diameter ≤10 µm | µg/m³ |
| `PM2.5` | Fine particulate matter, nominal aerodynamic diameter ≤2.5 µm | µg/m³ |
| `SO2` | Sulphur dioxide | µg/m³ |

¹ **Supported CO interpretation:** [EEA convention][EEA CO] + unchanged concentration scale
in the [authors' import code][AQ code]; contradicts the blanket µg/m³ heading. Upstream unit
fields were dropped: not a per-record unit audit. Convert explicitly when needed.

**Imputation:** authors describe hourly local-level/Kalman-smoother gap filling;
>5 consecutive missing hours leaves the day missing. Already-daily PM inputs
need no hourly aggregation; identical summaries do not prove constant hourly levels.
O₃ daily maximum ≠ MDA8; daily summaries cannot recover arbitrary hourly metrics
(including CO eight-hour exceedances).

### Climate and vegetation

Operations below are traced in the authors' [climate code][CL code] and
[station merge][AQ code], not recomputed from original hourly data. ERA5-Land preferred, ERA5 single-level fallback; **BLH from ERA5**.
Grid data are not on-site weather measurements; nearby sites can share inputs;
no per-value source flag.

| Column | Meaning | Daily operation | Unit |
| --- | --- | --- | --- |
| `CL_blh` | Boundary layer height | Mean | m |
| `CL_lai_hv` | High-vegetation leaf area index | Last of each 24-entry block | m²/m² |
| `CL_lai_lv` | Low-vegetation leaf area index | Last of each 24-entry block | m²/m² |
| `CL_rh` | Relative humidity from temperature/dewpoint | Mean | % |
| `CL_ssr` | Surface net solar radiation | Maximum hourly accumulation | J/m² |
| `CL_t2m` | Temperature at 2 m | Kelvin→Celsius, mean | °C |
| `CL_tp` | Total precipitation | Last block value; separate ERA5 ×24 correction | m (×1,000→mm) |
| `CL_winddir` | Wind class | Mode of hourly sectors | Unordered 1–4 |
| `CL_windspeed` | Speed from 10 m u/v components | Mean hourly speed | m/s |

**Wind conflict:** all 2,988,648 rows contain integer 1–4, no missing values.
Mappings for codes **1,2,3,4**: Zenodo **N,E,S,W**; code comments **S,W,N,E**;
function applied to cardinal u/v examples under [ECMWF wind-from convention][ECMWF wind]
**N,W,S,E**. Last mapping is code-derived, not validated for the delivered release.
**Use unordered categories; no compass labels/source-direction claims** until
checked against original hourly u/v or a release-specific build. Never average codes.

**Accumulation timing:** [ERA5/ERA5-Land accumulation conventions differ][ECMWF accumulation];
00 UTC may represent the previous day's accumulation. Original NetCDF time/step
axes and a release build record are not included in the daily tables. Use supplied covariates without
claiming verified local-day totals; validate alignment before lag-specific
analysis. Net radiation is neither W/m² irradiance nor incoming solar radiation;
do not silently relabel, reconstruct hourly fluxes or shift dates.

### Registry and geography

Registry: **744 rows, 744 unique IDs, 15 columns**. Coordinates ≠ geometry CRS.

| Fields | Definition / handling |
| --- | --- |
| `AirQualityStation` | Station ID; registry join key |
| `Longitude`, `Latitude` | WGS84 decimal degrees; longitude first |
| `Altitude` | Elevation, m |
| `AirQualityStationType` | `background` 444; `traffic` 202; `industrial` 98 |
| `AirQualityStationArea` | `urban` 408; `suburban` 212; `rural` 101; `rural-nearcity` 12; `rural-regional` 8; `rural-remote` 3 |
| `COD_RIP`, `COD_REG`, `COD_PROV` | Geographical division, region, province codes |
| `COD_CM` | Metropolitan city code; 0 outside; **not mountain community** |
| `COD_UTS` | Supra-municipal unit: province, metropolitan city or equivalent |
| `PRO_COM`, `PRO_COM_T` | Numeric municipality code; six-character text equivalent with leading zeros, **not temporary** |
| `COMUNE` | Municipality name; not a unique join key |
| `Area comune` | Municipality area, **m²** (÷1,000,000→km²), not the description's km² |

Static metadata do not reconstruct historical site, classification, instrument
or boundary changes. Preserve codes as identifiers; use official code–name lookups.
Document any aggregation of rural subtypes.

**Area verified:** all 744 `Area comune` values exactly match municipality
`Shape_Area` (check tolerance 0.001 m²); author merge renames that attribute.
Geometry RDS embeds **WGS84 / UTM 32N, EPSG:32632**, metre axes.
[ISTAT][ISTAT] confirms m² area, metre perimeter. These are supplied geometry attributes, not newly computed
geodesic areas or population weights. CRS checked via embedded WKT, not full RDS decoding.

### Uncertainty and joins

Uncertainty CSV: **50 columns = 2 keys + 48 `sd_<stat>_<pollutant>`**;
`sd_mean_NO2` matches `AQ_mean_NO2` (no `sd_AQ_` prefix). SD units match the target,
not squared units. Sparse rows represent days with at least one imputation;
within included rows, mean SD=0 indicates no missing hourly inputs for that
target, positive SD indicates imputation uncertainty. Missing SD, absent rows
and explicit zero are distinct; SD is not total measurement error, trend
uncertainty or a complete observed/imputed flag.

1. Concatenate daily CSVs; check (`AirQualityStation`, `time`) uniqueness.
2. Left-join registry by station; **many-to-one**, no unmatched station IDs.
3. Left-join SD by station/date; **one-to-one**, preserve matched-row indicator.
4. Assert unchanged row count. No inner SD join or blanket missing-SD→0 replacement.

### Municipality products

Separate unit: (`PRO_COM`, `time`); harmonise numeric/text keys before joining.
Documented 2013–2023 series use **2025 boundaries**, not historical boundaries.

| Product / fields | Meaning |
| --- | --- |
| NO₂: `mean_NO2_FRK`, `sd_NO2_FRK` | Fixed Rank Kriging prediction/uncertainty; 0.05° grid aggregated to municipalities, not local monitoring observations |
| Climate: `mean_<variable>`, `sd_<variable>` | Spatial means/within-municipality variation; 0.1° grids; spatial SD ≠ imputation SD |
| Metadata CSV | 7,896 rows, 12 fields: `COD_RIP`, `COD_REG`, `COD_PROV`, `COD_CM`, `COD_UTS`, `PRO_COM`, `PRO_COM_T`, `COMUNE`, plus fields below |
| `COMUNE_A` | Name in another language for bilingual municipalities |
| `CC_UTS` | Supra-municipal capital flag: 1=yes, 0=no; local values checked |
| `Shape_Leng`, `Shape_Area` | Perimeter m; area m² |

Full RDS schemas, time-series counts, dates and municipality code coverage remain
unchecked. Geometry attributes/CRS checks do not establish time-series completeness.

## Validation scope

**Completed, scoped evidence:** manifest/version/16 files; recorded size/MD5
matches; six identical 59-column daily headers; 50-column SD schema; registry
uniqueness/category counts; municipality CSV schema/count; area/CRS/admin definitions;
climate code investigation; full wind-code scan. These are not full value validation.

**Inference limits:** station averages ≠ national area/population averages or
personal exposure. No clinical outcomes: health relevance requires literature;
no direct diagnoses/deaths attribution. Account for serial/spatial dependence.

## Definition sources

The cited code revisions identify inspected implementations, not a certified
build manifest for dataset v.2.0.3. In the AQ repository, see `functions.R`
(import) and `EEA_merging_v103.R` (area rename/source merge); in the climate
repository, see `C3S_functions.R`, `C3S_H2D.R` and `C3S_fixing_tp_ERA5SL.R`.
ISTAT field definitions establish `COD_CM`, `COMUNE_A`, `CC_UTS` and geometry units.

[AQ code]: https://github.com/GRINS-Spoke0-WP2/AQ-EEA/tree/caddef9d77d916332093f3ddd4a1eb9d82512382
[CL code]: https://github.com/GRINS-Spoke0-WP2/WE-C3S/tree/74c8f55e14f80cb524b4aeea996f86cfab514f61
[ISTAT]: https://www.istat.it/wp-content/uploads/2026/02/Nota-tecnica-Litoraneita-e-Sezioni-litoranee.pdf
[EEA CO]: https://www.eea.europa.eu/en/analysis/maps-and-charts/carbon-monoxide-8-hour-mean-limit-value-for-the-protection-of-human-health-5
[ECMWF wind]: https://confluence.ecmwf.int/pages/viewpage.action?pageId=133262398
[ECMWF accumulation]: https://confluence.ecmwf.int/pages/viewpage.action?pageId=501029136
