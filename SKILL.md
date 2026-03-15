---
name: param-tune
description: 通用 ML 模型超参数自动调参工具。自动发现项目中的可调参数，
             循环执行：分析日志 → 更新搜索空间 → 推荐参数 → 更新配置 → 运行训练，
             直到搜索收敛。支持任意 ML 框架和任务类型。
             触发词：调参、tune、hyperparam、loss不收敛、开始调参
user-invocable: true
---

# ML 超参数自动调参工作流

## 使用方式
- `/param-tune` — 在当前目录启动
- `/param-tune ~/my_project` — 指定项目路径

TARGET_DIR = $ARGUMENTS（若为空则用当前目录）

启动前确认 TARGET_DIR 下存在训练脚本（`train.py` 或 `main.py`）和配置文件，否则报错退出。

---

## 启动检查

检查 `~/.claude/skills/param-tune/memory/search_state.json` 是否存在：
- **不存在** → 首次运行，执行 **初始化流程**
- **已存在** → 读取其中的 `tunable_params` 和 `search_space`，执行 **循环流程**

---

## 初始化流程（仅首次运行执行一次）

### Init — 环境感知与超参数自动发现
读取并执行 ~/.claude/skills/param-tune/modules/01_environment.md

展示自动发现的参数表，等待用户确认（可增删参数、调整范围）后，
将确认结果写入 `~/.claude/skills/param-tune/memory/search_state.json`，
然后进入循环流程的 Step 2。

---

## 循环流程

以下步骤循环执行，直到触发终止条件：

### Step 2 — 训练协议分析
读取并执行 ~/.claude/skills/param-tune/modules/02_protocol.md

若预检发现异常（NaN、梯度消失、初始值异常等），停止循环，等待用户处理。

### Step 3 — 验证评估
读取并执行 ~/.claude/skills/param-tune/modules/03_validation.md

**若尚无任何训练结果文件**（首次运行）：跳过曲线分析，直接进入 Step 4。
此时 reward = null，在全搜索空间内均匀采样生成第一组推荐。

### Step 3b — 可视化生成
读取并执行 ~/.claude/skills/param-tune/modules/05_visualization.md

### Step 4 — 记忆与搜索决策
读取并执行 ~/.claude/skills/param-tune/modules/04_memory.md

### Step 5 — 终止检查
读取 `~/.claude/skills/param-tune/memory/search_state.json` 中的 `termination_triggered`：
- `true` → 执行 **收尾流程**，退出循环
- `false` → 继续 Step 6

**收尾流程**（仅在终止时执行）：

**5a. 将最优参数写入配置文件**

从 `experiment_log.jsonl` 中找出 reward 最大的实验，读取其 `hyperparams`，
将每个 tunable 参数写入 TARGET_DIR 的配置文件（格式自动检测），其余字段保持不变。

写入前显示：
```
最优参数已写入配置文件：
  {param_1} : {旧值} → {新值}
  {param_2} : {旧值} → {新值}
```


### Step 6 — 更新配置并运行训练

**6a. 将推荐参数写入配置文件**

自动检测配置文件格式（YAML / JSON / TOML），
将 Module 4 推荐的每个 tunable 参数写入对应字段。
其余字段保持不变。

写入前显示 diff：
```
即将更新配置文件：
  {param_1} : {旧值} → {新值}
  {param_2} : {旧值} → {新值}
```

**6b. 读取训练脚本入口**

从 `search_state.json` 的 `train_script` 字段读取（由 Module 1 初始化时确定）。

**6c. 检测运行环境**

**步骤 1：判断是否为集群环境**

按优先级检测以下调度器命令是否可用（`which` 或 `command -v`）：

| 调度器 | 检测命令 | 提交命令 | 状态查询命令 |
|--------|----------|----------|--------------|
| SLURM  | `sbatch` | `sbatch {submit_script}` | `squeue -j {jobid} -h` |
| PBS/Torque | `qsub` | `qsub {submit_script}` | `qstat {jobid}` |
| LSF    | `bsub`   | `bsub < {submit_script}` | `bjobs {jobid}` |
| SGE    | `qsub`（区分于 PBS：检查 `qconf` 是否存在） | `qsub {submit_script}` | `qstat -j {jobid}` |

若无任何调度器可用 → 本地环境。

**步骤 2：查找提交脚本**

在 TARGET_DIR 下按优先级查找：
1. `submit_train.sh`
2. `submit.sh`
3. `run.pbs` / `run.lsf` / `run.sge`
4. 任意 `*.sh` 文件中含有调度器指令（`#SBATCH`、`#PBS`、`#BSUB`、`#$`）的文件

