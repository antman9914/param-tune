# MODULE 2: 训练协议分析

## 2.1 自动识别损失函数

从训练代码中扫描以下模式：

| 代码特征 | 损失函数 | 任务类型 |
|---------|---------|---------|
| `CrossEntropyLoss` / `cross_entropy` | 交叉熵 | 多分类 |
| `BCELoss` / `BCEWithLogitsLoss` | 二元交叉熵 | 二分类 |
| `MSELoss` / `mse_loss` | 均方误差 | 回归 |
| `L1Loss` / `HuberLoss` | L1/Huber | 回归 |
| `NLLLoss` | 负对数似然 | 分类 |
| 自定义 loss 类 | 标记为"自定义" | 需用户确认 |

任务类型结论将传递给 Module 3 用于 metric 选择。

## 2.2 识别优化器与正则化

扫描 optimizer 的初始化代码，提取：
- optimizer 类型（Adam / SGD / AdamW / RMSProp 等）
- optimizer 中的 `weight_decay` 值
- lr_scheduler 类型（StepLR / CosineAnnealing / ReduceLROnPlateau / None）

**双重正则化检测**：
若代码中同时存在 optimizer 的 `weight_decay > 0` 和手动 `loss += lambda * norm`，
输出 ⚠️ 警告：双重正则化会叠加，建议只保留一种。
暂停调参建议，等用户确认。

## 2.3 读取本次实验配置

**若 logs/ 目录下不存在任何实验日志（首次运行）**：
- 直接从 config 文件读取当前参数值（跳过 summary JSON 查找）
- 跳过 2.4 的训练稳定性预检，标注"预检结果：首次运行，无历史日志"
- 继续输出 Module 2 摘要后，进入 Step 3

否则，从最新实验日志的 summary JSON 或 config 文件读取当前实际使用的参数值。

只显示 **tunable_params**（来自 Module 1 的列表）中的参数：

```
本次实验配置（搜索参数）：
  <param_1> : <值>
  <param_2> : <值>
  ...

固定配置（仅供参考）：
  batch_size    : <值>
  epochs        : <值>
  optimizer     : <类型>
  lr_scheduler  : <类型或 None>
```

## 2.4 训练稳定性预检

从最新实验日志中读取第一个 epoch 的 loss，做以下检查：

- **分类任务**：初始 loss 应接近 `-log(1/num_classes)`
  - 偏差超过 2 倍 → 可能有数据或初始化问题，⚠️ 警告
- **回归任务**：初始 loss 应在目标值方差量级以内
- 任意 epoch 出现 NaN 或 inf → 🚫 停止，报告并等待用户处理
- 第一个 epoch 后 loss 完全不变 → ⚠️ 可能学习率为 0 或梯度未传播

预检失败时停止循环，不继续分析。

## 2.5 Module 2 输出摘要

- 损失函数类型
- 任务类型（传递给 Module 3）
- 双重正则化风险（有/无）
- 本次实验的搜索参数实际值
- 预检结果（通过 / 异常类型）
