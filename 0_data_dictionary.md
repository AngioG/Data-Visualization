# Data dictionary: air-quality indices and temporal aggregates

This document describes the variables produced by `00_prepare_v2.ipynb`. It is based on the notebook's implementation and default configuration, not on an independent validation of every source value.

## 1. What the notebook writes

| Output | Default path | Contents |
|---|---|---|
| Daily indices | `1_processed/daily_indices.csv.gz` | One row per station-day, containing derived indices and exceedance flags. It does **not** contain the source pollutant concentrations, intraday spread variables, AQI driver dummy columns, or climate variables. |
| Weekly aggregates | `1_processed/aggregated_week.csv` | Aggregated pollutant concentrations, derived indices, threshold summaries, and climate variables for each station and complete week. |
| Monthly aggregates | `1_processed/aggregated_month.csv` | Same kinds of variables, grouped by station and calendar month. |
| Seasonal aggregates | `1_processed/aggregated_season.csv` | Same kinds of variables, grouped by station and meteorological season. |
| Yearly aggregates | `1_processed/aggregated_year.csv` | Same kinds of variables, grouped by station and calendar year. |
| Reliability tables | `1_reliability/reliability_{week,month,season,year}.csv` | Valid-day counts and coverage diagnostics corresponding row-for-row to each matching aggregate table. |

The notebook also writes a source-stage `1_processed/raw.csv` and station metadata, but these are not the analytical output tables described here. Station metadata and station attributes are intentionally not documented in this dictionary. The aggregation code uses the station identifier as a grouping key by default, so each aggregate row still represents one station-period.

## 2. Row grain, dates, and temporal meaning

- **Daily working data:** one row per station and date (`time`). Source pollutant columns contain daily summaries of the underlying observations, where available.
- **Weekly:** `week` is the Monday starting the Monday-to-Sunday week.
- **Monthly:** `year` plus `month` identify a calendar month (`month` is 1 through 12).
- **Seasonal:** `year` plus `season` identify Spring, Summer, Autumn, or Winter. Winter is labelled with the year in which its **December** occurs, so January and February belong to the winter that began in the previous December.
- **Yearly:** `year` identifies the calendar year.

By default, periods that are not fully contained in the dates available in the input are dropped (`drop_partial_periods=True`). For example, a partial first or last week and a winter season that extends outside the data's date range are excluded. Coverage is assessed against the full calendar length of each period, not only the number of dates present in the file.

## 3. Source pollutants and units

The notebook standardises pollutant labels and uses these seven pollutants:

| Code in output | Pollutant | Unit used by the notebook | Used by derived indices/flags? |
|---|---|---|---|
| `CO` | Carbon monoxide | mg/m³ | WHO daily-mean exceedance flag; AQHI does not use CO. |
| `NO` | Nitrogen monoxide | µg/m³ | `NOx` and `NO2_NOx`. |
| `NO2` | Nitrogen dioxide | µg/m³ | `NOx`, `NO2_NOx`, `Ox_ppb`, AQI, AQHI, and NO₂ threshold flags. |
| `O3` | Ozone | µg/m³ in source data; converted to ppb where needed | `Ox_ppb`, AQI, AQHI, and ozone threshold flags. |
| `PM10` | Particulate matter with aerodynamic diameter up to 10 µm | µg/m³ | `RPM`, AQI, and PM₁₀ threshold flags. |
| `PM25` | Particulate matter with aerodynamic diameter up to 2.5 µm | µg/m³ | `RPM`, AQI, AQHI, and PM₂.₅ threshold flag. The code renames source `PM2.5` to `PM25`. |
| `SO2` | Sulphur dioxide | µg/m³ | AQI and sulphur-dioxide threshold flags. |

**NH₃ is deliberately excluded** by the notebook, regardless of whether it is present in the input files due to limited availability (one station / one usable year). Pollutant units are configuration-level assumptions in the notebook; the code explicitly treats CO as mg/m³ and the other included pollutants as µg/m³.

## 4. Daily source-statistic columns in the working dataset

Before calculating indices, `standardize_columns()` renames source pollutant columns so that pollutant tags are alphanumeric. For example, `AQ_mean_PM2.5` becomes `AQ_mean_PM25`.

