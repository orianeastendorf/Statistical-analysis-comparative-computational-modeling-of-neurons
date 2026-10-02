# Fit comparison and cell characterization scripts

Three scripts for working with Allen Institute NWB 1.0 recordings of human and mouse cells and NEURON-style model fits:

| Script | Purpose | Inputs |
|---|---|---|
| `compare_fits_RMSE.py` | First fit vs. best fit: raw voltage error (RMSE or SSE) per sweep, averaged per cell | fit CSVs + NWB |
| `compare_fits_per_features.py` | First fit vs. best fit: normalized electrophysiological feature error (via eFEL) per cell | fit CSVs + NWB |
| `cell_intrinsic_properties_database.py` | Characterization of the measured cells themselves (passive and active properties), human vs. mouse | NWB only |

The first two compare a model to measurements. The third has no model involved and does not import from the other two.


## 1. Software requirements

- **Python 3.8 or newer** (3.9+ recommended)
- Python packages:

| Package | Needed by | Notes |
|---|---|---|
| `numpy` | all | |
| `pandas` | all | Named aggregation (`.agg(name=(col, func))`) is used |
| `scipy` | `compare_fits_RMSE.py`, `compare_fits_by_feature.py` | Wilcoxon and paired t-test |
| `matplotlib` | all | |
| `seaborn` | all | Version **0.12 or newer** (`sns.boxplot(..., hue=..., legend=False)` is used) |
| `h5py` | all | Reads the NWB 1.0 files directly |
| `efel` | `compare_fits_by_feature.py`, `cell_intrinsic_properties_database.py` | Calls used: `efel.set_setting("Threshold", ...)` and `efel.get_feature_values(..., raise_warnings=False)` |

Install:

```bash
pip install numpy pandas scipy matplotlib seaborn h5py efel
```

`pynwb` is **not** needed. The classic Allen Institute NWB 1.0 format cannot be read by pynwb, so the scripts use `h5py`.

---

## 2. Input files

### 2.1 Fit comparison scripts (`compare_fits_RMSE.py`, `compare_fits_by_features.py`)

All input files must be in **one folder**, set via `DATA_DIR` in `compare_fits_RMSE.py`. For every cell with ID `{ID}`, three files are required:

```
best_fit_traces_{ID}.csv
first_fit_traces_{ID}.csv
{ID}_ephys.nwb
```

Cells are discovered automatically from the `best_fit_traces_*.csv` files. A cell is skipped with a warning if any of its three files is missing, or if the NWB sweep matching fails.

### 2.2 Intrinsic properties script (`cell_intrinsic_properties_database.py`)

Only NWB files are needed, in the folder set via `DATA_DIR` in `cell_intrinsic_properties_database.py` (its own setting, separate from the one above):

```
{ID}_ephys.nwb
```

Cells are discovered from all `*_ephys.nwb` files in that folder. No CSV files are used.

### 2.3 Cell ID and species (all scripts)

The species is read from the part of the cell ID **before the first underscore**:

| Cell ID | Prefix | Species |
|---|---|---|
| `h_001`, `h_562381187` | `h` | Human |
| `m_101`, `m_501234567` | `m` | Mouse |

Other prefixes are labeled `Unknown` and drawn in grey in the intrinsic properties plots. In the fit comparison plots, only `Human` and `Mouse` are colored (see `SPECIES_PALETTE`), so `Unknown` cells appear in the tables and statistics but not in the scatter plot. Extend `SPECIES_PREFIX_MAP` if you use different prefixes. Each script that needs it has its own copy (`compare_fits_features.py` imports it from `compare_fits.py`, `cell_intrinsic_properties.py` defines its own), so change it in both `compare_fits.py` and `cell_intrinsic_properties.py`.

### 2.4 Fit CSV files (`best_fit_traces_*.csv`, `first_fit_traces_*.csv`)

- One column `time_ms` plus one column per sweep.
- Sweep column names must follow the pattern `sweepNN_amp<sign><value>nA_mV`, for example:

```
time_ms, sweep00_amp-0.110nA_mV, sweep01_amp+0.090nA_mV, sweep02_amp+0.150nA_mV
```

