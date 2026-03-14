# MODULE 3: 验证评估 — Reward Function

## 3.1 自动选择 Primary Metric

根据 Module 2 传入的任务类型，自动选择：

| 任务类型 | Primary Metric | 方向 | 辅助 Metric |
|---------|---------------|------|------------|
| 多分类（均衡） | val_accuracy | ↑高 | Macro-F1 |
| 多分类（不均衡） | val_macro_f1 | ↑高 | per-class accuracy |
| 二分类（均衡） | val_accuracy | ↑高 | F1, AUC |
| 二分类（不均衡） | val_f1 | ↑高 | Precision, Recall |
| 回归 | val_mae 或 val_mse | ↓低 | R² |

**归一化处理**（统一为"越高越好"）：
- 分类：直接使用 [0,1] 的 accuracy / F1
- 回归：`normalized_metric = 1 - (val_mae / mean_abs_target)`，若无法计算则用 `1 / (1 + val_mse)`

**自动检测 metric 名称**：
从日志文件的 CSV 列名或 summary JSON 的 key 中，
匹配以下模式（大小写不敏感）：
- `val_acc*`, `val_accuracy` → 分类 metric
- `val_f1*`, `val_macro_f1` → F1 metric
- `val_loss` → 作为辅助指标
- `val_mae`, `val_mse`, `val_rmse` → 回归 metric

若自动检测失败，使用 `val_loss`（最小化）作为 fallback，并提示用户确认。

## 3.2 读取训练曲线并诊断

从最新实验日志（CSV 格式）中读取完整曲线。

**曲线形态诊断**（基于最后 20% epoch 的数据）：

| 形态 | 诊断 | 调参方向提示 |
|------|------|------------|
| val_loss 震荡（std > mean×0.15） | lr 过高 | lr ÷ 3~5 |
| train/val loss 前 30% epoch 几乎不动 | lr 过低 | lr × 3~10 |
| val_loss 持续高于 train_loss 且差距扩大 | 过拟合 | 增加正则化 |
| val/train loss 都高且平稳 | 欠拟合 | 减少正则化 |
| val_loss 先降后升（有明显拐点） | 过拟合（有最优点） | 记录 best_epoch，建议 early stopping |
| val_loss 在末尾仍持续下降 | 尚未收敛 | lr 偏低 或 训练轮数不足 |
| 正常收敛 | 无明显问题 | 在当前参数附近精细搜索 |

## 3.3 计算 Reward

```
overfit_ratio = min(max(0, (val_loss_final - train_loss_final) / train_loss_final), 3.0)

reward = 0.85 × primary_metric_normalized - 0.15 × overfit_ratio
```

注意：
- `primary_metric_normalized` 已统一为"越高越好"的 [0,1] 值
- `overfit_ratio` 用 max(0,...) 截断，val_loss < train_loss 时不额外奖励
- 若日志中无 train_loss，仅用 `reward = primary_metric_normalized`

输出本次评估结果：
```
实验评估：
  primary_metric ({metric_name}) : {值}
  train_loss_final               : {值}
  val_loss_final                 : {值}
  overfit_ratio                  : {值}
  curve_diagnosis                : {一句话}
  reward                         : {值}
```

## 3.4 Module 3 输出摘要

- reward 值
- primary_metric 名称和值
- 过拟合程度：无 / 轻微 / 中等 / 严重
- 曲线诊断结论
- 调参方向提示（传递给 Module 4）
