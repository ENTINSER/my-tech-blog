---
title: 'MiniTrain：基于 LoRA 的代码修复模型微调与 MLOps 实践'
date: 2026-07-19T11:00:00+08:00
draft: false
tags: ['LLM', 'LoRA', 'MLX', 'MLOps', 'MLflow']
categories: ['工程实践']
series: ['AI Agent 平台构建']
description: '介绍 MiniTrain：一个面向代码修复场景的大模型微调与发布平台。文章覆盖数据清洗（MinHash 去重、质量过滤）、Apple Silicon 上的 MLX LoRA 微调、模型评估与 MLflow 注册、Nginx 灰度发布与 A/B 测试，并给出在 Qwen2.5-1.5B-Instruct-4bit 上的实测数据。'
---

## 摘要

MiniTrain 是一个被设计用来“自愈”的 LLM 微调平台：当上游系统（如 [Agent Factory](/posts/agent-factory-distributed-agent-platform/)）检测到某类代码错误修复准确率下降时，MiniTrain 会接收触发、自动完成数据清洗 → LoRA 微调 → 评估 → MLflow 注册 → 灰度发布 → A/B 测试的完整闭环。本文以 Apple Silicon + MLX 为例，展示一个最小但完整的代码修复模型微调实验。

## 1. 为什么需要 MiniTrain

在 Agent Factory 运行一段时间后，我们发现两类问题：

1. **模式固化**：通用 base 模型对训练数据之外的新错误模式（如特定项目的命名约定、特定库的异常处理）修复率会下降。
2. **反馈闭环缺失**：评估服务已经知道哪些任务做错了，但这些信号没有回流到模型参数中。

MiniTrain 的目标就是把“评估信号”变成“新的 LoRA 适配器”，并通过灰度发布控制风险。

## 2. 整体流水线

```text
┌─────────────────────────────────────────────────────────────────────────────┐
│                              Agent Factory                                   │
│  (detects accuracy drop / recurring error type)                              │
└──────────────────────────────┬──────────────────────────────────────────────┘
                               │ POST /train-jobs
                               ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                         MiniTrain API (api/main.py)                          │
│   • Accepts training job requests                                            │
│   • Orchestrates data → train → eval → registry pipeline                     │
│   • Exposes /health, /train-jobs, /models, /metrics                          │
└──────────────────────────────┬──────────────────────────────────────────────┘
                               │ subprocess calls
                               ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│  data.pipeline ──► train.train ──► evaluate.benchmark ──► registry.mlflow   │
│                                                                              │
│  • Load raw data (file/PostgreSQL)   • LoRA fine-tuning       • Pass/fail   │
│  • MinHash dedup                     • DeepSpeed or PyTorch     • MLflow     │
│  • Quality/outlier filter            • Save final adapter       • register   │
└──────────────────────────────┬──────────────────────────────────────────────┘
                               │ registered model
                               ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│  deploy.nginx_config_generator          deploy.ab_test                       │
│  • Gray-release upstream weights        • Fisher / chi-squared test          │
│  • Stage plan (10% → 50% → 100%)        • Effect size & recommendation       │
└─────────────────────────────────────────────────────────────────────────────┘
```

## 3. 数据清洗：MinHash + 质量过滤

`data/pipeline.py` 负责把原始记录转换为 instruction/response 格式的训练集：

1. **加载**：支持 JSON/JSONL/CSV 文件或 PostgreSQL 查询。
2. **去重**：使用 MinHash + LSH 对 `buggy_code` 做近重复检测，阈值 0.8。
3. **质量过滤**：基于 expected_behavior 长度、buggy_code 可解析性、描述清晰度打分。
4. **异常值过滤**：保留代码行数在 3–200 之间的样本。
5. **拆分**：按 8:1:1 划分为 train/val/test。

在我们的实验中，共合并了 50 条样本：

| 来源 | 数量 |
|---|---|
| MiniTrain 自带样本 | 10 |
| CODESKILL 实验成功修复对 | 22 |
| 规则补充样本 | 18 |
| **最终训练集** | **40** |
| 验证集 / 测试集 | 5 / 5 |

## 4. MLX LoRA 微调实战

由于本地环境是 Apple Silicon，我们使用 `mlx-lm` 在 MLX 后端上直接微调，避免了 CUDA 依赖。

### 4.1 训练配置

