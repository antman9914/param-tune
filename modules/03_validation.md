# MODULE 3: 验证评估 — Reward Function

## 3.1 读取 metric_config

从 `~/.claude/skills/param-tune/memory/search_state.json` 中读取 `metric_config`，
获取本项目的 primary metric 配置（由 Module 1 在初始化时通过代码推理确定，此处直接使用）：

- `primary_metric_name`：metric 名称（如 `val_acc`）
- `primary_metric_direction`：`max` 或 `min`
- `peak_value.source`：数据来源类型（如 `json_summary`、`epoch_csv`）
- `peak_value.file_pattern`：文件路径或匹配模式
- `peak_value.field_or_column`：字段名或列名
- `peak_value.aggregation`：`direct`（直接读取）或 `max`/`min`（需聚合）
- `overfitting_diagnosis.available`：是否可做过拟合诊断
- `overfitting_diagnosis.train_metric`：train metric 峰值字段名（train 创新高时的 train 值）
- `overfitting_diagnosis.val_metric`：同一 epoch 的 val metric 字段名（train 创新高时的 val 值）
- `overfitting_diagnosis.source`：同 peak_value.source

## 3.2 读取 Primary Metric 值

按 `metric_config.peak_value` 描述的方式读取：

1. 定位最新一次训练的输出文件（按 `file_pattern` 匹配，取最新的）
2. 按 `aggregation` 字段决定读取方式：
   - `direct`：直接读取 `field_or_column` 的值
   - `max` / `min`：扫描文件中该列/字段的所有值，取最大/最小值
3. 若文件或字段不存在，报告具体原因，不静默忽略

**归一化**（统一为"越高越好"）：
- direction=max：直接使用原始值
- direction=min（回归误差）：`normalized = 1 / (1 + raw_value)`

## 3.3 读取训练曲线并诊断

从最新实验日志中读取完整训练曲线（若日志为 CSV，则读取全部 epoch 行；若为 JSON，则读取 epoch 级记录）。

**曲线形态诊断**（基于最后 20% epoch 的数据）：

| 形态 | 诊断 | 调参方向提示 |
|------|------|------------|
| val metric 震荡（std > mean×0.15） | lr 过高 | lr ÷ 3~5 |
| train/val metric 前 30% epoch 几乎不动 | lr 过低 | lr × 3~10 |
| val loss 持续高于 train loss 且差距扩大 | 过拟合 | 增加正则化 |
| val/train metric 都差且平稳不动 | 欠拟合 | 减少正则化或增加模型容量 |
| val metric 先升后降（有明显拐点） | 过拟合（有最优点） | 记录 best_epoch，建议 early stopping |
| val metric 在末尾仍持续改善 | 尚未收敛 | lr 偏低 或 训练轮数不足 |
| 正常收敛 | 无明显问题 | 在当前参数附近精细搜索 |

## 3.4 计算 Reward

```
reward = primary_metric_normalized
```

注意：
- `primary_metric_normalized` 已统一为"越高越好"的 [0,1] 值
- reward 不含过拟合惩罚项，过拟合信息通过曲线诊断（3.3）传递给 Module 4 影响搜索方向

**过拟合程度判断**（仅用于诊断，不影响 reward）：

若 `overfitting_diagnosis.available = true`，从数据源中读取 train metric 峰值及同 epoch 的 val metric：

> **注意**：train metric 通常在 train 模式（dropout 开启）下计算，val metric 在 eval 模式（dropout 关闭）下计算。因此比较的是「train 达到最充分状态时，模型在 val 上的表现」，绝对值差异中包含 dropout 引起的系统性偏差，应以**差距的相对大小和趋势**为主要依据，而不是绝对阈值。

| 条件 | 判断 |
|------|------|
| val metric 与 train metric 接近（差距 < dropout 预期偏差） | 无明显过拟合 |
| val metric 明显低于 train metric，且差距随训练扩大 | 中等，建议增加正则化 |
| val metric 远低于 train metric，val 曲线先升后降有明显拐点 | 严重，需增强正则化或 early stopping |

若 `overfitting_diagnosis.available = false`，仅凭曲线形态诊断，标注"仅凭曲线形态判断"。

输出本次评估结果：
```
实验评估：
  primary_metric ({metric_name}) : {值}
  overfitting_train_metric       : {值}（若可获取）
  overfitting_val_metric         : {值}（若可获取）
  过拟合诊断                      : {无 / 轻微 / 中等 / 严重}
  curve_diagnosis                : {一句话}
  reward                         : {值}
```

## 3.5 Module 3 输出摘要

- reward 值（= primary_metric）
- primary_metric 名称和值
- 过拟合程度诊断（不影响 reward，传递给 Module 4 作为搜索方向参考）
- 曲线诊断结论
- 调参方向提示（传递给 Module 4）