- The stimulus amplitude (in nA) is parsed from the column name with the regex `amp([+-]?[0-9.]+)nA` and used to find the matching NWB sweep. The amplitude in the name must therefore be correct.
- Voltages in **mV**, time in **ms**, on a **uniform time grid**.
- The best-fit and first-fit CSVs of a cell must contain the **same sweep column names**. The time grid of the best-fit file is used to cut the measured target traces.
- Sweep numbering should be zero-padded (`sweep00`, `sweep01`, ...), because `compare_fits_features.py` sorts the column names alphabetically.

### 2.5 NWB files (`{ID}_ephys.nwb`)

The files must be in the **classic Allen Institute NWB 1.0 layout** (HDF5), not the modern NWB 2 schema. The paths read by each script:

| HDF5 path | Used for | Fit scripts | Intrinsic script |
|---|---|---|---|
| `acquisition/timeseries/<sweep>/data` | Measured voltage, in **volts** (converted to mV) | yes | yes |
| `acquisition/timeseries/<sweep>/starting_time` (attribute `rate`) | Sampling rate in Hz | yes | yes |
| `acquisition/timeseries/<sweep>/aibs_stimulus_amplitude_pa` | Stimulus amplitude in pA, used for sweep matching | yes | no |
| `acquisition/timeseries/<sweep>/aibs_stimulus_description` | Stimulus description (sweeps containing `LS` are preferred) | yes | no |
| `stimulus/presentation/<sweep>/data` | Stimulus current, in **amperes** | yes | yes |
| `stimulus/presentation/<sweep>/starting_time` (attribute `rate`) | Sampling rate in Hz | yes | yes |
| `stimulus/presentation/<sweep>/aibs_stimulus_amplitude_pa` | Target amplitude for pulse detection | yes | no |

---

## 3. How the fit comparison scripts match fits to measurements

1. **Sweep matching by amplitude.** NWB sweeps are not numbered like the CSV columns (for example `Sweep_24`, `Sweep_34`). Each CSV column is matched to the NWB sweep whose `aibs_stimulus_amplitude_pa` is within **2 pA** of the amplitude in the column name. If several sweeps match, the one whose description contains `LS` (Long Square) is preferred, otherwise the first match is used.
2. **Pulse detection.** The current pulse is located in `stimulus/presentation` as the span between the first and last sample at the target amplitude (tolerance 1e-12 A).
3. **Window alignment.** The fit time window is placed symmetrically around the pulse: `buffer = (fit_duration - pulse_duration) / 2` before and after it. The measured voltage is then interpolated onto the fit time grid.

Assumptions and limitations:

- The experiment must use a **single square current pulse** per sweep (Allen Long Square protocol).
- A sweep with an amplitude of **0 pA** is not reliably supported, because the baseline also matches the target amplitude and the whole sweep would be taken as the pulse.
- The symmetric alignment assumes the fit traces were simulated with equal buffer before and after the pulse. It was checked against one example cell (607124114), where the buffer came out at about 100 ms. If your simulation protocol differs, the alignment will be off.
- Sweeps are classified as suprathreshold if the measured trace exceeds 0 mV anywhere (rough spike heuristic).

---

## 4. How `cell_intrinsic_properties_database.py` works

### 4.1 Sweep classification

It does not use amplitude matching or `aibs_stimulus_description` (which is empty in some files). Instead, each sweep's stimulus is analyzed generically:

1. The baseline is the median of the first `MIN_BASELINE_MS` (100 ms) of the stimulus trace.
2. Samples that deviate from the baseline by more than 2% of the largest deflection form the pulse. The longest contiguous segment is taken.
3. The sweep counts as **long square** only if that segment lasts between 800 and 1200 ms (`LONG_SQUARE_DURATION_RANGE_MS`). Ramps, short pulses and noise are thereby excluded.
4. The amplitude is the median deflection within the pulse. Sweeps below **-5 pA** are hyperpolarizing, above **+5 pA** depolarizing. Sweeps in between are ignored.

Sweeps that raise any error while being read are skipped silently, so check the printed sweep counts per cell.

### 4.2 Passive properties (hyperpolarizing sweeps)

| Property | Method |
|---|---|
| RMP (mV) | eFEL `voltage_base` |
| Rm / Rin (MOhm) | (mean voltage in the last 200 ms of the pulse minus `voltage_base`) / stimulus amplitude |
| tau_m (ms) | eFEL `decay_time_constant_after_stim` |
| Cm (pF) | tau_m / Rin (single-compartment RC approximation), per sweep |