```yaml
model: "mlx-community/Qwen2.5-1.5B-Instruct-4bit"
batch_size: 2
grad_accumulation_steps: 2
learning_rate: 2.0e-4
iters: 120
max_seq_length: 512
adapter_path: "checkpoints/lora-codefix"
lora_parameters:
  rank: 8
  scale: 2.0
  dropout: 0.05
  keys:
    - "self_attn.q_proj"
    - "self_attn.k_proj"
    - "self_attn.v_proj"
    - "self_attn.o_proj"
```

说明：`mlx-lm` 以 `iters` 控制总前向次数。在 `batch_size=2`、`grad_accumulation_steps=2` 的配置下，每 20 iters 约完成一次对 40 条训练样本的全量遍历。

### 4.2 训练结果

| 指标 | 数值 |
|---|---|
| 可训练参数 | 1.245M / 1543.714M（0.081%） |
| 初始 Loss | 2.311 |
| 最终 Loss | 0.108 |
| 测试 PPL | 1.652 |
| 总训练时间 | 约 74.6 秒（1.24 分钟） |
| 每轮遍历耗时 | 约 12.4 秒 |

```text
Iter 5:   Train loss 2.311
Iter 30:  Train loss 0.378
Iter 60:  Train loss 0.225
Iter 90:  Train loss 0.117
Iter 120: Train loss 0.108
```

## 5. 模型评估与对比

我们在 20 条固定测试用例上对比了 base 模型与 LoRA 适配器，使用贪婪解码（temperature=0）保证可复现。

| 指标 | Base 模型 | LoRA 微调后 | 变化 |
|---|---|---|---|
| 功能正确率 | 70.0% | 75.0% | +5.0% |
| 代码质量分 | 100.0 | 100.0 | 0.0 |
| 困惑度 PPL | 3.815 | 2.854 | -0.961 |

虽然 base 模型本身已经能处理大部分简单错误，但微调后仍在部分 null-pointer 与 type-error 用例上表现出更稳定的修复能力，同时困惑度明显下降，说明模型对代码修复指令的分布拟合更好。

## 6. MLflow 注册与灰度发布

微调完成后，通过 `registry/mlflow_integration.py` 注册到 MLflow：

```bash
python -m registry.mlflow_integration register \
  --model-path checkpoints/lora-codefix \
  --eval-report evaluate/benchmark_result.json
```

注册信息：

- 模型名：`code-fix-lora`
- 最新版本：version 2
- Tracking URI：`file:///Users/mingrun/minitrain/mlruns`
- 记录参数：base_model、lora_r、lora_alpha、learning_rate、training_samples 等
- 记录指标：accuracy、code_quality、ppl、accuracy_delta、training_time_minutes

随后使用 `deploy.nginx_config_generator` 生成三阶段灰度配置：

```text
Stage 1: 10% new model for 2h   (old weight 90, new weight 10)
Stage 2: 50% new model for 6h   (old weight 50, new weight 50)
Stage 3: 100% new model steady  (old weight 0,  new weight 100)
```

灰度期间可以用 `deploy.ab_test` 对控制组与实验组进行 Fisher 精确检验或卡方检验，得到 p-value、Cohen's h 效应量与是否继续灰度的建议。

## 7. 与 Agent Factory 的联动

Agent Factory 的 evaluation-service 在检测到某类错误平均分低于阈值时，会向 MiniTrain 发送：

```json
{
  "error_type": "TypeError",
  "avg_score": 62.5,
  "sample_count": 10,
  "trigger_time": "2026-07-19T08:11:53+00:00"
}
```

MiniTrain 通过新增的 `/training-trigger` 兼容入口，自动将其映射为完整的 `TrainJobRequest`，从而在不修改 Agent Factory 的前提下完成端到端联动。

## 8. 总结

MiniTrain 把“代码修复模型效果下降”这一问题，转化为一条可自动运行的 LoRA 微调 + 注册 + 灰度流水线。在 Apple Silicon 上，仅用约 1.24 分钟、1.245M 可训练参数，就把测试集困惑度从 3.82 降到 2.85，并将功能正确率从 70% 提升到 75%。对于需要在资源受限环境中快速迭代模型的团队，MLX + LoRA + MLflow 是一条值得尝试的轻量路径。

## 相关项目

- [Agent Factory：分布式 Agent 推理与任务编排平台](/posts/agent-factory-distributed-agent-platform/)
