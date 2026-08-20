# GPMTM — Graph-Prior Multi-Task Model (Anomaly Detection Based on Multi-KPI Spatiotemporal Correlation Analysis)

> **Authors**: S. He &lt;heshiwen@pmlabs.com.cn&gt;, L. Wang &lt;wangliangpeng@pmlabs.com.cn&gt; &nbsp;|&nbsp; **Org**: PML &nbsp;|&nbsp; **Date**: 2026-07

Multi-KPI anomaly detection leveraging knowledge-graph neighborhood features (spatial domain) and temporal correlation analysis, supporting single-device/multi-device anomaly identification across multiple scenarios (http/icmp/rtsp).

## 1. Data Analysis

#### 1. Dataset Overview (Labeled)

The project contains multi-device time series data for three protocols:

| Protocol | Dataset      | Rows   | Devices | Columns | KPI Features | Label |
|----------|--------------|--------|---------|---------|--------------|-------|
| **HTTP** | data_train   | 58,207 | 11      | 80      | 76           | Yes   |
|          | data_test    | 14,556 | 11      | 80      | 76           | Yes   |
|          | data_detect  | 26,023 | 11      | 79      | 76           | No    |
| **ICMP** | data_train   | 44,467 | 8       | 35      | 31           | Yes   |
|          | data_test    | 11,119 | 8       | 35      | 31           | Yes   |
|          | data_detect  | 25,055 | 8       | 34      | 31           | No    |
| **RTSP** | data_train   | 46,261 | 9       | 91      | 87           | Yes   |
|          | data_test    | 11,573 | 9       | 91      | 87           | Yes   |
|          | data_detect  | 24,773 | 9       | 90      | 87           | No    |

#### 2. Data Structure

Each dataset contains:
- **First 3 columns**: Timestamp (`time_value_min`), server address (`n2_addr`), device ID (`imei`)
- **Middle columns**: KPI features (performance metrics prefixed with `kpi_`)
- **Last column (train/test only)**: Label (`label`: 0 = normal, 1 = anomaly)

## 2. Project Structure

```
KG_MTAD/
├── main.py                        # Unified entry point: --mode train|test|detect|crossval
├── config.py                      # CLI argument definitions and default paths
├── train.py                       # Training mode
├── test.py                        # Testing mode
├── detect.py                      # Detection mode (label-free inference)
├── cv.py                          # Cross-validation mode (kfold/holdout/timeseries)
│
├── model/
│   ├── __init__.py
│   └── detector.py                # MultiTSAnomalyDetector model definition
│
├── data/
│   ├── __init__.py
│   ├── loader.py                  # load_data: multi-device grouping + global standardization
│   ├── detect_loader.py           # load_data_detect: label-free data loading
│   ├── sequences.py               # Breakpoint-aware time series sequence creation
│   └── adjacency.py               # Adjacency matrix (knowledge graph) loading
│
├── utils/
│   ├── __init__.py
│   └── transform.py               # De-normalization utilities
│
├── outputs/
│   ├── __init__.py
│   └── save.py                    # Prediction saving + evaluation metrics
│
├── visualization/
│   ├── __init__.py
│   └── graph.py                   # KPI knowledge graph topology visualization
│
├── analysis/
│   ├── __init__.py
│   └── contribution.py            # Anomaly contribution analysis (Top-K metrics)
│
└── README.md
```

## 3. Core Modes

The project supports four operating modes: **Training**, **Testing**, **Detection**, and **Cross-Validation**. Switch between them using the `--mode` parameter.

#### 1. Training Mode `--mode train`

Learns KPI temporal patterns from the training set while evaluating model performance on the test set.

**Features**:
- Train and save model weights
- Full evaluation on the test set
- Output all prediction results and visualization files

**Example**:
```bash
python main.py --mode train --scenario http --epochs 100 --batch_size 256 --hidden_dim 128
```

#### 2. Testing Mode `--mode test`

Loads a pre-trained model and performs inference and evaluation on the test set without updating model weights.

**Features**:
- Requires a pre-trained model (`model/{scenario}/anomaly_detector.ckpt`)
- Outputs full prediction results and evaluation metrics
- Used to verify model generalization ability

**Example**:

```bash
python main.py --mode test --scenario http
```

#### 3. Detection Mode `--mode detect`

Performs anomaly detection on unlabeled data. The output row count matches the input exactly.

**Features**:
- Does not rely on label columns; pure inference mode
- Rows that cannot be inferred due to missing data are left blank
- Output row count matches input exactly

**Example**:
```bash
python main.py --mode detect --scenario http
```

#### 4. Cross-Validation Mode `--mode crossval`

Loads merged labeled data, automatically splits train/validation sets internally, and outputs complete results from the best-fold model.

