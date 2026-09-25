# Forecasting Surface Drifter Trajectories in the Gulf Stream

## Research plan

### 1. Research questions and objectives

This project investigates whether the recent trajectory of a surface drifter can be used to predict where it will travel next in the Gulf Stream region in period July to September.

The primary research question is:

> Given the recent hourly track of a drifter in the Gulf Stream, how accurately can we predict its position 1 hour, 1 day, and 1 week ahead on drifters that were not used for model training?

The project has four objectives:

1. Construct reliable multi-horizon position forecasts at lead times of 1, 24, and 168 hours.
2. Compare the proposed model with two physically meaningful baselines: constant present velocity and advection by the regional mean flow.
3. Measure how forecast skill changes with lead time, season, location, data quality, and drogue status.
4. Produce a reproducible forecasting and evaluation pipeline that can be applied to another longitude-latitude box without rewriting the analysis.

Secondary questions include:

- How much recent trajectory history is useful: 6, 24, 48, or 72 hours?
- Does using recent velocity history improve forecasts beyond using only the latest observed velocity?
- Are predictions less accurate near strong Gulf Stream gradients, meanders, or eddies?
- Does forecast performance change after a drifter loses its drogue?

### 2. Data, region, and data description

#### Study region

The initial region of interest is the following Gulf Stream box:

| Boundary | Value |
| --- | ---: |
| Western longitude | 80°W (`-80`) |
| Eastern longitude | 60°W (`-60`) |
| Southern latitude | 30°N |
| Northern latitude | 45°N |

This box captures the Gulf Stream along the east coast of the United States and its northeastward extension into the North Atlantic. The boundaries may be refined after the full coverage audit, but all code will accept the region as parameters rather than hard-coded constants.

#### Dataset

The project uses the NOAA Global Drifter Program hourly dataset, version 2.01. It contains quality-controlled, interpolated hourly locations, surface velocities, sea-surface temperatures, uncertainty estimates, and drifter metadata. The global CloudDrift representation contains 19,396 trajectories and 197,214,787 hourly observations.

This is one NOAA dataset, not two separate datasets. In its CloudDrift/Xarray representation, variables are organised along two linked dimensions:

- **Trajectory dimension (`traj`)**: one entry per drifter, including `id`, `rowsize`, deployment information, start and end dates, location system, and drogue-loss date.
- **Observation dimension (`obs`)**: many hourly entries per drifter, including `time`, `lat`, `lon`, eastward velocity `ve`, northward velocity `vn`, sea-surface temperature `sst`, uncertainty estimates, quality flags, and `drogue_status`.

Our core modelling variables are:

| Variable | Meaning | Unit or type |
| --- | --- | --- |
| `id` | Unique drifter identifier | integer |
| `time` | Observation time | UTC datetime |
| `lat`, `lon` | Position | degrees |
| `ve`, `vn` | Eastward and northward velocity | m/s |
| `sst` | Fitted sea-surface temperature | K |
| `drogue_status` | Whether the drogue is present | boolean |

### 3. Importance and significance

Accurate surface-trajectory forecasts support search and rescue, oil-spill response, marine-debris tracking, fisheries management, and the interpretation of Lagrangian ocean observations. The Gulf Stream is an important test region because it is a fast, narrow western boundary current with strong spatial gradients, meanders, and eddies. These features create transport pathways that are economically and scientifically important while also making trajectories difficult to predict.

Short forecasts may be dominated by the drifter's current velocity, whereas errors at one day or one week can grow rapidly as the drifter encounters curved flow, changing current speed, or an eddy. Comparing a learned model against simple physical baselines will show whether recent trajectory history contains useful predictive information beyond persistence and climatology.

### 4. Background and existing approaches

Elipot et al. (2016) developed the hourly GDP position and velocity product by fitting local trajectory models to irregular satellite fixes while accounting for location error. The resulting hourly resolution retains high-frequency motions that are not resolved safely by the older six-hourly product.

Several families of methods are relevant to this project:

- **Persistence or constant-velocity models** assume that the latest velocity continues unchanged. They are simple but can be difficult to beat at short lead times.
- **Mean-flow advection models** estimate a spatial, and optionally seasonal, climatological velocity field and integrate a particle through that field.
- **Statistical time-series models** use recent positions or velocities to extrapolate future motion.
- **Machine-learning sequence models** learn nonlinear relationships in recent drifter motion. Aksamit et al. (2020), for example, combined recurrent learning with a reduced physical drifter model. More recent work by Grossi et al. (2025) found that simple neural networks did not consistently beat autoregressive baselines on observed Gulf of Mexico trajectories, while a spatiotemporal graph model showed more promise. This supports using strong baselines and held-out trajectories rather than assuming that a more complex model will automatically perform better.

Our modelling approach will use engineered recent-track features with gradient-boosted regression. This gives a practical and interpretable test of whether recent velocity, acceleration, turning, season, and location improve on the baselines. A GRU, LSTM, will be considered as an extension if the feature-based model and data audit justify the additional complexity.

### 5. Proposed method

#### 5.1 Cohort construction and quality control

1. Define a common study window for each year, from 1 September at 00:00 UTC through 30 November at 23:00 UTC.
2. Identify all drifter IDs with at least one valid observation inside the Gulf Stream box (`80°W–60°W`, `30°N–45°N`) during this window.
3. For each selected drifter, retrieve all available observations within the same September–November window, regardless of location. Include observations before its first appearance in the box and after it leaves the box.
4. Sort observations by `id` and `time`, remove duplicate drifter–time records, and align trajectories to the same hourly UTC time grid. Leave unavailable observations missing rather than assuming that every drifter has a complete record.
5. Remove invalid positions and velocities, examine uncertainty and quality flags, and retain drogue status for separate evaluation of drogued and undrogued observations.
6. Split trajectories at missing or unacceptably large time gaps and at boundaries between annual study windows. Do not split or truncate a trajectory simply because it crosses the geographic boundary.
7. Construct forecast samples only when the required history and target observations are available within an uninterrupted segment of the same September–November window. Forecast origins and targets may lie outside the Gulf Stream box.
8. Extend the spatial coverage of the mean-flow baseline to support trajectories outside the selection box, using training data only and a documented fallback where coverage is insufficient.

