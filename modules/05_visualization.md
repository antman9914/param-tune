# MODULE 5: 训练过程可视化

## 5.1 检测 wandb 可用性

读取 TARGET_DIR 的 TRAIN_SCRIPT，检查是否包含 `import wandb` 或 `wandb.init`。

- 若使用 wandb → 执行 5.2
- 若未使用 wandb → 执行 5.3

## 5.2 通过 wandb API 读取训练曲线

从 TRAIN_SCRIPT 中提取 `wandb.init(project=...)` 的 project 名称。若未找到，使用 TARGET_DIR 目录名作为 project 名（`PROJECT_NAME`）。

执行以下命令获取最新 run 的训练历史：

```bash
python3 - <<'EOF'
import wandb, json, sys
try:
    api = wandb.Api()
    runs = api.runs("PROJECT_NAME", order="-created_at")
    if not runs:
        print(json.dumps({"error": "no runs found"})); sys.exit(0)
    run = runs[0]
    history = run.history().to_dict(orient="records")
    fields = list(run.history().columns)
    print(json.dumps({"run_id": run.id, "run_name": run.name, "history": history, "fields": fields}))
except Exception as e:
    print(json.dumps({"error": str(e)}))
EOF
```

- 若返回 `{"error": ...}` 或命令失败 → 回退到 5.3
- 否则使用返回的 history 列表 → 进入 5.4

## 5.3 从本地日志文件读取训练曲线

在 TARGET_DIR 下按优先级查找 epoch 级日志：
1. `metric_config.peak_value.file_pattern` 对应文件（若 aggregation=max/min，说明有 epoch 级数据）
2. `logs/*.csv`（取最新文件）
3. `train_log.csv`、`history.csv`

若所有路径均不含 epoch 级数据（如 json_summary 且无 CSV）：
→ 输出 `[可视化] 未找到 epoch 级曲线数据，跳过可视化`，退出本模块。

若找到，读取全部 epoch 行，记为 history 列表，进入 5.4。

## 5.4 生成训练曲线图

确认目录存在（若不存在则创建）：`~/.claude/skills/param-tune/memory/figures/`

从 metric_config 读取：
- `overfitting_diagnosis.val_metric` → val 曲线字段名（`VAL_FIELD`）
- `overfitting_diagnosis.train_metric` → train 曲线字段名（`TRAIN_FIELD`，若 available=false 则为 null）
- `primary_metric_name` → 纵轴标签

图表保存路径：
- 顺序模式：`~/.claude/skills/param-tune/memory/figures/round_{NNN}.png`
- 并行模式（候选 id 已知）：`~/.claude/skills/param-tune/memory/figures/round_{NNN}_{candidate_id}.png`

执行绘图：

```bash
python3 - <<'PYEOF'
import matplotlib; matplotlib.use('Agg')
import matplotlib.pyplot as plt, json

history     = HISTORY_LIST       # list[dict]，由 5.2 或 5.3 传入
val_field   = "VAL_FIELD"        # 替换为实际字段名
train_field = "TRAIN_FIELD"      # 若无则为 None
metric_name = "PRIMARY_METRIC"   # 替换为 primary_metric_name
figure_path = "FIGURE_PATH"      # 替换为实际保存路径
title       = "ROUND_LABEL"      # 如 "Round 007 — exp_007"

epochs = list(range(len(history)))
fig, ax = plt.subplots(figsize=(8, 5))

val_vals = [d.get(val_field) for d in history]
if any(v is not None for v in val_vals):
    ax.plot(epochs, val_vals, label=f'val {metric_name}', color='steelblue')

if train_field:
    train_vals = [d.get(train_field) for d in history]
    if any(v is not None for v in train_vals):
        ax.plot(epochs, train_vals, label=f'train {metric_name}', color='coral', linestyle='--')

ax.set_xlabel('Epoch'); ax.set_ylabel(metric_name)
ax.set_title(title); ax.legend(); ax.grid(True, alpha=0.3)
plt.tight_layout(); plt.savefig(figure_path, dpi=150)
print('saved')
PYEOF
```

## 5.5 输出结果

- 若生成成功（输出 `saved`）：
  ```
  [可视化] 训练曲线已保存：{figure_path}
  ```
- 若失败（matplotlib 缺失、数据为空、字段不匹配等）：输出具体原因，**不中断主循环**。
