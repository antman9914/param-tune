# MODULE 1: 环境感知与超参数自动发现

## 1.1 确定项目路径与训练入口

TARGET_DIR = 用户参数（或当前目录）。以下所有扫描均在此目录下进行。

**定位训练脚本**：按优先级查找 TARGET_DIR 下的入口文件：
`train.py` → `main.py` → `run.py` → `scripts/train.py` → `train.sh` → `main.sh`

若以上均不存在，列出 TARGET_DIR 下所有可执行脚本（`.py` / `.sh` / `.r` / `.R` 等）：
- 若唯一，自动选择
- 若有多个，询问用户

将最终确定的路径记为 `TRAIN_SCRIPT`，在 1.6 中写入 `search_state.json`。

## 1.2 扫描配置文件

按优先级读取配置：
1. `config.yaml` / `config.json` / `config.toml` / `config.py`
2. `configs/` 目录下的文件
3. 训练脚本中的 `argparse` 定义（`add_argument` 调用）
4. 训练脚本顶部的全局变量赋值

## 1.3 自动发现可调超参数

从上述来源中提取**所有数值型参数**，不做关键词过滤，逐一分析。

**对每个参数，结合以下信息进行推理：**
- 参数名称及所在的代码上下文（optimizer 初始化？loss 函数？数据增强？）
- 参数的注释或文档字符串（如有）
- 当前取值（作为量级参考，不作为范围的唯一依据）
- 该参数在深度学习中的通用语义和典型取值分布

**对于不熟悉的参数**（自定义参数、不常见的正则化项、特定论文引入的超参数等），
使用 WebSearch 查询相关文献或官方文档，确认该参数的典型取值范围和 scale，
再给出推荐范围，并在"推断依据"列注明参考来源。

**对每个参数判断三件事：**

1. **是否可调**：区分三类
   - `✓ 是`：对模型性能有直接影响、在训练过程中可以自由调整的超参数（优化器参数、正则化参数、损失函数权重、数据增强强度等）
   - `✗ 否`：由数据或环境决定不应搜索的参数（`num_classes`、`input_dim`、`seed`、`num_workers`、`device` 等）
   - `? 询问`：结构性参数，调整代价大或需要重新设计网络（`hidden_dim`、`num_layers`、`num_heads`、`batch_size` 等），默认不搜索，但展示给用户决定

2. **Scale**：
   - 若参数的合理取值横跨多个数量级，或对数变化比线性变化更有意义 → `log`
   - 若参数本身是比例或概率，线性变化更自然 → `linear`

3. **初始搜索范围**：
   - 基于该参数的 ML 领域知识给出合理的初始范围
   - 若当前值超出典型范围，以当前值为中心扩展，并说明原因
   - 对于不熟悉的自定义参数，根据其量级和上下文给出保守的初始范围，并在备注中说明不确定性

输出自动发现结果表（包含所有数值型参数，不遗漏）：

```
自动发现的参数（请确认）：
┌─────────────────┬───────────┬────────┬──────────────┬─────────┬─────────────────────────┐
│ 参数名          │ 当前值    │ 是否调 │ 初始范围     │ Scale   │ 推断依据                │
├─────────────────┼───────────┼────────┼──────────────┼─────────┼─────────────────────────┤
│ learning_rate   │ 0.01      │ ✓ 是   │ [1e-5, 1e-1] │ log     │ Adam 优化器学习率典型范围│
│ weight_decay    │ 0.0       │ ✓ 是   │ [1e-6, 1e-1] │ log     │ L2 正则化系数            │
│ dropout_rate    │ 0.1       │ ✓ 是   │ [0.0, 0.8]   │ linear  │ Dropout 概率             │
│ hidden_dim      │ 64        │ ? 询问  │ [32, 256]    │ log/int │ 结构参数，调整代价较大  │
│ batch_size      │ 32        │ ? 询问  │ [8, 256]     │ log/int │ 影响有效学习率，可选搜索│
│ epochs          │ 100       │ ✗ 否   │ -            │ -       │ 训练预算，不作为超参搜索│
└─────────────────┴───────────┴────────┴──────────────┴─────────┴─────────────────────────┘
```

"推断依据"列必须填写，说明为该参数选择此范围和 scale 的理由。
对于不常见的自定义参数，在备注中标注"⚠️ 不确定，建议用户确认范围"。

若用户对某参数的分类或范围有异议，以用户意见为准并更新表格。

## 1.4 扫描数据信息

从数据加载代码或配置中提取：
- 样本数量（影响正则化强度建议）
- 输入维度（影响 dropout 重要性）
- 任务类型（分类/回归，用于 1.5 中 primary metric 的选择）
- 类别是否均衡（用于 1.5 中 primary metric 的选择）

若无法确定，填"未知"，不猜测。

## 1.5 训练脚本代码分析（仅首次运行执行）

**目标**：通过阅读并理解训练脚本的代码逻辑，一次性提取后续所有轮次所需的静态信息，存入 `search_state.json`。后续循环模块直接使用，不再重复扫描源代码。

### 1.5.1 读取训练脚本

读取 `TRAIN_SCRIPT`（由 1.1 确定）的完整代码。

### 1.5.2 Loss 函数合理性评估

