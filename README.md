# MCJSP-DRL 指南

基于 PPO + HGNN 的混合作业车间调度（MCJSP）方案。支持先做初始排产，发生插单 / 故障时用同一策略滚动重排；也可切换到 CP-SAT 兜底。输入 / 输出统一为 JSON / Excel / 张量，便于与生产系统对接。

---


## 项目目标
- 初始排产：在批加工、缓冲、交期、互斥机组、订单屏障等约束下优化 makespan，并兼顾利用率 / 迟交 / WIP。
- 在线滚动重排：基于快照和事件（插单、机故障）快速重排，策略与训练环境一致。
- 兜底求解：CP-SAT 重排提供确定性解或可行性保障，可与 PPO 对比。

---

## 核心能力
- 模型：HGNN 双塔编码（机器 / 工序）+ PPO Actor/Critic，动作空间为 (工序, 机器, 作业) 组合。
- 约束：批加工、缓冲容量、订单屏障、互斥机组、交期惩罚、非法动作惩罚。
- 动态事件：插单 / 故障快照重置，滚动重排；支持 `rush_orders` 直接追加订单。
- 导出 / 可视化：统一导出 Excel / Gantt / JSON，Visdom 实时训练曲线。

---

## 目录与组件速览
- 配置：`config.json`（主配置），`config_annotation.jsonc`（带注释示例）。
- 模型：`PPO_model.py`、`graph/hgnn.py`、`mlp.py`。
- 环境：`env/mcjsp_env.py`、`env/order_case_generator.py`、`env/load_data.py`。
- 训练 / 推理：`train.py`、`validate.py`、`scheduling_plant.py`。
- 重排链路：`one_click_reschedule.py`（集成 PPO / CP-SAT 两种模式）。
- 工具 / 基线：`utils/exporter.py`、`utils/cp_rescheduler.py`、`utils/reschedule.py`、`heuristic_lpt.py`、`random_baseline.py`。

项目结构示意（简化）：
```text
.
├── config.json
├── config_annotation.jsonc
├── train.py / validate.py / scheduling_plant.py
├── env/
│   ├── mcjsp_env.py
│   ├── order_case_generator.py
│   └── load_data.py
├── graph/hgnn.py
├── utils/
│   ├── exporter.py
│   ├── cp_rescheduler.py
│   └── reschedule.py
└── one_click_reschedule.py
```

---

## 配置要点（`config.json`）
- `env_paras`：`batch_size`、`num_mas`、批加工规则、缓冲容量、互斥机组、订单屏障、交期开关与权重、非法动作惩罚、调试开关。
- `model_paras` / `train_paras`：HGNN 维度与层数、PPO 超参、优化器、训练 / 验证频率、Visdom。
- `data_paras`：随机生成范围或固定数据目录。
- `auto_tune`：按 tardy / key_util / wip 目标自动调节奖励权重（需 `collect_debug_stats=true`）。
- `rolling_paras` / `reschedule_defaults`：重排默认路径、CP-SAT / PPO 推理参数（由 `one_click_reschedule.py` 读取）。
---

## 训练与验证流程
- 数据：默认生成，也可用固定 JSON；每 `parallel_iter` 轮刷新 batch。
- Rollout：运行至完成或触发 `max_steps_per_iter`；超步可尝试 `debug_unfinished` / `force_finish`。
- 更新：按 `update_timestep` 触发 PPO 更新，记录 loss / reward / entropy / ratio / adv 等；可用 Visdom。
- 验证：每 `save_timestep` 推理无采样，指标为 `makespan / lb_norm`，自动保存更优模型。
- 自动调优：验证后按 tardy / key_util / wip 目标微调 `w_tardy` / `w_key_util` / `w_wip`。
- 故障排查：可查看 `debug_dump/rollout_dump.pt` 或导出的训练统计。

---