The general column pattern is `AQ_<statistic>_<pollutant>`. The six statistics are:

| Statistic token | Meaning within a day | Example |
|---|---|---|
| `min` | Minimum of the available underlying daily observations | `AQ_min_NO2` |
| `q1` | First quartile, 25th percentile | `AQ_q1_NO2` |
| `mean` | Arithmetic mean | `AQ_mean_NO2` |
| `med` | Median, 50th percentile | `AQ_med_NO2` |
| `q3` | Third quartile, 75th percentile | `AQ_q3_NO2` |
| `max` | Maximum of the available underlying daily observations | `AQ_max_NO2` |

These are **within-day** summaries, not statistics calculated across multiple days. For example, `AQ_q1_NO2` is the first quartile of NO₂ observations for that station-day. For CO, values are interpreted in mg/m³; for the other listed pollutants, they are interpreted in µg/m³.

The fields above exist in the notebook's working daily table where the corresponding source field is available. They are not written to the final daily indices-only file. The weekly/monthly/seasonal/yearly tables summarise these data using the output variables described below.

### Handling particulate-matter data that already have daily resolution

Some particulate-matter (`PM10`, `PM25`) observations may already be daily values rather than hourly observations. The notebook infers a daily-resolution PM row when `AQ_min_PM10 == AQ_max_PM10` or `AQ_min_PM25 == AQ_max_PM25`, respectively.

- `PM10_daily_only`: `1` when the PM10 row meets that equality test, `0` when it does not, and missing when min/max are unavailable.
- `PM25_daily_only`: equivalent flag for PM2.5.
- `<P>_iqr`: intraday interquartile range, defined as `AQ_q3_<P> - AQ_q1_<P>` when nonnegative.
- For rows inferred to be daily-resolution PM, the notebook masks the corresponding intraday PM statistics (`min`, `q1`, `med`, `q3`, `max`) and the IQR to missing after calculating the indices. This prevents these values from being treated as observed intraday variation or peaks in later aggregation.

The equality test is an operational heuristic: the code treats min equal to max as evidence of daily resolution. It does not independently verify the source sampling frequency. As a result, any truly hourly day with a constant observed concentration would also satisfy the test.

The daily IQR variables can be `CO_iqr`, `NO_iqr`, `NO2_iqr`, `O3_iqr`, `PM10_iqr`, `PM25_iqr`, and `SO2_iqr` when the respective q1 and q3 columns exist. They are internal working variables and are not included in the daily indices-only file.

## 5. Derived daily air-quality variables

These are computed per station-day from the source daily statistics. They are proxies derived from daily summaries, not necessarily official regulatory index values.

