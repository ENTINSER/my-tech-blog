---
title: '情感陪护数字人系统：OpenClaw + Kimi + DeepFace 的 ReAct Agent 实践'
date: 2026-07-17T09:36:58+08:00
draft: false
tags: ['AI Agent', 'Digital Human', 'Emotion Recognition', 'OpenClaw', 'ReAct', 'Kimi']
categories: ['工程实践']
series: ['AI Agent 工程实践']
description: '介绍一个基于数字人和大语言模型的情感陪护系统：采用情绪驱动的 ReAct Agent 架构，集成 OpenClaw Gateway 实现 AI 对话与情绪感知双向通信，支持多用户、多模态感知与连续情感记忆。'
---

## 摘要

过去我独立完成了一套**情感陪护数字人系统**，目标是把“情绪识别 + 大模型对话 + 3D/2D 数字人 + 语音合成”串成一个可运行、可管理、可扩展的完整产品。系统采用**情绪驱动的 ReAct Agent 架构**：感知层融合文本与面部情绪，策略层根据情绪状态动态选择回应策略，执行层调用 Kimi API 生成回复、Edge TTS 合成语音，并通过 **OpenClaw Gateway** 与数字人前端做双向实时通信。

本文将介绍这套系统的核心架构、关键模块设计、OpenClaw 集成方式，以及我在实现过程中遇到的几个真实权衡。

## 1. 为什么做这个项目

大语言模型擅长“对答如流”，但缺少两样东西：

1. **对对话者状态的感知**：它看不到你的表情，也记不住你上周的低落；
2. **持续一致的陪伴体验**：每次对话都是新会话，关系无法累积。

我想做一个更接近“陪伴”而非“问答”的系统：

- 用户说话时，摄像头同时捕捉面部表情；
- 系统把文本情绪与面部情绪融合，判断当前需要“安抚”“共情”“引导”还是“活跃气氛”；
- 大模型根据策略、记忆和情绪上下文生成回复；
- 数字人用语音+表情把回复演出来；
- 整个状态被记录下来，形成连续的情感档案。

## 2. 整体架构

系统采用前后端分离 + 本地 Python 服务 + OpenClaw Gateway 的组合：

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
| 前端 | Next.js 14 + React 18 + Tailwind CSS + Ant Design | 管理后台 + 数字人交互页 |
| 后端 | Node.js + Express + Prisma + SQLite | REST API + WebSocket |
| 情感识别 | Python + Flask + OpenCV + DeepFace | 本地多模态情绪分析 |
| 大模型 | Kimi API (k2p5) | 对话生成与情绪理解 |
| 语音 | Edge TTS / SAG (ElevenLabs) | 文本转语音 |
| 数字人 | Three.js + Live2D + PixiJS | 3D/2D 虚拟形象渲染 |
| 外部集成 | OpenClaw Gateway | WebSocket 双向通信与工具编排 |

## 3. ReAct Agent 核心流程

系统的 Agent 不是简单地把用户输入丢给 LLM，而是分五步走：

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

策略引擎根据融合后的情绪状态选择四种模式之一：

| 策略 | 触发条件 | 回应风格 |
|---|---|---|
| `caring` | 负面情绪 | 关怀、安抚、降低压迫感 |
| `interactive` | 正面情绪 | 轻松、互动、鼓励表达 |
| `guiding` | 中性情绪 | 引导话题、提供建议 |
| `balanced` | 通用/不确定 | 平稳、自然、不偏不倚 |

策略会作为 system prompt 的一部分注入，影响 LLM 的语气和内容走向。

### 3.2 记忆系统

记忆不是简单地把对话历史拼进 prompt，而是分三类：

1. **对话历史**：最近 N 轮上下文；
2. **情绪历史**：每次交互的情绪标签、强度、时间戳；
3. **语义检索记忆**：对长期对话做嵌入，检索与当前话题相关的过往片段。

这样即使单次会话很长，也能在上下文有限的情况下保留“长期关系感”。

## 4. OpenClaw 双向集成

OpenClaw 在本项目里扮演的是**工具编排与双向通信网关**角色。

### 4.1 为什么用 OpenClaw

如果不用网关，系统会变成这样：

- 前端直接连后端 WebSocket；
- 后端再去调 Kimi API；
- 情感核心、TTS、记忆检索各写各的接口；
- 新增一个工具就要改后端路由。

OpenClaw 把这些问题统一了： skills 化封装工具，Gateway 负责与 LLM 的多轮交互，后端只需维护一条 WebSocket 连接。

