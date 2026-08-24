# Probabilistic Prediction of Laser-Track Geometry

**NSF Future Manufacturing Data Challenge submission by The Regularizers**

Vedangi Bengali and Dhawal Chaudhari  
Texas A&M University

This repository contains an end-to-end, reproducible pipeline for predicting spatially
varying laser-track geometry in directed energy deposition (DED). The approach uses
in-situ melt-pool thermal imagery together with masked scanning electron microscopy
(SEM) measurements of the surrounding substrate.

The model predicts local track geometry as a function of scan position rather than
reducing each track to a single average width. Predictions are probabilistic and include
nine quantiles for the left boundary, right boundary, and width at 0.2 mm intervals.

> **Key result:** Using a feature-selection rule fixed entirely from leave-one-track-out
> (LOTO) validation, the selected thermal-plus-SEM model achieved a held-out Track 21
> MAE of **0.1233 mm**, compared with **0.2907 mm** for a constant-prediction baseline.
> The analysis also found no reliable bin-to-bin predictive skill after track-level
> variation was removed. This null result—and the tests used to validate it—is a central
> finding of the project.

## Contents

- [Project overview](#project-overview)
- [Methodology](#methodology)
- [Evaluation protocol](#evaluation-protocol)
- [Results](#results)
- [Installation](#installation)
- [Reproducing the analysis](#reproducing-the-analysis)
- [Prediction-file format](#prediction-file-format)
- [Repository outputs](#repository-outputs)
- [Limitations and scope](#limitations-and-scope)
- [Reproducibility](#reproducibility)
- [Acknowledgments and disclosure](#acknowledgments-and-disclosure)

## Project overview

The goal of the challenge is to infer the final local geometry of a laser track from a
sequence of thermal images recorded during laser scanning. Our pipeline:

1. extracts spatially resolved track geometry from profilometer height maps;
2. derives melt-pool and thermal-field descriptors from the thermal recordings;
3. constructs substrate descriptors from SEM images after masking the processed track;
4. trains quantile-regression models for local width and boundary position;
5. evaluates generalization with grouped, leave-one-track-out validation; and
6. calibrates predictive intervals using conformalized quantile regression (CQR).

The challenge dataset is available from
[Zenodo (DOI: 10.5281/zenodo.21285367)](https://doi.org/10.5281/zenodo.21285367).

## Methodology

### Geometry representation

For scan position \(x\), the local geometry is represented by:

- left boundary \(y_{\mathrm{left}}(x)\);
- right boundary \(y_{\mathrm{right}}(x)\); and
- width \(w(x) = y_{\mathrm{right}}(x)-y_{\mathrm{left}}(x)\).

The output grid uses 0.2 mm bin centers over the 20–100 mm scan interval. All geometry
values are reported in millimeters.

### Input modalities

- **Thermal:** descriptors of the melt pool and surrounding thermal field.
- **SEM:** substrate-morphology features computed only outside the processed track.
- **Thermal + SEM:** the combined descriptor set.

For SEM feature extraction, the final laser track is masked with an additional 0.30 mm
safety margin. Per-tile normalization reduces brightness-based track fingerprinting, the
burned-in data bar is cropped, and the complete `Scale_` image set is used consistently
across all tracks.

### Probabilistic model

Nine quantile models are fit at quantiles 0.1 through 0.9. Quantile crossing is corrected
by sorting the predicted quantile ladder. CQR uses errors from a held-out track so that
the correction reflects cross-condition generalization error.

## Evaluation protocol

### Held-out test policy

Track 21 is treated as the final held-out test track. It must not be used to select a
feature set, model class, or interval-calibration method.

The primary configuration is selected using only the LOTO folds over the remaining
tracks:

> Select the feature set with the lowest mean LOTO continuous ranked probability score
> (CRPS), and use CQR intervals calibrated on held-out tracks.

This rule selects **`thermal+sem`**. Its Track 21 result is therefore the primary
selection-independent result.

The three plausible LOTO selection rules are not stable:

| LOTO selection rule | Selected features | Track 21 MAE |
|---|---:|---:|
| Lowest mean CRPS | `thermal+sem` | 0.1233 mm |
| Lowest mean MAE | `sem` | 0.1875 mm |
| Best coverage | `sem` | 0.1875 mm |

With only three model-selection folds, the choice of feature set is uncertain. We report
that instability rather than treating the selected configuration as definitive.

### Selection-informed comparison

For transparency, the thermal-only model with raw intervals achieved a Track 21 MAE of
**0.1032 mm**, CRPS of 0.0915 mm, coverage of 0.736, and calibration error of 0.054.
However, no LOTO rule selected this configuration; it was identified after examining
Track 21. These values are therefore descriptive and **not an unbiased final-test
estimate**.

The packaged figures, `predictions_track21.csv`, and `feature_importance.*` use the
thermal-only configuration selected with `--headline thermal`. Both the primary and
selection-informed results are retained in `outputs/metrics.json`.

## Results

### Primary held-out result

| Thermal + SEM with CQR | Track 21 result |
|---|---:|
| MAE | **0.1233 mm** |
| CRPS | 0.0871 mm |
| 80% interval coverage | **0.907** |
| Calibration error | 0.107 |
| Constant-prediction baseline MAE | 0.2907 mm |

The primary model reduces MAE by approximately 58% relative to the constant-prediction
baseline.

### Modality ablation

Each evaluation is repeated with thermal features, SEM features, and both modalities.

| Feature set | Features | Within-track MAE | Within-track \|r\| | LOTO MAE | LOTO CRPS | Track 21 MAE | Track 21 raw coverage |
|---|---:|---:|---:|---:|---:|---:|---:|
| `thermal` | 33 | 0.1087 mm | 0.071 | 0.2383 mm | 0.1881 mm | 0.1032 mm | 0.736 |
| `sem` | 16 | 0.1139 mm | 0.075 | 0.2241 mm | 0.1625 mm | 0.1875 mm | 0.450 |
| `thermal+sem` | 49 | 0.1103 mm | 0.041 | 0.2375 mm | 0.1603 mm | 0.1233 mm | 0.354 |

Predicting each track's own mean width produces a within-track MAE of 0.1044 mm. No
feature set improves on this baseline for local bin-to-bin variation.

### Local-skill investigation

We tested whether the absence of local predictive skill could be explained by common
pipeline failures.

| Possible explanation | Diagnostic | Finding |
|---|---|---|
| Noisy ground truth | Split-half reliability | Unlikely: reliability 0.918–0.956; estimated \|r\| ceiling ≈ 0.97 |
| Thermal misregistration | 51-lag sweep, surrogate null, and transfer test | No reproducible improvement |
| Inappropriate target | Seven geometry definitions | No target produced positive local skill |
| Inappropriate spatial scale | 0.2, 0.4, and 1.0 mm grids | No scale recovered local skill |
| Inappropriate model class | Tree, linear, and hybrid models | No tested model recovered local skill |
| Missing substrate information | Masked SEM ablation | No robust local improvement |

#### Ground-truth reliability

Each bin contains approximately 50 height-map columns. Measuring width independently
from two half-bin samples gives split-half correlations of 0.848–0.916 and
Spearman–Brown reliabilities of 0.918–0.956. Estimated full-bin measurement noise is
27–40 µm, compared with observed width variation of 94–168 µm. The target therefore has
substantial measurable variation that the current feature set does not explain.

#### Registration

A thermal-lag sweep increases the best observed correlation to approximately 0.2, but
the same search over 800 circular-shift surrogates—preserving each series'
autocorrelation—produces a mean maximum correlation of 0.146 and a 95th percentile of
0.204. Optimal lags vary from −19 to −2 bins across tracks. A lag selected from the
other tracks improves only two of four held-out tracks, and all resulting skill scores
remain negative.

#### Target and scale search

The search covers 21 target-scale combinations: width, left and right boundaries,
centerline, left and right edge roughness, and waviness at 0.2, 0.4, and 1.0 mm. Every
combination performs below a constant predictor; the best result is `width_mm@0.2mm`
with a skill score of −0.0421.

#### SEM alignment

Masked SEM features do not recover local skill. SEM-derived and profilometer-derived
widths correlate by only +0.03 to +0.19 at 0.2 mm, and no global shift resolves the
disagreement. Because the released tiles can only be placed on the physical axis to
within several millimeters—roughly ten times the target-grid spacing—the local substrate
hypothesis remains **untested rather than refuted**.

### Model comparison

| Model class | LOTO MAE | Track 21 MAE |
|---|---:|---:|
| Gradient-boosted trees | 0.2317 mm | 0.1082 mm |
| Linear quantile regression | 0.2754 mm | 0.2964 mm |
| Linear anchor + tree residual | 0.2807 mm | 0.3282 mm |

Tree models cannot extrapolate beyond the response range represented in training, which
is especially limiting for the Track 8 fold. The tested linear alternatives extrapolate
but do so in the wrong direction: a trend fitted to two process conditions does not
reliably predict a third. Raw-frame, temporal, hierarchical, and physics-informed models
remain promising directions for future work.

### Uncertainty calibration

| Evaluation | 80% coverage | Mean interval width | Calibration error |
|---|---:|---:|---:|
| LOTO, raw | 0.403 | 0.303 mm | 0.285 |
| LOTO, CQR | 0.693 | 0.759 mm | — |
| Track 21, `thermal+sem` + CQR (primary) | 0.907 | 0.658 mm | 0.107 |
| Track 21, `thermal` raw (selection-informed) | 0.736 | 0.395 mm | 0.054 |

For the thermal-only configuration, nominal coverages of 0.2, 0.4, 0.6, and 0.8 yield
empirical coverages of 0.24, 0.48, 0.57, and 0.74, respectively.

### Extracted ground truth

| Track | Raster NaN fraction | Usable bins | Mean width | Width SD | Crown rise | Reliability |
|---:|---:|---:|---:|---:|---:|---:|
| 8 | 0.369 | 381/400 | 0.849 mm | 0.162 mm | 8.5 µm | 0.949 |
| 10 | 0.516 | 348/400 | 0.516 mm | 0.135 mm | 4.3 µm | 0.920 |
| 14 | 0.511 | 374/400 | 0.471 mm | 0.136 mm | 5.0 µm | 0.956 |
| 21 | 0.555 | 333/400 | 0.325 mm | 0.098 mm | 1.5 µm | 0.918 |

## Installation

### Requirements

- Python 3.9 or newer
- Approximately 1.9 GB of available disk space
- CPU only; no GPU or pretrained model is required

The submitted environment used Python 3.9.6 with:

| Package | Version |
|---|---:|
| NumPy | 2.0.2 |
| SciPy | 1.13.1 |
| pandas | 2.3.3 |
| scikit-learn | 1.6.1 |
| Matplotlib | 3.9.4 |
| Pillow | 11.3.0 |
| h5py | 3.14.0 |
| python-docx | 1.2.0 |
| python-pptx | 1.0.2 |

`python-docx` and `python-pptx` are needed only to regenerate the report and slide deck.

### Environment setup

```bash
python3 -m venv .venv
source .venv/bin/activate

python -m pip install --upgrade pip
python -m pip install \
  numpy scipy pandas scikit-learn matplotlib pillow h5py \
  python-docx python-pptx zenodo_get
```

## Reproducing the analysis

### 1. Obtain the repository and data

```bash
git clone https://github.com/abhishekhanchate/nsf-fmrg-data-challenge.git
cd nsf-fmrg-data-challenge

zenodo_get 10.5281/zenodo.21285367 -o data_zips
mkdir -p data/raw

for archive in thermal height_maps sem; do
  unzip -q "data_zips/${archive}.zip" -d data/raw
done
```

### 2. Confirm the expected data layout

```text
data/raw/
├── thermal/
│   └── Thermal_{8,10,14,21}.mat
├── height_maps/
│   └── Heightmap_{8,10,14,21}.ASC
└── sem/
    └── SEM_{8,10,14,21}/
        ├── Scale_*.tif
        └── PlainImages/
```

`run_pipeline.py` must be in the repository root alongside
`src/nsf_fmrg_data.py`. The organizer-provided loader is used without modification.

### 3. Run the pipeline

```bash
# Optional smoke test
python run_pipeline.py --data-root data/raw --tracks 8 10

# Full analysis
python run_pipeline.py --data-root data/raw --headline thermal
```

Alternatively, open `TheRegularizers_Notebook.ipynb` and run all cells.

The `--headline` option controls which configuration is used in the figures and
`predictions_track21.csv`; it does not change the metrics computed for any
configuration. The packaged artifacts use `thermal`. The unbiased `thermal+sem` result
is always stored under `selection_protocol` in `outputs/metrics.json`.

### 4. Regenerate the report and presentation

```bash
python make_report.py \
  --out outputs \
  --docx TheRegularizers_Report.docx

python make_slides.py \
  --out outputs \
  --pptx TheRegularizers_Presentation.pptx
```

Typical runtime after downloading the data is approximately eight minutes on an Apple
M-series laptop using one CPU process. Peak memory usage is approximately 3 GB. The
download is about 0.67 GB and expands to approximately 1.2 GB.

## Prediction-file format

`TheRegularizers_Predictions.csv` contains machine-readable spatial predictions.

| Column | Description |
|---|---|
| `track_id` | Track identifier: 8, 10, 14, or 21 |
| `x_mm` | Scan-direction position in physical coordinates; 0.2 mm bin centers over 20–100 mm |
| `split` | `LOTO_out_of_fold` or `FINAL_held_out` |
| `descriptor` | Geometry component: `left`, `right`, or `width` |
| `pred_q10_mm` … `pred_q90_mm` | Nine predicted quantiles in millimeters |
| `reference_mm` | Profilometer-derived reference; blank when no usable cross-section exists |
| `reference_valid` | Indicates whether a reference value is available for the bin |

For `LOTO_out_of_fold`, each track is predicted by a model that did not observe that
track during training. `FINAL_held_out` denotes Track 21 predictions from the model fit
on Tracks 8, 10, and 14.

Left and right are \(y\)-positions in the height-map coordinate system, with \(y=0\) at
the first profilometer row. Width equals right minus left. All values are in millimeters.

## Repository outputs

### Submission artifacts

| Path | Description |
|---|---|
| `TheRegularizers_Report.pdf` | Three-page technical report |
| `TheRegularizers_Presentation.pptx` | Presentation deck |
| `TheRegularizers_Notebook.ipynb` | Executable end-to-end notebook |
| `TheRegularizers_Predictions.csv` | Machine-readable spatial predictions |
| `code/` | Pipeline and document-generation scripts |
| `outputs/` | Generated metrics, tables, predictions, and figures |

### Generated analysis files

| File | Contents |
|---|---|
| `metrics.json` | Complete metrics, ablations, selection protocol, reliability, lag analysis, target search, and model comparison |
| `features_and_targets.csv` | Extracted features and geometry for every 0.2 mm bin |
| `predictions_track21.csv` | Per-bin Track 21 quantile predictions |
| `feature_importance.csv` / `.png` | Median-model feature importances |
| `ground_truth_extraction.png` | Height-map segmentation diagnostic |
| `sem_masking.png` | SEM masking diagnostic |
| `measured_width_all_tracks.png` | Measured width functions for all tracks |
| `within_track_cv.png` | Out-of-fold within-track spatial validation |
| `local_skill_search.png` | Target-by-scale local-skill comparison |
| `ablation.png` | Modality ablation |
| `prediction_track_{8,10,14,21}.png` | LOTO-fold and held-out prediction plots |

## Limitations and scope

- **Local predictive skill was not established.** The model predicts cross-condition
  width differences but does not reliably explain fine-scale bin-to-bin variation.
- **Model selection is unstable.** Three LOTO selection criteria choose different
  feature sets because only three selection folds are available.
- **SEM placement is coarse.** Tile localization is several millimeters uncertain, so
  the substrate hypothesis cannot be tested conclusively at a 0.2 mm spatial scale.
- **Track-to-power labels are not assumed.** The dataset paper reports four laser powers
  but does not publish a track-ID-to-power mapping.
- **The model search is not exhaustive.** Raw-frame, temporal, hierarchical, and
  physics-informed approaches remain untested.

### Height-map segmentation note

These samples are bead-on-plate remelt tracks whose crowns rise only a few micrometers.
The original bump-threshold rule detected a bead in only 6 of 400 bins for Track 8. We
therefore segment the track using surface finish, corroborated by the height dome and
thermal melt-pool width. This is the only substantive modification to the supplied data
pipeline; `run_pipeline.py` remains the single source of truth for all reported results.

## Reproducibility

All models use `random_state=0`, no pretrained models or external weights are used, and
the pipeline has no additional stochastic step. Reported floating-point values are
rounded to six decimal places, allowing `metrics.json` to reproduce byte-for-byte across
reruns. Without rounding, different BLAS implementations can differ in the final one or
two units in the last place for correlation coefficients.

## Acknowledgments and disclosure

### External resources

- **Dataset:** NSF Future Manufacturing Data Challenge dataset,
  [DOI 10.5281/zenodo.21285367](https://doi.org/10.5281/zenodo.21285367).
- **Organizer code:** `src/nsf_fmrg_data.py`, used without modification.
- **Libraries:** NumPy, SciPy, pandas, scikit-learn, Matplotlib, Pillow, and h5py.
- **Pretrained models:** None.

### Generative-AI assistance

Anthropic Claude assisted with code implementation, diagnostic plotting, document
generation, and drafting. The authors specified and verified all modeling decisions,
validation procedures, diagnostics, and physical interpretations. Every numerical value
in the report, presentation, and repository documentation is generated programmatically
from `outputs/metrics.json`; no reported result is manually transcribed.

---

For detailed experimental context, see `TheRegularizers_Report.pdf`. For a complete,
executable workflow, see `TheRegularizers_Notebook.ipynb` or run `run_pipeline.py`.
