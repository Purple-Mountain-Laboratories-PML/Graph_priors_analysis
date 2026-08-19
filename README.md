# GPMTM — 知识图谱先验多任务模型（多指标时空域关联异常检测）

> **Author**: jswanglp &lt;wangliangpeng@pmlabs.com&gt; &nbsp;|&nbsp; **Org**: PML &nbsp;|&nbsp; **Date**: 2026-07

融合多指标知识图谱邻域特征（空域）和时域关联分析，支持单设备/多设备多场景（http/icmp/rtsp）的异常识别。

## 一、项目数据分析

#### 1. 数据集概览（具有标签的）

项目包含三种协议的多设备时间序列数据：

| 协议     | 数据集      | 行数   | 设备数 | 列数 | KPI特征数 | 标签列 |
| -------- | ----------- | ------ | ------ | ---- | --------- | ------ |
| **HTTP** | data_train  | 58,207 | 11     | 80   | 76        | 是     |
|          | data_test   | 14,556 | 11     | 80   | 76        | 是     |
|          | data_detect | 26,023 | 11     | 79   | 76        | 否     |
| **ICMP** | data_train  | 44,467 | 8      | 35   | 31        | 是     |
|          | data_test   | 11,119 | 8      | 35   | 31        | 是     |
|          | data_detect | 25,055 | 8      | 34   | 31        | 否     |
| **RTSP** | data_train  | 46,261 | 9      | 91   | 87        | 是     |
|          | data_test   | 11,573 | 9      | 91   | 87        | 是     |
|          | data_detect | 24,773 | 9      | 90   | 87        | 否     |

#### 2. 数据结构

每个数据集包含：
- **前3列**：时间戳 (`time_value_min`)、服务器地址 (`n2_addr`)、设备ID (`imei`)
- **中间列**：KPI特征（以 `kpi_` 开头的性能指标）
- **最后列（训练集和测试集）**：标签 (`label`：0=正常，1=异常)

## 二、项目结构

`
KG_MTAD/
├── main.py                        # 统一入口：--mode train|test|detect|crossval
├── config.py                      # 命令行参数定义与默认路径
├── train.py                       # 训练模式
├── test.py                        # 测试模式
├── detect.py                      # 识别模式（无标签推理）
├── cv.py                          # 交叉验证模式（kfold/holdout/timeseries）
│
├── model/
│   ├── __init__.py
│   └── detector.py                # MultiTSAnomalyDetector 模型定义
│
├── data/
│   ├── __init__.py
│   ├── loader.py                  # load_data：多设备分组 + 全局标准化 + 序列化
│   ├── detect_loader.py           # load_data_detect：无标签数据加载
│   ├── sequences.py               # 断点感知时序序列创建
│   └── adjacency.py               # 邻接矩阵（知识图谱）加载
│
├── utils/
│   ├── __init__.py
│   └── transform.py               # 反标准化工具
│
├── outputs/
│   ├── __init__.py
│   └── save.py                    # 预测结果保存 + 评估指标计算
│
├── visualization/
│   ├── __init__.py
│   └── graph.py                   # KPI 知识图谱拓扑可视化
│
├── analysis/
│   ├── __init__.py
│   └── contribution.py            # 异常贡献度分析（Top-K 指标 + 拓扑邻居）
│
└── README.md
`

## 三、核心模式

项目支持四种运行模式：**训练**、**测试**、**识别**、**交叉验证**。以 `--mode` 参数切换。

#### 1. 训练模式 `--mode train`

从训练集学习 KPI 时序模式，同时在测试集上评估模型性能。

**特点**：
- 训练模型并保存权重
- 对测试集做完整评估
- 输出所有预测结果和可视化文件

**命令示例**：
```bash
python main.py --mode train --scenario http --epochs 100 --batch_size 256 --hidden_dim 128
```

#### 2. 测试模式 `--mode test`

加载已训练模型，对测试集做推理和评估，不更新模型权重。

**特点**：
- 依赖已训练好的模型（`model/{scenario}/anomaly_detector.ckpt`）
- 输出完整预测结果和评估指标
- 用于验证模型泛化能力

**命令示例**：

```bash
python main.py --mode test --scenario http
```

#### 3. 识别模式 `--mode detect`

对无标签数据做异常识别，输入多少行就输出多少行。

**特点**：
- 不依赖标签列，纯推理模式
- 数据缺失无法识别的行 label 留空
- 输出与输入行数完全一致

**命令示例**：
```bash
python main.py --mode detect --scenario http
```

#### 4. 交叉验证模式 `--mode crossval`

加载合并的带标签数据，内部自动划分训练/验证集，输出最佳折模型的完整结果。

**特点**：
- 支持三种划分方法：kfold / holdout / timeseries
- 输出每折的训练集和验证集指标（均值 ± 标准差）
- 使用最佳折模型对验证集做完整预测，输出与训练模式相同的文件

