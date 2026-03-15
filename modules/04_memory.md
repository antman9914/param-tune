# MODULE 4: 记忆与搜索决策

## 4.1 将本次实验追加到历史日志

**若当前轮次尚无训练结果（首次运行，Step 3 返回 reward = null）**：跳过 4.1–4.4，直接进入 4.5 生成推荐配置。

将以下内容 append 到 `~/.claude/skills/param-tune/memory/experiment_log.jsonl`（每次一行）：

```json
{
  "exp_id": "exp_NNN",
  "candidate_id": "<exploration | candidate_A | candidate_B | ...>",
  "timestamp": "<ISO 8601>",
  "hyperparams": {
    "<param_1>": <值>,
    "<param_2>": <值>
  },
  "results": {
    "primary_metric_name": "<metric名>",
    "primary_metric_value": <值>,
    "reward": <值>,
    "overfitting_train_value": <值或null>,
    "overfitting_val_value": <值或null>
  },
  "curve_diagnosis": "<Module 3 的一句话诊断>",
  "tuning_hint": "<Module 3 的调参方向提示>"
}
```

`overfitting_train_value` 和 `overfitting_val_value` 从 `metric_config.overfitting_diagnosis` 描述的字段读取；若 `overfitting_diagnosis.available = false`，则填 `null`。

exp_id 编号 = experiment_count + 1，格式 exp_001, exp_002 ...

追加完成后，将 `experiment_count` 加 1 并写回 `search_state.json`。

`candidate_id` 取值规则：
- exploration 阶段：填 `"exploration"`
- multi_focused 阶段：填当前候选的 id（如 `"candidate_A"`）

## 4.2 历史趋势分析

读取 `~/.claude/skills/param-tune/memory/experiment_log.jsonl` 的所有记录。