| Variable | Meaning and calculation | Base pollutants | Interpretation and caveats |
|---|---|---|---|
| `RPM` | Ratio of the daily-mean PM2.5 concentration to the daily-mean PM10 concentration: `AQ_mean_PM25 / AQ_mean_PM10`. | PM25, PM10 | Dimensionless. It is missing if PM10 is not positive or if the calculated ratio exceeds the configured `rpm_max=1.0`. The notebook does not otherwise clip the ratio. |
| `NOx` | NOx expressed as NO₂-equivalent mass: `1.533 × AQ_mean_NO + AQ_mean_NO2`. The factor is the ratio of molar masses, 46.006 / 30.006. | NO, NO2 | Daily-mean-derived concentration in µg/m³ on an NO₂-equivalent basis. This is a calculated mass-equivalent quantity, not a directly measured NOx column. |
| `NO2_NOx` | `AQ_mean_NO2 / NOx`, only where NOx is positive. | NO, NO2 | Dimensionless share on the mass-equivalent basis used by `NOx`. The notebook comments that lower values may indicate fresher combustion influence and higher values more aged air; this is a heuristic interpretation, not a unique source attribution. |
| `Ox_ppb` | Ozone plus NO₂ after converting each daily mean from µg/m³ to ppb: `O3_ppb + NO2_ppb`. | O3, NO2 | Units are ppb. The conversion uses molar volume 24.055 L/mol at 293.15 K and 101.325 kPa, with molar masses 47.998 g/mol for O₃ and 46.006 g/mol for NO₂. This is an oxidant proxy, not a direct ozone measurement. |
| `AQI_lo` | Lower-bound, EEA-inspired daily air-quality class. Each available pollutant's **daily mean** is converted to a class from 1 to 6; the maximum class across pollutants is retained. | PM25, PM10, NO2, O3, SO2 | Only defined when at least `aqi_min_pollutants=2` of these pollutants are available. `lo` uses daily means even for gases. It is a notebook-defined EEA-inspired proxy, not a claim of an official EEA hourly AQI. |
| `AQI_hi` | Upper-bound/worst-hour-oriented EEA-inspired class. PM2.5 and PM10 are assessed on daily means; NO₂, O₃, and SO₂ are assessed on daily maxima. The maximum pollutant class is retained. | PM25, PM10, NO2, O3, SO2 | Requires at least two available pollutant classes. Daily maximum values approximate the worst hour only where the source data preserve intraday/hourly information. PM uses daily means by design. |
| `AQI_hi_driver` | Categorical code naming the pollutant that attains the highest class in `AQI_hi`. | PM25, PM10, NO2, O3, SO2 | Available in the working daily data. If pollutants tie for the highest class, the code's tie-breaking order is `PM25`, `PM10`, `NO2`, `O3`, `SO2`. It is not included in the saved daily indices-only file, but is used to calculate driver shares in aggregate tables. |
| `AQI_hi_n` | Number of non-missing pollutant classes used to calculate `AQI_hi`. | PM25, PM10, NO2, O3, SO2 | Diagnostic count, from 0 to 5. Internal working daily variable; not included in the saved daily indices-only file or current aggregation outputs. |
| `AQI_hi_drv_<P>` | Per-day 0/1 indicator for whether pollutant `<P>` is the `AQI_hi_driver`, for `<P>` in `PM25`, `PM10`, `NO2`, `O3`, `SO2`. | PM25, PM10, NO2, O3, SO2 | Missing when the driver cannot be assigned. These indicators are used to produce aggregate `AQI_hi_drv_<P>_share` variables. They are not in the saved daily indices-only file. |
| `AQHI_lo` | Canadian AQHI-formula proxy computed from daily means for O₃ and NO₂ (converted to ppb) and PM2.5 (µg/m³). | O3, NO2, PM25 | A continuous index value, not the 1-to-6 AQI class. Requires all three inputs. Intended as a lower bound on a daily hourly-AQHI summary under the formula's convexity assumptions. |
| `AQHI_hi` | Same Canadian AQHI-formula proxy, using daily maxima for O₃, NO₂ and, when available, PM2.5. If the PM2.5 maximum is missing, the code falls back to the PM2.5 daily mean. | O3, NO2, PM25 | An upper-bound-oriented proxy. It is **not a strict upper bound** when PM2.5 only has daily resolution and its mean substitutes for the unobserved hourly maximum. Requires O₃ and NO₂ maxima and the PM2.5 daily mean. |
| `AQHI_ln` | Optional estimate of the daily mean of hourly AQHI, using a fitted lognormal distribution for hourly concentrations based on the daily median and quartiles, clipped to the daily min/max. | O3, NO2, PM25 | Not generated by the notebook's default run. It is calculated only when `build_daily_dataset(..., aqhi_lognormal=True)` is used. Run `check_lognormal_fit()` first and inspect its diagnostic; the assumption is not guaranteed to fit the hourly values. |

### AQI class cut-points used by the notebook

The code applies the following pollutant concentration upper bounds (µg/m³). Values at or below the first cut-point are class 1; values above the last cut-point are class 6. For intermediate classes, the value is assigned to the first upper bound it does not exceed.

| Pollutant | Class 1 upper bound | Class 2 upper bound | Class 3 upper bound | Class 4 upper bound | Class 5 upper bound |
|---|---:|---:|---:|---:|---:|
| PM25 | 5 | 15 | 50 | 90 | 140 |
| PM10 | 15 | 45 | 120 | 195 | 270 |
| NO2 | 10 | 25 | 60 | 100 | 150 |
| O3 | 60 | 100 | 120 | 160 | 180 |
| SO2 | 20 | 40 | 125 | 190 | 275 |

