# Adversarial Robustness in IoT Intrusion Detection Systems

A multi-layer adversarial defense framework for IoT network intrusion detection, trained and evaluated on the CICIoT2023 dataset. This project investigates the vulnerability of deep neural network classifiers to adversarial attacks and proposes a structured defense pipeline to improve robustness across eight IoT attack categories.

---

## Table of Contents

- [Overview](#overview)
- [Dataset](#dataset)
- [Project Structure](#project-structure)
- [Requirements](#requirements)
- [Installation](#installation)
- [Pipeline](#pipeline)
  - [Phase 1: Baseline Model Training](#phase-1-baseline-model-training)
  - [Phase 2: Adversarial Attack Evaluation](#phase-2-adversarial-attack-evaluation)
  - [Phase 3: Multi-Layer Defense](#phase-3-multi-layer-defense)
  - [Phase 4: SHAP Explainability](#phase-4-shap-explainability)
- [Results](#results)
- [Attack Categories](#attack-categories)
- [Defense Architecture](#defense-architecture)
- [File Reference](#file-reference)

---

## Overview

IoT devices are increasingly targeted by sophisticated network attacks, yet their resource constraints make comprehensive onboard security impractical. This project applies machine learning-based anomaly detection with a four-layer adversarial defense strategy to classify malicious IoT traffic and resist evasion attacks.

The pipeline covers:

1. Training a deep neural network (DNN) baseline classifier on imbalanced IoT traffic data
2. Evaluating model vulnerability to FGSM, BIM, PGD, and C&W adversarial attacks
3. Applying a multi-layer defense including adversarial training, input transformations, ensemble voting, and adversarial detection
4. Using SHAP to explain model decisions and identify vulnerable features

---

## Dataset

**CICIoT2023** — Canadian Institute for Cybersecurity IoT Attack Dataset

- 63 CSV files of labeled network traffic
- 39 raw features per sample
- 8 mapped attack categories (see [Attack Categories](#attack-categories))
- Download: [https://www.unb.ca/cic/datasets/iotdataset-2023.html](https://www.unb.ca/cic/datasets/iotdataset-2023.html)

Place all 63 CSV files in a single directory before running the pipeline.

---

## Project Structure

```
project/
│
├── combine_first_15.py          # Combine first 15 CSV files alphabetically
├── combine_random_csvs.py       # Randomly select N CSVs with guaranteed class coverage
│
├── train_model.py               # Phase 1: Baseline DNN training
├── phase2_adversarial.py        # Phase 2: Adversarial attack evaluation
├── phase3_defense.py            # Phase 3: Multi-layer defense training and evaluation
├── phase4_shap.py               # Phase 4: SHAP explainability analysis
│
├── combined_output.csv          # Combined dataset (generated)
│
├── baseline_model.keras         # Saved baseline model (generated)
├── scaler.pkl                   # Feature scaler (generated)
├── imputer.pkl                  # Missing value imputer (generated)
├── label_encoder.pkl            # Label encoder (generated)
├── X_test.npy                   # Test features (generated)
├── y_test.npy                   # Test labels (generated)
│
├── X_adv_cw.npy                 # C&W adversarial examples (generated)
├── X_cw_subset.npy              # C&W clean subset (generated)
├── y_cw_subset.npy              # C&W labels (generated)
│
├── adv_model_seed0.weights.h5   # Ensemble model 1 (generated)
├── adv_model_seed42.weights.h5  # Ensemble model 2 (generated)
├── adv_model_seed123.weights.h5 # Ensemble model 3 (generated)
├── adversarial_detector.pkl     # Adversarial detector (generated)
│
├── phase2_results.csv           # Phase 2 attack results (generated)
├── eval_a_layers123.csv         # Phase 3 layers 1-3 results (generated)
├── eval_c_full_pipeline.csv     # Phase 3 full pipeline results (generated)
│
└── shap_plots/                  # SHAP plots directory (generated)
    ├── plot1_baseline_clean.png
    ├── plot2_baseline_adv.png
    ├── plot3_adv_clean.png
    ├── plot4_adv_adv.png
    ├── plot5_vulnerability.png
    └── plot6_defense_stability.png
```

---

## Requirements

- Python 3.12
- TensorFlow 2.16.2
- CUDA-compatible GPU (recommended)

### Key Dependencies

```
tensorflow==2.16.2
scikit-learn
imbalanced-learn
adversarial-robustness-toolbox[tensorflow]
shap==0.46.0
pandas
numpy<2.0.0
scipy
joblib
matplotlib
```

---

## Installation

```bash
# Clone the repository
git clone https://github.com/yourusername/iot-adversarial-defense.git
cd iot-adversarial-defense

# Create and activate a virtual environment
python -m venv .env
source .env/bin/activate        # Mac/Linux
.env\Scripts\activate           # Windows

# Install dependencies
pip install tensorflow==2.16.2
pip install scikit-learn imbalanced-learn pandas numpy==1.26.4 scipy joblib matplotlib
pip install adversarial-robustness-toolbox[tensorflow]
pip install shap==0.46.0
```

> Note: shap 0.51+ requires numpy>=2.0 which conflicts with TensorFlow 2.16.2. Use shap==0.46.0 with numpy==1.26.4.

---

## Pipeline

Run the scripts in order. All scripts must be run from the same working directory where the CSV files and generated artifacts are stored.

### Phase 1: Baseline Model Training

**Script:** `train_model.py`

Trains a deep neural network on the combined IoT traffic dataset with class imbalance handling.

```bash
# Step 1: Combine datasets
# Randomly select 15 with guaranteed class coverage
python combine_random_csvs.py

# Step 2: Train the baseline model
python train_model.py
```

**What it does:**
- Maps 39 raw labels into 8 attack categories
- Splits data 60% train / 10% validation / 40% test (validation carved before resampling)
- Undersamples DDoS (capped at 500K) and DoS (capped at 300K)
- Oversamples Brute Force and Web Attack to 30K each using RandomOverSampler
- Trains a 256-128-64-32 DNN with BatchNormalization and Dropout
- Applies class weights to handle remaining imbalance
- Saves model, scaler, imputer, label encoder, and test set

**Model Architecture:**
```
Input (39 features)
Dense(256, relu) → BatchNorm → Dropout(0.3)
Dense(128, relu) → BatchNorm → Dropout(0.3)
Dense(64,  relu) → BatchNorm → Dropout(0.3)
Dense(32,  relu)
Dense(8,   softmax)
```

**Class Weights:**
| Class | Weight |
|---|---|
| Benign | 5.35 |
| Brute Force | 10.00 |
| DDoS | 1.00 |
| DoS | 1.00 |
| Mirai | 2.23 |
| Reconnaissance | 8.51 |
| Spoofing | 10.00 |
| Web Attack | 10.00 |

**Outputs:** `baseline_model.keras`, `scaler.pkl`, `imputer.pkl`, `label_encoder.pkl`, `X_test.npy`, `y_test.npy`

---

### Phase 2: Adversarial Attack Evaluation

**Script:** `phase2_adversarial.py`

Evaluates the baseline model's vulnerability to four adversarial attack methods across five epsilon values.

```bash
python phase2_adversarial.py
```

**What it does:**
- Samples 1,000 per class (8,000 total) from the test set for evaluation
- Wraps the model using the Adversarial Robustness Toolbox (ART)
- Runs four attacks: FGSM, BIM, PGD, and C&W
- Reports accuracy and attack success rate per attack and epsilon value
- Saves adversarial examples for use in Phase 3

**Attacks evaluated:**
| Attack | Epsilons | Batch Size |
|---|---|---|
| FGSM | 0.01, 0.05, 0.10, 0.15, 0.20 | 512 |
| BIM | 0.01, 0.05, 0.10, 0.15, 0.20 | 512 |
| PGD | 0.01, 0.05, 0.10, 0.15, 0.20 | 256 |
| C&W | N/A (L2 norm) | 32 |

**Outputs:** `phase2_results.csv`, `X_adv_cw.npy`, `X_cw_subset.npy`, `y_cw_subset.npy`

---

### Phase 3: Multi-Layer Defense

**Script:** `phase3_defense.py`

Trains and evaluates a four-layer adversarial defense system.

```bash
python phase3_defense.py
```

**Defense Layers:**

| Layer | Method | Description |
|---|---|---|
| 1 | Adversarial Training | Three ensemble models trained on clean + FGSM + PGD examples using curriculum learning (eps: 0.01 → 0.05 → 0.10) |
| 2 | Input Transformations | Feature squeezing (12-bit) and Gaussian smoothing to reduce adversarial perturbations |
| 3 | Ensemble Voting | Majority vote across 3 adversarially trained models (seeds: 0, 42, 123) |
| 4 | Adversarial Detector | Gradient Boosting classifier trained to distinguish clean from adversarial inputs |

**Evaluation A** — Layers 1+2+3 against FGSM, BIM, PGD, C&W

**Evaluation C** — Full pipeline (all 4 layers) against BIM and C&W, with flagged samples routed for human review

**Outputs:** `adv_model_seed{0,42,123}.weights.h5`, `adversarial_detector.pkl`, `eval_a_layers123.csv`, `eval_c_full_pipeline.csv`

---

### Phase 4: SHAP Explainability

**Script:** `phase4_shap.py`

Applies SHAP (SHapley Additive exPlanations) to interpret model decisions on clean and adversarial data.

```bash
python phase4_shap.py
```

**What it does:**
- Uses DeepExplainer with a 200-sample background set (25 per class)
- Explains 400 samples (50 per class) from the test set
- Computes SHAP values for baseline and adversarially trained models on clean and adversarial data
- Generates 6 plots comparing feature importance and vulnerability

**Plots generated:**
| Plot | Description |
|---|---|
| plot1_baseline_clean.png | Baseline model feature importance on clean data |
| plot2_baseline_adv.png | Baseline model feature importance on adversarial data |
| plot3_adv_clean.png | Defended model feature importance on clean data |
| plot4_adv_adv.png | Defended model feature importance on adversarial data |
| plot5_vulnerability.png | Top 15 most vulnerable features (baseline model) |
| plot6_defense_stability.png | Feature importance shift comparison: baseline vs defended |

**Outputs:** 6 PNG plots, SHAP value arrays saved as `.npy` files

---

### SHAP Analysis (Phase 4)

- Most important features: `rst_flag_number`, `Protocol Type`, `Header_Length`, `Time_To_Live`, `syn_flag_number`
- Most vulnerable features: `rst_flag_number`, `Header_Length`, `psh_flag_number`
- Defense stability improvement: **17.4%** reduction in average feature importance shift under attack

---

## Attack Categories

| Category | Raw Labels |
|---|---|
| Benign | BENIGN |
| DDoS | DDOS-* (16 subtypes) |
| DoS | DOS-* (4 subtypes) |
| Mirai | MIRAI-*, BACKDOOR_MALWARE |
| Reconnaissance | RECON-*, VULNERABILITYSCAN |
| Spoofing | DNS_SPOOFING, MITM-ARPSPOOFING |
| Brute Force | DICTIONARYBRUTEFORCE |
| Web Attack | SQLINJECTION, XSS, COMMANDINJECTION, UPLOADING_ATTACK, BROWSERHIJACKING |

---

## Defense Architecture

```
Incoming Traffic
      │
      ▼
[Layer 4] Adversarial Detector (Gradient Boosting)
      │
      ├── Adversarial? ──► Flag for Human Review
      │
      └── Clean?
            │
            ▼
      [Layer 2] Input Transformations
      (Feature Squeezing + Gaussian Smoothing)
            │
            ▼
      [Layer 3] Ensemble Voting
      (3 Adversarially Trained DNNs → Majority Vote)
            │
            ▼
      [Layer 1] Adversarially Trained Models
      (Curriculum: ε=0.01 → 0.05 → 0.10)
            │
            ▼
      Final Classification
```

---

## File Reference

| File | Phase | Description |
|---|---|---|
| `combined_output.csv` | Pre-processing | Combined dataset from selected CSVs |
| `baseline_model.keras` | Phase 1 | Trained baseline DNN |
| `scaler.pkl` | Phase 1 | StandardScaler fitted on training data |
| `imputer.pkl` | Phase 1 | SimpleImputer fitted on training data |
| `label_encoder.pkl` | Phase 1 | LabelEncoder for 8 attack categories |
| `X_test.npy` | Phase 1 | Scaled test features (3.97M rows) |
| `y_test.npy` | Phase 1 | Test labels |
| `phase2_results.csv` | Phase 2 | Attack accuracy and success rates |
| `X_adv_cw.npy` | Phase 2 | C&W adversarial examples |
| `adv_model_seed*.weights.h5` | Phase 3 | Ensemble model weights |
| `adversarial_detector.pkl` | Phase 3 | Trained adversarial detector |
| `eval_a_layers123.csv` | Phase 3 | Layers 1-3 evaluation results |
| `eval_c_full_pipeline.csv` | Phase 3 | Full pipeline evaluation results |
| `shap_plots/` | Phase 4 | SHAP visualization plots |