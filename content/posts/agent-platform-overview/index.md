---
title: '四智能体协作平台：架构、模块与数据流'
date: 2026-07-14T10:00:00+08:00
draft: false
tags: ['AI Agent', 'LangGraph', 'Architecture']
categories: ['工程实践']
series: ['AI Agent 平台构建']
description: '介绍基于 LangGraph 的 D-B-E-C 四阶段智能体协作平台，包括架构设计、模块职责、数据流与部署方式。'
---

## 背景

在大模型能力快速进化的背景下，一个自然的想法是：**能否让 AI 自动把自然语言需求转化为可运行、可测试的项目代码？** 为此我搭建了一个四智能体协作平台，采用 D-B-E-C（Designer-Developer-Evaluator-Critic）工作流，把“需求 → 设计 → 开发 → 评估 → 改写”这一链路自动化。

## 整体架构

平台采用三层架构：

- **编排层**：LangGraph 实现 D-B-E-C 状态机；
- **执行层**：Celery Worker 异步执行 LLM 调用、Docker 测试；
- **存储层**：Redis（队列/缓存）、Qdrant（向量检索）、SQLite（checkpoint）、本地文件系统（项目代码）。

## 四阶段职责

| 阶段 | 模块 | 职责 |
|------|------|------|
| D（Designer） | `src/core/orchestrator/nodes.py` | 分析需求、检索 RAG 案例、输出 file_plan 与 required_elements |
| B（Developer） | `src/core/B.py` | 按 file_plan 生成代码骨架、填充实现、import 干跑、Docker 测试 |
| E（Evaluator） | `src/core/orchestrator/nodes.py` + evaluator | 对生成代码从五维度评分 |
| C（Critic/Rewriter） | `src/core/orchestrator/nodes.py` | 根据评估结果改写需求或 file_plan，驱动下一轮迭代 |

## 数据流

```text
用户需求
   ↓
Go API Gateway → Celery Producer → Redis Queue
   ↓
Celery Worker (solo 模式) → LangGraph D-B-E-C 编排
   ↓
LLM Provider (Kimi/OpenAI)  ←→  RAG Retriever (Qdrant)
   ↓
生成代码 → 本地 projects/<project>/
   ↓
Docker Sandbox 运行 pytest
   ↓
MultiMetricGate 评估
   ↓
结果写回 task_store
```

## 关键模块

- `src/core/orchestrator/core.py`：LangGraph StateGraph 定义，节点与路由。
- `src/core/B.py`：代码生成核心，包含 file_plan 拓扑排序、chunk 拆分、AST 校验、Docker 测试、`__init__.py` 自动生成。
- `src/core/llm.py`：统一 LLM chat 接口，封装 `max_completion_tokens`、`finish_reason`、token 统计。
- `src/core/docker_sandbox.py`：Docker 容器内运行 pytest，隔离环境。
- `src/vector_store/rag_retriever.py`：基于 Qdrant 的代码/文档检索。
- `src/tasks/agent_task.py`：Celery 任务入口，桥接 LangGraph 与任务队列。
- `api/`：Go 实现的 RESTful Gateway，负责任务提交与状态查询。

## 部署方式

本地开发只需三条命令：

```bash
redis-server
venv/bin/celery -A src.tasks worker -l info -P solo
go run ./api/main.go
```

提交任务：

```bash
curl -X POST http://localhost:8080/api/v1/tasks \
  -H "Content-Type: application/json" \
  -d '{"requirements": "你的需求描述"}'
```

## 生产化建议

- Worker 使用 Linux + Docker，避免 macOS fork+MPS 问题；
- Redis 改为集群或 Sentinel；
- Qdrant 使用服务端模式；
- 增加 metrics 服务与链路追踪；
- HITL 默认批准改为默认拒绝或接入人工审批系统。

## 小结

这个平台本身并不是终点，而是一个**可扩展的代码生成试验床**。后续无论更换更强的 LLM、接入新的 RAG 策略，还是尝试方法级生成，都可以在这个架构上迭代。
