---
title: '情感陪护数字人系统：基于 OpenClaw 的多模态 ReAct Agent 设计与实现'
date: 2026-07-17T09:36:58+08:00
draft: false
tags: ['AI Agent', 'Digital Human', 'Emotion Recognition', 'OpenClaw', 'ReAct', 'Kimi']
categories: ['工程实践']
series: ['AI Agent 工程实践']
description: '介绍一套基于数字人与大语言模型的情感陪护系统：采用情绪驱动的 ReAct Agent 架构，集成 OpenClaw Gateway 实现情绪感知与 AI 对话的双向通信，支持文本与面部多模态情感分析、连续情感记忆与多用户隔离部署。'
---

## 摘要

本文介绍一套基于数字人和大语言模型的情感陪护系统。该系统采用**情绪驱动的 ReAct Agent 架构**，通过感知层融合文本情绪与面部情绪，经由策略引擎动态选择回应策略，并调用 Kimi API 生成回复、Edge TTS 合成语音，最终由数字人前端完成语音与表情的同步渲染。系统通过 **OpenClaw Gateway** 实现情绪状态与 AI 决策的双向实时通信，并支持多用户隔离、连续情感记忆与完整的后台管理能力。

系统的主要技术栈包括：Next.js + React 前端、Node.js + Express + Prisma + SQLite 后端、Python + Flask + DeepFace 情感识别服务、Three.js / Live2D 数字人渲染，以及 OpenClaw Gateway 工具编排。项目当前包含 14 个 E2E 测试文件，覆盖 109 个用例。

## 1. 问题背景

大语言模型在自然语言问答任务中表现出色，但在情感陪护场景中面临两个核心局限：

1. **缺乏对对话者状态的感知**：纯文本输入无法获取用户的面部表情、语调、生理信号等多模态信息，导致回复可能忽视用户当前情绪状态。
2. **缺乏连续一致的陪伴体验**：多数对话系统按会话重置上下文，难以形成长期、稳定的用户画像与情感档案，关系连续性较弱。

为缓解上述问题，本系统的设计目标是构建一个多模态、可记忆、可扩展的情感陪护 Agent：在对话过程中持续感知用户情绪，依据情绪状态选择回应策略，并通过数字人形象将回复以语音与表情形式呈现，同时将交互状态持久化以形成连续的情感档案。

## 2. 整体架构

系统采用前后端分离 + 本地 Python 情感服务 + OpenClaw Gateway 的三层架构：

```
┌─────────────────────────────────────────────────────────────┐
│                       用户浏览器                              │
│              数字人页面 (Next.js + Three.js/Live2D)          │
└───────────────────────┬─────────────────────────────────────┘
                        │ WebSocket
┌───────────────────────▼─────────────────────────────────────┐
│                  Node.js 后端服务                             │
│   ReAct Agent：Perception → Memory → Policy → Action → LLM   │
└───────────┬───────────────────────┬─────────────────────────┘
            │ HTTP                  │ WebSocket
┌───────────▼──────────┐   ┌────────▼─────────┐
│  Python 情感核心     │   │ OpenClaw Gateway │
│  DeepFace + OpenCV   │   │ 工具编排 / LLM   │
└──────────────────────┘   └────────┬─────────┘
                                    │
                          ┌─────────▼─────────┐
                          │   Kimi API (k2p5) │
                          └───────────────────┘
```

### 2.1 技术栈

| 层级 | 技术 | 说明 |
|---|---|---|
| 前端 | Next.js 14 + React 18 + Tailwind CSS + Ant Design | 管理后台与数字人交互界面 |
| 后端 | Node.js + Express + Prisma + SQLite | REST API + WebSocket 服务 |
| 情感识别 | Python + Flask + OpenCV + DeepFace | 本地文本与面部多模态情感分析 |
| 大模型 | Kimi API（k2p5，OpenAI 兼容接口） | 对话生成与情绪理解 |
| 语音 | Edge TTS / SAG（ElevenLabs） | 文本转语音 |
| 数字人 | Three.js + Live2D + PixiJS | 3D/2D 虚拟形象渲染 |
| 外部集成 | OpenClaw Gateway | WebSocket 双向通信与工具编排 |

