# Forecasting Surface Drifter Trajectories in the Gulf Stream

## Research plan

### 1. Research questions and objectives

This project investigates whether the recent trajectory of a surface drifter can be used to predict where it will travel next in the Gulf Stream region.

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

For example, a drifter may have one trajectory-level entry with `id = 12345` and `rowsize = 8,000`, linked to 8,000 consecutive hourly observations in the `obs` dimension. This storage format is called a contiguous ragged array because different drifters have different numbers of observations. When the same data are requested as a flat table through NOAA ERDDAP, the drifter ID and relevant metadata are repeated across its hourly rows, so the result looks like an ordinary table.

The raw satellite fixes were not necessarily recorded exactly once per hour. The hourly product estimates positions and velocities on a regular one-hour grid from irregular and noisy source observations. Consequently, uncertainty and quality variables will be retained during preprocessing rather than treating every interpolated observation as equally reliable.

The core modelling variables are:

| Variable | Meaning | Unit or type |
| --- | --- | --- |
| `id` | Unique drifter identifier | integer |
| `time` | Observation time | UTC datetime |
| `lat`, `lon` | Position | degrees |
| `ve`, `vn` | Eastward and northward velocity | m/s |
| `sst` | Fitted sea-surface temperature | K |
| `err_lat`, `err_lon` | Position uncertainty | degrees |
| `err_ve`, `err_vn` | Velocity 95% confidence intervals | m/s |
| `drogue_status` | Whether the drogue is present | boolean |

The initial inspection script is [`inspect_gulf_stream.py`](inspect_gulf_stream.py). It queries a small subset from NOAA ERDDAP without downloading the full global archive.

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

Our first modelling approach will use engineered recent-track features with gradient-boosted regression. This gives a practical and interpretable test of whether recent velocity, acceleration, turning, season, and location improve on the baselines. A GRU, LSTM, or temporal convolutional model will be considered as an extension if the feature-based model and data audit justify the additional complexity.

### 5. Proposed method

#### 5.1 Cohort construction and quality control

1. Select observations whose forecast origin lies inside the Gulf Stream box.
2. Sort each trajectory by `id` and `time`.
3. Break trajectories at missing or unacceptably large time gaps.
4. Require enough uninterrupted history and future observations for each forecast horizon.
5. Remove invalid positions and velocities and examine the supplied uncertainty and quality flags.
6. Retain drogue status so that drogued and undrogued forecasts can be evaluated separately. The primary analysis will prioritise drogued observations because undrogued drifters are more affected by direct wind slip.
7. Preserve future locations even when a drifter leaves the study box, provided the global trajectory remains available. A spatial buffer will be used for the mean-flow baseline so forecasts near the region boundary can still be integrated.

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
- cyclical hour-of-day and day-of-year features;
- sea-surface temperature and recent temperature change;
- local and seasonal mean-flow velocity;
- uncertainty measures and drogue status.

Separate direct models will initially be trained for 1-, 24-, and 168-hour displacements. This avoids recursively feeding earlier prediction errors through 168 one-hour steps.

#### 5.3 Baselines

**Baseline 1: constant present velocity.** The latest observed `ve` and `vn` are held constant and used to advect the current position for the full lead time:

\[
\hat{p}_{t+h} = \operatorname{advance}(p_t, v_{e,t}, v_{n,t}, h).
\]

**Baseline 2: mean-flow advection.** A gridded velocity climatology will be estimated using training drifters only. Mean `ve` and `vn` will be calculated by spatial cell and season or month. Starting at the observed position, the prediction will be advanced in hourly steps, looking up the mean velocity at each new position. Sparse cells will use a documented fallback such as a coarser grid or smoothed neighbouring estimate.

#### 5.4 Train, validation, and test design

The split will be performed by complete drifter ID, not by randomly splitting hourly rows:

- 70% of drifters for training;
- 15% for validation and model selection;
- 15% held out for final testing.

All overlapping windows from a given drifter will remain in the same partition. This prevents information from the same physical trajectory appearing in both training and testing. If coverage permits, a second evaluation will hold out a later time period to test temporal generalisation.

All preprocessing parameters, including scaling, mean-flow fields, feature thresholds, and imputation values, will be fitted on the training partition only.

#### 5.5 Models and evaluation

The planned modelling sequence is:

1. Constant-velocity baseline.
2. Seasonal mean-flow baseline.
3. Regularised linear or autoregressive model.
4. Gradient-boosted regression using recent-track features.
5. Optional sequence model if it offers a justified extension.

The primary evaluation measure will be great-circle distance between the predicted and observed position. For each horizon and model, we will report:

- median position error in kilometres;
- mean position error and RMSE;
- 75th and 90th percentile error;
- error distributions and error versus lead time;
- skill relative to each baseline;
- bootstrap confidence intervals resampled by drifter ID.

A simple relative skill score will be calculated as:

\[
\operatorname{Skill} = 1 - \frac{\operatorname{Error}_{model}}
                              {\operatorname{Error}_{baseline}}.
\]

Positive skill indicates improvement over the selected baseline. Performance will also be stratified by season, subregion, speed regime, trajectory uncertainty, and drogue status.

### 6. Initial analysis and planned visualisations

A preliminary read of the official NOAA endpoint used the proposed box and the period 1–3 January 2020. It returned 762 hourly observations from 11 unique drifters. Ten drifters contributed all 72 possible hourly observations, while one contributed 42 observations because it entered or left the selected box during the period. In this small diagnostic sample, the observed velocity components ranged approximately from `-2.26` to `0.96 m/s` eastward and from `-0.91` to `1.02 m/s` northward. This confirms the expected repeated-measures structure: many hourly rows belong to each drifter.

This three-day subset is only a pipeline check and is not evidence that the full region has sufficient spatial or temporal coverage. The next analysis will audit the complete Gulf Stream subset using:

1. a map of all drifter tracks, coloured by time or speed;
2. spatial heatmaps of observation counts and unique drifter counts per grid cell;
3. annual and monthly counts of observations and unique drifters;
4. trajectory-duration and time-gap distributions;
5. velocity, speed, sea-temperature, and uncertainty distributions;
6. maps comparing drogued and undrogued coverage;
7. example forecast cases showing the observed path and both baseline predictions.

The coverage analysis will determine the final spatial grid, seasonal grouping, history length, and whether the initial geographic box should be refined.

### 7. Timeline and work plan

| Period | Planned work | Output |
| --- | --- | --- |
| Week 1 | Confirm research question, region, data access, and responsibilities | Research plan and reproducible sample query |
| Weeks 2–3 | Download or stream the regional subset; perform quality checks and coverage analysis | Clean regional dataset and exploratory figures |
| Weeks 3–4 | Construct uninterrupted track segments, forecast origins, targets, and ID-level splits | Modelling table and documented split |
| Weeks 4–5 | Implement and validate both baselines | Baseline error curves at 1, 24, and 168 hours |
| Weeks 5–7 | Develop linear and gradient-boosted models; tune using validation drifters | Selected main model and ablation results |
| Weeks 7–8 | Evaluate on held-out drifters; analyse performance by season, location, and drogue status | Final metrics, confidence intervals, and error maps |
| Weeks 8–9 | Optional sequence-model extension and sensitivity analysis | Extension results and model comparison |
| Week 10 | Finalise report, code, figures, reproducibility checks, and presentation | Final submitted product |

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