若集群环境下找不到提交脚本，提示用户并退出。

**步骤 3：运行训练**

- **集群环境**：

  提交作业并捕获 job ID：
  ```
  SLURM: jobid = sbatch {submit_script} 的输出中提取数字
  PBS:   jobid = qsub {submit_script} 的输出（格式如 12345.hostname）
  LSF:   jobid = bsub < {submit_script} 输出中 "Job <NNNN>" 的数字
  SGE:   jobid = qsub {submit_script} 输出中 "Your job NNNN" 的数字
  ```

  输出：`已提交作业 {jobid}，开始轮询状态...`

  **轮询循环**（每 60 秒一次）：
  ```
  执行状态查询命令：
    SLURM: squeue -j {jobid} -h → 若输出为空，作业已结束
    PBS:   qstat {jobid}        → 若返回非零退出码，作业已结束
    LSF:   bjobs {jobid}        → 若输出含 DONE 或 EXIT，作业已结束
    SGE:   qstat -j {jobid}     → 若返回非零退出码，作业已结束
  ```

  轮询期间每次输出：`[{时间戳}] 作业 {jobid} 运行中，已等待 {N} 分钟...`

  作业结束后：
  1. 读取提交脚本中 `--output`（SLURM）/`-o`（PBS/LSF/SGE）指定的日志文件路径
  2. 检查日志末尾是否含 `Error`、`Traceback`、`CANCELLED`、`FAILED` 等关键词
  3. 若发现报错，输出日志末尾 20 行，停止循环，等待用户处理

- **本地环境**：
  ```bash
  cd TARGET_DIR && python {训练脚本}
  ```
  （若脚本不是 `.py`，去掉 `python` 前缀直接执行）

训练完成后，根据 `metric_config.peak_value.file_pattern` 确认新的训练结果文件已生成。若文件未出现，报告并停止循环。

**6d. 写入本轮运行日志**

将本轮的完整分析结果写入：
`~/.claude/skills/param-tune/memory/run_logs/round_{NNN}.md`

其中 NNN = 当前 experiment_count（三位补零）。若 `run_logs/` 目录不存在，先创建它。

日志内容格式：

```markdown
# Round {NNN} — {ISO timestamp}

## 实验配置
| 参数 | 值 |
|------|----|
| {param_1} | {值} |
| {param_2} | {值} |

## 训练结果
| 指标 | 值 |
|------|----|
| {primary_metric}（train 峰值 epoch 处）| {值} |
| {primary_metric}（末值）| {值} |
| overfitting_train_metric（train 峰值）| {值}（若可获取）|
| overfitting_val_metric（train 峰值对应 epoch）| {值}（若可获取）|
| reward | {值} |

## 曲线诊断
{Module 3 的一句话诊断}

## 搜索状态
- 阶段：{exploration / multi_focused}
- 候选区域：{candidate_id 或 N/A}
- 已完成实验数：{N}
- 当前最优：{exp_id}（reward={值}）

## 下一轮推荐
| 参数 | 值 | 推荐理由 |
|------|----|---------|
| {param_1} | {值} | {理由} |
```

若本轮触发终止，在日志末尾追加：

```markdown
## 终止报告
{全局最优实验的参数和 reward}
```

**6e. 回到 Step 2，开始下一轮**

---

## 并行循环流程（parallel_mode = true 时替代循环流程）

当 `search_phase = multi_focused` 且 `parallel_mode = true` 时，使用以下并行循环，直到触发终止条件。每轮同时为所有活跃候选各提交一个作业。

### P1 — 搜索决策与推荐

**若所有活跃候选的 recent_rewards 均为空（并行首轮，尚无 multi_focused 阶段的实验）**：直接执行 Module 4.5，在各候选 focus 范围内各生成一组推荐配置，跳过 4.2–4.4。

**否则**：执行 Module 4.2–4.5（趋势分析、focus 收窄、终止判断、为所有活跃候选生成推荐配置）。

### P2 — 并行提交

**P2a. 训练脚本 patch 检查（仅首次执行一次）**

搜索 TRAIN_SCRIPT 是否已包含 `--pt-config` 字符串。若不存在，执行以下一次性 patch：