### 2.2 运行环境要求

根据项目文档，系统的运行环境要求如下：

- **Node.js** >= 18.0
- **npm** >= 9.0
- **Python** >= 3.11
- **OpenClaw CLI** 已安装
- **内存**：至少 8GB（用于运行本地情感识别模型）
- **操作系统**：macOS 或 Linux（Windows 需使用 WSL2）

## 3. ReAct Agent 设计

系统的 Agent 并非简单将用户输入转发给 LLM，而是遵循感知 → 记忆 → 策略 → 执行 → 生成的五阶段流程：

```
用户输入 + 面部数据
    ↓
[感知层 Perception]  ← 文本情绪 + 面部情绪
    ↓
[记忆检索 MemoryStore]  ← 历史对话 + 情绪档案
    ↓
[策略引擎 PolicyEngine]  ← 选择当前策略
    ↓
[执行层 ActionExecutor]  ← 预响应动作 + TTS
    ↓
[LLM 生成]  ← Kimi API 生成最终回复
    ↓
[响应输出]  ← 文本 + 语音 + 表情同步
```

### 3.1 情绪策略

策略引擎根据融合后的情绪状态，从以下四种模式中选取一种注入 system prompt：

| 策略 | 触发条件 | 回应风格 |
|---|---|---|
| `caring` | 负面情绪 | 关怀、安抚、降低压迫感 |
| `interactive` | 正面情绪 | 轻松、互动、鼓励表达 |
| `guiding` | 中性情绪 | 引导话题、提供建议 |
| `balanced` | 通用/不确定 | 平稳、自然、不偏不倚 |

策略选择结果作为 system prompt 的一部分，影响 LLM 的语气、情感色彩与内容走向。system prompt 的设计示例如下：

```text
You are an emotional companion digital human. The user's current emotional state is {emotion_state}. 
Adopt the {strategy} response style. Use warm, natural language and consider the conversation history.
```

### 3.2 记忆系统

记忆系统分为三类，以在有限上下文内保留长期关系感：

1. **对话历史**：最近 N 轮对话直接进入上下文窗口。
2. **情绪历史**：每次交互的情绪标签、强度、时间戳构成情绪时间序列。
3. **语义检索记忆**：对历史对话进行嵌入，按当前话题检索 Top-K 相关片段。

通过分层记忆，系统能够在单轮上下文受限的情况下，仍调用与用户相关的历史信息。

## 4. OpenClaw 双向集成

OpenClaw Gateway 在本系统中承担**工具编排与双向通信网关**角色。其核心价值在于将情感识别、TTS、记忆检索、LLM 调用等能力统一封装为 OpenClaw Skill，并通过单一 WebSocket 通道与后端交互，降低新增工具时的接口维护成本。

### 4.1 数据流

```
摄像头面部数据
    ↓ HTTP
Python 情感核心 (DeepFace)
    ↓ WebSocket
OpenClaw Gateway ───────────────→ Kimi API
    ↓ WebSocket                           ↑
Node 后端 ←─────────────────────────────┘
    ↓ WebSocket
数字人前端
```

OpenClaw 实现两个关键能力：

- **情绪实时同步**：DeepFace 检测到的表情数据通过 Gateway 注入 LLM 上下文；
- **AI 建议注入**：LLM 生成的策略、回复文本、TTS 参数通过 Gateway 回传至后端与前端。

### 4.2 配置示例

项目中的 `openclaw-config.json` 主要配置如下：

```json
{
  "llm": {
    "provider": "openai-compatible",
    "model": "k2p5",
    "baseUrl": "https://api.kimi.com/coding",
    "apiKey": "${env.KIMI_API_KEY}",
    "timeoutSeconds": 60,
    "maxTokens": 2048,
    "temperature": 0.7
  },
  "memory": {
    "enabled": true,
    "storage": "file",
    "path": "./memory"
  },
  "agents": {
    "digital-human": {
      "systemPrompt": "...",
      "tools": [
        "llm-kimi",
        "tts-local",
        "emotion-deepface",
        "memory-context",
        "proactive-care"
      ]
    }
  }
}
```