### 4.2 数据流

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

- **情绪实时同步**：DeepFace 检测到的表情通过 Gateway 注入到 LLM 上下文中；
- **AI 建议注入**：LLM 生成的策略、回复、TTS 参数通过 Gateway 回传给后端和前端。

### 4.3 配置示例

项目的 `openclaw-config.json` 大致结构如下：

```json
{
  "llm": {
    "provider": "openai-compatible",
    "model": "kimi-k2.5",
    "baseUrl": "https://api.moonshot.cn/v1",
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
      "tools": ["memory_search", "memory_update"]
    }
  }
}
```

启动时只需：

```bash
openclaw gateway config.apply openclaw-config.json
openclaw gateway start
```

## 5. 多用户管理

系统支持多用户同时使用，每个用户拥有独立的端口组、配置目录和服务进程。

### 5.1 端口分配策略

- 扫描范围：`7778 - 65000`
- 每组 4 个端口，间隔 100 避免冲突：

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
| 进程 | 每个用户独立启动 OpenClaw / Backend / Frontend / Emotion |
| 配置 | 独立配置目录，路径按 userId 分目录 |
| 数据 | SQLite 按用户分库，或按 userId 字段隔离 |
| 代理 | 统一代理入口 `/user/{userId}/*` 路由到对应端口 |

这种设计让系统可以从“单机个人版”平滑扩展到“多租户服务版”。

## 6. 启动与使用

项目提供了一键启动脚本：

```bash
./scripts/start-all.sh
```

它会自动拉起：

- OpenClaw Gateway（端口 7778）
- 情感识别服务（端口 5002）
- 后端 API（端口 3001）
- 前端管理后台（端口 3000）

启动完成后访问 `http://localhost:3000/dashboard`。

### 6.1 项目结构

```
emotion-system-with-openclaw/
├── frontend/              # Next.js 前端
├── backend/               # Node.js 后端 + Prisma + SQLite
├── emotional-core/        # Python 情感识别服务
├── manager/               # 多用户服务管理
├── skills/                # OpenClaw Skill 扩展
├── scripts/               # 一键启动脚本
├── tests/                 # E2E 测试（Playwright）
├── docs/                  # 项目文档
└── openclaw-config.json   # OpenClaw 网关配置
```

## 7. 实现过程中的几个权衡

### 7.1 本地情绪识别 vs 云端 API

一开始考虑过用云端视觉 API 识别表情，优点是准确率高，缺点是：

- 延迟高（每次截图都要上传）；
- 成本高（视频流持续调用）；
- 隐私风险大。

最终选择本地 **DeepFace**，在 M 系列 Mac 上跑 `Facenet` 轻量模型，单帧推理约 200-500ms，足够实时。

### 7.2 长记忆 vs 短记忆

如果把所有历史对话都塞进 prompt，token 消耗会爆炸。我的做法是：

- **短记忆**：最近 6-10 轮对话直接进上下文；
- **长记忆**：对历史对话做嵌入，按当前话题检索 Top-K 相关片段；
- **情绪摘要**：定期让 LLM 生成一段“用户情绪摘要”，替代原始对话。

### 7.3 语音合成的延迟

Edge TTS 免费且中文效果好，但首次合成有约 1-2s 延迟。为了提升体验，我做了两件事：

1. 在 LLM 生成过程中**预加载** TTS 服务连接；
2. 对常见欢迎语、安慰语做**缓存**，避免重复合成。

## 8. 还能怎么扩展

这套架构可以自然延伸出几个方向：

- **更多感官**：加入语音情绪识别、心率/皮肤电等生理信号；
- **主动关怀**：根据情绪历史和用户画像，在适当时机主动发起对话；
- **个性化数字人**：允许用户自定义形象、音色、性格参数；
- **多语言**：切换 LLM 与 TTS 模型即可支持英文、日文等。

## 9. 总结

这套情感陪护数字人系统本质上是在探索一个问题：**当 LLM 拥有“情绪感知”和“连续记忆”之后，能不能成为一个更像人的陪伴者？**

从工程角度看，它的价值在于把多个独立能力（视觉、语言、语音、数字人、Agent、网关）整合成一个可运行的产品；从产品角度看，它提醒我们：**技术只是载体，真正决定体验的是“策略设计”和“记忆连续性”**。

如果你对这个项目感兴趣，可以在 [Gitee 仓库](https://gitee.com/entinser/emotion-system-with-openclaw) 查看完整代码与文档。