若记录数 ≤ 1，跳过此步直接进入 4.3。

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
search_phase == "exploration" → 全范围随机采样，每轮结束后评估是否切换
search_phase == "multi_focused" → 多起点 focused 搜索
```

### exploration 阶段的切换评估（每轮 exploration 结束后执行）

**最少轮次保护**：若 exploration 实验数 < 3，直接继续 exploration，不做评估。

**强制切换上限**：若 exploration 实验数 ≥ 12，无论评估结果如何，强制切换至 multi_focused。

**3 ≤ 实验数 < 12 时**，评估以下两项标准，**全部满足则切换**：

1. **奖励区分度**：`max(rewards) - min(rewards) ≥ 0.02`
   → 说明搜索空间内存在可区分的高低值区域，有方向性信号

2. **参数覆盖度**：至少有 1 个 tunable 参数在已有实验中覆盖了 ≥ 3 个不同子区间
   → 避免采样点全部聚集在一处，导致锚点选择缺乏全局视野
   （子区间定义：将该参数的搜索范围等分为 4 段；log scale 参数按对数均匀划分；落入的段数 ≥ 3 即满足）

**候选 B 可行性**不作为切换的必要条件，而是决定切换后使用**双候选**还是**单候选**模式：
- 在满足质量门槛（`reward ≥ max_reward × 0.95`）的实验中，
  存在与候选 A 归一化距离 ≥ 0.2 的实验 → 双候选模式
- 否则 → single-candidate 模式（只保留候选 A）

**若两项标准未全部满足**，继续 exploration，并在推荐下一组参数时，
优先选择当前**覆盖最少的子区域**（最远离已有采样点的区域），提升后续评估成功率。

输出评估摘要：
```
[Exploration 评估 — 第 N 轮]
奖励区分度：{max:.4f} - {min:.4f} = {spread:.4f}  {"✓" if >= 0.02 else "✗（继续探索）"}
参数覆盖度：{param_1} 覆盖 {M}/4 子区间  {"✓" if >= 3 else "✗"}
→ {"切换至 multi_focused（双候选）" / "切换至 multi_focused（单候选，无合适的候选B）" / "继续 exploration，下一轮优先覆盖：{区域描述}"}
```

### exploration → multi_focused 切换（评估通过时执行）

**从历史实验中选出 2 个候选起点（最远点采样）：**

1. 将每个参数归一化到 [0, 1]：
   - log scale 参数：`norm = (log(v) - log(min)) / (log(max) - log(min))`
   - linear 参数：`norm = (v - min) / (max - min)`

2. 候选 A = reward 最高的实验

3. 候选 B = 在满足质量门槛的实验中，与候选 A 归一化距离最远的实验（欧氏距离）：
   `dist(A, B) = sqrt(sum((norm_A_i - norm_B_i)^2))`

   **质量门槛**：候选 B 的 anchor 实验必须满足 `reward >= candidate_A.reward × 0.95`。
   排除不满足此条件的实验后再做最远点选择。

   以下情况只保留 1 个候选（single-candidate 模式）：
   - 所有满足质量门槛的其他实验与 A 的距离 < 0.2（归一化空间内距离太近）
   - 没有任何实验满足质量门槛（其余实验 reward 均低于 candidate_A.reward × 0.95）

4. 为每个候选分别计算 focus 范围：
   - log scale：`[anchor × 0.4, anchor × 2.5]`（大约 ±1 个数量级的一半）
   - linear：`[anchor - 0.25×range, anchor + 0.25×range]`，并 clip 到 [min, max]

5. 将 `search_phase` 更新为 `"multi_focused"`，写回 `search_state.json`。

6. 写入 search_state.json 的 `candidates` 字段：

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

7. **询问用户是否启用并行搜索模式**：
   > 是否并行运行所有候选区域？（每轮同时提交 N 个作业，需要集群环境）[y/N]

   - 若用户选 y 且 `scheduler != local` → 将 `"parallel_mode": true` 写入 search_state.json，后续切换至 SKILL.md 的并行循环流程
   - 其余情况 → `"parallel_mode": false`，继续顺序循环

### multi_focused 阶段的搜索逻辑

读取 `current_candidate_idx`，当前工作在对应的候选区域内：
- 推荐的参数值必须落在当前候选的 focus 范围内

**每轮追加实验后，立即从 `experiment_log.jsonl` 重建当前候选的 `recent_rewards`**：

```
candidate.recent_rewards = [
    entry.results.reward
    for entry in experiment_log
    if entry.candidate_id == current_candidate.id
]
```

不依赖 search_state.json 中的历史 append，而是每轮从 log 派生，确保状态与日志始终一致。
将重建后的 `recent_rewards` 写回 search_state.json 的对应候选字段。

**候选切换**：若当前候选触发终止条件（见 4.4），将其 `converged = true`，
`current_candidate_idx += 1`，切换到下一个候选，**不终止整体搜索**。

**整体终止**：所有候选的 `converged` 均为 `true` 时，整体搜索结束。

## 4.3b Focus Range 动态收窄

**触发条件**：multi_focused 阶段，且当前候选在本轮结束后累计实验数 ≥ 5。

**在并行模式下**：对每个活跃候选（converged=false）分别执行一次以下收窄逻辑，而非仅针对 current_candidate。

**每轮结束后执行一次**（在推荐下一组参数前）：

### 第一步：识别高价值实验

从 `experiment_log.jsonl` 中取出当前候选的所有实验，计算：

```
candidate_best = max(当前候选所有实验的 reward)
high_reward_threshold = candidate_best - 0.02
high_reward_exps = [exp for exp in 当前候选实验 if exp.reward >= high_reward_threshold]
```

若 `len(high_reward_exps) < 3`：跳过本步骤（高价值样本不足，暂不收窄）。

### 第二步：计算各参数的高价值区间

对每个 tunable 参数，在 `high_reward_exps` 中取参数值：

- **log scale 参数**：
  ```
  param_min = min(high_reward_exps[param])
  param_max = max(high_reward_exps[param])
  new_focus_min = param_min / 1.3   # 向外扩 buffer
  new_focus_max = param_max * 1.3
  ```

- **linear 参数**：
  ```
  param_min = min(high_reward_exps[param])
  param_max = max(high_reward_exps[param])
  focus_width = current_focus_max - current_focus_min
  new_focus_min = param_min - 0.15 * focus_width
  new_focus_max = param_max + 0.15 * focus_width
  ```

### 第三步：应用约束并决定是否更新

对每个参数的新 focus 范围：

1. **Clip 到当前 focus**：`new_focus_min = max(new_focus_min, current_focus_min)`，`new_focus_max = min(new_focus_max, current_focus_max)`（只能收窄，不能扩张）
2. **最小宽度保护**：若新 focus 宽度 < 当前 focus 宽度的 20%，保持当前 focus 不变（防止过度收窄导致采样空间崩溃）
   - log scale 宽度 = `log(max) - log(min)`
   - linear 宽度 = `max - min`
3. **只有真正收窄才更新**：若新 `focus_min > current_focus_min` 或 `new_focus_max < current_focus_max`，则更新 search_state.json 中该候选的对应参数 focus 范围

### 第四步：输出收窄日志

若任意参数的 focus 发生了变化，输出：

```
[Focus 收窄] 候选 {candidate_id}，基于 {len(high_reward_exps)} 个高价值实验（reward ≥ {threshold:.4f}）：
  {param_1}: [{旧min}, {旧max}] → [{新min}, {新max}]
  {param_2}: 无变化
  ...