Class labels used in the notebook are: 1 = Good, 2 = Fair, 3 = Moderate, 4 = Poor, 5 = Very poor, 6 = Extremely poor. The code computes an overall AQI by taking the highest class among the pollutants that are available. CO and NO are not AQI inputs.

### Canadian AQHI formula used by the notebook

The implemented expression is:

`AQHI = (10 / 10.4) × 100 × [exp(0.000537 × O3_ppb) - 1 + exp(0.000871 × NO2_ppb) - 1 + exp(0.000487 × PM25_ug_m3) - 1]`

O₃ and NO₂ are converted to ppb; PM2.5 remains in µg/m³. The `lo` and `hi` labels refer to which daily statistics are inserted, not different formula coefficients.

## 6. Daily exceedance flags (`EX_*`)

Each flag is created from a pollutant statistic and a threshold. For the threshold dictionary below, the code uses the strict comparison **value > threshold**. A flag is `1` when the threshold is exceeded, `0` when the available value does not exceed it, and missing (`NaN`) when the required concentration statistic is missing. The AQI/AQHI class flags are the exception: they use **index >= 4**.

| Flag | Input statistic | Threshold / test | Meaning in this notebook |
|---|---|---|---|
| `EX_PM10_EU50` | PM10 daily mean | > 50 µg/m³ | Daily-mean PM10 above the code's EU 24-hour limit value. The notebook's comment says up to 35 exceedances per year are allowed under the cited framework; this flag only identifies exceedance days. |
| `EX_SO2_EU125` | SO2 daily mean | > 125 µg/m³ | Daily-mean SO₂ above the code's EU 24-hour limit value. The notebook's comment says up to 3 exceedances per year are allowed; this flag itself does not apply the annual allowance. |
| `EX_NO2_H200` | NO2 daily maximum | > 200 µg/m³ | Daily maximum above the code's EU hourly NO₂ limit value. The notebook's comment says up to 18 exceedances per year are allowed; the daily maximum is the available proxy for an hourly exceedance. |
| `EX_SO2_H350` | SO2 daily maximum | > 350 µg/m³ | Daily maximum above the code's EU hourly SO₂ limit value. The notebook's comment says up to 24 exceedances per year are allowed. |
| `EX_O3_H180` | O3 daily maximum | > 180 µg/m³ | Daily maximum above the code's ozone information threshold. |
| `EX_O3_H240` | O3 daily maximum | > 240 µg/m³ | Daily maximum above the code's ozone alert threshold. |
| `EX_PM25_WHO15` | PM25 daily mean | > 15 µg/m³ | Daily mean above the WHO 2021 24-hour PM2.5 guideline value configured by the notebook. |
| `EX_PM10_WHO45` | PM10 daily mean | > 45 µg/m³ | Daily mean above the WHO 2021 24-hour PM10 guideline value configured by the notebook. |
| `EX_NO2_WHO25` | NO2 daily mean | > 25 µg/m³ | Daily mean above the WHO 2021 24-hour NO₂ guideline value configured by the notebook. |
| `EX_SO2_WHO40` | SO2 daily mean | > 40 µg/m³ | Daily mean above the WHO 2021 24-hour SO₂ guideline value configured by the notebook. |
| `EX_CO_WHO4` | CO daily mean | > 4 mg/m³ | Daily mean above the WHO 2021 24-hour CO guideline value configured by the notebook. CO is handled in mg/m³. |
| `EX_AQI_lo_ge4` | `AQI_lo` | `AQI_lo >= 4` | The lower-bound AQI proxy is in class 4, 5, or 6. Described in code comments as a certain exceedance of this index-class threshold. |
| `EX_AQI_hi_ge4` | `AQI_hi` | `AQI_hi >= 4` | The upper-bound/worst-hour-oriented AQI proxy is in class 4, 5, or 6. Described as a possible exceedance. |
| `EX_AQHI_lo_ge4` | `AQHI_lo` | `AQHI_lo >= 4` | The lower-bound AQHI proxy is at least 4. |
| `EX_AQHI_hi_ge4` | `AQHI_hi` | `AQHI_hi >= 4` | The upper-bound-oriented AQHI proxy is at least 4. |