1. 读取 TRAIN_SCRIPT，找到所有 `import` 语句的末尾位置。在其后插入：
   ```python
   # --- param-tune parallel support (auto-generated) ---
   import argparse as _pt_ap
   _pt_p = _pt_ap.ArgumentParser(add_help=False)
   _pt_p.add_argument('--pt-config', default='config.yaml', dest='pt_config')
   _pt_p.add_argument('--pt-run-id', default='', dest='pt_run_id')
   _pt_args, _ = _pt_p.parse_known_args()
   _PT_CONFIG = _pt_args.pt_config
   _PT_RUN_ID = _pt_args.pt_run_id
   # -----------------------------------------------------
   ```

2. 找到 config 文件加载语句（含 `'config.yaml'` 或 `"config.yaml"` 的行），将硬编码路径替换为 `_PT_CONFIG`。

3. 找到 summary 输出文件路径构造处（含 `_summary.json` 或 summary 文件名拼接的行），在文件名前插入候选 id 前缀：
   - 示例：`f"logs/{ts}_summary.json"` → `f"logs/{(_PT_RUN_ID + '_') if _PT_RUN_ID else ''}{ts}_summary.json"`

patch 完成后输出：`[Patch] 已为 {TRAIN_SCRIPT} 添加 --pt-config / --pt-run-id 支持`

**P2b. 生成候选配置与提交脚本**

对每个活跃候选（converged=false），基于 P1 的推荐：

1. 将推荐参数写入 `TARGET_DIR/config_{candidate_id}.yaml`（复制完整 config 结构，仅 tunable 参数替换为推荐值）。

2. 写入候选专属提交脚本 `TARGET_DIR/submit_{candidate_id}.sh`：
   - 复制 submit_script 的全部 `#SBATCH` 指令
   - 将 `--job-name` 改为原名称加 `_{candidate_id}` 后缀
   - 将 `--output` / `--error` 路径改为 `logs/slurm_{candidate_id}_%j.out` / `logs/slurm_{candidate_id}_%j.err`
   - 将执行命令改为：
     ```bash
     cd TARGET_DIR
     python TRAIN_SCRIPT --pt-config config_{candidate_id}.yaml --pt-run-id {candidate_id}
     ```

写入前输出：
```
即将并行提交（本轮 N 个活跃候选）：
  candidate_A : {param_1}={值}, {param_2}={值}, ...
  candidate_B : {param_1}={值}, {param_2}={值}, ...
```

**P2c. 同时提交所有候选作业**

依次执行 `sbatch submit_{candidate_id}.sh`，收集所有 job ID，输出：
```
已并行提交 N 个作业：
  candidate_A → {job_id_A}
  candidate_B → {job_id_B}
```

### P3 — 轮询等待所有作业完成

每 60 秒查询一次所有 job ID 的状态（`squeue -j {id1},{id2} -h`）。每次输出：
```
[时间戳] 运行中 — candidate_A({job_id_A})✓  candidate_B({job_id_B})...，已等待 N 分钟
```
（已完成标 ✓，仍在运行标 ...）

所有作业完成后，逐候选检查对应 SLURM 日志末尾 20 行，若发现 Error / Traceback / FAILED / CANCELLED，输出日志内容，**停止循环，等待用户处理**。

### P4 — 逐候选评估与记录

对每个活跃候选，依次执行：

1. **Step 2**（训练协议分析）：从 `TARGET_DIR/config_{candidate_id}.yaml` 读取本次实验配置。
2. **Step 3**（验证评估）：在 `TARGET_DIR/logs/` 中匹配 `{candidate_id}_*_summary.json`（取最新文件）。
3. **Step 3b**（可视化）：保存图表为 `figures/round_{NNN}_{candidate_id}.png`。
4. **Module 4.1**（记录实验）：将本候选结果追加到 experiment_log.jsonl，`candidate_id` = 当前候选 id。

所有候选均记录完毕后，`experiment_count` 已累计增加 N。

### P5 — 终止检查

读取 `search_state.json` 中的 `termination_triggered`：
- `true` → 执行收尾流程（同 Step 5a），退出循环
- `false` → 继续 P6

### P6 — 写入本轮运行日志

为本轮每个候选写一条日志：
`~/.claude/skills/param-tune/memory/run_logs/round_{NNN}_{candidate_id}.md`

格式同 Step 6d，额外在搜索状态行标注：`并行模式 | 本轮活跃候选数: N`

→ 回到 P1，开始下一轮

---

## 中途修改搜索空间

若用户在循环过程中希望新增或移除 tunable 参数，
无需重启，直接编辑 `~/.claude/skills/param-tune/memory/search_state.json`：
- 在 `tunable_params` 中增删参数名
- 在 `search_space` 中增删对应的范围定义（参考 Module 1 的格式）
下一轮循环读取时自动生效，Module 1 不需要重跑。