**Features**:
- Supports three split methods: kfold / holdout / timeseries
- Outputs per-fold training and validation metrics (mean ± std)
- Uses the best-fold model for full validation set prediction

**Split Method Comparison**:

| Method | CLI Parameter | Train/Val Ratio | Description |
|--------|--------------|-----------------|-------------|
| **kfold** | `--cv_method kfold --cv_folds 5` | 80% / 20% (default 5 folds) | Stratified K-fold CV, shuffled and evenly split |
| **holdout** | `--cv_method holdout --cv_split_ratio 0.8` | 80% / 20% | Single stratified random split |
| **timeseries** | `--cv_method timeseries --cv_split_ratio 0.8` | 80% / 20% | Per-device chronological split, no shuffling |

**Examples**:

```bash
# kfold cross-validation
python main.py --mode crossval --scenario http --cv_method kfold --cv_folds 5

# holdout (save best model)
python main.py --mode crossval --scenario icmp --cv_method holdout --cv_save_model

# timeseries 70/30
python main.py --mode crossval --scenario rtsp --cv_method timeseries --cv_split_ratio 0.7
```

### Input/Output File Summary

| Filename | train | crossval | test | detect | Description |
|----------|:-----:|:--------:|:----:|:------:|-------------|
| **Input Files** ||||||
| `data_train.csv` | ✅ | — | — | ✅(scaler) | Training data (labeled) |
| `data_test.csv` | ✅ | — | ✅ | — | Test data (labeled) |
| `data_withlabels.csv` | — | ✅ | — | — | Merged labeled data (CV) |
| `data_detect.csv` | — | — | — | ✅ | Data for detection (unlabeled) |
| `adjmat.csv` | ✅ | ✅ | ✅ | ✅ | Knowledge graph adjacency matrix |
| **Output Files** ||||||
| `system_predictions.csv` | ✅ | ✅ | ✅ | — | System-level predictions (timestamp+IP+IMEI+label) |
| `metrics_predictions.csv` | ✅ | ✅ | ✅ | — | KPI-level binary predictions |
| `evaluation_results.csv` | ✅ | ✅ | ✅ | — | Evaluation metrics (Acc/Recall/F1/AUC) |
| `kpi_topology_graph.png` | ✅ | ✅ | ✅ | ✅ | KPI topology graph (weighted in GAT mode) |
| `anomaly_contribution.csv` | ✅ | ✅ | ✅ | ✅ | Top-K contribution metric analysis |
| `learned_adjacency.csv/.npy` | ✅(GAT) | ✅(GAT) | — | — | Adjacency matrix learned by GAT |
| `cv_summary.csv` | — | ✅ | — | — | CV summary (per-fold metrics + mean/std) |
| `data_detect_output.csv` | — | — | — | ✅ | Detection results (original data + labels) |
| `anomaly_detector.ckpt.*` | ✅ | ✅(optional) | — | — | Model weights (`--cv_save_model`) |
| `model_meta.json` | ✅ | ✅(optional) | — | — | Model hyperparameter configuration |

**Notes**:
- `✅(GAT)` means output only in GAT mode (`--adj_type gat`)
- `✅(optional)` means requires the `--cv_save_model` parameter to output
- All output file paths: `outputs/{scenario}/`, model file paths: `model/{scenario}/`

## 4. Ablation Studies

Use the `--adj_type` parameter to control the adjacency matrix type and evaluate the contribution of the knowledge graph spatial branch to model performance.

#### 1. Supported Types

| `--adj_type` | Description | Ablation Purpose |
|---|---|---|
| `original` (default) | Original knowledge graph adjacency matrix | Baseline experiment |
| `zero` | All-zero adjacency matrix | Disable spatial branch, retain temporal LSTM only |
| `one` | All-one adjacency matrix | Fully connected graph, verify topology importance |
| `random` | Random adjacency matrix (fixed seed=42) | Noise baseline |
| `gat` | GAT-based dynamic adjacency weight learning | Learn topology via Graph Attention Networks |

#### 2. Run Commands

```bash
# Experiment 1: Original adjacency matrix (baseline)
python main.py --mode crossval -s http --cv_method holdout

# Experiment 2: All-zero adjacency matrix (disable spatial branch)
python main.py --mode crossval -s http --cv_method holdout --adj_type zero

# Experiment 3: All-one adjacency matrix (fully connected)
python main.py --mode crossval -s http --cv_method holdout --adj_type one

# Experiment 4: Random adjacency matrix (noise baseline)
python main.py --mode crossval -s http --cv_method holdout --adj_type random
```

#### 3. Default Paths

- **Data directory**: `project_root/data/{scenario}/` (contains CSV and adjacency matrix)
- **Model directory**: `project_root/model/{scenario}/` (weights .ckpt + model_meta.json)
- **Results directory**: `project_root/outputs/{scenario}/`