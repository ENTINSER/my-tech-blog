---
title: '四智能体协作平台：基于 LangGraph 的 D-B-E-C 代码生成系统架构'
date: 2026-07-14T10:00:00+08:00
draft: false
tags: ['AI Agent', 'LangGraph', 'Architecture', 'Code Generation', 'Celery']
categories: ['工程实践']
series: ['AI Agent 平台构建']
description: '介绍基于 LangGraph 的 D-B-E-C 四阶段智能体协作平台，用于将自然语言需求转化为可运行、可测试的项目代码。文章阐述平台的分层架构、四阶段状态机、关键模块职责、数据流与部署方式，并讨论当前局限与未来工作。'
---

## 摘要

本文介绍一套基于 LangGraph 的四智能体协作平台，采用 D-B-E-C（Designer-Developer-Evaluator-Critic）四阶段工作流，目标是将自然语言需求自动转化为可运行、可测试的项目代码。平台采用三层架构：LangGraph 编排层负责状态机与 Agent 路由，Celery Worker 执行层负责异步 LLM 调用与 Docker 测试，Redis/Qdrant/SQLite 构成存储层。文章详细阐述四个阶段的设计意图、关键模块职责、端到端数据流、本地部署方式以及生产化建议。

## 1. 问题背景

随着大语言模型代码生成能力的提升，将自然语言需求直接转化为可运行项目代码成为可能。然而，单次 LLM 调用难以同时完成需求分析、架构设计、代码实现、测试验证与错误修复等多重任务。主要原因包括：

1. **上下文长度有限**：单次生成难以覆盖复杂项目的全部文件与约束。
2. **多能力耦合**：设计、开发、评估、改写需要不同的 prompt 策略与工具链。
3. **错误累积**：代码生成中的单个错误可能在后续文件中被放大，需要反馈循环进行修正。

为解决上述问题，平台将代码生成过程拆分为四个专职 Agent，通过 LangGraph 状态机进行编排，形成“需求 → 设计 → 开发 → 评估 → 改写”的闭环。

## 2. 设计目标

平台的核心设计目标包括：

- **职责分离**：每个 Agent 只负责一个明确阶段，降低单阶段 prompt 复杂度。
- **可验证性**：每轮开发完成后通过 Docker 运行 pytest，获得客观反馈。
- **可迭代性**：Evaluator 与 Critic 的输出驱动下一轮设计或开发，支持多轮改写。
- **可扩展性**：通过 Celery 任务队列与模块化设计，支持水平扩展与新 Agent 接入。
- **可观测性**：通过 task_store 与日志记录完整生成轨迹，便于调试与审计。

## 3. 整体架构

平台采用经典的三层架构：

```text
┌───────────────────────────────────────────────┐
│                 用户 / 客户端                   │
└───────────────────┬───────────────────────────┘
                    │ HTTP
┌───────────────────▼───────────────────────────┐
│              Go API Gateway                   │
│         任务提交 / 状态查询 / 结果获取          │
└───────────────────┬───────────────────────────┘
                    │
┌───────────────────▼───────────────────────────┐
│           Celery + Redis 任务队列             │
└───────────────────┬───────────────────────────┘
                    │
┌───────────────────▼───────────────────────────┐
│            Celery Worker (solo)               │
│   LangGraph D-B-E-C 状态机编排                │
│   ├─ Designer (D)                             │
│   ├─ Developer (B)                            │
│   ├─ Evaluator (E)                            │
│   └─ Critic (C)                               │
└───────────┬───────────────────────────────────┘
            │ LLM / RAG / Docker
┌───────────▼───────────────────────────────────┐
│  Kimi/OpenAI API  ←→  Qdrant RAG  ←→  Docker  │
└───────────────────────────────────────────────┘
```

### 3.1 技术栈

| 层级 | 技术 | 说明 |
|---|---|---|
| API 网关 | Go | RESTful 任务提交与状态查询 |
| 任务队列 | Celery + Redis | 异步任务调度与结果缓存 |
| 编排引擎 | LangGraph | D-B-E-C 状态机定义与路由 |
| 代码生成 | Kimi / OpenAI API | LLM 驱动的设计与开发 |
| RAG 检索 | Qdrant | 代码示例与文档向量检索 |
| 测试沙箱 | Docker | 隔离运行 pytest |
| 持久化 | SQLite / 本地文件系统 | checkpoint 与生成代码 |

## 4. D-B-E-C 四阶段工作流

四个阶段构成一个可循环的状态机：

