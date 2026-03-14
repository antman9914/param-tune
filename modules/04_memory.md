# MODULE 4: 记忆与搜索决策

## 4.1 将本次实验追加到历史日志

将以下内容 append 到 `~/.claude/skills/param-tune/memory/experiment_log.jsonl`（每次一行）：

```json
{
  "exp_id": "exp_NNN",
  "timestamp": "<ISO 8601>",
  "hyperparams": {
    "<param_1>": <值>,
    "<param_2>": <值>
  },
  "results": {
    "train_loss_final": <值>,
    "val_loss_final": <值>,
    "primary_metric_name": "<metric名>",
    "primary_metric_value": <值>,
    "overfit_ratio": <值>,
    "reward": <值>
  },
  "curve_diagnosis": "<Module 3 的一句话诊断>",
  "tuning_hint": "<Module 3 的调参方向提示>"
}
```

exp_id 编号 = experiment_count + 1，格式 exp_001, exp_002 ...

## 4.2 历史趋势分析

读取 `~/.claude/skills/param-tune/memory/experiment_log.jsonl` 的所有记录。

若只有 1 条（当前轮首次），跳过此步直接进入 4.3。

**对每个 tunable 参数做单参数影响分析**（若该参数有 ≥3 个不同取值）：
- 列出取值 vs reward
- 判断趋势：单调增 / 单调减 / 有峰值区间
- 若有明显峰值区间，标记为"高价值区间"

**协同效应检测**（实验数 ≥ 6）：
- 检查是否存在某两个参数的取值组合普遍对应低 reward
- 若发现，记录为注意事项，推荐时避免这类组合

## 4.3 搜索阶段管理（含多起点）

### 阶段定义

```
experiment_count < 5  → exploration（全范围随机采样）
experiment_count >= 5 → multi_focused（多起点 focused 搜索）
```

### exploration → multi_focused 切换（仅在 experiment_count 首次达到 5 时执行）

**从历史实验中选出 2 个候选起点（最远点采样）：**

1. 将每个参数归一化到 [0, 1]：
   - log scale 参数：`norm = (log(v) - log(min)) / (log(max) - log(min))`
   - linear 参数：`norm = (v - min) / (max - min)`

2. 候选 A = reward 最高的实验

3. 候选 B = 与候选 A 归一化距离最远的实验（欧氏距离）：
   `dist(A, B) = sqrt(sum((norm_A_i - norm_B_i)^2))`
   若所有其他实验与 A 的距离 < 0.2（归一化空间），则只保留 1 个候选（距离太近没有意义）

4. 为每个候选分别计算 focus 范围：
   - log scale：`[anchor × 0.4, anchor × 2.5]`（大约 ±1 个数量级的一半）
   - linear：`[anchor - 0.25×range, anchor + 0.25×range]`，并 clip 到 [min, max]

5. 写入 search_state.json 的 `candidates` 字段：

```json
"candidates": [
  {
    "id": "candidate_A",
    "anchor_exp": "exp_NNN",
    "focus": {
      "<param>": {"focus_min": <值>, "focus_max": <值>}
    },
    "recent_rewards": [],
    "converged": false
  },
  {
    "id": "candidate_B",
    "anchor_exp": "exp_MMM",
    "focus": { ... },
    "recent_rewards": [],
    "converged": false
  }
],
"current_candidate_idx": 0
```

### multi_focused 阶段的搜索逻辑

读取 `current_candidate_idx`，当前工作在对应的候选区域内：
- 推荐的参数值必须落在当前候选的 focus 范围内
- 每轮将本次 reward 追加到当前候选的 `recent_rewards`

**候选切换**：若当前候选触发终止条件（见 4.4），将其 `converged = true`，
`current_candidate_idx += 1`，切换到下一个候选，**不终止整体搜索**。

**整体终止**：所有候选的 `converged` 均为 `true` 时，整体搜索结束。

## 4.4 终止条件检查

**确定窗口大小 K 和阈值 δ**（按当前搜索阶段）：
- exploration 阶段：K=3，δ=0.02
- multi_focused 阶段：K=5，δ=0.01

**检查当前候选的 recent_rewards**：

**若长度 < K**：跳过终止判断，继续。

**若长度 ≥ K**：
```
window = current_candidate.recent_rewards[-K:]
improvement = max(window) - min(window)
```

**采样均匀性检查**：对 window 对应的最近 K 次实验，
若任意一个 tunable 参数的取值标准差 < 该参数当前 focus 范围的 5%，
视为"集中采样"，不触发终止。

**终止判断**：
- improvement < δ 且采样均匀 → 当前候选 `converged = true`
- 否则 → 继续当前候选

将更新后状态写回 `~/.claude/skills/param-tune/memory/search_state.json`。

## 4.5 生成推荐配置

**若仍有未收敛的候选**：

在当前候选的 focus 范围内推荐下一组参数，优先覆盖该候选区域内的空白子区域。

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
推荐实验配置（实验 #N+1）   [候选区域: {candidate_id} | 阶段: multi_focused]
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
{param_1} : {值}    [{推荐理由}]
{param_2} : {值}    [{推荐理由}]
...
推荐理由综述：{2-3句话}
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

候选切换时额外输出：
```
候选 {id} 已收敛（best_reward={值}），切换到候选 {next_id}（锚点：{exp_id}）
```

**若所有候选均已收敛**：

读取 `experiment_log.jsonl` 中的全部记录，找出 `reward` 字段最大的实验作为全局最优。
**不以窗口内最优为准，而是扫描所有历史实验取全局最大值。**

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
搜索终止 — 所有候选区域已收敛
共进行实验：{N} 次   候选数量：{M}
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
各候选区域最优结果：
  候选 A（锚点 {exp_id}）: reward={值}  {参数}
  候选 B（锚点 {exp_id}）: reward={值}  {参数}

全局最优实验：{exp_id}（来自 experiment_log.jsonl 全量扫描）
  {每个 tunable 参数}: {该实验的实际值}
  reward        : {值}
  primary_metric: {值}

后续建议：
- 若两个候选 reward 差距 < 0.02，说明存在多个等价最优区域，可信度更高
- 若差距较大，建议在胜出候选附近继续缩小搜索范围
- 若需进一步提升，可考虑将 hidden_dim 等结构参数纳入搜索
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```