**划分方法对比**：

| 方法 | 命令行参数 | 训练/验证比例 | 说明 |
|------|-----------|--------------|------|
| **kfold** | `--cv_method kfold --cv_folds 5` | 80% / 20%（默认 5 折） | 分层 K 折交叉验证，打乱后均分，每折异常比例一致 |
| **holdout** | `--cv_method holdout --cv_split_ratio 0.8` | 80% / 20% | 单次分层随机划分 |
| **timeseries** | `--cv_method timeseries --cv_split_ratio 0.8` | 80% / 20% | 每设备内按时序划分，不打乱 |

**命令示例**：

```bash
# kfold 交叉验证
python main.py --mode crossval --scenario http --cv_method kfold --cv_folds 5

# holdout（保存最佳模型）
python main.py --mode crossval --scenario icmp --cv_method holdout --cv_save_model

# timeseries 70/30
python main.py --mode crossval --scenario rtsp --cv_method timeseries --cv_split_ratio 0.7
```

### 输入输出文件汇总

| 文件名 | train | crossval | test | detect | 说明 |
|--------|:-----:|:--------:|:----:|:------:|------|
| **输入文件** |||||
| `data_train.csv` | ✅ | — | — | ✅(scaler) | 训练数据（带标签） |
| `data_test.csv` | ✅ | — | ✅ | — | 测试数据（带标签） |
| `data_withlabels.csv` | — | ✅ | — | — | 合并的带标签数据（CV） |
| `data_detect.csv` | — | — | — | ✅ | 待识别数据（无标签） |
| `adjmat.csv` | ✅ | ✅ | ✅ | ✅ | 知识图谱邻接矩阵 |
| **输出文件** |||||
| `system_predictions.csv` | ✅ | ✅ | ✅ | — | 系统级预测（时间戳+IP+IMEI+标签） |
| `metrics_predictions.csv` | ✅ | ✅ | ✅ | — | KPI 指标级二值预测 |
| `evaluation_results.csv` | ✅ | ✅ | ✅ | — | 评估指标（Acc/Recall/F1/AUC） |
| `kpi_topology_graph.png` | ✅ | ✅ | ✅ | ✅ | KPI 拓扑关系图（GAT 模式为加权图） |
| `anomaly_contribution.csv` | ✅ | ✅ | ✅ | ✅ | 异常时刻 Top-K 贡献指标分析 |
| `learned_adjacency.csv/.npy` | ✅(GAT) | ✅(GAT) | — | — | GAT 学习到的邻接矩阵 |
| `cv_summary.csv` | — | ✅ | — | — | 交叉验证汇总（每折指标 + 均值/标准差） |
| `data_detect_output.csv` | — | — | — | ✅ | 识别结果（原数据 + 预测标签） |
| `anomaly_detector.ckpt.*` | ✅ | ✅(可选) | — | — | 模型权重（`--cv_save_model`） |
| `model_meta.json` | ✅ | ✅(可选) | — | — | 模型超参数配置 |

**说明**：
- `✅(GAT)` 表示仅 GAT 模式（`--adj_type gat`）输出
- `✅(可选)` 表示需加 `--cv_save_model` 参数才输出
- 所有输出文件路径：`outputs/{scenario}/`，模型文件路径：`model/{scenario}/`

## 四、消融实验

通过 `--adj_type` 参数控制邻接矩阵类型，评估知识图谱空域分支对模型性能的贡献。

#### 1. 支持类型

| `--adj_type` | 说明 | 消融目的 |
|---|---|---|
| `original`（默认） | 知识图谱原始邻接矩阵 | 基线实验 |
| `zero` | 全0邻接矩阵 | 关闭空域分支，仅保留时域LSTM |
| `one` | 全1邻接矩阵 | 全连接图，验证拓扑结构的重要性 |
| `random` | 随机邻接矩阵（固定seed=42） | 噪声基线，对比随机关联的效果 |
| `gat` | 预留，用于GAT动态学习邻接权重 | 未来扩展 |

#### 2. 运行命令

```bash
# 实验1：原始邻接矩阵（基线）
python main.py --mode crossval -s http --cv_method holdout

# 实验2：全0邻接矩阵（关闭空域分支）
python main.py --mode crossval -s http --cv_method holdout --adj_type zero

# 实验3：全1邻接矩阵（全连接）
python main.py --mode crossval -s http --cv_method holdout --adj_type one

# 实验4：随机邻接矩阵（噪声基线）
python main.py --mode crossval -s http --cv_method holdout --adj_type random
```

#### 3. 默认路径

- **数据目录**: `项目根目录/data/{scenario}/`（含 CSV 与邻接矩阵）
- **模型目录**: `项目根目录/model/{scenario}/`（权重 .ckpt + model_meta.json）
- **结果目录**: `项目根目录/outputs/{scenario}/`