```
          ┌─────────────┐
          │   Start     │
          └──────┬──────┘
                 ↓
          ┌─────────────┐
          │  Designer   │
          │  (D) 设计    │
          └──────┬──────┘
                 ↓
          ┌─────────────┐
          │  Developer  │
          │  (B) 开发    │
          └──────┬──────┘
                 ↓
          ┌─────────────┐
          │  Evaluator  │
          │  (E) 评估    │
          └──────┬──────┘
                 ↓
       ┌─────────────────┐
       │  是否满足退出条件？ │
       └────────┬────────┘
                │
       ┌────────┴────────┐
       ↓                 ↓
   ┌─────────┐      ┌──────────┐
   │  Finish │      │  Critic  │
   │  结束    │      │ (C) 改写  │
   └─────────┘      └────┬─────┘
                         │
                         └────→ Designer / Developer
```

### 4.1 各阶段职责

| 阶段 | 模块 | 输入 | 输出 | 核心职责 |
|---|---|---|---|---|
| D（Designer） | `src/core/orchestrator/nodes.py` | 用户需求、RAG 案例 | `file_plan`、`required_elements` | 分析需求、检索相似案例、输出文件计划与必需元素 |
| B（Developer） | `src/core/B.py` | `file_plan`、必需元素 | 项目代码、`__init__.py` | 按文件计划生成代码骨架、填充实现、AST 校验、Docker 测试 |
| E（Evaluator） | `src/core/orchestrator/nodes.py` + evaluator | 生成代码、pytest 结果 | 多维度评分、失败项 | 从多个维度评估代码质量 |
| C（Critic/Rewriter） | `src/core/orchestrator/nodes.py` | 评分与失败项 | 改写后的需求或 `file_plan` | 根据评估结果驱动下一轮迭代 |

### 4.2 状态转移条件

- **D → B**：Designer 成功输出 `file_plan`。
- **B → E**：Developer 完成代码生成并通过 import 干跑。
- **E → Finish**：评分满足退出阈值且无失败项。
- **E → C**：评分未达标或存在失败项。
- **C → D/B**：Critic 判断需要修改设计或重新生成代码。

平台支持设置最大迭代次数，避免无限循环。

## 5. 关键模块职责

### 5.1 `src/core/orchestrator/core.py`

LangGraph `StateGraph` 的定义文件，负责：

- 定义全局状态结构（需求、file_plan、代码、评分、迭代次数等）；
- 注册 D/B/E/C 四个节点；
- 定义条件边与循环边，实现状态机路由；
- 管理 checkpoint，支持失败恢复与轨迹回放。

### 5.2 `src/core/B.py`

代码生成核心模块，职责包括：

- **file_plan 拓扑排序**：按依赖关系确定文件生成顺序；
- **chunk 拆分**：将大文件拆分为多次生成任务；
- **AST 校验**：检测语法错误、未定义符号、空方法体；
- **Docker 测试**：在隔离容器中运行 pytest；
- **`__init__.py` 自动生成**：扫描 `src/*.py` 顶层定义，自动导出符号。

### 5.3 `src/core/llm.py`

统一 LLM 调用接口，封装：

- `max_completion_tokens` 控制；
- `finish_reason` 检查；
- Token 使用统计；
- 多 provider 切换（Kimi / OpenAI）。

### 5.4 `src/core/docker_sandbox.py`

Docker 测试沙箱，职责包括：

- 为每次代码生成创建隔离容器；
- 安装依赖并运行 pytest；
- 捕获 stdout/stderr 与测试结果；
- 清理容器与临时文件。

### 5.5 `src/vector_store/rag_retriever.py`

基于 Qdrant 的向量检索模块，用于：

- 存储历史成功案例的代码与需求嵌入；
- 在 Designer 阶段检索相似需求，辅助 file_plan 生成；
- 在 Developer 阶段检索相关代码片段，作为 few-shot 示例。

### 5.6 `src/tasks/agent_task.py`

Celery 任务入口，职责包括：

- 接收 API Gateway 提交的任务；
- 启动 LangGraph 编排流程；
- 将中间状态与最终结果写回 `task_store`；
- 处理超时、重试与异常。

### 5.7 `api/`

Go 实现的 RESTful Gateway，提供：

- `POST /api/v1/tasks`：提交新任务；
- `GET /api/v1/tasks/{id}`：查询任务状态；
- `GET /api/v1/tasks/{id}/result`：获取生成结果。

## 6. 端到端数据流

一次完整的代码生成任务数据流如下：