Values are averaged over all hyperpolarizing sweeps of a cell. The reported SD is the population SD across sweeps (`np.std`, `ddof=0`).

### 4.3 Active properties (depolarizing sweeps)

| Property | Method |
|---|---|
| Rheobase (pA) | Smallest tested amplitude with at least one spike (limited by the amplitude steps of the protocol) |
| Threshold (mV) | eFEL `AP_begin_voltage`, pooled over all spikes in all spiking sweeps |
| AP peak (mV) | eFEL `peak_voltage`, pooled the same way |
| Halfwidth (ms) | eFEL `AP_width`, pooled the same way |
| Max. frequency (Hz) | Maximum of eFEL `mean_frequency` over all spiking sweeps. One value per cell, no error bar |

The spike threshold for eFEL is `SPIKE_THRESHOLD_MV` (default -20 mV).

If a cell has no spiking sweep, `Rheobase_pA` is `NaN` but `MaxFreq_Hz` is `0.0`, because the running maximum starts at zero.

### 4.4 Species-level aggregation

The main figure shows one bar per species: the mean of the per-cell values, with the **SEM** (SD of the cell means with `ddof=1`, divided by the square root of the number of cells) as the error bar. With a single cell in a group the SEM is `NaN` and no error bar is drawn. No significance tests are performed in this script.

---

## 5. Configuration

### `compare_fits_RMSE.py`

| Setting | Meaning |
|---|---|
| `DATA_DIR` | Folder with all CSV and NWB files. **change it to your own.** |
| `OUTPUT_DIR` | Output folder, default `./output` (relative to the directory you run the script from) |
| `SPECIES_PREFIX_MAP` | Cell ID prefix to species label |
| `ERROR_METRIC` | `"rmse"` or `"sse"` |
| `MIN_N_FOR_TEST` | Minimum cells per group for a significance test (default 5) |

### `compare_fits_per_feature.py`

This script imports `DATA_DIR`, `OUTPUT_DIR` and `SPECIES_PREFIX_MAP` from `compare_fits_RMSE.py`, so those are set in one place. Its own settings:

| Setting | Meaning |
|---|---|
| `STIM_START_MS`, `STIM_END_MS` | Stimulus window **inside the fit traces** (default 100 ms and 1100 ms, meaning a 1000 ms pulse with a 100 ms buffer before and after, so fit traces of 1200 ms). Change these if your traces use a different layout. |
| `SPIKE_THRESHOLD_MV` | eFEL spike threshold, default -20 mV |
| `FEATURE_NAMES` | eFEL features to compare (voltage_base, steady_state_voltage_stimend, Spikecount, mean_frequency, AP_amplitude, AP_width, time_to_first_spike, ISI_CV, AHP_depth, adaptation_index2) |
| `MIN_N_FOR_TEST` | Same meaning as above |

Feature values are only used if they are valid in **all three** traces (target, first fit, best fit) of a sweep. Features that are undefined in any of them, for example spike features in a subthreshold sweep, are dropped for that sweep.

### `cell_intrinsic_properties_database.py`

| Setting | Meaning |
|---|---|
| `DATA_DIR` | Folder with the `*_ephys.nwb` files. **change it to your own.** Independent of the `DATA_DIR` in `compare_fits.py`. |
| `OUTPUT_DIR` | Output folder, default `./output` |
| `SPIKE_THRESHOLD_MV` | eFEL spike threshold, default -20 mV |
| `LONG_SQUARE_DURATION_RANGE_MS` | Accepted pulse duration for long-square classification, default (800, 1200) |
| `MIN_BASELINE_MS` | Baseline window before the stimulus, default 100 ms |
| `SPECIES_PREFIX_MAP` | Cell ID prefix to species label |

---

## 6. Running

`compare_fits_RMSE.py` and `compare_fits_by_feature.py` must be in the **same folder** (the feature script imports from `compare_fits`). `cell_intrinsic_properties_database.py` can be anywhere. Run each from its folder:

```bash
python compare_fits_RMSE.py
python compare_fits_by_feature.py
python cell_intrinsic_properties_database.py
```

All three are independent of each other's outputs. Note that all use `OUTPUT_DIR = "./output"` by default, so their results end up in the same folder if run from the same directory. The file names do not collide.

---