```

若没有任何参数发生变化，不输出（静默跳过）。

## 4.4 终止条件检查

**仅在 multi_focused 阶段执行**（exploration 阶段通过评估条件或达到12轮上限时切换，无需终止判断）。

**K = 5**（multi_focused 阶段固定值）。

**在并行模式下**：对每个活跃候选分别独立执行以下终止判断（使用各自的 recent_rewards），而非仅针对 current_candidate。

**从重建后的 `recent_rewards` 中做终止判断**：

**若长度 < K**：跳过终止判断，继续。

**若长度 ≥ K**：
```
recent_rewards = current_candidate.recent_rewards

candidate_best = max(recent_rewards)
first_best_idx = recent_rewards.index(candidate_best)   # 仅取第一次出现的下标
post_best_count = len(recent_rewards) - first_best_idx - 1
```

**终止判断**：

- `post_best_count >= K`
  → 自候选首次达到 `candidate_best` 起，后续已完成 K 次实验且无一严格超越该值
  → 当前候选 `converged = true`
- 否则：继续当前候选

> **说明**：收敛与否取决于"自首次达到最优以来经过了多少轮"，与后续轮次是否再次达到该最优值无关。在终止报告中应准确描述为"距首次达到最优已过 K 轮"，而非"某轮达到最优触发了收敛"。

将更新后状态写回 `~/.claude/skills/param-tune/memory/search_state.json`。

## 4.5 生成推荐配置

**若仍有未收敛的候选**：

读取 search_state.json 中的 `parallel_mode`：

**顺序模式（parallel_mode = false）**：

在当前候选（`current_candidate_idx` 对应）的 focus 范围内推荐下一组参数，优先覆盖该候选区域内的空白子区域。

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

**并行模式（parallel_mode = true）**：

为所有 converged=false 的候选分别生成推荐配置，各自在其 focus 范围内采样，优先覆盖本候选区域内的空白子区域。

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
推荐实验配置（本轮并行，共 N 个活跃候选）   [阶段: multi_focused]
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
[候选区域: candidate_A]
  {param_1} : {值}    [{推荐理由}]
  {param_2} : {值}    [{推荐理由}]

[候选区域: candidate_B]
  {param_1} : {值}    [{推荐理由}]
  {param_2} : {值}    [{推荐理由}]

推荐理由综述：{2-3句话，说明各候选的探索方向}
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

本轮某候选收敛时额外输出（该候选不再出现在下一轮推荐中）：
```
候选 {id} 已收敛（best_reward={值}），下一轮不再提交该候选
```

**若所有候选均已收敛**：

将 `search_state.json` 中的 `termination_triggered` 设为 `true`，`search_phase` 设为 `"terminated"`，写回文件。

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
