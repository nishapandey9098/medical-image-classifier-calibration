# Knowing When Not to Predict: Calibration, Uncertainty and Selective Referral for Medical Image Classifiers

[![Open In Colab](https://colab.research.google.com/assets/colab-badge.svg)](https://colab.research.google.com/github/nishapandey9098/medical-image-classifier-calibration/blob/main/Project2_Calibration_Referral.ipynb)

> A small, reproducible study on public low-resolution datasets. **Not validated for clinical use.**

## 1. Research questions

1. Are the confidence scores of small medical image classifiers honest (calibrated)?
2. Do temperature scaling, MC dropout or a deep ensemble make them more honest, and give a better "I am not sure" signal?
3. If the model refers its least confident cases to a clinician, how much more reliable are the cases it still answers?

## 2. Data and inputs

- **Datasets:** PneumoniaMNIST (chest X-ray, 2 classes) and RetinaMNIST (retinal fundus photo, 5 DR grades) from [MedMNIST](https://medmnist.com/), 64×64 pixels, official train / validation / test split.
- **Models from Project 1** ([link to your Project 1 repository]): for each dataset the setting with the best mean *validation* score, five models (seeds 0 to 4): PneumoniaMNIST = `pretrained+aug`, RetinaMNIST = `pretrained+aug+clahe`. Their saved predictions (raw scores) for the validation and test images are in `inputs_from_project1/`.
- **New models trained here:** 5 ResNet-18 models per dataset with a dropout layer (p = 0.3) in front of the last layer, same setting, hyperparameters and seeds as the Project 1 models.

## 3. Methods

| Method | Description |
|---|---|
| single | One Project 1 model (5 seeds) |
| single + TS | The same, after temperature scaling |
| dropout (1 pass) | A dropout-trained model used normally (control) |
| MC dropout | A dropout-trained model, 30 random passes averaged |
| MC dropout + TS | The same, after temperature scaling |
| ensemble | The 5 Project 1 models averaged (one ensemble) |
| ensemble + TS | The same, after temperature scaling |

- **Temperature scaling:** one temperature T per model (or ensemble), fitted on the validation set by minimising the negative log-likelihood. The test set was never used for fitting.
- **Calibration scores (test set):** expected calibration error (ECE, 15 equal-width bins), negative log-likelihood (NLL) and Brier score. Also reliability diagrams.
- **Referral quality:** risk-coverage curves (error rate on the answered cases as the model answers fewer cases), the area under that curve (AURC), accuracy at 90 / 80 / 70 / 50% coverage (and QWK for RetinaMNIST), and a comparison of three uncertainty scores (highest probability, entropy, mutual information).
- **Threshold transfer:** the confidence threshold was chosen on the validation set to answer 90 / 80 / 70% of the validation cases and then applied to the test set.
- **Statistics:** tables show mean ± standard deviation over seeds. Differences between methods (A minus B) use a **paired bootstrap over the test images** (1000 redraws, seeds averaged within each redraw); the interval is the 2.5th to 97.5th percentile. This captures the luck of *which test images we have*, not a different patient population.

## 4. Results

### PneumoniaMNIST

#### Table 1: all methods (mean ± standard deviation over seeds)

| method | models | ECE | NLL | Brier | accuracy | AUC | AURC |
|---|---|---|---|---|---|---|---|
| single | 5 | 0.058 ± 0.011 | 0.314 ± 0.067 | 0.143 ± 0.021 | 0.913 ± 0.013 | 0.984 ± 0.005 | 0.018 ± 0.006 |
| single + TS | 5 | 0.052 ± 0.010 | 0.279 ± 0.063 | 0.139 ± 0.021 | 0.913 ± 0.013 | 0.984 ± 0.005 | 0.018 ± 0.006 |
| dropout (1 pass) | 5 | 0.061 ± 0.017 | 0.304 ± 0.076 | 0.146 ± 0.036 | 0.909 ± 0.026 | 0.986 ± 0.004 | 0.018 ± 0.008 |
| MC dropout | 5 | 0.059 ± 0.017 | 0.301 ± 0.076 | 0.145 ± 0.036 | 0.909 ± 0.026 | 0.986 ± 0.004 | 0.018 ± 0.008 |
| MC dropout + TS | 5 | 0.056 ± 0.020 | 0.278 ± 0.079 | 0.143 ± 0.036 | 0.909 ± 0.026 | 0.986 ± 0.004 | 0.018 ± 0.008 |
| ensemble | 1 | 0.048 | 0.251 | 0.130 | 0.912 | 0.988 | 0.015 |
| ensemble + TS | 1 | 0.045 | 0.253 | 0.130 | 0.912 | 0.988 | 0.015 |

#### Table 2: differences A minus B with 95% bootstrap intervals over the test images

Each cell shows the difference A minus B with its 95% bootstrap interval. Lower is better for ECE, NLL, Brier and AURC; higher is better for accuracy.

| comparison (A vs B) | what it tests | ECE | NLL | Brier | AURC | accuracy | A clearly better on | A clearly worse on |
|---|---|---|---|---|---|---|---|---|
| single + TS vs single | temperature scaling on a single model | -0.007 [-0.010, -0.003] | -0.035 [-0.049, -0.022] | -0.004 [-0.005, -0.002] | +0.000 [+0.000, +0.000] | +0.000 [+0.000, +0.000] | ECE, NLL, Brier | - |
| ensemble vs single | ensemble of 5 vs a single model | -0.011 [-0.018, -0.001] | -0.063 [-0.081, -0.045] | -0.014 [-0.017, -0.010] | -0.003 [-0.004, -0.002] | -0.002 [-0.010, +0.006] | ECE, NLL, Brier, AURC | - |
| ensemble + TS vs single + TS | ensemble vs single model, both temperature-scaled | -0.007 [-0.013, +0.005] | -0.026 [-0.037, -0.016] | -0.010 [-0.013, -0.006] | -0.003 [-0.004, -0.002] | -0.002 [-0.010, +0.006] | NLL, Brier, AURC | - |
| ensemble + TS vs ensemble | temperature scaling on the ensemble | -0.003 [-0.005, +0.003] | +0.002 [+0.001, +0.003] | +0.000 [+0.000, +0.000] | +0.000 [-0.000, +0.000] | +0.000 [+0.000, +0.000] | - | NLL, Brier |
| MC dropout vs dropout (1 pass) | 30 MC passes vs 1 pass of the same dropout-trained models | -0.001 [-0.004, +0.001] | -0.003 [-0.005, -0.002] | -0.000 [-0.000, -0.000] | +0.000 [-0.000, +0.000] | -0.000 [-0.001, +0.001] | NLL, Brier | - |
| MC dropout vs single | MC dropout vs a standard single model | +0.001 [-0.005, +0.007] | -0.013 [-0.028, +0.000] | +0.002 [-0.004, +0.008] | -0.000 [-0.002, +0.001] | -0.004 [-0.011, +0.002] | - | - |
| MC dropout + TS vs MC dropout | temperature scaling on MC dropout | -0.003 [-0.006, -0.001] | -0.023 [-0.034, -0.014] | -0.002 [-0.003, -0.001] | -0.000 [-0.000, +0.000] | +0.000 [+0.000, +0.000] | ECE, NLL, Brier | - |
| MC dropout vs ensemble | MC dropout vs the ensemble | +0.012 [+0.002, +0.019] | +0.049 [+0.033, +0.068] | +0.016 [+0.010, +0.022] | +0.002 [+0.001, +0.004] | -0.003 [-0.013, +0.006] | - | ECE, NLL, Brier, AURC |

#### Accuracy on the answered cases at different coverage levels

| method | answer 100% | answer 90% | answer 80% | answer 70% | answer 50% |
|---|---|---|---|---|---|
| single | 0.913 | 0.946 | 0.967 | 0.982 | 0.991 |
| single + TS | 0.913 | 0.946 | 0.967 | 0.982 | 0.991 |
| dropout (1 pass) | 0.909 | 0.943 | 0.968 | 0.982 | 0.996 |
| MC dropout | 0.909 | 0.943 | 0.968 | 0.982 | 0.996 |
| MC dropout + TS | 0.909 | 0.943 | 0.968 | 0.982 | 0.996 |
| ensemble | 0.912 | 0.950 | 0.968 | 0.986 | 0.997 |
| ensemble + TS | 0.912 | 0.950 | 0.968 | 0.984 | 0.997 |

#### AURC by uncertainty score (lower is better)

| method | highest probability | (negative) entropy | (negative) mutual information |
|---|---|---|---|
| single | 0.018 | 0.018 | – |
| MC dropout | 0.018 | 0.018 | 0.019 |
| ensemble | 0.015 | 0.015 | 0.017 |

– = not defined for a single deterministic model (no random passes or ensemble members).

#### Threshold chosen on validation, applied to test

| method | target coverage (validation) | real coverage (test) | accuracy of answered cases | accuracy of referred cases | accuracy without referral |
|---|---|---|---|---|---|
| single | 0.900 | 0.797 | 0.968 | 0.699 | 0.913 |
| single | 0.800 | 0.696 | 0.982 | 0.758 | 0.913 |
| single | 0.700 | 0.606 | 0.989 | 0.796 | 0.913 |
| MC dropout | 0.900 | 0.798 | 0.967 | 0.679 | 0.909 |
| MC dropout | 0.800 | 0.704 | 0.981 | 0.740 | 0.909 |
| MC dropout | 0.700 | 0.628 | 0.991 | 0.772 | 0.909 |
| ensemble | 0.900 | 0.796 | 0.968 | 0.693 | 0.912 |
| ensemble | 0.800 | 0.694 | 0.988 | 0.738 | 0.912 |
| ensemble | 0.700 | 0.607 | 0.992 | 0.788 | 0.912 |

### RetinaMNIST

#### Table 1: all methods (mean ± standard deviation over seeds)

| method | models | ECE | NLL | Brier | accuracy | QWK | AURC |
|---|---|---|---|---|---|---|---|
| single | 5 | 0.132 ± 0.026 | 1.239 ± 0.051 | 0.591 ± 0.011 | 0.553 ± 0.011 | 0.579 ± 0.043 | 0.251 ± 0.016 |
| single + TS | 5 | 0.092 ± 0.012 | 1.158 ± 0.011 | 0.574 ± 0.008 | 0.553 ± 0.011 | 0.579 ± 0.043 | 0.251 ± 0.016 |
| dropout (1 pass) | 5 | 0.120 ± 0.042 | 1.229 ± 0.058 | 0.581 ± 0.019 | 0.539 ± 0.028 | 0.547 ± 0.071 | 0.256 ± 0.017 |
| MC dropout | 5 | 0.112 ± 0.036 | 1.205 ± 0.048 | 0.579 ± 0.016 | 0.541 ± 0.023 | 0.550 ± 0.065 | 0.256 ± 0.016 |
| MC dropout + TS | 5 | 0.095 ± 0.030 | 1.178 ± 0.051 | 0.573 ± 0.011 | 0.541 ± 0.023 | 0.550 ± 0.065 | 0.256 ± 0.016 |
| ensemble | 1 | 0.042 | 1.044 | 0.526 | 0.595 | 0.588 | 0.216 |
| ensemble + TS | 1 | 0.057 | 1.063 | 0.529 | 0.598 | 0.589 | 0.216 |

#### Table 2: differences A minus B with 95% bootstrap intervals over the test images

Each cell shows the difference A minus B with its 95% bootstrap interval. Lower is better for ECE, NLL, Brier and AURC; higher is better for accuracy.

| comparison (A vs B) | what it tests | ECE | NLL | Brier | AURC | accuracy | A clearly better on | A clearly worse on |
|---|---|---|---|---|---|---|---|---|
| single + TS vs single | temperature scaling on a single model | -0.040 [-0.044, -0.019] | -0.081 [-0.105, -0.059] | -0.017 [-0.022, -0.012] | -0.000 [-0.001, +0.001] | +0.000 [+0.000, +0.000] | ECE, NLL, Brier | - |
| ensemble vs single | ensemble of 5 vs a single model | -0.090 [-0.098, -0.037] | -0.195 [-0.226, -0.166] | -0.065 [-0.070, -0.060] | -0.035 [-0.047, -0.022] | +0.042 [+0.019, +0.065] | ECE, NLL, Brier, AURC, accuracy | - |
| ensemble + TS vs single + TS | ensemble vs single model, both temperature-scaled | -0.035 [-0.053, -0.000] | -0.095 [-0.118, -0.073] | -0.044 [-0.053, -0.036] | -0.035 [-0.047, -0.022] | +0.045 [+0.022, +0.067] | ECE, NLL, Brier, AURC, accuracy | - |
| ensemble + TS vs ensemble | temperature scaling on the ensemble | +0.014 [-0.020, +0.036] | +0.018 [+0.008, +0.029] | +0.003 [+0.001, +0.006] | +0.000 [-0.002, +0.002] | +0.003 [-0.005, +0.012] | - | NLL, Brier |
| MC dropout vs dropout (1 pass) | 30 MC passes vs 1 pass of the same dropout-trained models | -0.007 [-0.016, +0.006] | -0.023 [-0.036, -0.012] | -0.002 [-0.004, -0.001] | -0.000 [-0.002, +0.002] | +0.003 [-0.003, +0.009] | NLL, Brier | - |
| MC dropout vs single | MC dropout vs a standard single model | -0.020 [-0.037, +0.004] | -0.034 [-0.088, +0.016] | -0.012 [-0.030, +0.006] | +0.005 [-0.012, +0.020] | -0.011 [-0.031, +0.008] | - | - |
| MC dropout + TS vs MC dropout | temperature scaling on MC dropout | -0.017 [-0.027, -0.005] | -0.027 [-0.039, -0.017] | -0.006 [-0.008, -0.004] | +0.000 [-0.001, +0.001] | +0.000 [+0.000, +0.000] | ECE, NLL, Brier | - |
| MC dropout vs ensemble | MC dropout vs the ensemble | +0.070 [+0.023, +0.085] | +0.161 [+0.109, +0.217] | +0.053 [+0.035, +0.071] | +0.040 [+0.020, +0.059] | -0.053 [-0.085, -0.022] | - | ECE, NLL, Brier, AURC, accuracy |

#### Accuracy on the answered cases at different coverage levels

| method | answer 100% | answer 90% | answer 80% | answer 70% | answer 50% |
|---|---|---|---|---|---|
| single | 0.553 | 0.581 | 0.613 | 0.648 | 0.735 |
| single + TS | 0.553 | 0.581 | 0.614 | 0.646 | 0.737 |
| dropout (1 pass) | 0.539 | 0.566 | 0.598 | 0.631 | 0.726 |
| MC dropout | 0.541 | 0.565 | 0.598 | 0.631 | 0.729 |
| MC dropout + TS | 0.541 | 0.564 | 0.599 | 0.631 | 0.727 |
| ensemble | 0.595 | 0.625 | 0.650 | 0.689 | 0.790 |
| ensemble + TS | 0.598 | 0.625 | 0.650 | 0.696 | 0.790 |

#### QWK on the answered cases

| method | answer 100% | answer 90% | answer 80% | answer 70% | answer 50% |
|---|---|---|---|---|---|
| single | 0.579 | 0.604 | 0.638 | 0.662 | 0.719 |
| single + TS | 0.579 | 0.607 | 0.641 | 0.661 | 0.725 |
| dropout (1 pass) | 0.547 | 0.578 | 0.607 | 0.632 | 0.681 |
| MC dropout | 0.550 | 0.575 | 0.610 | 0.632 | 0.698 |
| MC dropout + TS | 0.550 | 0.575 | 0.613 | 0.636 | 0.694 |
| ensemble | 0.588 | 0.664 | 0.678 | 0.683 | 0.766 |
| ensemble + TS | 0.589 | 0.666 | 0.678 | 0.699 | 0.754 |

#### AURC by uncertainty score (lower is better)

| method | highest probability | (negative) entropy | (negative) mutual information |
|---|---|---|---|
| single | 0.251 | 0.253 | – |
| MC dropout | 0.256 | 0.258 | 0.295 |
| ensemble | 0.216 | 0.219 | 0.268 |

– = not defined for a single deterministic model (no random passes or ensemble members).

#### Threshold chosen on validation, applied to test

| method | target coverage (validation) | real coverage (test) | accuracy of answered cases | accuracy of referred cases | accuracy without referral |
|---|---|---|---|---|---|
| single | 0.900 | 0.880 | 0.589 | 0.287 | 0.553 |
| single | 0.800 | 0.787 | 0.619 | 0.307 | 0.553 |
| single | 0.700 | 0.703 | 0.647 | 0.329 | 0.553 |
| MC dropout | 0.900 | 0.881 | 0.572 | 0.322 | 0.541 |
| MC dropout | 0.800 | 0.785 | 0.606 | 0.309 | 0.541 |
| MC dropout | 0.700 | 0.695 | 0.629 | 0.343 | 0.541 |
| ensemble | 0.900 | 0.890 | 0.632 | 0.295 | 0.595 |
| ensemble | 0.800 | 0.800 | 0.650 | 0.375 | 0.595 |
| ensemble | 0.700 | 0.645 | 0.717 | 0.373 | 0.595 |

### Temperature scaling of the single models

| dataset | seed | T | ECE val raw | ECE val TS | ECE test raw | ECE test TS | NLL test raw | NLL test TS | acc test raw | acc test TS |
|---|---|---|---|---|---|---|---|---|---|---|
| PneumoniaMNIST | 0 | 1.327 | 0.011 | 0.009 | 0.065 | 0.056 | 0.334 | 0.273 | 0.913 | 0.913 |
| PneumoniaMNIST | 1 | 0.929 | 0.014 | 0.010 | 0.050 | 0.052 | 0.328 | 0.345 | 0.905 | 0.905 |
| PneumoniaMNIST | 2 | 1.202 | 0.005 | 0.006 | 0.047 | 0.039 | 0.228 | 0.204 | 0.929 | 0.929 |
| PneumoniaMNIST | 3 | 1.257 | 0.013 | 0.016 | 0.073 | 0.064 | 0.405 | 0.339 | 0.897 | 0.897 |
| PneumoniaMNIST | 4 | 1.269 | 0.012 | 0.011 | 0.057 | 0.047 | 0.275 | 0.234 | 0.921 | 0.921 |
| RetinaMNIST | 0 | 1.341 | 0.090 | 0.097 | 0.132 | 0.080 | 1.275 | 1.169 | 0.557 | 0.557 |
| RetinaMNIST | 1 | 1.086 | 0.111 | 0.082 | 0.098 | 0.082 | 1.180 | 1.158 | 0.540 | 0.540 |
| RetinaMNIST | 2 | 1.283 | 0.121 | 0.121 | 0.167 | 0.109 | 1.239 | 1.146 | 0.547 | 0.547 |
| RetinaMNIST | 3 | 1.471 | 0.107 | 0.074 | 0.145 | 0.091 | 1.303 | 1.147 | 0.570 | 0.570 |
| RetinaMNIST | 4 | 1.097 | 0.080 | 0.099 | 0.117 | 0.098 | 1.198 | 1.168 | 0.550 | 0.550 |

![Reliability diagrams, PneumoniaMNIST](figures/reliability_pneumoniamnist.png)
![Reliability diagrams, RetinaMNIST](figures/reliability_retinamnist.png)
![Calibration error of every method](figures/ece_by_method.png)
![Risk-coverage curves](figures/risk_coverage.png)
![One raw single model (seed 0): confidence, validation and test reliability](figures/step4_raw_model.png)
![The same model before and after temperature scaling](figures/step5_temperature_scaling.png)

## 5. Key findings

- Single models were overconfident on RetinaMNIST (mean confidence 0.683 against accuracy 0.553; test ECE 0.132) and, less so, on PneumoniaMNIST (0.966 against 0.913; ECE 0.058).
- Temperature scaling lowered the test ECE of single models by 0.040 on RetinaMNIST (95% CI −0.044 to −0.019) but only by 0.007 on PneumoniaMNIST (−0.010 to −0.003), where the validation images were easier than the test images (ECE of the raw models 0.005 to 0.014 on validation against 0.047 to 0.073 on test); for two classes it cannot change the referral ranking, so AURC stayed at 0.018.
- The ensemble of five models was the best-calibrated method on both datasets (lowest ECE, NLL and Brier score); against a single model on RetinaMNIST it changed ECE by −0.090 (−0.098 to −0.037), NLL by −0.195 (−0.226 to −0.166) and accuracy by +0.042 (+0.019 to +0.065), and temperature scaling on top of it gave no clear further gain (NLL was slightly worse on both datasets).
- MC dropout improved NLL over one pass of the same dropout models (RetinaMNIST −0.023, CI −0.036 to −0.012), but it was not clearly better or worse than a standard single model on either dataset and was clearly worse than the ensemble (RetinaMNIST ECE +0.070, CI +0.023 to +0.085).
- Answering only the 80% most confident cases raised the accuracy of the ensemble from 0.912 to 0.968 on PneumoniaMNIST and from 0.595 to 0.650 on RetinaMNIST (QWK 0.588 to 0.678); a threshold chosen on the validation set to answer 80% of the cases answered only 69% of the PneumoniaMNIST test cases, but 80% on RetinaMNIST.

## 6. Limitations

- The test sets are small (624 and 400 images), so calibration scores are noisy; the bootstrap intervals show how noisy. ECE depends on the binning and is itself biased on small samples; NLL and Brier score are reported as well.
- Temperature scaling is fitted on a small validation set (524 and 120 images). If the validation set differs from the test set (as it appeared to for PneumoniaMNIST in Project 1), the fitted temperature may not transfer.
- The ensemble is a single ensemble of 5 models, so it has no seed-to-seed variability; the dropout models used one dropout rate and one number of passes (not tuned).
- The Project 1 models were chosen by validation loss, which selected very early epochs on RetinaMNIST (Project 1 report).
- Referral risk uses the 0/1 error. For RetinaMNIST (ordered grades) this ignores how far a wrong grade is from the true grade.
- Low-resolution images, two public datasets, one architecture, no external validation.
- Not clinically validated and not suitable for medical decisions.

## 7. How to reproduce

1. Click the **Open in Colab** badge and choose a GPU runtime (needed only for the MC dropout step).
2. Point `P1_DIR` in cell 1c to the Project 1 inputs: either your own Project 1 output folder, or the `inputs_from_project1` folder of this repository (upload it to Colab or Drive).
3. Set `QUICK_TEST = False` and choose Runtime, Run all (about 15 minutes).

Exact library versions and settings are in `environment.json`.

## 8. Repository contents

| File | What it is |
|---|---|
| `Project2_Calibration_Referral.ipynb` | Full code and explanations, with the outputs of my run |
| `results_tables.md`, `table1_*.csv`, `table2_*.csv` | Method comparison and paired differences |
| `selective_*.csv`, `uncertainty_scores_*.csv`, `threshold_transfer_*.csv` | Referral analyses |
| `step5_temperature_scaling_single_models.csv` | Fitted temperatures and before/after scores |
| `figures/` | Plots used in this README |
| `inputs_from_project1/` | `results.csv` and the 10 prediction files used from Project 1 |
| `environment.json` | Software versions and settings |

## 9. References and acknowledgements

- C. Guo, G. Pleiss, Y. Sun, K. Q. Weinberger. *On calibration of modern neural networks.* ICML, 2017 (temperature scaling, ECE).
- B. Lakshminarayanan, A. Pritzel, C. Blundell. *Simple and scalable predictive uncertainty estimation using deep ensembles.* NeurIPS, 2017.
- Y. Gal, Z. Ghahramani. *Dropout as a Bayesian approximation: representing model uncertainty in deep learning.* ICML, 2016 (MC dropout).
- Y. Geifman, R. El-Yaniv. *Selective classification for deep neural networks.* NeurIPS, 2017 (risk-coverage curves).
- Y. Ovadia et al. *Can you trust your model's uncertainty? Evaluating predictive uncertainty under dataset shift.* NeurIPS, 2019.
- G. W. Brier. *Verification of forecasts expressed in terms of probability.* Monthly Weather Review, 1950.
- B. Efron. *Bootstrap methods: another look at the jackknife.* Annals of Statistics, 1979.
- J. Yang et al. *MedMNIST v2: a large-scale lightweight benchmark for 2D and 3D biomedical image classification.* Scientific Data, 2023.
- D. Kermany et al. *Identifying medical diagnoses and treatable diseases by image-based deep learning.* Cell, 2018 (source of PneumoniaMNIST).
- R. Liu et al. *DeepDRiD: Diabetic Retinopathy, Grading and Image Quality Estimation Challenge.* Patterns, 2022 (source of RetinaMNIST).
- Please also check the license terms of each dataset on the MedMNIST website.

## 10. Author

Nisha Pandey · [your email] · [link to your CV or LinkedIn]