#### 5.2 Prediction samples and targets

For each valid forecast origin at time `t`, the input will be the previous `L` hourly observations, initially `L = 24` or `48`. Targets will be defined at three horizons:

| Horizon | Target time |
| --- | --- |
| Short | `t + 1 hour` |
| Medium | `t + 24 hours` |
| Long | `t + 168 hours` |

Instead of directly predicting latitude and longitude, the model will predict local eastward and northward displacement from the forecast origin. This avoids longitude wrap-around and makes the target represent movement in physical distance. Predicted displacements will then be converted back to geographic coordinates.

Candidate predictors include:

- latest position, velocity, speed, and direction;
- means, standard deviations, and trends of `ve` and `vn` over recent windows;
- acceleration and recent turning angle;
- cyclical hour-of-day features and the position of the observation within the September–November study window;
- sea-surface temperature and recent temperature change;
- local September–November mean-flow velocity;
- uncertainty measures and drogue status.

Separate direct models will initially be trained for 1-, 24-, and 168-hour displacements. This avoids recursively feeding earlier prediction errors through 168 one-hour steps.

### 6. Preliminary Research

The analyses in [03_preliminary_analysis.ipynb](03_preliminary_analysis.ipynb) examine data completeness, forecast-horizon feasibility, and a simple prediction benchmark. The results below were checked against the four existing processed files, which contain **July–September observations from 2007–2022**. They are preliminary findings; the September–November cohort described in Section 5 still needs to be extracted and evaluated.

#### 6.1 Data completeness

The existing extract contains **2,482,661 hourly observations**. No missing or non-finite values were found in longitude, latitude, or either velocity component, and there were no duplicate trajectory–time records. SST missingness varies by period:

| Study period | Hourly observations | Unique trajectory IDs within period | Missing SST |
| --- | ---: | ---: | ---: |
| 2007–2010 | 616,023 | 251 | 3.05% |
| 2011–2014 | 640,974 | 320 | 6.15% |
| 2015–2018 | 587,026 | 231 | 3.30% |
| 2019–2022 | 638,638 | 240 | 0.40% |

Position and velocity are therefore available for the initial models. SST can be tested as an additional predictor, with missing values handled using training data only. Completeness alone does not establish measurement accuracy.

#### 6.2 Availability of future observations

![Percentage of origins with an exact future observation, by study period and forecast lead time](images/forecast_horizon_availability.png)

Future observations were matched using the same trajectory ID and an exact timestamp offset, rather than a row shift. Across the four periods, availability is **99.92–99.94% at 1 hour**, **98.28–98.60% at 24 hours**, and **90.20–91.27% at 168 hours**. This supports investigating all three proposed horizons. These percentages measure endpoint availability; requiring uninterrupted input history and valid observations throughout each forecast segment will further restrict the usable samples.

#### 6.3 A 24-hour constant-velocity benchmark

![Median, mean, and 90th-percentile 24-hour constant-velocity position errors for four study periods](images/constant_velocity_24h_error.png)

The baseline extrapolates the latest eastward and northward velocities for 24 hours and compares the predicted position with the exact future observation using great-circle distance. Median error ranges from **12.45 to 13.66 km**, while the 90th percentile reaches **29.70–33.34 km**. The larger upper-tail errors motivate reporting more than an average and testing whether recent trajectory history improves difficult forecasts. These results use all available 24-hour pairs and provide an exploratory benchmark, not a held-out model evaluation.

### Expected deliverables

- Parameterised code for loading any longitude-latitude region.
- A documented Gulf Stream trajectory dataset and preprocessing pipeline.
- Implementations of the constant-velocity and mean-flow baselines.
- A trained multi-horizon forecasting model.
- Held-out evaluation with error-versus-lead-time plots.
- A final report explaining model skill, limitations, and practical implications.

### References

- Aksamit, N. O., Sapsis, T. P., & Haller, G. (2020). Machine-learning mesoscale and submesoscale surface dynamics from Lagrangian ocean drifter trajectories. *Journal of Physical Oceanography, 50*(5), 1179–1196. <https://doi.org/10.1175/JPO-D-19-0238.1>
- Elipot, S., Lumpkin, R., Perez, R. C., Lilly, J. M., Early, J. J., & Sykulski, A. M. (2016). A global surface drifter data set at hourly resolution. *Journal of Geophysical Research: Oceans, 121*, 2937–2966. <https://doi.org/10.1002/2016JC011716>
- Elipot, S., Sykulski, A., Lumpkin, R., Centurioni, L., & Pazos, M. (2022). A dataset of hourly sea surface temperature from drifting buoys. *Scientific Data, 9*, 567. <https://doi.org/10.1038/s41597-022-01670-2>
- Grossi, M. D., Jegelka, S., Lermusiaux, P. F. J., & Özgökmen, T. M. (2025). Surface drifter trajectory prediction in the Gulf of Mexico using neural networks. *Ocean Modelling, 196*, 102543. <https://doi.org/10.1016/j.ocemod.2025.102543>
- NOAA Global Drifter Program. Hourly location, current velocity, and temperature collected from Global Drifter Program drifters world-wide, version 2.01. <https://doi.org/10.25921/x46c-3620> (accessed 21 September 2026).