**Important interpretation note:** these are the thresholds and labels implemented in this notebook. The code does not reconstruct every legally required averaging interval from hourly data, nor does it itself enforce annual allowed-exceedance counts. In particular, it intentionally does not make an hourly CO > 10 mg/m³ flag, because daily summary statistics are insufficient to reconstruct the legally defined maximum daily 8-hour CO metric. The hourly-threshold flags use each day's maximum as the comparison statistic, which is a proxy for whether at least one observed hour could have exceeded the threshold when intraday source data are available.

## 7. Climate variables (`CL_*`)

The input climate columns are carried into the working daily table and aggregated separately from pollutants and air-quality indices. They are useful covariates; they are not pollutant concentrations and do not enter the AQI/AQHI formulas in this notebook.

| Daily input variable | Meaning in the notebook/source dataset | Unit or encoding | Period-level output(s) |
|---|---|---|---|
| `CL_lai_hv` | Constant fixed value of leaf area index for high vegetation | m²/m² | `CL_lai_hv_mean` |
| `CL_lai_lv` | Constant fixed value of leaf area index for low vegetation | m²/m² | `CL_lai_lv_mean` |
| `CL_rh` | Daily mean relative humidity | % | `CL_rh_mean`, `CL_rh_std` |
| `CL_ssr` | Daily maximum of surface solar radiation | J/m² | `CL_ssr_mean`, `CL_ssr_std`; these summarise daily maxima across the period |
| `CL_t2m` | Daily mean temperature at 2 metres | °C | `CL_t2m_mean`, `CL_t2m_std` |
| `CL_tp` | Daily total precipitation | m | `CL_tp_sum`, `CL_tp_wet_days` |
| `CL_winddir` | Daily modal wind direction | `1=N`, `2=E`, `3=S`, `4=W` | `CL_winddir_N_share`, `CL_winddir_E_share`, `CL_winddir_S_share`, `CL_winddir_W_share`, `CL_winddir_mode` |
| `CL_windspeed` | Daily mean wind speed | m/s | `CL_windspeed_mean`, `CL_windspeed_std` |
| `CL_blh` | Daily mean atmospheric boundary-layer height | m | `CL_blh_mean`, `CL_blh_std` |

For climate variables with `mean` and `std`, the period-level mean and standard deviation are calculated over the daily values. Thus, for example, `CL_ssr_mean` is the mean of daily maximum solar-radiation values in the period, not a mean of hourly radiation.

- `CL_tp_sum` is the sum of daily precipitation over the period, in metres.
- `CL_tp_wet_days` counts days with precipitation **greater than or equal to** 0.001 m, which is 1 mm.
- `CL_winddir_<sector>_share` is the fraction (0 to 1, not 0 to 100) of valid daily wind-direction values assigned to that sector during the period.
- `CL_winddir_mode` is the period's most frequent daily direction code: 1 = N, 2 = E, 3 = S, 4 = W. In a tie, the implementation takes the first direction in this order: N, E, S, W. It is missing if all wind-direction values in the period are missing.

Climate columns are not coverage-masked by the notebook's `min_coverage` rule. This does not mean every source climate value is guaranteed to be present; it means the aggregation code does not set climate summaries to missing solely because their valid-day coverage is below the configured threshold.

## 8. Period-level aggregate variables

The same naming patterns are used in `aggregated_week.csv`, `aggregated_month.csv`, `aggregated_season.csv`, and `aggregated_year.csv`. They summarise the daily working table. They are **not** all calculated from the same pooled set of sub-daily measurements.

### 8.1 Pollutant concentrations

For each available pollutant `P` in `CO`, `NO`, `NO2`, `O3`, `PM10`, `PM25`, `SO2`, the aggregate table may contain:

| Pattern | Meaning |
|---|---|
| `P_mean` | Mean over the period of the available **daily means** for pollutant P. |
| `P_median` | Median across the period's daily-mean values. |
| `P_std` | Standard deviation across the period's daily-mean values, representing day-to-day variability. It is not the standard deviation of hourly observations. |
| `P_p95` | 95th percentile across the period's daily-mean values. |
| `P_max_mean` | Mean across the period of the available daily maxima for P. Since intraday PM maxima are masked for rows inferred to be daily-resolution, these statistics use PM rows with intraday information. |
| `P_max_peak` | Maximum of the available daily maxima for P within the period. It describes the peak of the daily maxima, not necessarily a rigorously reconstructed regulatory averaging metric. |
| `P_iqr_mean` | Mean across the period of the daily intraday interquartile range (`q3 - q1`). For PM, daily-resolution rows are excluded by masking their IQR. |

The `P_max_mean`, `P_max_peak`, and `P_iqr_mean` columns are only created where the respective source max or IQR data exist. The code's `present` pollutant list is based on availability of `AQ_mean_P`.

### 8.2 Derived indices and ratios

For every derived variable that exists in the daily working data, the aggregate table contains these period summaries:

- `RPM_mean`, `RPM_median`, `RPM_std`, `RPM_p95`
- `NOx_mean`, `NOx_median`, `NOx_std`, `NOx_p95`
- `NO2_NOx_mean`, `NO2_NOx_median`, `NO2_NOx_std`, `NO2_NOx_p95`
- `Ox_ppb_mean`, `Ox_ppb_median`, `Ox_ppb_std`, `Ox_ppb_p95`
- `AQI_lo_mean`, `AQI_lo_median`, `AQI_lo_std`, `AQI_lo_p95`
- `AQI_hi_mean`, `AQI_hi_median`, `AQI_hi_std`, `AQI_hi_p95`
- `AQHI_lo_mean`, `AQHI_lo_median`, `AQHI_lo_std`, `AQHI_lo_p95`
- `AQHI_hi_mean`, `AQHI_hi_median`, `AQHI_hi_std`, `AQHI_hi_p95`
- Optional `AQHI_ln_mean`, `AQHI_ln_median`, `AQHI_ln_std`, `AQHI_ln_p95` if the notebook is run with lognormal AQHI enabled.

For these columns, `mean`, `median`, `std`, and `p95` are calculated over the corresponding **daily index values** within the period. The aggregate AQI values remain summaries of daily class numbers, so their mean or percentile should not be mistaken for a new official AQI class.

The categorical daily variable `AQI_hi_driver` and `AQI_hi_n` are not directly aggregated into scalar period variables. Instead, the code aggregates the driver indicators described below.

### 8.3 AQI upper-bound driver shares

For each pollutant `P` among `PM25`, `PM10`, `NO2`, `O3`, and `SO2`, the aggregate output can contain `AQI_hi_drv_<P>_share`.

This is the fraction of valid station-days in the period on which pollutant P was identified as setting the maximum class for `AQI_hi`. It ranges from 0 to 1. Shares can sum to less than 1 where no driver could be assigned on some days; when every day with a valid driver has exactly one assigned pollutant, the five shares sum to 1 over those valid driver-days. Ties are handled using the priority order specified under `AQI_hi_driver`.

### 8.4 Aggregated exceedance flags

Each daily flag `EX_*` may produce two period-level columns:

| Pattern | Meaning |
|---|---|
| `EX_*_n` | Number of valid days in the period on which that flag is 1. Missing daily flags are not counted as valid days; zero-valued valid flags contribute zero to the sum. |
| `EX_*_pct` | Percentage of valid days in the period on which the flag is 1: `100 × number of exceedance days / number of non-missing daily flags`. |

The percentage is out of valid days for that flag, not necessarily all calendar days. It is expressed on a 0-to-100 scale. The flags count days flagged by the notebook's daily rule; for example, `EX_O3_H180_n` counts days whose available daily maximum ozone exceeded the configured 180 µg/m³ threshold. It does not apply any annual allowable-exceedance rule.

The flag names are the 15 daily flags listed in Section 6.

## 9. Reliability tables

Every `reliability_{frequency}.csv` uses the same row keys and row order as `aggregated_{frequency}.csv`. In the default station-grouped run, it includes the following fields:

| Variable / pattern | Meaning |
|---|---|
| `n_days_expected` | Number of calendar days in the full target period: 7 for a complete week, 28 to 31 for a month, the calendar length of the season, or 365/366 for a year. |
| `<P>_n_valid` | Number of valid station-days for pollutant P's daily-mean series. P is one of the pollutant codes listed in Section 3. |
| `<INDEX>_n_valid` | Number of valid station-days for a derived daily index, such as `RPM_n_valid`, `AQI_hi_n_valid`, or `AQHI_lo_n_valid`. `AQHI_ln_n_valid` occurs only when that optional index was computed. |
| `EX_*_n_valid` | Number of station-days with a non-missing value for the corresponding daily flag, including both 0 and 1 values. It is the denominator used for the matching `EX_*_pct`. |
| `PM10_n_daily_only` | Number of days in the period flagged as PM10 daily-resolution by the notebook's min-equals-max test. |
| `PM25_n_daily_only` | Number of days in the period flagged as PM2.5 daily-resolution by the notebook's min-equals-max test. |
| `<P-or-index>_n_months` | Annual-table diagnostic only. For pollutant daily means and derived-index values, counts how many calendar months in the year have at least one valid day for that variable. Optional-index fields appear only if the corresponding index was computed. |

`n_months` fields are produced only for `frequency="year"` and for the level series (pollutant daily means and derived indices), not for climate variables or daily exceedance flags. In the notebook's default run the aggregation is grouped by station, so `n_stations` is not expected in the reliability output. The function can add `n_stations` if it is used differently to pool across stations, but that is not the default invocation.

### Coverage masking

The default `min_coverage=0.5` means that for a station-period, all aggregate statistics associated with a pollutant or derived index are set to missing if the number of valid daily values divided by `n_days_expected` is **less than 0.5**. Exactly 50% coverage passes. The reliability file retains the valid-day count, allowing the reason for missing period summaries to be distinguished from a genuinely missing source variable.

Coverage masking follows each variable's valid-day count. For example, missing `AQI_hi` values do not automatically imply missing `NO2_mean` if NO₂ itself has enough coverage. Climate variables are exempt from this masking rule. The daily-only PM counts are retained as a separate diagnostic and are not themselves coverage counts for concentration summaries.

## 10. What is and is not present in each final dataset

### Saved daily indices-only file

Expected key fields: `AirQualityStation`, `time`.

The output selects the following derived fields when available: `RPM`, `NOx`, `NO2_NOx`, `Ox_ppb`, `AQI_lo`, `AQI_hi`, `AQI_hi_driver`, `AQHI_lo`, `AQHI_hi`, and optional `AQHI_ln`. It then includes every available `EX_*` flag. It **does not** include the pollutant concentration columns (`AQ_*`), IQR or daily-only indicators, `AQI_hi_n`, `AQI_hi_drv_<P>` dummy columns, or `CL_*` climate covariates.

### Weekly/monthly/seasonal/yearly aggregate files

Expected keys are the station identifier plus the time keys described in Section 2. Values include the pollutant aggregate patterns, derived index aggregate patterns, AQI driver shares, exceedance counts/percentages, and climate aggregate variables described above. There is no direct period-level field for the categorical `AQI_hi_driver`; use the `AQI_hi_drv_<P>_share` columns instead.

### Reliability files

Contain the same station/time-period keys plus data-validity counts, PM daily-resolution counts, and, for years, month-availability counts. They are designed to be joined to the corresponding aggregate table by identical row order and keys.

## 11. Missing data and implementation caveats