## 7. Outputs (in `OUTPUT_DIR`)

**`compare_fits_RMSE.py`**

- `summary_per_sweep.csv`: error per cell and sweep, including `is_suprathreshold`
- `summary_per_cell.csv`: error averaged per cell, with absolute and percent improvement
- `stats_report.txt`: descriptive statistics and tests, overall and per species
- `scatter_first_vs_best.png`, `slopegraph_by_species.png`, `boxplot_improvement.png`

**`compare_fits_by_feature.py`**

- `features_long.csv`: every cell x sweep x feature with target, first and best values and normalized errors
- `features_per_cell.csv`: mean normalized feature error per cell
- `features_stats_report.txt`
- `features_scatter_first_vs_best.png`, `features_slopegraph_by_species.png`, `features_boxplot_improvement.png`, `features_per_feature_barplot.png`

**`cell_intrinsic_properties_database.py`**

- `intrinsic_properties_per_cell.csv`: per cell, mean and SD of RMP, Rin, tau_m, Cm, Threshold, AP_peak, Halfwidth, plus `Rheobase_pA` and `MaxFreq_Hz`
- `intrinsic_properties_by_species.csv`: group mean and SEM per species for the eight plotted properties, with `n_cells`
- `intrinsic_properties_by_species_barplot.png`: main figure, 8 panels, one bar per species (mean ± SEM)
- `intrinsic_properties_per_cell_barplot.png`: supplementary figure, one bar per cell (error bars are SD across sweeps or spikes)

The docstring at the top of `cell_intrinsic_properties_database.py` still lists a single `intrinsic_properties_barplot.png`. That file is not produced. The four files above are what the code actually writes.

## 8. Interpretation notes

- **Spike timing and RMSE.** In spiking sweeps, a 1 ms timing offset can produce more than 50 mV of pointwise difference at the spike flank, so the voltage RMSE can be dominated by timing even for a good fit. Compare `error_first` and `error_best` separately for sub- and suprathreshold sweeps (the script prints this split), and consider the feature-based script as the more robust measure.
- **Feature normalization.** Each feature error is divided by the standard deviation of that feature's target values across the dataset, with a lower bound of 10% of its mean absolute target value. With few cells this scale can be unstable. This is an empirical scale, not a trial-to-trial variability estimate.
- **Statistics in the fit scripts.** Tests are paired (first vs. best fit, one value per cell): Wilcoxon signed-rank and paired t-test. Groups with fewer than `MIN_N_FOR_TEST` cells get descriptive statistics and individual values only, with no p-value. Even n = 5 only has adequate power for very large effects. Sweeps are averaged per cell first, so each cell is one observation.
- **Small groups in the intrinsic script.** The SEM is based on the spread of the cell means. With n = 1 it is undefined, and with n = 2 it is valid but unstable, so treat species bars from very few cells as descriptive only.
- **Rin and Cm are estimates.** Rin is a simple Ohmic estimate from the end of the pulse, and Cm assumes a single RC compartment, so Cm will be biased for cells with extended dendrites.

---

## 9. Troubleshooting

| Message | Likely cause |
|---|---|
| `No 'best_fit_traces_*.csv' files found in ...` | `DATA_DIR` in `compare_fits_RMSE.py` is wrong or the file names do not match |
| `No '*_ephys.nwb' files found in ...` | `DATA_DIR` in `cell_intrinsic_properties_database.py` is wrong |
| `[WARNING] Incomplete files for ...` | One of the three required files is missing for that cell |
| `[ERROR] ...: NWB matching failed (No sweep with amplitude ~... pA found.)` | No NWB sweep has a matching amplitude. Check the amplitude in the CSV column name and the units |
| `No stimulus plateau at target amplitude found in ...` | The stimulus trace does not contain the expected amplitude, or the NWB layout differs |
| `No valid feature values extracted ...` | Check `STIM_START_MS`/`STIM_END_MS` and `SPIKE_THRESHOLD_MV` |
| `0 hyperpolarizing, 0 depolarizing long-square sweeps found.` | No sweep has a pulse of 800 to 1200 ms, or sweeps are failing to read. Check `LONG_SQUARE_DURATION_RANGE_MS` and the NWB layout |
| `ModuleNotFoundError: compare_fits` | The feature script is not in the same folder as `compare_fits_RMSE.py` (see section 8) |