理解训练脚本中的 loss 计算逻辑（包括主 loss、辅助 loss、正则化项等所有参与反向传播的部分），判断整体设计是否合理。

评估无固定检查列表，基于对代码的理解进行开放性判断。若发现任何可能影响训练稳定性或导致优化目标与任务目标不一致的问题，记录为 ⚠️ 警告，并说明原因。判断无问题则记录为"✓ 无明显风险"。

评估完成后立即决策：
- 若发现会**根本性影响调参有效性**的问题（如优化目标与任务目标不一致、loss scale 异常等）：输出具体问题说明，**终止初始化，等待用户修复后重新运行**。
- 若无显著风险，或仅有轻微提示（如某项正则化较强但不影响搜索方向）：简要记录，继续初始化。

### 1.5.3 理解训练输出

通过理解代码逻辑，回答以下三个问题：

**问题 A：训练结束后写出了什么文件？**
阅读训练脚本的结尾部分（训练循环结束后），找出所有写出的文件及其格式（CSV、JSON、pickle、checkpoint、日志文件、数控制台输出等）。对于每个输出文件，理解它包含的内容结构。

**问题 B：哪个指标最适合作为调参目标（primary metric）？**
结合 1.4 确定的任务类型，在训练脚本实际计算并输出的指标中，选出最能反映模型泛化能力的一个。理解标准：
- 分类任务：优先选择验证集上的准确率或 F1（不均衡时），而不是 loss
- 回归任务：优先选择验证集上的 MAE 或 RMSE
- 若脚本只输出 loss，则以 val_loss 为目标（方向：min）
- 确定方向（`max` 或 `min`）

**问题 C：如何从输出文件中提取 primary metric 的峰值？**
峰值（而非末值）才能反映模型在训练过程中达到的最佳状态。推理方式：
- 若脚本在训练结束时显式计算并保存了该 metric 的峰值（如 `best_val_acc`、`best_f1` 等），则直接读取该字段，记录其所在文件和字段路径
- 若脚本只保存了每个 epoch 的 metric 值（如 CSV 中的逐 epoch 记录），则记录需要扫描该文件并取 `max` 或 `min`
- 若两者都有，优先用显式保存的峰值字段

同时推理过拟合诊断所需的字段：在训练脚本中，是否同时保存了 train metric 的峰值（即 train 创新高时的值）及该 epoch 对应的 val metric？若能则记录这两个字段；若不能则标记为"仅凭曲线形态诊断"。

### 1.5.3 输出 metric_config

将以上推理结果写入 `search_state.json` 的 `metric_config` 字段。内容应完整描述"如何从训练输出中读取 primary metric 峰值"，字段名和结构根据实际项目自适应，下方为参考结构：

```json
"metric_config": {
  "primary_metric_name": "<metric 的语义名称，如 val_acc>",
  "primary_metric_direction": "<max 或 min>",
  "peak_value": {
    "source": "<文件类型或来源，如 json_summary / epoch_csv / checkpoint / stdout>",
    "file_pattern": "<文件路径或匹配模式，如 logs/*_summary.json>",
    "field_or_column": "<字段名或列名，如 best_val_acc>",
    "aggregation": "<若需聚合则填 max 或 min，否则填 direct>"
  },
  "overfitting_diagnosis": {
    "available": true,
    "train_metric": "<对应 epoch 的 train metric 字段，如 best_train_acc_at_best_val>",
    "val_metric": "<同上的 val metric 字段，如 best_val_acc>",
    "source": "<同 peak_value.source>"
  },
  "analysis_note": "<一两句话说明推理过程，引用代码中的关键行或逻辑>"
}
```

## 1.6 初始化搜索空间

根据 1.3 和 1.5 的分析结果，生成初始 search_state.json（Module 1 仅在首次运行时执行，此时文件必然不存在）：

```json
{
  "experiment_count": 0,
  "search_phase": "exploration",
  "best_reward": null,
  "best_experiment": null,
  "termination_triggered": false,
  "train_script": "<由 1.1 确定的相对路径>",
  "submit_script": "<提交脚本路径，集群环境下由 Step 6c 确定；本地环境填 null>",
  "scheduler": "<slurm / pbs / lsf / sge / local>",
  "tunable_params": ["<由 1.3 确定>"],
  "fixed_params": {},
  "search_space": {
    "<每个 tunable 参数名>": {
      "scale": "<log 或 linear>",
      "min": <初始下界>,
      "max": <初始上界>,
      "focus_min": null,
      "focus_max": null,
      "excluded_regions": []
    }
  },
  "metric_config": { "<由 1.5.3 填写>" },
  "task_type": "<由 1.4 确定>",
  "n_classes": "<若适用，否则省略>",
  "candidates": [],
  "current_candidate_idx": 0
}
```

注：`candidates` 和 `current_candidate_idx` 在 exploration 阶段为空/0，切换到 multi_focused 时由 4.3 填充。`submit_script` 和 `scheduler` 在 Step 6c 首次检测环境时写入，初始化阶段可暂时省略。

## 1.7 Module 1 输出摘要

- 可调参数列表及当前范围
- 数据特征标记（小数据集 / 高维 / 不均衡）
- 任务类型（分类 / 回归）
- Primary metric 及读取方式（来自 metric_config）
- 搜索阶段及已完成实验数
