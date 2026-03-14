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

若预检发现异常（NaN、双重正则化），停止循环，等待用户处理。

### Step 3 — 验证评估
读取并执行 ~/.claude/skills/param-tune/modules/03_validation.md

**若 logs/ 目录下没有任何实验日志**：跳过曲线分析，直接进入 Step 4。
此时 reward = null，为首次运行，直接基于参数名规则生成第一组推荐。

### Step 4 — 记忆与搜索决策
读取并执行 ~/.claude/skills/param-tune/modules/04_memory.md

### Step 5 — 终止检查
读取 `~/.claude/skills/param-tune/memory/search_state.json` 中的 `termination_triggered`：
- `true` → 输出最终报告，退出循环，不再训练
- `false` → 继续 Step 6

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

**6b. 检测训练脚本入口**

按优先级查找：`train.py` → `main.py` → `run.py` → `scripts/train.py`

**6c. 运行训练**

```bash
cd TARGET_DIR && python {训练脚本}
```

训练完成后，确认 logs/ 下出现了新的结果文件。

**6d. 回到 Step 2，开始下一轮**

---

## 中途修改搜索空间

若用户在循环过程中希望新增或移除 tunable 参数（如加入 `hidden_dim`），
无需重启，直接编辑 `~/.claude/skills/param-tune/memory/search_state.json`：
- 在 `tunable_params` 中增删参数名
- 在 `search_space` 中增删对应的范围定义
下一轮循环读取时自动生效，Module 1 不需要重跑。