启动命令：

```bash
openclaw gateway config.apply openclaw-config.json
openclaw gateway start
```

### 4.3 集成的 Skill 列表

根据项目配置，OpenClaw 当前启用的 Skill 包括：

| Skill | 功能 |
|---|---|
| `llm-kimi` | 调用 Kimi API 生成对话回复 |
| `tts-local` | 本地 Edge TTS 语音合成 |
| `emotion-deepface` | 本地 DeepFace 面部情绪识别 |
| `memory-context` | 上下文与长期记忆检索 |
| `proactive-care` | 主动关怀触发逻辑 |

## 5. 多用户管理

系统支持多用户同时使用，每个用户拥有独立的端口组、配置目录与服务进程。

### 5.1 端口分配策略

端口扫描范围为 `7778 - 65000`，每组 4 个端口，间隔 100 以避免冲突：

```
基础端口: 7778
├─ OpenClaw: 7778
├─ Backend:  7878 (+100)
├─ Frontend: 7978 (+200)
└─ Emotion:  8078 (+300)
```

### 5.2 用户隔离

| 资源 | 隔离方式 |
|---|---|
| 进程 | 每个用户独立启动 OpenClaw / Backend / Frontend / Emotion 服务 |
| 配置 | 独立配置目录，按 userId 分目录存储 |
| 数据 | SQLite 按用户分库，或按 userId 字段隔离 |
| 代理 | 统一代理入口 `/user/{userId}/*` 路由到对应端口 |

该设计使系统能够从单机个人版平滑扩展到多租户服务版。

## 6. 工程实现与部署

### 6.1 一键启动

项目提供 `scripts/start-all.sh`，启动后会拉起以下服务：

- OpenClaw Gateway（端口 7778）
- 情感识别服务（端口 5002）
- 后端 API（端口 3001）
- 前端管理后台（端口 3000）

启动后访问 `http://localhost:3000/dashboard`。

### 6.2 项目结构

```
emotion-system-with-openclaw/
├── frontend/                  # Next.js 前端应用
├── backend/                   # Node.js 后端服务
├── emotional-core/            # Python 情感识别服务
├── manager/                   # 多用户服务管理
├── skills/                    # OpenClaw Skill 扩展
├── scripts/                   # 一键启动脚本
├── tests/e2e/                 # E2E 测试（Playwright）
├── docs/                      # 项目文档
└── openclaw-config.json       # OpenClaw 网关配置
```

### 6.3 测试覆盖

项目当前使用 Playwright 进行端到端测试，共 14 个测试文件，109 个用例：

| 序号 | 文件 | 功能覆盖 |
|---:|---|---|
| 01 | `basic.spec.js` | 基础页面加载 |
| 02 | `admin.spec.js` | 管理员后台 |
| 03 | `api.spec.js` | 基础 API |
| 04 | `agent.spec.js` | Agent 核心 |
| 05 | `chat.spec.js` | 数字人对话 |
| 06 | `knowledge.spec.js` | 知识库管理 |
| 07 | `strategies.spec.js` | 策略管理 |
| 08 | `user-features.spec.js` | 用户注册与数字人 |
| 09 | `prompts.spec.js` | Prompt 实验室 |
| 10 | `annotations.spec.js` | 数据标注平台 |
| 11 | `metrics.spec.js` | 性能监控 |
| 12 | `debug.spec.js` | Agent 调试控制台 |
| 13 | `emotion-history.spec.js` | 情绪历史查看 |
| 14 | `profile.spec.js` | 个人资料管理 |

## 7. 关键设计权衡

### 7.1 本地情绪识别 vs 云端 API

在设计情感识别模块时，曾对比本地 DeepFace 与云端视觉 API：

| 维度 | 本地 DeepFace | 云端视觉 API |
|---|---|---|
| 隐私性 | 数据不上传 | 需上传视频/图像帧 |
| 成本 | 无按次调用费用 | 按调用量计费 |
| 延迟 | 本地推理，无网络往返 | 受网络与排队影响 |
| 准确率 | 依赖模型与光照条件 | 通常更高 |