1. **Missing stays missing.** Daily flag values are `NaN` when their underlying statistic is missing. The period flag percentage uses only non-missing flags in its denominator. A period with no valid daily flag observations has no meaningful percentage.
2. **Indices require their inputs.** `RPM` requires PM25 and PM10 means; `NOx` and `NO2_NOx` require NO and NO₂; `Ox_ppb` requires O₃ and NO₂; AQHI requires O₃, NO₂, and PM25; AQI requires at least two of its five configured pollutants.
3. **Daily resolution limits peak interpretation.** The source dataset may contain particulate matter at daily rather than hourly resolution. The notebook masks the intraday PM statistics for rows inferred to be daily-only. Consequently, PM peak and IQR summaries only use days with intraday information, while the PM daily mean continues to be used for level statistics and indices.
4. **The daily maximum is a proxy for hourly thresholds.** The code compares `AQ_max_NO2`, `AQ_max_SO2`, or `AQ_max_O3` against the configured hourly threshold. This is meaningful as a worst-observed-within-day proxy only when the underlying daily max represents the relevant hourly observations.
5. **Regulatory labels are comments/configuration, not a full compliance engine.** The notebook flags threshold days but does not aggregate legal exceedance allowances into compliance conclusions. Check the relevant regulation's averaging period, annual count allowance, units, and valid-data requirements before interpreting flags as legal non-compliance.
6. **AQI and AQHI are approximations from daily summaries.** `AQI_lo`, `AQI_hi`, `AQHI_lo`, and `AQHI_hi` bracket or proxy hourly-based values using the daily statistics available; they should not be described as direct hourly-index observations. The AQHI upper-bound caveat for daily-resolution PM2.5 is especially important.
7. **`AQHI_ln` is off by default.** The notebook calls `build_daily_dataset(df_measurements)` without `aqhi_lognormal=True`. The separate `check_lognormal_fit(df_daily)` call is diagnostic only and does not activate the optional index.
8. **Aggregation summaries refer to daily values.** `P_std`, `P_p95`, and the equivalent derived-index statistics describe variability across daily means or daily index values, not dispersion across the hourly observations within the aggregation period.
9. **Station-period keys matter.** Unless the aggregation function is called with different grouping arguments, aggregate rows are station-specific. Do not interpret them as regional or national summaries without a separate pooling procedure and appropriate handling of stations that do not measure every pollutant.

## 12. Quick lookup: base inputs by derived variable

| Derived variable or flag family | Required base pollutant(s) / statistic |
|---|---|
| `RPM` | PM25 daily mean and PM10 daily mean |
| `NOx` | NO daily mean and NO2 daily mean |
| `NO2_NOx` | NO and NO2 daily means, with positive calculated NOx |
| `Ox_ppb` | O3 and NO2 daily means, converted to ppb |
| `AQI_lo` | Available daily means among PM25, PM10, NO2, O3, SO2; at least 2 needed |
| `AQI_hi` | PM daily means plus NO2, O3, SO2 daily maxima; at least 2 available classes needed |
| `AQI_hi_driver`, `AQI_hi_drv_<P>_share` | Same pollutant classes as `AQI_hi` |
| `AQHI_lo` | O3 daily mean, NO2 daily mean, PM25 daily mean |
| `AQHI_hi` | O3 daily max, NO2 daily max, PM25 daily max when available, otherwise PM25 daily mean |
| `AQHI_ln` (optional) | O3, NO2, PM25 daily min/q1/median/q3/max plus daily mean for fit diagnostics; requires enabling the option |
| `EX_PM10_EU50`, `EX_PM10_WHO45` | PM10 daily mean |
| `EX_SO2_EU125`, `EX_SO2_WHO40` | SO2 daily mean |
| `EX_NO2_H200` | NO2 daily maximum |
| `EX_SO2_H350` | SO2 daily maximum |
| `EX_O3_H180`, `EX_O3_H240` | O3 daily maximum |
| `EX_PM25_WHO15` | PM25 daily mean |
| `EX_NO2_WHO25` | NO2 daily mean |
| `EX_CO_WHO4` | CO daily mean in mg/m³ |
| `EX_AQI_lo_ge4`, `EX_AQI_hi_ge4` | Corresponding computed AQI proxy |
| `EX_AQHI_lo_ge4`, `EX_AQHI_hi_ge4` | Corresponding computed AQHI proxy |

---

**Naming convention summary:** `AQ_*` fields are source daily pollutant summaries in the working table; `<P>_*` fields in the aggregate tables are temporal summaries for a pollutant; `EX_*` fields are daily exceedance flags, while `EX_*_n` and `EX_*_pct` are period summaries; `CL_*` fields are climate variables; and derived index names (`RPM`, `NOx`, `NO2_NOx`, `Ox_ppb`, `AQI_*`, `AQHI_*`) are calculated from the pollutant inputs listed above.
