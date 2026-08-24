NSF FUTURE MANUFACTURING DATA CHALLENGE — SUBMISSION
====================================================================
Team:       The Regularizers
Members:    Vedangi Bengali, Dhawal Chaudhari
University: Texas A&M University

FILES
  TheRegularizers_Report.pdf         3-page report (Arial 10pt, 1in margins)
  TheRegularizers_Presentation.pptx  slide deck
  TheRegularizers_Notebook.ipynb     executable end-to-end notebook
  TheRegularizers_Predictions.csv    machine-readable spatial predictions
  README.txt                         this file
  code/                              run_pipeline.py + document generators
  outputs/                           all metrics and figures

SOFTWARE AND VERSIONS
  Python 3.9.6 (CPU only, no GPU required)
  numpy 2.0.2 | scipy 1.13.1 | pandas 2.3.3 | scikit-learn 1.6.1
  matplotlib 3.9.4 | pillow 11.3.0 | h5py 3.14.0
  python-docx 1.2.0 | python-pptx 1.0.2  (document generation only)

INSTALLATION AND EXECUTION
  python3 -m venv .venv && source .venv/bin/activate
  pip install numpy scipy pandas scikit-learn matplotlib pillow h5py \
              python-docx python-pptx
  pip install zenodo_get && zenodo_get 10.5281/zenodo.21285367 -o data_zips
  mkdir -p data/raw
  for z in thermal height_maps sem; do unzip -q data_zips/$z.zip -d data/raw; done
  cp code/*.py .
  python run_pipeline.py --data-root data/raw --headline thermal
  # or open TheRegularizers_Notebook.ipynb and run all cells

INPUT / OUTPUT FOLDERS
  input :  data/raw/thermal, data/raw/height_maps, data/raw/sem
  output:  outputs/  (metrics.json, figures, CSVs, prediction file)

EXECUTION TIME AND HARDWARE
  ~8 minutes total on an Apple M-series laptop, single process, CPU only.
  Peak memory ~3 GB. Download is 0.67 GB; extracted data ~1.2 GB.

RANDOM SEEDS AND PRETRAINED MODELS
  Every model uses random_state=0. No pretrained models or external
  weights are used. The pipeline has no other stochastic step, and all
  reported floats are rounded to 6 dp, so metrics.json reproduces
  byte-identically across reruns and machines.

EXTERNAL DATA, CODE AND MODELS
  Data:  Zenodo 10.5281/zenodo.21285367 (challenge dataset) only.
  Code:  src/nsf_fmrg_data.py, the organizers' loader from the challenge
         repository, used unmodified. Everything else is our own, built
         on numpy / scipy / pandas / scikit-learn / matplotlib / pillow.
  No external pretrained models.

GENERATIVE AI USE
  Anthropic Claude was used to assist with code implementation,
  diagnostic plotting, document generation and drafting of the report,
  slides and this file. All modelling choices, validation design,
  diagnostics and physical interpretation were specified and verified by
  the authors. Every number in the report, slides and this README is
  generated programmatically from outputs/metrics.json by the submitted
  code; none is transcribed by hand.

PREDICTION FILE FORMAT (TheRegularizers_Predictions.csv)
  track_id        8, 10, 14 or 21
  x_mm            position along the scan direction, actual part
                  coordinates, 0.2 mm bin centres over 20-100 mm
  split           LOTO_out_of_fold = predicted by a model that never saw
                  that track; FINAL_held_out = track 21 from the model
                  trained on tracks 8/10/14
  descriptor      left | right | width   (geometry component)
  pred_q10_mm ... pred_q90_mm   nine predicted quantiles, millimetres
  reference_mm    our profilometer-derived reference (blank where the
                  height map yielded no usable cross-section)
  reference_valid whether a reference value exists for that bin
  Cross-track convention: left/right are y positions in height-map
  coordinates, y=0 at the first profilometer row; width = right - left.
  All units are millimetres.

====================================================================
The remainder of this file is generated from outputs/metrics.json and
documents the results in detail.
====================================================================

Probabilistic prediction of laser-track geometry in directed energy
deposition from in-situ melt-pool thermal imaging and masked SEM
substrate morphology, validated leave-one-track-out with track 21 as the
held-out final test.  Dataset DOI 10.5281/zenodo.21285367.

## How track 21 was used — read this first

Track 21 is the held-out final test, so it may be touched **once**. Choosing a feature set or an interval type by looking at its track-21 score turns it into a validation set and biases the headline number. We therefore fixed a selection rule that uses **only** the leave-one-track-out folds:

> feature set = lowest mean CRPS over the leave-one-track-out folds; intervals = conformalised (CQR), calibrated on held-out tracks. Both decided without reference to track 21.

That rule selects **`thermal+sem`**. Its track-21 score is the primary, unbiased result:

| Primary (unbiased) — thermal+sem + CQR | Value |
|---|---|
| MAE | **0.1233 mm** |
| CRPS | 0.0871 mm |
| 80% interval coverage | **0.907** (nominal 0.80) |
| Calibration error | 0.107 |
| Constant-prediction baseline | 0.2907 mm (2.4× worse) |

**The selection is not stable.** The three sensible LOTO rules pick different feature sets:

| LOTO rule | picks | its track-21 MAE |
|---|---|---|
| lowest LOTO mean CRPS | `thermal+sem` | 0.1233 mm |
| lowest LOTO mean MAE | `sem` | 0.1875 mm |
| best LOTO coverage | `sem` | 0.1875 mm |

With only three cross-validation folds, model selection is itself unreliable — that is a finding, not an inconvenience.

**Reported for transparency, but NOT unbiased:** the `thermal`-only configuration with raw intervals scores MAE **0.1032 mm**, CRPS 0.0915 mm, coverage 0.736, calibration error 0.054 — better than the primary result on every count. But `thermal` is chosen by **no** LOTO criterion; it was identified as best *after* seeing track 21. The figures, `predictions_track21.csv` and `feature_importance.*` in this package use that configuration (`--headline thermal`), so treat those track-21 numbers as validation-informed. Both sets are in `metrics.json`.

## Modality ablation

Every evaluation is run three times. SEM features come only from substrate, with the processed track masked out plus a 0.30 mm margin.

| Feature set | n | Within-track MAE | within \|r\| | LOTO MAE | LOTO CRPS | FINAL MAE | FINAL cov. |
|---|---|---|---|---|---|---|---|
| `thermal` | 33 | 0.1087 mm | 0.071 | 0.2383 mm | 0.1881 | 0.1032 mm | 0.736 |
| `sem` | 16 | 0.1139 mm | 0.075 | 0.2241 mm | 0.1625 | 0.1875 mm | 0.450 |
| `thermal+sem` ← | 49 | 0.1103 mm | 0.041 | 0.2375 mm | 0.1603 | 0.1233 mm | 0.354 |

Predicting each track's own mean width gives a within-track MAE of 0.1044 mm — **no feature set beats it.**

## The null result, and the alternatives we eliminated

No configuration shows local (bin-to-bin) predictive skill. A negative finding is only worth anything if the boring explanations were ruled out, so each was tested directly.

| Could the null be caused by…? | Tested how | Answer |
|---|---|---|
| A noisy ground truth | split-half reliability | **No** — 0.918–0.956, ceiling \|r\| ≈ 0.97 |
| Misaligned thermal data | 51-lag sweep + surrogate null + transfer test | **No** |
| The wrong target | 7 definitions | **No** |
| The wrong spatial scale | 0.2 / 0.4 / 1.0 mm | **No** |
| The wrong model class | trees / linear / hybrid | **No** |
| A missing sensor | SEM ablation with masking | **No** |

**Reliability.** Splitting the ~50 height-map columns per bin into two halves and measuring width from each gives split-half correlation 0.848–0.916 (Spearman-Brown 0.918–0.956). Full-bin measurement noise is 27–40 µm against a real width variation of 94–168 µm, i.e. 4–8% of the variance. **The target is well measured — there is large headroom that the model does not reach, not a ceiling it has hit.**

**Registration.** Sweeping the thermal lag raises the best correlation to ~0.2, but the same search over 800 circular-shift surrogates (which preserve each series' autocorrelation) already returns 0.146 on average (95th pct 0.204). Optimal lags disagree across tracks (-19 to -2 bins), and a lag chosen on the other tracks improves 2/4 held-out tracks; all skills remain negative.

**Target and scale.** 21 combinations — width, left and right boundary, centre-line, edge roughness each side, waviness — at 0.2, 0.4 and 1.0 mm, on every track. All score below a constant predictor; the best is `width_mm@0.2mm` at -0.0421.

**SEM.** Adding masked substrate features does not recover local skill. The SEM and the profilometer both measure how wide the track is, so they should agree; at 0.2 mm they correlate only +0.03 to +0.19, and no global shift fixes it. The released tiles cannot be placed on the physical axis to better than a few millimetres — 10× coarser than the target grid, so the substrate hypothesis is **untested rather than refuted**.

## Model bake-off

Trees cannot predict outside their training range, which is what breaks the track-8 fold, so a model that extrapolates ought to win. It does not.

| Model class | LOTO MAE | FINAL MAE |
|---|---|---|
| gradient-boosted trees | 0.2317 mm | 0.1082 mm |
| linear quantile regression | 0.2754 mm | 0.2964 mm |
| linear anchor + tree residual | 0.2807 mm | 0.3282 mm |

Linear models do extrapolate — in the wrong direction. A trend through two process conditions does not predict a third. This is not a proof that no model can do better: raw-frame, temporal, hierarchical and physics-informed formulations remain untested.

## Uncertainty

Nine quantile models (q = 0.1 … 0.9), crossing removed by sorting the ladder. Conformalised quantile regression (CQR) is calibrated on a **held-out track**, so the correction reflects cross-condition error.

| | 80% coverage | Interval width | Calibration error |
|---|---|---|---|
| LOTO, raw | 0.403 | 0.303 mm | 0.285 |
| LOTO, CQR | 0.693 | 0.759 mm | — |
| FINAL `thermal+sem`, CQR (primary) | 0.907 | 0.658 mm | 0.107 |
| FINAL `thermal`, raw (selection-informed) | 0.736 | 0.395 mm | 0.054 |

Coverage curve for the `thermal` configuration, nominal 0.2/0.4/0.6/0.8 → empirical 0.24/0.48/0.57/0.74.

## Ground truth extracted from the height maps

| Track | Raster NaN | Usable bins | Mean width | Width SD | Crown rise | Reliability |
|---|---|---|---|---|---|---|
| 8 | 0.369 | 381/400 | 0.849 mm | 0.162 mm | 8.5 µm | 0.949 |
| 10 | 0.516 | 348/400 | 0.516 mm | 0.135 mm | 4.3 µm | 0.920 |
| 14 | 0.511 | 374/400 | 0.471 mm | 0.136 mm | 5.0 µm | 0.956 |
| 21 | 0.555 | 333/400 | 0.325 mm | 0.098 mm | 1.5 µm | 0.918 |

## Reproducing from scratch

Python 3.9+, ~1.9 GB free disk (0.67 GB download). Runtime after download: **~8 min**.

```bash
git clone https://github.com/abhishekhanchate/nsf-fmrg-data-challenge.git
cd nsf-fmrg-data-challenge
python3 -m venv .venv && source .venv/bin/activate
pip install numpy scipy pandas scikit-learn matplotlib pillow h5py \
            python-docx python-pptx

pip install zenodo_get
zenodo_get 10.5281/zenodo.21285367 -o data_zips
mkdir -p data/raw
for z in thermal height_maps sem; do unzip -q data_zips/$z.zip -d data/raw; done

cp code/*.py .
python run_pipeline.py --data-root data/raw --tracks 8 10   # smoke test
python run_pipeline.py --data-root data/raw --headline thermal   # full run
python make_report.py  --out outputs --docx TheRegularizers_Report.docx
python make_slides.py  --out outputs --pptx TheRegularizers_Presentation.pptx
python make_readme.py  --out outputs --md   README.txt
```

`--headline` selects which configuration the figures and `predictions_track21.csv` use; it does not change any metric. The packaged outputs use `thermal`. The primary unbiased result (`thermal+sem`) is in `metrics.json` under `selection_protocol` regardless of the flag.

### Expected data layout

```text
data/raw/thermal/Thermal_{8,10,14,21}.mat
data/raw/height_maps/Heightmap_{8,10,14,21}.ASC
data/raw/sem/SEM_{8,10,14,21}/            (Scale_*.tif used; PlainImages/ present)
```

`run_pipeline.py` must sit in the repo root next to `src/nsf_fmrg_data.py`.

### Determinism

All models use `random_state=0` and the pipeline performs no other stochastic operation. Reported floats are rounded to 6 decimal places so `metrics.json` is byte-identical across reruns; without rounding, different BLAS builds differ in the last 1–2 ulp (~1e-16) on correlation coefficients.

## Outputs

| File | Contents |
|---|---|
| `metrics.json` | every metric, the ablation, selection protocol, reliability, lag check, skill search and bake-off |
| `features_and_targets.csv` | all features + extracted geometry per 0.2 mm bin |
| `predictions_track21.csv` | per-bin quantile ladder for the final test |
| `feature_importance.{csv,png}` | importances of the median model |
| `ground_truth_extraction.png` | how the bead is segmented |
| `sem_masking.png` | how the track is masked out of each SEM tile |
| `measured_width_all_tracks.png` | measured w(x) for all four tracks |
| `within_track_cv.png` | out-of-fold within-track spatial CV |
| `local_skill_search.png` | 21 target × scale combinations |
| `ablation.png` | three-way modality comparison |
| `prediction_track_{8,10,14,21}.png` | LOTO folds and the final test |

## Scope notes

- **SEM is used with the track masked out.** The tiles image the region containing the finished track, so it is located and excluded with a 0.30 mm margin before any feature is computed. Tiles are normalised per-tile so brightness cannot fingerprint a track, and the `Scale_` set is used for all four tracks (the only complete set — track 14's `PlainImages` holds twelve `Scale_` copies and one `Plain_` file) with the burned-in data bar cropped.
- **No track-ID → laser-power mapping is claimed.** The dataset paper states four powers were investigated but does not publish which track is which.
- **`run_pipeline.py` is the single source of truth.** The one substantive change to the supplied pipeline was the ground-truth width extraction: these are bead-on-plate remelt tracks, so the crown rises only a few microns and the original bump-threshold rule found a bead in 6 of 400 bins on track 8. It was replaced with segmentation on surface finish, corroborated by the height dome and by the thermal melt-pool width.
