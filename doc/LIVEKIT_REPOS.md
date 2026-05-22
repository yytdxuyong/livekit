# LiveKit 生态 GitHub 仓库全景

本文档汇总 LiveKit 生态中所有相关的 GitHub 仓库、本地方位及其功能说明。

---

## 目录

1. [核心服务（Server-side）](#1-核心服务server-side)
2. [Agent 框架](#2-agent-框架)
3. [核心协议与基础库](#3-核心协议与基础库)
4. [Server SDK](#4-server-sdk)
5. [客户端 SDK（Client-side）](#5-客户端-sdkclient-side)
6. [UI 组件](#6-ui-组件)
7. [Agent 插件](#7-agent-插件)
8. [CLI 与 DevOps](#8-cli-与-devops)
9. [示例项目](#9-示例项目)
10. [文档与网站](#10-文档与网站)
11. [本地代码与外部仓库的关系](#11-本地代码与外部仓库的关系)
12. [生产部署服务一览](#12-生产部署服务一览)

---

## 1. 核心服务（Server-side）

| 仓库 | 说明 | 本地方位 |
|------|------|---------|
| [livekit/livekit](https://github.com/livekit/livekit) | **LiveKit Server** — 核心 WebRTC 媒体服务器，提供房间管理、参与者管理、音视频路由等功能 | `livekit/` |
| [livekit/sip](https://github.com/livekit/sip) | **SIP Bridge** — SIP ↔ WebRTC 桥接服务，将传统电话网络（PSTN）与 LiveKit 房间互通 | `livekit-sip/` |
| [livekit/egress](https://github.com/livekit/egress) | **Egress** — 录制/推流服务，将房间音视频录制为文件或推流到 RTMP | 独立仓库（Go module 依赖） |
| [livekit/ingress](https://github.com/livekit/ingress) | **Ingress** — 外部媒体输入服务，接收 RTMP、WHIP 等外部流输入到 LiveKit 房间 | 独立仓库 |

### 本地代码概述

```
d:\xuyong\source\AITest\livekit\
├── livekit/         ← github.com/livekit/livekit 主服务
├── livekit-sip/     ← github.com/livekit/sip SIP 桥接
└── livekit-agents/  ← github.com/livekit/agents Agent 框架
```

---

## 2. Agent 框架

| 仓库 | 说明 | 本地方位 |
|------|------|---------|
| [livekit/agents](https://github.com/livekit/agents) | **Agents Python SDK** — AI Agent 框架，支持 STT/LLM/TTS 流水线，多 Agent 协作 | `livekit-agents/` |
| [livekit/agents-js](https://github.com/livekit/agents-js) | **Agents Node.js SDK** — TypeScript 版 Agent 框架 | 独立仓库 |
| [livekit/agent-skills](https://github.com/livekit/agent-skills) | **Agent Skills** — AI 编码助手技能包，为 Cursor/Copilot 提供 Agent 开发最佳实践 | 独立仓库 |

---

## 3. 核心协议与基础库

这些是 LiveKit 生态的底层基础设施，作为 Go module 依赖引入。

| 仓库 | 说明 | 依赖来源 |
|------|------|---------|
| [livekit/protocol](https://github.com/livekit/protocol) | **协议定义** — Protobuf 定义（livekit_models.proto、livekit_room.proto 等）、rpc 生成代码、工具库（logger、guid、tracer） | `livekit`、`livekit-sip` |
| [livekit/psrpc](https://github.com/livekit/psrpc) | **psrpc** — 基于 Redis Pub/Sub 的异步 RPC 框架，用于 LiveKit Server 与各服务间的通信 | `livekit`、`livekit-sip` |
| [livekit/sipgo](https://github.com/livekit/sipgo) | **sipgo** — SIP 协议栈（Go 实现，livekit fork），基于 emiago/sipgo 改造，添加事务层独立包和 RFC 6026 支持 | `livekit-sip` |
| [livekit/media-sdk](https://github.com/livekit/media-sdk) | **media-sdk** — 媒体处理库，包含 RTP 收发、SRTP 加密、DTMF 检测、Opus 编解码（cgo/libopus）、混音器、SDP 交换 | `livekit-sip` |
| [livekit/mediatransportutil](https://github.com/livekit/mediatransportutil) | **mediatransportutil** — 媒体传输工具库，包含端口范围、ICE 配置等 | `livekit`、`livekit-sip` |

### 在 go.mod 中的依赖声明

```go
// livekit-sip/go.mod
github.com/livekit/protocol v1.45.9-...
github.com/livekit/psrpc v0.7.1
github.com/livekit/sipgo v0.13.2-...
github.com/livekit/media-sdk v0.0.0-...
github.com/livekit/mediatransportutil v0.0.0-...
github.com/livekit/server-sdk-go/v2 v2.16.4-...
github.com/livekit/mageutil v0.0.0-...
```

---

## 4. Server SDK

| 仓库 | 说明 |
|------|------|
| [livekit/server-sdk-go](https://github.com/livekit/server-sdk-go) | **Go Server SDK** — 用 Go 语言调用 LiveKit API |
| [livekit/node-sdks](https://github.com/livekit/node-sdks) | **Node.js Server SDK** |
| [livekit/python-sdks](https://github.com/livekit/python-sdks) | **Python Server SDK** |
| [livekit/rust-sdks](https://github.com/livekit/rust-sdks) | **Rust Server SDK** |
| [livekit/server-sdk-ruby](https://github.com/livekit/server-sdk-ruby) | Ruby Server SDK |
| [livekit/server-sdk-kotlin](https://github.com/livekit/server-sdk-kotlin) | Java/Kotlin Server SDK |
| [agence104/livekit-server-sdk-php](https://github.com/agence104/livekit-server-sdk-php) | PHP Server SDK（社区维护） |
| [pabloFuente/livekit-server-sdk-dotnet](https://github.com/pabloFuente/livekit-server-sdk-dotnet) | .NET Server SDK（社区维护） |

---

## 5. 客户端 SDK（Client-side）

| 仓库 | 平台 | 语言 |
|------|------|------|
| [livekit/client-sdk-js](https://github.com/livekit/client-sdk-js) | **Browser** (Web) | JavaScript/TypeScript |
| [livekit/client-sdk-swift](https://github.com/livekit/client-sdk-swift) | **Swift** (iOS/macOS) | Swift |
| [livekit/client-sdk-android](https://github.com/livekit/client-sdk-android) | **Android** | Kotlin/Java |
| [livekit/client-sdk-flutter](https://github.com/livekit/client-sdk-flutter) | **Flutter** (跨平台) | Dart |
| [livekit/client-sdk-react-native](https://github.com/livekit/client-sdk-react-native) | **React Native** | TypeScript |
| [livekit/client-sdk-unity](https://github.com/livekit/client-sdk-unity) | **Unity** (游戏引擎) | C# |
| [livekit/client-sdk-unity-web](https://github.com/livekit/client-sdk-unity-web) | **Unity WebGL** | C# |
| [livekit/rust-sdks](https://github.com/livekit/rust-sdks) | **Rust** | Rust |
| [livekit/node-sdks](https://github.com/livekit/node-sdks) | **Node.js** | JavaScript/TypeScript |
| [livekit/python-sdks](https://github.com/livekit/python-sdks) | **Python** | Python |
| [livekit/client-sdk-cpp](https://github.com/livekit/client-sdk-cpp) | **C++** | C++ |
| [livekit/client-sdk-esp32](https://github.com/livekit/client-sdk-esp32) | **ESP32** (物联网) | C++ |

---

## 6. UI 组件

| 仓库 | 说明 |
|------|------|
| [livekit/components-js](https://github.com/livekit/components-js) | React UI 组件（VideoConference、ParticipantTile、ControlBar 等） |
| [livekit/components-android](https://github.com/livekit/components-android) | Android Compose UI 组件 |
| [livekit/components-swift](https://github.com/livekit/components-swift) | SwiftUI UI 组件 |
| [livekit/components-flutter](https://github.com/livekit/components-flutter) | Flutter UI 组件 |

---

## 7. Agent 插件

以下插件均在 `livekit-agents/livekit-plugins/` 目录下，对应 `livekit/agents-plugins` 仓库中的子包。

### 7.1 LLM（大语言模型）

| 插件目录 | 提供商 | 功能 |
|---------|--------|------|
| `livekit-plugins-openai` | OpenAI | GPT 系列、Realtime API |
| `livekit-plugins-anthropic` | Anthropic | Claude 系列 |
| `livekit-plugins-google` | Google | Gemini 系列 |
| `livekit-plugins-aws` | AWS | Amazon Bedrock / Nova |
| `livekit-plugins-azure` | Azure | Azure OpenAI |
| `livekit-plugins-deep-seek` | DeepSeek | DeepSeek V3/R1 |
| `livekit-plugins-groq` | Groq | 快推理 LLM |
| `livekit-plugins-cerebras` | Cerebras | Cerebras LLM |
| `livekit-plugins-baseten` | Baseten | Baseten 托管模型 |
| `livekit-plugins-nvidia` | NVIDIA | NVIDIA Nemo |

### 7.2 STT（语音转文本）

| 插件目录 | 提供商 |
|---------|--------|
| `livekit-plugins-deepgram` | Deepgram (nova-3) |
| `livekit-plugins-assemblyai` | AssemblyAI |
| `livekit-plugins-google` | Google Cloud Speech |
| `livekit-plugins-azure` | Azure Speech |
| `livekit-plugins-aws` | AWS Transcribe |
| `livekit-plugins-cartesia` | Cartesia Sonic |
| `livekit-plugins-openai` | OpenAI Whisper |

### 7.3 TTS（文本转语音）

| 插件目录 | 提供商 |
|---------|--------|
| `livekit-plugins-cartesia` | Cartesia Sonic |
| `livekit-plugins-elevenlabs` | ElevenLabs |
| `livekit-plugins-openai` | OpenAI TTS |
| `livekit-plugins-google` | Google Cloud TTS |
| `livekit-plugins-azure` | Azure TTS |
| `livekit-plugins-aws` | AWS Polly |
| `livekit-plugins-neuphonic` | Neuphonic |
| `livekit-plugins-rime-ai` | Rime AI |
| `livekit-plugins-inworld` | Inworld AI |
| `livekit-plugins-asyncai` | Async AI |
| `livekit-plugins-cambai` | Camb AI |
| `livekit-plugins-clova` | Clova Voice |

### 7.4 头像（Avatar）

| 插件目录 | 提供商 |
|---------|--------|
| `livekit-plugins-anam` | Anam.ai |
| `livekit-plugins-avatario` | Avatario |
| `livekit-plugins-avatartalk` | Avatartalk |
| `livekit-plugins-bey` | Bey.dev |
| `livekit-plugins-bithuman` | BitHuman |
| `livekit-plugins-tavus` | Tavus |
| `livekit-plugins-did` | D-ID |
| `livekit-plugins-simli` | Simli |

### 7.5 其他

| 插件目录 | 功能 |
|---------|------|
| `livekit-plugins-silero` | Silero VAD（语音活动检测） |
| `livekit-plugins-turn-detector` | 多语言轮次检测 |
| `livekit-plugins-hamming` | Hamming 通话分析平台 |
| `livekit-plugins-ultravox` | Ultravox Realtime API |
| `livekit-plugins-fal` | Fal.ai 图像生成 |
| `livekit-plugins-browser` | 浏览器自动化（BrowserAgent） |
| `livekit-plugins-baseten` | Baseten 模型托管 |

### 7.6 工具库（非插件，livekit-agents 子包）

| 目录 | 功能 |
|------|------|
| `livekit-blingfire` | 文本分词（BlingFire C 库绑定） |
| `livekit-blockguard` | 内容安全过滤（C 扩展） |
| `livekit-durable` | 持久化数据帧（跨进程共享内存） |

---

## 8. CLI 与 DevOps

| 仓库 | 说明 |
|------|------|
| [livekit/livekit-cli](https://github.com/livekit/livekit-cli) | **CLI 工具** — 命令行管理工具，创建房间、生成 token、创建 SIP Trunk 等 |
| [livekit/livekit-server-sdk-php](https://github.com/agence104/livekit-server-sdk-php) | PHP SDK（社区维护） |
| [livekit/livekit-server-sdk-dotnet](https://github.com/pabloFuente/livekit-server-sdk-dotnet) | .NET SDK（社区维护） |

---

## 9. 示例项目

| 仓库 | 说明 |
|------|------|
| [livekit-examples/agent-starter-python](https://github.com/livekit-examples/agent-starter-python) | Python Agent 入门模板 |
| [livekit-examples/agent-starter-node](https://github.com/livekit-examples/agent-starter-node) | Node.js Agent 入门模板 |
| [livekit-examples/agent-starter-react](https://github.com/livekit-examples/agent-starter-react) | React 前端入门模板 |
| [livekit-examples/agent-starter-swift](https://github.com/livekit-examples/agent-starter-swift) | SwiftUI 前端入门模板 |
| [livekit-examples/agent-starter-android](https://github.com/livekit-examples/agent-starter-android) | Android 前端入门模板 |
| [livekit-examples/agent-starter-flutter](https://github.com/livekit-examples/agent-starter-flutter) | Flutter 前端入门模板 |
| [livekit-examples/agent-starter-react-native](https://github.com/livekit-examples/agent-starter-react-native) | React Native 前端模板 |
| [livekit-examples/agent-starter-embed](https://github.com/livekit-examples/agent-starter-embed) | Web Embed 模板 |
| [livekit-examples/outbound-caller-python](https://github.com/livekit-examples/outbound-caller-python) | 外呼机器人示例 |
| [livekit-examples/vision-demo](https://github.com/livekit-examples/vision-demo) | Gemini Vision 视觉 Agent |
| [livekit-examples/python-agents-examples](https://github.com/livekit-examples/python-agents-examples) | Python Agents 更多示例 |
| [livekit-examples/agent-starter-js](https://github.com/livekit-examples/agent-starter-js) | JS/TS Agent 入门模板 |

---

## 10. 文档与网站

| 仓库 | 说明 |
|------|------|
| [livekit/docs](https://github.com/livekit/docs) | 官方文档 |
| [livekit/mcp](https://github.com/livekit/mcp) | Docs MCP Server — 让 AI 编码助手检索 LiveKit 文档 |
| [livekit/agents-playground](https://agents-playground.livekit.io) | Agents 游乐场（在线体验） |
| [livekit](https://livekit.io) | 官网 |
| [livekit/cloud](https://cloud.livekit.io) | LiveKit Cloud 云服务 |

---

## 11. 本地代码与外部仓库的关系

### 11.1 本地已有的完整仓库

```
d:\xuyong\source\AITest\livekit\
├── livekit/              ← github.com/livekit/livekit（核心服务器）
│   ├── cmd/               ← 服务入口
│   ├── pkg/               ← 核心代码
│   │   ├── config/        ← 配置管理
│   │   ├── service/       ← API 服务层（EgressService、SIPService 等）
│   │   ├── rtc/           ← 实时通信核心
│   │   └── telemetry/     ← 遥测
│   └── doc/               ← 设计文档
│
├── livekit-sip/           ← github.com/livekit/sip（SIP 桥接）
│   ├── cmd/               ← 服务入口
│   ├── pkg/
│   │   ├── config/        ← 配置管理
│   │   ├── sip/           ← SIP 核心逻辑
│   │   ├── service/       ← psrpc 服务层
│   │   └── stats/         ← 监控指标
│   └── doc/               ← 设计文档
│
└── livekit-agents/        ← github.com/livekit/agents（Agent 框架）
    ├── livekit-agents/    ← 核心 Agent 框架（Python）
    ├── livekit-plugins/   ← 40+ 插件
    ├── examples/          ← 示例代码
    └── doc/               ← 设计文档
```

### 11.2 依赖引入（Go module cache）

`livekit` 和 `livekit-sip` 通过 `go.mod` 引入的依赖，自动下载到 `$GOPATH/pkg/mod/github.com/livekit/` 目录下：

| 依赖 | 作用 | 被谁依赖 |
|------|------|---------|
| `livekit/protocol` | Protobuf 定义、rpc 生成代码、工具库 | `livekit` + `livekit-sip` |
| `livekit/psrpc` | Redis Pub/Sub RPC 框架 | `livekit` + `livekit-sip` |
| `livekit/sipgo` | SIP 协议栈 | `livekit-sip` |
| `livekit/media-sdk` | 媒体处理库 | `livekit-sip` |
| `livekit/mediatransportutil` | 媒体传输工具 | `livekit` + `livekit-sip` |
| `livekit/server-sdk-go` | Go Server SDK | `livekit-sip` |

### 11.3 依赖引入（Python）

`livekit-agents` 通过 `pyproject.toml` 引入的依赖，自动下载到 Python venv 中：

| 依赖 | 作用 |
|------|------|
| `livekit` | Python LiveKit SDK |
| `livekit-api` | Python LiveKit Server API |
| `livekit-plugins-*` | 各插件包 |

### 11.4 不在本地的仓库

以下仓库是独立服务，不在本地代码中：

| 仓库 | 原因 |
|------|------|
| `livekit/egress` | 独立录制服务，作为 Go module 依赖引入 |
| `livekit/ingress` | 独立输入服务 |
| `livekit/agents-js` | Node.js 版 Agent 框架 |
| `livekit/client-sdk-*` | 各平台客户端 SDK |
| `livekit/components-*` | UI 组件库 |
| `livekit/livekit-cli` | CLI 工具 |
| 所有 `livekit-examples/*` | 示例项目 |

---

## 12. 生产部署服务一览

### 12.1 服务拓扑

```
                        ┌──────────────────────┐
                        │   负载均衡 / DNS      │
                        └──────┬───────┬───────┘
                               │       │
                 ┌─────────────┘       └─────────────┐
                 ▼                                   ▼
        ┌────────────────┐                  ┌────────────────┐
        │  LiveKit Server │                  │  LiveKit Server │
        │  (livekit)      │                  │  (livekit)      │
        │  节点 1         │                  │  节点 2         │
        └────┬───────┬────┘                  └────┬───────┬────┘
             │       │                            │       │
             │       └────────────────────────────┘       │
             ▼                                            ▼
     ┌─────────────────────────────────────────────────────────┐
     │                  Redis (共享状态)                        │
     │       psrpc 消息总线 + SIP 配置 + Room 状态             │
     └─────────────────────────────────────────────────────────┘
             │              │              │
             ▼              ▼              ▼
     ┌────────────┐ ┌────────────┐ ┌────────────────┐
     │  SIP Bridge │ │   Egress   │ │   Ingress      │
     │ (livekit-sip)│ │ (livekit/  │ │ (livekit/      │
     │             │ │  egress)   │ │  ingress)      │
     └────────────┘ └────────────┘ └────────────────┘
             │
             ▼
     ┌────────────────┐      ┌──────────────────────┐
     │  AI Agent      │      │  Client App          │
     │ (livekit-agents)│      │ (client-sdk-*)       │
     │ Python 进程    │      │ iOS/Android/Web      │
     └────────────────┘      └──────────────────────┘
```

### 12.2 服务清单

| 服务名 | 二进制/进程名 | 对应代码库 | 部署方式 | 必需/可选 |
|-------|-------------|-----------|---------|----------|
| **LiveKit Server** | `livekit-server` | `livekit/livekit` | 单二进制 | **必需** |
| **Redis** | `redis-server` | N/A（外部依赖） | 独立部署 | **必需** |
| **SIP Bridge** | `livekit-sip` | `livekit/sip` | 单二进制 | 可选（需 SIP 时） |
| **Egress** | `livekit-egress` | `livekit/egress` | 单二进制 | 可选（需录制时） |
| **Ingress** | `livekit-ingress` | `livekit/ingress` | 单二进制 | 可选（需外部输入时） |
| **AI Agent** | `python agent.py` | `livekit/agents` | Python 进程 | 可选（需 AI 时） |

### 12.3 各服务的端口与通信方式

```
服务间通信矩阵:

┌──────────────┬────────────────┬──────────────────┬──────────────────┐
│ 服务         │ 监听端口        │ 对外通信协议      │ 内部通信方式      │
├──────────────┼────────────────┼──────────────────┼──────────────────┤
│ LiveKit Server │ 7880 (HTTP)  │ WebSocket (7881) │ psrpc (Redis)    │
│              │ 7881 (WS)      │ HTTP API (7880)  │                  │
│              │ 7882 (UDP/TCP) │                   │                  │
│              │ ← WebRTC ICE   │                   │                  │
├──────────────┼────────────────┼──────────────────┼──────────────────┤
│ SIP Bridge   │ 5060 (UDP/TCP) │ SIP (5060)       │ psrpc (Redis)    │
│              │ 5061 (TLS)     │                   │                  │
│              │ 10000-20000    │                   │                  │
│              │ (UDP/RTP)      │                   │                  │
├──────────────┼────────────────┼──────────────────┼──────────────────┤
│ Egress       │ 动态           │ WebSocket (7881) │ psrpc (Redis)    │
│              │                │ → LiveKit Server │                  │
├──────────────┼────────────────┼──────────────────┼──────────────────┤
│ Ingress      │ 1935 (RTMP)    │ RTMP / WHIP      │ psrpc (Redis)    │
│              │ 动态 (WHIP)    │                   │                  │
├──────────────┼────────────────┼──────────────────┼──────────────────┤
│ AI Agent     │ 无（Client）   │ WebSocket (7881) │ 无               │
│              │                │ → LiveKit Server │                  │
├──────────────┼────────────────┼──────────────────┼──────────────────┤
│ Redis        │ 6379           │ Redis 协议       │ N/A              │
└──────────────┴────────────────┴──────────────────┴──────────────────┘
```

### 12.4 最低部署方案 vs 完整部署方案

```
┌──────────────────────────────────────────────────────────────────┐
│  最低部署方案（仅核心功能）                                        │
│                                                                  │
│  ┌──────────┐  ┌───────┐  ┌──────────────────────┐             │
│  │ LiveKit  │  │ Redis │  │  Client App + Agent  │             │
│  │ Server   │  │       │  │  (同一进程/设备)      │             │
│  └──────────┘  └───────┘  └──────────────────────┘             │
│                                                                  │
│  适用: 开发测试、小规模私有部署                                    │
└──────────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────────┐
│  完整部署方案（生产环境）                                          │
│                                                                  │
│  ┌──────────┐  ┌───────┐  ┌──────────┐  ┌───────┐  ┌────────┐ │
│  │ LiveKit  │  │ Redis │  │ SIP      │  │Egress │  │Ingress │ │
│  │ Server   │  │       │  │ Bridge   │  │       │  │        │ │
│  │ (多节点) │  │       │  │          │  │       │  │        │ │
│  └──────────┘  └───────┘  └──────────┘  └───────┘  └────────┘ │
│                                                                  │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────────────┐  │
│  │ AI Agent     │  │ Client Apps  │  │  SIP Trunk (PSTN)   │  │
│  │ (多实例)     │  │ (多平台)     │  │  / 外部运营商        │  │
│  └──────────────┘  └──────────────┘  └──────────────────────┘  │
│                                                                  │
│  适用: 生产环境、需要 SIP/录制/多平台                              │
└──────────────────────────────────────────────────────────────────┘
```

### 12.5 服务启动方式

```
# 1. LiveKit Server (必需)
livekit-server --config=config.yaml
# 或 Docker:
docker run livekit/livekit-server --config=config.yaml

# 2. Redis (必需)
redis-server

# 3. SIP Bridge (可选，需要 SIP 功能时)
livekit-sip --config=sip_config.yaml
# 或 Docker:
docker run --network host livekit/sip --config=sip_config.yaml

# 4. Egress (可选，需要录制功能时)
livekit-egress --config=egress_config.yaml

# 5. Ingress (可选，需要 RTMP 输入时)
livekit-ingress --config=ingress_config.yaml

# 6. AI Agent (可选)
python my_agent.py start
```

### 12.6 服务规模建议

```
┌─────────────────────────────────────────────────────────────┐
│  部署规模建议 (以 1000 并发通话为参考)                        │
│                                                              │
│  ┌──────────────────┬──────────┬──────────┬──────────┐      │
│  │ 服务              │ 小型     │ 中型     │ 大型     │      │
│  │                  │ ~100并发 │ ~1000并发 │ ~5000并发│      │
│  ├──────────────────┼──────────┼──────────┼──────────┤      │
│  │ LiveKit Server   │ 1节点    │ 2-3节点  │ 5+节点   │      │
│  │ Redis            │ 1节点    │ 1主2从   │ 集群模式  │      │
│  │ SIP Bridge       │ 1节点    │ 2节点    │ 3+节点   │      │
│  │ Egress           │ 按需     │ 1-2节点  │ 3+节点   │      │
│  │ AI Agent         │ N/A      │ 按需     │ N/A      │      │
│  └──────────────────┴──────────┴──────────┴──────────┘      │
│                                                              │
│  资源估算 (单节点):                                            │
│  LiveKit Server: 4核/8GB                                     │
│  SIP Bridge:     4核/4GB (5060 + RTP端口)                   │
│  Redis:          2核/4GB                                     │
│  Egress:         按录制并发，每路 ~0.5核                      │
└─────────────────────────────────────────────────────────────┘
```

### 12.7 依赖关系图

```
必需:

  Client App ──WebSocket──→ LiveKit Server ──Redis──→ Redis
                                │
  AI Agent  ─────WebSocket──────┘


可选 (与 LiveKit Server 通过 psrpc/Redis 通信):

  SIP Bridge ──psrpc──→ LiveKit Server
  Egress     ──psrpc──→ LiveKit Server (加入 Room 需 WebSocket)
  Ingress    ──psrpc──→ LiveKit Server

依赖关系:
  LiveKit Server ──必需──→ Redis
  SIP Bridge     ──必需──→ Redis（共享 psrpc 消息总线）
  Egress         ──必需──→ Redis
  Ingress        ──必需──→ Redis
  所有服务        ──可选──→ LiveKit Server（配置管理、房间管理）
```

| 分类 | 数量 |
|------|------|
| 核心服务 | 4 |
| Agent 框架 | 3 |
| 核心协议与基础库 | 5 |
| Server SDK | 8 |
| 客户端 SDK | 12 |
| UI 组件 | 4 |
| Agent 插件 | 40+ |
| CLI 与 DevOps | 3 |
| 示例项目 | 12+ |
| 文档与网站 | 4 |
| **总计** | **95+** |

## 附录：B. 各仓库主要语言

| 语言 | 仓库 | 占比 |
|------|------|------|
| **Go** | 核心服务、基础库、Server SDK | ~60% |
| **Python** | Agents 框架、插件 | ~20% |
| **TypeScript/JavaScript** | 客户端 SDK、UI 组件 | ~10% |
| **其他** | Swift、Kotlin、Dart、C#、Rust、C++ | ~10% |