```text
1. 用户通过 curl / UI 提交需求
   ↓
2. Go API Gateway 将请求写入 Redis 任务队列
   ↓
3. Celery Worker 消费任务，启动 LangGraph D-B-E-C 状态机
   ↓
4. Designer 检索 Qdrant 中的相似案例，输出 file_plan
   ↓
5. Developer 按 file_plan 逐文件生成代码
   ↓
6. import 干跑 → AST 校验 → Docker 中运行 pytest
   ↓
7. Evaluator 根据 pytest 结果与代码质量多维度评分
   ↓
8. 若未达标，Critic 生成改写建议，返回 Designer 或 Developer
   ↓
9. 若达标，生成结果打包并写回 task_store
   ↓
10. 用户通过 API 查询并下载生成项目
```

## 7. 部署方式

### 7.1 本地开发部署

本地开发需要启动三个服务：

```bash
# 1. 启动 Redis
redis-server

# 2. 启动 Celery Worker（solo 模式避免 macOS fork 问题）
venv/bin/celery -A src.tasks worker -l info -P solo

# 3. 启动 Go API Gateway
go run ./api/main.go
```

提交任务示例：

```bash
curl -X POST http://localhost:8080/api/v1/tasks \
  -H "Content-Type: application/json" \
  -d '{"requirements": "一个支持用户注册和登录的 Flask 博客系统"}'
```

### 7.2 生产化建议

- **Worker 环境**：使用 Linux + Docker，避免 macOS 下 fork 与 MPS 的兼容性问题；
- **Redis**：由单机实例升级为集群或 Sentinel 模式，提高可用性；
- **Qdrant**：使用服务端部署，替代嵌入式模式，便于共享检索索引；
- **可观测性**：增加 metrics 服务（如 Prometheus）与分布式链路追踪；
- **人工审批**：将 HITL（Human-in-the-Loop）默认批准改为默认拒绝，或接入人工审批系统；
- **资源隔离**：为 Docker 测试沙箱设置 CPU/内存上限，防止恶意或错误代码耗尽资源。

## 8. 局限性与未来工作

### 8.1 当前局限

1. **模型对精确逻辑的遵循能力有限**：当前模型在状态机循环、异常条件、精确阈值控制等方面容易出错。
2. **跨文件一致性维护困难**：多文件接口约定需要规则引擎兜底，LLM 难以在长时间生成中保持一致。
3. **缺少大规模定量评估**：平台目前缺少对生成成功率、代码质量、任务耗时的系统性基准测试。
4. **RAG 案例依赖历史数据**：初期案例库较小时，检索对设计阶段的辅助作用有限。

### 8.2 未来方向

- **方法级生成**：从文件级生成细粒度到方法级生成，降低单次输出长度与错误率；
- **RAG 增强**：引入更多高质量代码示例与 LangGraph 官方文档；
- **结构化反思**：让 Critic 输出分类化的失败原因（如接口不一致、逻辑错误、测试缺失），便于针对性改写；
- **模型升级**：在更强模型上验证当前架构的 ceiling；
- **基准测试**：建立固定任务集，持续评估不同模型与配置下的成功率与耗时。

## 9. 结论

本文介绍了一套基于 LangGraph 的 D-B-E-C 四智能体协作平台。通过将代码生成任务拆分为设计、开发、评估、改写四个阶段，并辅以 Docker 测试、RAG 检索与 Celery 异步调度，平台实现了从自然语言需求到可运行项目代码的自动化链路。

该平台的价值在于提供了一个可扩展的代码生成试验床：无论是更换更强的 LLM、接入新的 RAG 策略，还是尝试更细粒度的方法级生成，都可以在当前架构上迭代。其当前瓶颈主要在于模型对精确逻辑与跨文件一致性的遵循能力，而非架构本身。

---

## 附录

### A. 模块职责汇总

| 模块 | 职责 |
|---|---|
| `src/core/orchestrator/core.py` | LangGraph 状态机定义与路由 |
| `src/core/orchestrator/nodes.py` | Designer、Evaluator、Critic 节点实现 |
| `src/core/B.py` | Developer 代码生成、AST 校验、Docker 测试 |
| `src/core/llm.py` | 统一 LLM 调用接口与 Token 统计 |
| `src/core/docker_sandbox.py` | Docker 中隔离运行 pytest |
| `src/vector_store/rag_retriever.py` | Qdrant 向量检索 |
| `src/tasks/agent_task.py` | Celery 任务入口 |
| `api/main.go` | Go RESTful Gateway |

### B. 本地启动命令

```bash
redis-server
venv/bin/celery -A src.tasks worker -l info -P solo
go run ./api/main.go
```

### C. 技术栈

| 层级 | 技术 |
|---|---|
| API 网关 | Go |
| 任务队列 | Celery + Redis |
| 编排引擎 | LangGraph |
| 代码生成 | Kimi / OpenAI API |
| RAG 检索 | Qdrant |
| 测试沙箱 | Docker |
| 持久化 | SQLite + 本地文件系统 |