## 数据格式与目录约定
- Case JSON：包含 `routes`（品类工艺与机台 / 时长）、`orders`（family / qty / priority / due_date / release_time）、可选 `jobs`（已展开）与 `precedence_edges`（订单 DAG）；`priority` 数值越小越紧急。
- 排程文件：`utils/exporter.py` 导出的 JSON / Excel 含 `by_machine_rows`，需有 `abs_ope` / `machine` / `start` / `end`（有 `job_id` / `k_rel` 更佳）；`.pt` / `.pth` 张量形状 `[B, num_opes, 4]`，列为 `done_flag, machine, start, end`。
- 快照 / 事件：`{ "t_now": float, "completed_ops": [...], "running_ops": [...], "machine_blocks": [{"machine": i, "until": t}] }`，可带 `rush_orders` 或 `new_jobs`。
- 目录约定：输入默认在 `resched_input/`；推理 / 重排输出写入 `resched_output/` 或 `save/*` 时间戳目录（如 `save/test_*`、`save/rolling_*`、`save/cp_sat_*`）。

### 示例

简化快照 / 事件：
```json
{
  "t_now": 120.0,
  "completed_ops": [{ "job": 0, "k_rel": 0 }],
  "running_ops": [
    { "job": 1, "k_rel": 0, "machine": 2, "start": 110.0, "end": 140.0 }
  ],
  "machine_blocks": [{ "machine": 3, "until": 200.0 }],
  "rush_orders": [
    { "family": "A", "qty": 2, "priority": 1, "release_time": 120.0, "due_date": 260.0 }
  ]
}
```

---



## 初始运行流程
### 步骤 1：训练
- `python train.py`（快速体验可调小 `train_paras.max_iterations`）。

### 步骤 2：推理导出
- 批量：`python scheduling_plant.py --config config.json --model_dir ./save/train_xxx`
- 单例：`python scheduling_plant.py --case ./data_test/foo.json --model_dir ./save/train_xxx`





---
## 动态重排流程
### 步骤 1：准备输入
- Case（订单数据）、旧排程（JSON/Excel/.pt 均可）、当前时间 `t_now`，可选事件/插单（`rush_orders` / `new_jobs`）。

### 步骤 2：执行一键重排
- PPO 重排：`python one_click_reschedule.py --case ... --schedule ... --t_now ... --mode ppo`
- CP-SAT 兜底：`python one_click_reschedule.py --case ... --schedule ... --t_now ... --mode cp_sat`

### 步骤 3：导出与评估
- 自动导出 Excel / Gantt / JSON，`metrics.json` 含 makespan、求解状态等。
### 常用脚本速查
- 训练：`python train.py`
- 推理导出：`python scheduling_plant.py --model_dir ./save/train_xxx`
- 一键重排（CP-SAT 默认，可选 PPO）：`python one_click_reschedule.py --case ... --schedule ... --t_now ... [--mode cp_sat|ppo]`
---

## 参数提示（示例）
- 批加工：21 / 22 / 23 号机批量 2~5，`batch_time_mode = "max"`。
- 缓冲 / 等待：`buffer_capacity = 100`控制凑批等待。
- 互斥：`mutual_exclusive_rules` 限制同组设备同时只允许指定品类，如 `{machines:[21,22,23], categories:["A","B"]}`。
- 订单屏障：`order_barrier_steps` 指定品类在某工序前需完成全部订单，例如 `"A":[8]` 表示 A 品类第 8 道工序前需完成所有 A 订单。
- 订单优先级：`priority` 数值越小越紧急；PPO 可用 `prio_alpha` / `prio_norm` / `prio_small_is_high` 加软偏置，CP-SAT 可设 `cp_objective=priority` 按优先级加权完工。
- 交期：开启 `enable_due_date` 并调 `w_tardy`，归一化方式 `tardy_norm` 可设 `due` / `lb`。
- 奖励：自动调优仅 tardy / key_util / wip。
