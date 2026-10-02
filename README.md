# Fit comparison scripts: first fit vs. best fit

Two scripts that compare a first and a best (optimized) NEURON-style model fit against measured voltage traces from an NWB file, separately for human and mouse cells.

| Script | What it compares |
|---|---|
| `compare_fits.py` | Raw voltage error (RMSE or SSE) per sweep, averaged per cell |
| `compare_fits_features.py` | Normalized electrophysiological feature error (via eFEL) per cell |


---

## 1. Software requirements

- **Python 3.8 or newer** (3.9+ recommended)
- Python packages:

| Package | Needed by | Notes |
|---|---|---|
| `numpy` | both | |
| `pandas` | both | Named aggregation (`.agg(name=(col, func))`) is used |
| `scipy` | both | Wilcoxon and paired t-test |
| `matplotlib` | both | |
| `seaborn` | both | Version **0.12 or newer** (`sns.boxplot(..., hue=..., legend=False)` is used) |
| `h5py` | both | Reads the NWB 1.0 files directly |
| `efel` | `compare_fits_features.py` only | The calls used are `efel.set_setting("Threshold", ...)` and `efel.get_feature_values(..., raise_warnings=False)` |

Install:

```bash
pip install numpy pandas scipy matplotlib seaborn h5py efel
```

`pynwb` is **not** needed. The classic Allen Institute NWB 1.0 format cannot be read by pynwb, so the scripts use `h5py`.

---

## 2. Input files

All input files must be in **one folder**, set via `DATA_DIR` in `compare_fits.py`. For every cell with ID `{ID}`, three files are required:

```
best_fit_traces_{ID}.csv
first_fit_traces_{ID}.csv
{ID}_ephys.nwb
```

Cells are discovered automatically from the `best_fit_traces_*.csv` files. A cell is skipped with a warning if any of its three files is missing, or if the NWB sweep matching fails.

### 2.1 Cell ID and species

The species is read from the part of the cell ID **before the first underscore**:

| Cell ID | Prefix | Species |
|---|---|---|
| `h_001` | `h` | Human |
| `m_101` | `m` | Mouse |

Other prefixes are labeled `Unknown`. Extend `SPECIES_PREFIX_MAP` in `compare_fits.py` if you use different prefixes. Note that the plots only color `Human` and `Mouse` (see `SPECIES_PALETTE`), so `Unknown` cells appear in the tables and statistics but not in the scatter plot.

### 2.2 Fit CSV files (`best_fit_traces_*.csv`, `first_fit_traces_*.csv`)

- One column `time_ms` plus one column per sweep.
- Sweep column names must follow the pattern `sweepNN_amp<sign><value>nA_mV`, for example:

```
time_ms, sweep00_amp-0.110nA_mV, sweep01_amp+0.090nA_mV, sweep02_amp+0.150nA_mV
```

- The stimulus amplitude (in nA) is parsed from the column name with the regex `amp([+-]?[0-9.]+)nA` and used to find the matching NWB sweep. The amplitude in the name must therefore be correct.
- Voltages in **mV**, time in **ms**, on a **uniform time grid**.
- The best-fit and first-fit CSVs of a cell must contain the **same sweep column names**. The time grid of the best-fit file is used to cut the measured target traces.
- Sweep numbering should be zero-padded (`sweep00`, `sweep01`, ...), because `compare_fits_features.py` sorts the column names alphabetically.

### 2.3 NWB files (`{ID}_ephys.nwb`)

The files must be in the **classic Allen Institute NWB 1.0 layout** (HDF5), not the modern NWB 2 schema. The scripts read these paths:

| HDF5 path | Used for |
|---|---|
| `acquisition/timeseries/<sweep>/data` | Measured voltage, in **volts** (converted to mV) |
| `acquisition/timeseries/<sweep>/starting_time` (attribute `rate`) | Sampling rate in Hz |
| `acquisition/timeseries/<sweep>/aibs_stimulus_amplitude_pa` | Stimulus amplitude in pA, used for sweep matching |
| `acquisition/timeseries/<sweep>/aibs_stimulus_description` | Stimulus description (sweeps containing `LS` are preferred) |
| `stimulus/presentation/<sweep>/data` | Stimulus current, in **amperes** |
| `stimulus/presentation/<sweep>/starting_time` (attribute `rate`) | Sampling rate in Hz |
| `stimulus/presentation/<sweep>/aibs_stimulus_amplitude_pa` | Target amplitude for pulse detection |

---

## 3. How the scripts match fits to measurements

1. **Sweep matching by amplitude.** NWB sweeps are not numbered like the CSV columns (for example `Sweep_24`, `Sweep_34`). Each CSV column is matched to the NWB sweep whose `aibs_stimulus_amplitude_pa` is within **2 pA** of the amplitude in the column name. If several sweeps match, the one whose description contains `LS` (Long Square) is preferred, otherwise the first match is used.
2. **Pulse detection.** The current pulse is located in `stimulus/presentation` as the span between the first and last sample at the target amplitude (tolerance 1e-12 A).
3. **Window alignment.** The fit time window is placed symmetrically around the pulse: `buffer = (fit_duration - pulse_duration) / 2` before and after it. The measured voltage is then interpolated onto the fit time grid.

Assumptions and limitations that follow from this:

- The experiment must use a **single square current pulse** per sweep (Allen Long Square protocol).
- A sweep with an amplitude of **0 pA** is not reliably supported, because the baseline also matches the target amplitude and the whole sweep would be taken as the pulse.
- The symmetric alignment assumes the fit traces were simulated with equal buffer before and after the pulse. It was checked against one example cell (607124114), where the buffer came out at about 100 ms. If your simulation protocol differs, the alignment will be off.
- Sweeps are classified as suprathreshold if the measured trace exceeds 0 mV anywhere (rough spike heuristic).

---

## 4. Configuration

### `compare_fits.py`

| Setting | Meaning |
|---|---|
| `DATA_DIR` | Folder with all CSV and NWB files. **Currently a hard-coded Windows path, change it to your own.** |
| `OUTPUT_DIR` | Output folder, default `./output` (relative to the directory you run the script from) |
| `SPECIES_PREFIX_MAP` | Cell ID prefix to species label |
| `ERROR_METRIC` | `"rmse"` or `"sse"` |
| `MIN_N_FOR_TEST` | Minimum cells per group for a significance test (default 5) |

### `compare_fits_features.py`

This script imports `DATA_DIR`, `OUTPUT_DIR` and `SPECIES_PREFIX_MAP` from `compare_fits.py`, so those are set in one place. Its own settings:

| Setting | Meaning |
|---|---|
| `STIM_START_MS`, `STIM_END_MS` | Stimulus window **inside the fit traces** (default 100 ms and 1100 ms, meaning a 1000 ms pulse with a 100 ms buffer before and after, so fit traces of 1200 ms). Change these if your traces use a different layout. |
| `SPIKE_THRESHOLD_MV` | eFEL spike threshold, default -20 mV |
| `FEATURE_NAMES` | eFEL features to compare (voltage_base, steady_state_voltage_stimend, Spikecount, mean_frequency, AP_amplitude, AP_width, time_to_first_spike, ISI_CV, AHP_depth, adaptation_index2) |
| `MIN_N_FOR_TEST` | Same meaning as above |

Feature values are only used if they are valid in **all three** traces (target, first fit, best fit) of a sweep. Features that are undefined in any of them, for example spike features in a subthreshold sweep, are dropped for that sweep.

---

## 5. Running

Both scripts must be in the **same folder** (the feature script imports from `compare_fits`). Run them from that folder:

```bash
python compare_fits.py
python compare_fits_features.py
```

They are independent of each other, and the feature script does not need the outputs of the first one.

---

## 6. Outputs (in `OUTPUT_DIR`)

**`compare_fits.py`**

- `summary_per_sweep.csv`: error per cell and sweep, including `is_suprathreshold`
- `summary_per_cell.csv`: error averaged per cell, with absolute and percent improvement
- `stats_report.txt`: descriptive statistics and tests, overall and per species
- `scatter_first_vs_best.png`, `slopegraph_by_species.png`, `boxplot_improvement.png`

**`compare_fits_features.py`**

- `features_long.csv`: every cell x sweep x feature with target, first and best values and normalized errors
- `features_per_cell.csv`: mean normalized feature error per cell
- `features_stats_report.txt`
- `features_scatter_first_vs_best.png`, `features_slopegraph_by_species.png`, `features_boxplot_improvement.png`, `features_per_feature_barplot.png`

---

## 7. Interpretation notes

- **Spike timing and RMSE.** In spiking sweeps, a 1 ms timing offset can produce more than 50 mV of pointwise difference at the spike flank, so the voltage RMSE can be dominated by timing even for a good fit. Compare `error_first` and `error_best` separately for sub- and suprathreshold sweeps (the script prints this split), and consider the feature-based script as the more robust measure.
- **Feature normalization.** Each feature error is divided by the standard deviation of that feature's target values across the dataset, with a lower bound of 10% of its mean absolute target value. With few cells this scale can be unstable. This is an empirical scale, not a trial-to-trial variability estimate.
- **Statistics.** Tests are paired (first vs. best fit, one value per cell): Wilcoxon signed-rank and paired t-test. Groups with fewer than `MIN_N_FOR_TEST` cells get descriptive statistics and individual values only, with no p-value. Even n = 5 only has adequate power for very large effects.
- **Aggregation.** Sweeps are averaged per cell first, so the statistics treat each cell as one observation.

---

## 8. Using the English file names

The English translations are named `compare_fits_en.py` and `compare_fits_features_en.py`. The feature script still contains `from compare_fits import ...`. Either:

- rename `compare_fits_en.py` to `compare_fits.py`, or
- change the import line in `compare_fits_features_en.py` to `from compare_fits_en import ...`.

---

## 9. Troubleshooting

| Message | Likely cause |
|---|---|
| `No 'best_fit_traces_*.csv' files found in ...` | `DATA_DIR` is wrong or the file names do not match |
| `[WARNING] Incomplete files for ...` | One of the three required files is missing for that cell |
| `[ERROR] ...: NWB matching failed (No sweep with amplitude ~... pA found.)` | No NWB sweep has a matching amplitude. Check the amplitude in the CSV column name and the units |
| `No stimulus plateau at target amplitude found in ...` | The stimulus trace does not contain the expected amplitude, or the NWB layout differs |
| `No valid feature values extracted ...` | Check `STIM_START_MS`/`STIM_END_MS` and `SPIKE_THRESHOLD_MV` |
| `ModuleNotFoundError: compare_fits` | The feature script is not in the same folder as `compare_fits.py` (see section 8) |