最终选择本地 DeepFace，主要基于隐私保护与长期运行成本的考虑。本地部署在 M 系列 Mac 上可运行 `Facenet` 轻量模型，满足实时性要求。

### 7.2 长记忆 vs 短记忆

若将所有历史对话直接拼入 prompt，上下文消耗会随时间线性增长。系统采用分层记忆策略：

- **短记忆**：最近 6–10 轮对话直接进入上下文。
- **长记忆**：对历史对话做嵌入，按当前话题检索相关片段。
- **情绪摘要**：定期由 LLM 生成用户情绪摘要，替代部分原始对话。

该策略在上下文长度与记忆完整性之间取得平衡。

### 7.3 语音合成的延迟优化

Edge TTS 具备免费、中文效果较好的特点，但首次合成存在一定冷启动延迟。为缓解该问题，系统采取两项措施：

1. 在 LLM 生成过程中预建立 TTS 服务连接；
2. 对常见欢迎语、安慰语等高频回复做本地缓存，避免重复合成。

## 8. 局限性与未来工作

### 8.1 当前局限

1. **情感感知维度有限**：当前主要依赖文本情绪与面部情绪，未集成语音情绪、生理信号（心率、皮肤电等）。
2. **情绪识别受环境因素影响**：DeepFace 的准确率受光照、角度、遮挡等因素影响，复杂场景下可能不稳定。
3. **缺乏定量用户研究**：系统当前的评估主要依赖 E2E 测试用例，尚未进行大规模真实用户体验评估或情感陪伴效果度量。
4. **LLM 幻觉风险**：在记忆缺失或情绪推断不确定时，模型可能生成不恰当的回复。

### 8.2 未来方向

- **多模态扩展**：引入语音情绪识别与生理传感器数据，提升情绪判断维度。
- **主动关怀**：基于情绪历史与用户画像，在合适时机主动发起对话。
- **个性化数字人**：支持用户自定义形象、音色与性格参数。
- **多语言支持**：通过切换 LLM 与 TTS 模型支持英文、日文等语言。

## 9. 结论

本文介绍了一套基于数字人和大语言模型的情感陪护系统。该系统通过情绪驱动的 ReAct Agent 架构，将文本/面部情绪感知、动态策略选择、LLM 对话生成、TTS 语音合成与数字人渲染整合为一个可运行、可管理、可扩展的工程产品。OpenClaw Gateway 的使用显著简化了多模块间的双向通信与工具编排，而多用户隔离设计则为后续服务化部署提供了基础。

从工程角度看，该系统的价值在于验证了“多模态感知 + 连续记忆 + 策略化生成”在情感陪护场景中的可行性；从产品设计角度看，其体验上限取决于情绪感知的准确度与记忆系统的连续性。未来可通过引入更多感知维度、优化情绪识别鲁棒性，以及开展真实用户评估来进一步提升系统可用性。

---

## 附录

### A. 技术栈汇总

| 层级 | 技术 | 说明 |
|---|---|---|
| 前端 | Next.js 14 + React 18 + Tailwind CSS + Ant Design | 管理后台与数字人交互界面 |
| 后端 | Node.js + Express + Prisma + SQLite | REST API + WebSocket 服务 |
| 情感识别 | Python + Flask + OpenCV + DeepFace | 本地多模态情感分析 |
| 大模型 | Kimi API（k2p5） | 对话生成与情绪理解 |
| 语音 | Edge TTS / SAG（ElevenLabs） | 文本转语音 |
| 数字人 | Three.js + Live2D + PixiJS | 3D/2D 虚拟形象渲染 |
| 外部集成 | OpenClaw Gateway | WebSocket 双向通信与工具编排 |

### B. 运行环境要求

- Node.js >= 18.0
- npm >= 9.0
- Python >= 3.11
- OpenClaw CLI
- 内存 >= 8GB
- macOS / Linux（Windows 使用 WSL2）

### C. E2E 测试覆盖

- 测试文件数：14
- 测试用例数：109
- 测试框架：Playwright
