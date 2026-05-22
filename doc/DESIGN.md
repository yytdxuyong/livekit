# LiveKit Server 设计文档

## 1. 概述

LiveKit Server 是一个高性能的 WebRTC SFU（Selective Forwarding Unit）服务器，使用 Go 语言编写。它接收发布者（Publisher）的实时音视频流，并将其选择性转发给订阅者（Subscriber），而不进行混流或转码。项目支持多节点分布式部署、**Agent/AI Worker 集成**、Egress/Ingress 录制与推流、SIP 互通等功能。

### 技术栈

| 类别 | 技术选型 |
|------|----------|
| 语言 | Go 1.26 |
| WebRTC | pion/webrtc v4 |
| 信令 | WebSocket + Protobuf (psrpc) |
| 分布式路由 | Redis Pub/Sub |
| 依赖注入 | Google Wire |
| 监控 | Prometheus |
| Agent 框架 | psrpc JobRequest/JobTerminate |
| Worker 通信 | WebSocket + Protobuf (双协议: Protobuf Binary / JSON) |

---

## 2. 整体架构

系统采用分层架构，从上到下分为以下层次：

```
┌──────────────────────────────────────────────────────┐
│                  cmd/server (入口)                     │
│              CLI 命令解析、配置加载、启动               │
├──────────────────────────────────────────────────────┤
│                 pkg/service (服务层)                   │
│   HTTP API、RoomManager、SignalServer、               │
│   AgentService（Worker 注册/调度）、                   │
│   AgentDispatchService、Egress/Ingress/SIP/WHIP       │
├──────────────────────────────────────────────────────┤
│                  pkg/routing (路由层)                  │
│   LocalRouter / RedisRouter、消息通道、                │
│   节点选择器 (selector)                                │
├──────────────────────────────────────────────────────┤
│                   pkg/rtc (RTC 层)                     │
│   Room、Participant、Transport、                       │
│   agentClient（启动 Job）、AgentStore                 │
│   SubscriptionManager、UpTrackManager                 │
├──────────────────────────────────────────────────────┤
│                   pkg/sfu (SFU 层)                     │
│   Forwarder、DownTrack、Receiver、Buffer、            │
│   StreamAllocator、BWE、Pacer                         │
├──────────────────────────────────────────────────────┤
│              pkg/agent (Agent Worker 框架)             │
│   Worker、Client、WorkerRegisterer、                  │
│   WorkerSignalHandler、WorkerPingHandler              │
├──────────────────────────────────────────────────────┤
│              其他辅助模块                              │
│   pkg/config、pkg/telemetry、pkg/metric               │
└──────────────────────────────────────────────────────┘
```

---

## 3. Agent / AI 大模型交互系统（重点）

### 3.1 架构概览

LiveKit 的 Agent 系统允许**外部 AI Worker 程序**（例如运行 LLM 推理的程序）作为特殊的"程序化参与者"加入房间，实现对音视频流的实时处理。这是 LiveKit 与大模型交互的核心机制。

```
                          ┌────────────────────────────┐
                          │     AI Worker (Python/JS)   │
                          │  ┌──────────────────────┐   │
                          │  │  LLM 推理引擎         │   │
                          │  │  (GPT/Claude/Whisper) │   │
                          │  └──────────────────────┘   │
                          │  ┌──────────────────────┐   │
                          │  │  Worker SDK           │   │
                          │  │  - WebSocket Client   │   │
                          │  │  - Proto 编解码       │   │
                          │  └──────────┬───────────┘   │
                          └─────────────┼───────────────┘
                                        │ WebSocket
                          ┌─────────────▼───────────────┐
                          │     LiveKit Server          │
                          │  ┌──────────────────────┐   │
                          │  │  AgentService         │   │
                          │  │  - Worker 注册        │   │
                          │  │  - Job 调度           │   │
                          │  │  - 负载均衡           │   │
                          │  └──────────────────────┘   │
                          │  ┌──────────────────────┐   │
                          │  │  Room + Participant   │   │
                          │  │  Agent 作为参与者加入  │   │
                          │  │  获取音视频 Track      │   │
                          │  └──────────────────────┘   │
                          └─────────────────────────────┘
```

### 3.2 Job 类型（与 AI 场景对应）

Agent 系统定义了三种 Job 类型，每种对应不同的 AI 应用场景：

| Job 类型 | 枚举值 | 触发时机 | AI 典型场景 |
|----------|--------|----------|------------|
| `JT_ROOM` | Room Agent | 房间创建时 | 会议记录、实时翻译、内容审核 |
| `JT_PUBLISHER` | Publisher Agent | 参与者首次发布 Track 时 | 实时字幕生成、语音转文字（ASR）、视频内容分析 |
| `JT_PARTICIPANT` | Participant Agent | 参与者加入房间时 | 虚拟助手、角色扮演 AI、客服机器人 |

### 3.3 Worker 生命周期

AI Worker 与 LiveKit Server 的完整交互流程：

```
┌─ Worker 生命周期 ────────────────────────────────────────────────┐
│                                                                   │
│  ① WebSocket 连接 (Worker → Server)                               │
│     │ 携带 JWT Token 进行身份认证                                  │
│     ▼                                                             │
│  ② Register 阶段 (Handshake)                                     │
│     Worker → Server:  RegisterWorkerRequest                       │
│         { type: JT_ROOM, agentName: "transcriber", version: "1" } │
│     Server → Worker:  RegisterWorkerResponse                      │
│         { workerId: "AW_xxx", serverInfo: {...} }                 │
│     超时: 10s                                                     │
│     ▼                                                             │
│  ③ Worker 就绪                                                   │
│     Worker → Server:  UpdateWorkerStatus                          │
│         { status: WS_AVAILABLE, load: 0.3, jobCount: 1 }          │
│     ▼                                                             │
│  ④ Job 分配 (当房间中有匹配的 Job 时)                              │
│     Server → Worker:  AvailabilityRequest                         │
│         { job: {type: JT_ROOM, room: {...}} }                    │
│     Worker → Server:  AvailabilityResponse                        │
│         { available: true, participantIdentity: "PI_xxx" }        │
│     Server → Worker:  JobAssignment                               │
│         { job: {...}, token: "eyJ..." }          ← 包含加入房间的 Token │
│     ▼                                                             │
│  ⑤ Worker 作为 Participant 加入房间                               │
│     Worker 使用 token 通过 SignalServer 加入房间                   │
│     获得音视频 Track 订阅/发布权限                                  │
│     Worker → Server:  UpdateJobStatus                             │
│         { status: JS_RUNNING }                                    │
│     ▼                                                             │
│  ⑥ AI 处理循环                                                    │
│     Worker 通过 WebRTC Track 获取音频/视频帧                        │
│     → 送入 AI 模型推理 (ASR/LLM/CV)                                │
│     → 将结果通过 DataChannel 发回或发布为新 Track                    │
│     ▼                                                             │
│  ⑦ Job 终止                                                       │
│     Server → Worker:  JobTermination                              │
│         { jobId: "AJ_xxx", reason: ROOM_CLOSED }                  │
│     或 Worker → Server:  UpdateJobStatus                          │
│         { status: JS_SUCCESS }                                    │
│                                                                   │
└───────────────────────────────────────────────────────────────────┘
```

### 3.4 核心组件详解

#### 3.4.1 Worker ([worker.go](file:///d:/xuyong/source/AITest/livekit/livekit/pkg/agent/worker.go))

服务端表示的 Worker 实例，管理一个 WebSocket 连接的生命周期：

```go
type Worker struct {
    WorkerPingHandler              // Ping/Pong 心跳处理
    WorkerRegistration             // 注册信息（ID, AgentName, Namespace, JobType, 权限, Deployment）

    apiKey    string               // API Key（用于生成 Agent Token）
    apiSecret string               // API Secret

    load   float32                 // 当前负载 (0.0 ~ 1.0)
    status livekit.WorkerStatus   // WS_AVAILABLE / WS_FULL

    runningJobs  map[JobID]*Job   // 运行中的 Job
    availability map[JobID]chan *AvailabilityResponse  // 等待可用性响应的 Job
}
```

**核心方法**:
- `AssignJob(ctx, job, hook)` — 向 Worker 分配 Job，流程：
  1. 发送 `AvailabilityRequest` 询问 Worker 是否接受
  2. 等待 `AvailabilityResponse`（超时 10s）
  3. Worker 接受后，调用 `BuildAgentToken` 生成加入房间的 Token
  4. 发送 `JobAssignment`（包含 Job 详情和 Token）
- `TerminateJob(jobID, reason)` — 终止 Job，发送 `JobTermination` 消息
- `Load()` / `Status()` — 返回负载信息，用于调度器的负载均衡

#### 3.4.2 AgentClient ([client.go](file:///d:/xuyong/source/AITest/livekit/livekit/pkg/agent/client.go))

RTC 层使用的 Agent 客户端，封装了 psrpc 调用：

```go
type Client interface {
    LaunchJob(ctx, desc *JobRequest) *IncrementalDispatcher[*Job]
    TerminateJob(ctx, jobID, reason) (*JobState, error)
}
```

**`LaunchJob` 的调度逻辑**:
1. 从缓存（TTL=1min）获取可用的 Agent Name + Namespace
2. 通过 psrpc `JobRequest` RPC 向 `AgentService` 发起请求
3. 使用 `WorkerPool(50)` 并发向多个 Namespace 发起调用
4. `AgentService` 通过 `selectWorkerWeightedByLoad` 选择负载最低的 Worker

**缓存失效**: `WorkerRegistered` Pub/Sub 事件会触发缓存刷新。

#### 3.4.3 AgentService / AgentHandler ([agentservice.go](file:///d:/xuyong/source/AITest/livekit/livekit/pkg/service/agentservice.go))

服务层组件，负责：

| 职责 | 实现 |
|------|------|
| Worker WebSocket 接入 | `ServeHTTP` → `HandleConnection` |
| Worker 注册握手 | `HandshakeAgentWorker`（10s 超时，三步握手） |
| Worker 注册到调度表 | `registerWorker` → 注册 psrpc `JobRequestTopic` |
| Job 分配负载均衡 | `selectWorkerWeightedByLoad`（按可用容量加权随机选择） |
| Job 终止 | `JobTerminate` → Worker.TerminateJob |
| Worker 健康检查 | 通过 `CheckEnabled` RPC 返回可用 Worker 列表 |
| 连接排空 | `DrainConnections`（优雅关闭） |

**负载均衡算法** (`selectWorkerWeightedByLoad`):
```
1. 筛选 status == WS_AVAILABLE 的 Worker
2. 计算每个 Worker 的可用容量: max(0, 1 - Load())
3. 按可用容量加权随机选择
4. 若全部满载则返回错误
```

**调度注册表**:
```go
// 按 (agentName, namespace, jobType, deployment) 分组
namespaceWorkers map[workerKey][]*agent.Worker
// 按 jobID 快速查找 Worker
jobToWorker map[JobID]*agent.Worker
```

#### 3.4.4 AgentDispatchService ([agent_dispatch_service.go](file:///d:/xuyong/source/AITest/livekit/livekit/pkg/service/agent_dispatch_service.go))

管理 Agent Dispatch 的创建/删除/查询：

- `CreateDispatch` — 创建 AgentDispatch，先通过 `RoomAllocator` 确保房间存在，再通过 psrpc 发送给 Room 所在节点
- `DeleteDispatch` — 删除 Dispatch 及其关联的所有 Job
- `ListDispatch` — 列出房间的所有 Dispatch

#### 3.4.5 Room 中的 Agent 集成 ([room.go](file:///d:/xuyong/source/AITest/livekit/livekit/pkg/rtc/room.go))

Room 持有 `agentClient` 和 `agentStore`，在适当时机启动 Agent：

| 触发条件 | 代码位置 | 调用的方法 |
|----------|----------|-----------|
| 房间创建 | `NewRoom` | `createAgentDispatchesFromRoomAgent()` → `launchRoomAgents()` |
| 参与者加入 | `Join` | `launchTargetAgents(ads, participant, JT_PARTICIPANT)` |
| 首次发布 Track | `onTrackPublished` | `launchTargetAgents(ads, p, JT_PUBLISHER)` + `launchTargetAgents(ads, p, JT_PARTICIPANT)` |
| 通过 Dispatch API | 动态触发 | 重新启动对应类型的 Agent |

**Agent 参与者追踪**: Room 维护 `agentParticpants map[ParticipantIdentity]*agentJob` 来追踪由 Agent Job 产生的参与者。当 Job 需要等待参与者离开时，通过 `waitForParticipantLeaving` 进行同步。

#### 3.4.6 Worker 信号分发 ([worker.go](file:///d:/xuyong/source/AITest/livekit/livekit/pkg/agent/worker.go))

通过 `DispatchWorkerSignal` 进行消息类型分发：

```
WorkerMessage (oneof)
├── Register        → HandleRegister()
├── Availability    → HandleAvailability()
├── UpdateJob       → HandleUpdateJob()
├── Ping            → HandlePing() → 回复 Pong
├── UpdateWorker    → HandleUpdateWorker()
├── MigrateJob      → HandleMigrateJob()
└── SimulateJob     → HandleSimulateJob()
```

### 3.5 Worker 通信协议

#### 3.5.1 传输层
- WebSocket 长连接
- 支持 Protobuf Binary（BinaryMessage）和 JSON（TextMessage）两种编码
- 自动检测：首次收到 BinaryMessage 则切换为 Protobuf

#### 3.5.2 Worker 消息结构（Worker → Server）

```protobuf
message WorkerMessage {
  oneof message {
    RegisterWorkerRequest register = 1;
    AvailabilityResponse availability = 2;
    UpdateJobStatus update_job = 3;
    WorkerPing ping = 4;
    UpdateWorkerStatus update_worker = 5;
    MigrateJobRequest migrate_job = 6;
    SimulateJobRequest simulate_job = 7;
  }
}
```

#### 3.5.3 Server 消息结构（Server → Worker）

```protobuf
message ServerMessage {
  oneof message {
    RegisterWorkerResponse register = 1;
    AvailabilityRequest availability = 2;
    JobAssignment assignment = 3;
    JobTermination termination = 4;
    WorkerPong pong = 5;
  }
}
```

### 3.6 Worker 注册流程详解

```
Worker SDK                        AgentService/AgentHandler
    │                                      │
    │── WebSocket Connect ────────────────>│  (JWT Token 认证)
    │                                      │
    │── RegisterWorkerRequest ────────────>│  HandleRegister()
    │   { type, agentName,                │  ├─ 校验 JobType 合法性
    │     namespace, version,             │  ├─ 校验 Deployment 合法性
    │     allowedPermissions }            │  ├─ 设置默认权限（若未提供）
    │                                      │  ├─ 记录 AgentName/Namespace/JobType
    │                                      │  └─ registered = true
    │<── RegisterWorkerResponse ──────────│
    │   { workerId, serverInfo }          │
    │                                      │
    │── UpdateWorkerStatus ──────────────>│  registerWorker()
    │   { status, load, jobCount }        │  ├─ 放入 namespaceWorkers 表
    │                                      │  ├─ 注册 psrpc JobRequestTopic
    │                                      │  └─ Publish WorkerRegistered 事件
    │                                      │
    │  [Worker 进入就绪状态，等待 Job 分配]   │
```

### 3.7 Job 分配流程详解

```
Room.agentClient           AgentService        AgentHandler        AI Worker
     │                         │                    │                   │
     │─LaunchJob──────────────>│                    │                   │
     │                         │                    │                   │
     │  ┌─ getDispatcher()     │                    │                   │
     │  ├─ 检查缓存             │                    │                   │
     │  └─ checkEnabled() ────>│──────────────────>│                   │
     │                         │<─ CheckEnabled Resp│                   │
     │                         │ (namespaces,       │                   │
     │                         │  agentNames)       │                   │
     │                         │                    │                   │
     │  ┌─ ForEach namespace   │                    │                   │
     │  │                      │                    │                   │
     │  │─ JobRequest RPC ────>│──JobRequest()─────>│                   │
     │  │                      │                    │                   │
     │  │                      │  selectWorkerWeightedByLoad(key)       │
     │  │                      │  ┌─ 按(agentName,ns,jobType,           │
     │  │                      │  │  deployment) 查找worker列表          │
     │  │                      │  ├─ 过滤 status==AVAILABLE             │
     │  │                      │  ├─ 按 max(0,1-load) 加权随机选择       │
     │  │                      │  └─ 返回选中的 Worker                   │
     │  │                      │                    │                   │
     │  │                      │  AssignJob(job)───>│                   │
     │  │                      │                    │─Avail Request────>│
     │  │                      │                    │   { job, room }   │
     │  │                      │                    │                   │
     │  │                      │                    │<─Avail Response───│
     │  │                      │                    │ {available: true, │
     │  │                      │                    │  identity: "xx" }  │
     │  │                      │                    │                   │
     │  │                      │                    │ BuildAgentToken() │
     │  │                      │                    │ ┌─ 构建JWT Token  │
     │  │                      │                    │ │  - 包含身份信息  │
     │  │                      │                    │ │  - 包含权限      │
     │  │                      │                    │ │  - 包含Attributes│
     │  │                      │                    │ └─ 返回Token      │
     │  │                      │                    │                   │
     │  │                      │                    │─JobAssignment────>│
     │  │                      │                    │ {job, token}       │
     │  │                      │                    │                   │
     │  │<─ JobRequestResp ────│<──────────────────│                   │
     │  │   { state }          │                    │                   │
     │  │                      │                    │                   │
     │  │       [Worker 使用 Token 通过 SignalServer        │            │
     │  │        加入房间，成为 Agent Participant]          │            │
```

### 3.8 Token 生成

Agent Worker 加入房间时使用的 Token 由 `protoagent.BuildAgentToken` 生成：

```go
token, err := protoagent.BuildAgentToken(
    apiKey,           // Worker 连接时认证的 API Key
    apiSecret,        // 对应的 Secret
    job.Room.Name,    // 房间名
    participantIdentity,   // Worker 指定的身份
    participantName,       // Worker 指定的名称
    participantMetadata,   // Worker 指定的元数据
    attributes,            // 包含 AgentName（lk.agent_name）
    permissions,           // Worker 注册时声明的权限
)
```

关键在于 `attributes[AgentNameAttributeKey] = w.AgentName`，客户端可通过此属性识别 Agent 参与者。

### 3.9 Worker 负载管理

Worker 通过定期发送 `UpdateWorkerStatus` 来报告负载状态：

```
Worker SDK:
  sendStatus() 每 2s 执行一次
    ├─ 计算当前负载 = sum(每个 Job 的 Load())
    ├─ 若无 Job: load = DefaultWorkerLoad (默认 0.0)
    ├─ 若 load > JobLoadThreshold (默认 0.8): status = WS_FULL
    └─ 否则: status = WS_AVAILABLE
```

每个 Job 可以自定义 `Load()` 方法返回自身资源消耗，Worker 汇总所有 Job 负载。
`targetLoad` 配置项（默认 0.7）控制调度器的目标负载水位。

### 3.10 典型 AI 场景的数据流

```
                          ┌──────────────────────────────────┐
                          │         AI Worker Process         │
                          │                                   │
   User (Publisher)       │  ① 订阅音频 Track                 │
   ───语音/视频───>       │  ② 获取音频帧 (PCM/Opus)          │
                          │  ③ Whisper/ASR → 文字             │
                          │  ④ GPT/Claude → AI 回复           │
                          │  ⑤ TTS → 合成语音                 │
   User (Subscriber)      │  ⑥ 发布音频 Track 到房间           │
   <──AI 语音回复───      │  ⑦ 通过 DataChannel 发送结构化数据  │
                          │                                   │
                          └──────────────────────────────────┘
```

**关键代码路径**:
- 订阅其他参与者: `SubscriptionManager.SetSubscribedTrack()`
- 获取音频帧: SFU `DownTrack.WriteRTP()` → `buffer.Buffer`
- 发布结果: `ParticipantImpl.AddTrack()` → `UpTrackManager`
- 发送数据: `ParticipantImpl.SendDataPacket()` → DataChannel

### 3.11 Agent 测试框架

[testutils/server.go](file:///d:/xuyong/source/AITest/livekit/livekit/pkg/agent/testutils/server.go) 提供了完整的模拟测试框架：

```go
// 创建测试服务器
server := testutils.NewTestServer(bus)

// 模拟一个 AI Worker
worker := server.SimulateAgentWorker(
    testutils.WithLabel("transcriber"),
    testutils.WithJobAssignmentHandler(func(job *livekit.Job) JobLoad { ... }),
    testutils.WithJobAvailabilityHandler(func(req AgentJobRequest) { req.Accept() }),
)
worker.Register("transcriber", livekit.JobType_JT_ROOM)

// 监听事件
jobAssignments := worker.JobAssignments.Observe()
```

`AgentWorker` 实现了 `SignalConn` 接口，通过 channel 模拟 WebSocket 通信，支持：
- `RegisterWorkerResponses` — Worker 注册响应事件
- `AvailabilityRequests` — 可用性请求事件
- `JobAssignments` — Job 分配事件
- `JobTerminations` — Job 终止事件
- `WorkerPongs` — Ping/Pong 事件

---

## 4. 其他核心模块

### 4.1 入口层 (cmd/server/)

**文件**: [main.go](file:///d:/xuyong/source/AITest/livekit/livekit/cmd/server/main.go) | [commands.go](file:///d:/xuyong/source/AITest/livekit/livekit/cmd/server/commands.go)

CLI 应用程序入口，使用 `urfave/cli v3` 框架。主要功能：

- **`startServer`**: 启动服务器主流程
- **`generateKeys`**: 生成 API Key/Secret 密钥对
- **`printPorts`**: 打印服务器监听端口
- **`listNodes`**: 列出集群中所有节点

### 4.2 服务层 (pkg/service/)

#### 4.2.1 LivekitServer ([server.go](file:///d:/xuyong/source/AITest/livekit/livekit/pkg/service/server.go))

服务器主结构体，聚合了所有子服务：

| 字段 | 说明 |
|------|------|
| `ioService` | IO 信息服务 |
| `rtcService` | RTC 连接服务 |
| `whipService` | WHIP 协议服务 |
| `agentService` | **Agent 调度服务** |
| `roomManager` | 房间管理器 |
| `signalServer` | 信令服务器 |
| `router` | 消息路由器 |
| `turnServer` | TURN 服务器 |

#### 4.2.2 RoomManager ([roommanager.go](file:///d:/xuyong/source/AITest/livekit/livekit/pkg/service/roommanager.go))

房间管理器，负责房间的创建/删除/查找、参与者会话管理、ICE 配置缓存。

#### 4.2.3 依赖注入 ([wire.go](file:///d:/xuyong/source/AITest/livekit/livekit/pkg/service/wire.go))

使用 Google Wire 进行依赖注入。Agent 相关的注入链：

```
Config.Agents → agent.Config
    → agent.NewAgentClient → agent.Client (注入到 Room)
    → NewAgentService → AgentService (HTTP handler)
    → NewAgentDispatchService → AgentDispatchService (API handler)
```

#### 4.2.4 其他服务

| 服务 | 说明 |
|------|------|
| `RoomService` | Twirp RPC 房间管理 API |
| `RTCService` | RTC 连接验证 |
| `EgressService` | 录制/转推服务 |
| `IngressService` | RTMP/WHIP 推流服务 |
| `SIPService` | SIP 互通服务 |
| `IOInfoService` | TURN/STUN 配置 |
| `WHIPService` | WHIP 协议支持 |
| `SignalServer` | psrpc 信令服务器 |

#### 4.2.5 存储层

| 存储 | 说明 |
|------|------|
| `LocalStore` | 本地内存存储 |
| `RedisStore` | Redis 持久化存储 |

### 4.3 路由层 (pkg/routing/)

路由层负责消息在多节点之间传递，实现分布式 SFU 集群。

| 实现 | 说明 |
|------|------|
| `LocalRouter` | 单节点本地路由 |
| `RedisRouter` | 基于 Redis Pub/Sub 的分布式路由 |

**Redis 数据结构**:
- `nodes` (Hash): node_id → Node proto
- `room_node_map` (Hash): room_name → node_id

#### 节点选择器 (selector/)

| 选择器 | 说明 |
|------|------|
| `AnySelector` | 任意可用节点 |
| `CPULoadSelector` | 基于 CPU 负载 |
| `RegionAwareSelector` | 区域感知选择 |
| `SysloadSelector` | 系统负载选择 |

### 4.4 RTC 层 (pkg/rtc/)

#### 4.4.1 Room ([room.go](file:///d:/xuyong/source/AITest/livekit/livekit/pkg/rtc/room.go))

核心数据结构:
```go
type Room struct {
    participants      map[ParticipantIdentity]LocalParticipant
    agentDispatches   map[string]*agentDispatch
    agentClient       agent.Client          // Agent 调度客户端
    agentStore        AgentStore            // Agent 持久化存储
    agentParticpants  map[ParticipantIdentity]*agentJob
    trackManager      *RoomTrackManager
    dataMessageCache  *utils.TimeSizeCache
}
```

#### 4.4.2 ParticipantImpl ([participant.go](file:///d:/xuyong/source/AITest/livekit/livekit/pkg/rtc/participant.go))

管理 WebRTC PeerConnection、上行 Track、下行订阅、SDP 协商、ICE 连接、信令消息处理。

#### 4.4.3 TransportManager / PCTransport

管理参与者的 WebRTC 传输层，封装 Pion WebRTC 的 PeerConnection。

#### 4.4.4 SubscriptionManager ([subscriptionmanager.go](file:///d:/xuyong/source/AITest/livekit/livekit/pkg/rtc/subscriptionmanager.go))

管理订阅/取消订阅、订阅对账（Reconciliation）、视频/音频订阅数量限制。

#### 4.4.5 UpTrackManager ([uptrackmanager.go](file:///d:/xuyong/source/AITest/livekit/livekit/pkg/rtc/uptrackmanager.go))

管理上行 Track 发布、订阅权限控制。

#### 4.4.6 其他 RTC 组件

| 组件 | 说明 |
|------|------|
| `MediaTrack` | 已发布媒体轨道封装 |
| `RoomTrackManager` | 房间级 Track 注册表 |
| `DynacastManager` | 动态广播管理 |
| `ParticipantSupervisor` | 参与者状态监控 |
| `MigrationDataCache` | 迁移数据缓存 |
| `DataTrack` | 数据通道 Track |
| `MediaLossProxy` | 音频丢包代理 |

### 4.5 SFU 层 (pkg/sfu/)

#### 4.5.1 Forwarder ([forwarder.go](file:///d:/xuyong/source/AITest/livekit/livekit/pkg/rtc/forwarder.go))

上行 Track 到下行 Track 的转发器，核心功能：
- 视频层选择（Spatial/Temporal Layer）
- 层切换（GOP 对齐、暂停/恢复）
- 关键帧请求管理（PLI/FIR）
- RTCP 处理
- 带宽分配协作

#### 4.5.2 DownTrack ([downtrack.go](file:///d:/xuyong/source/AITest/livekit/livekit/pkg/sfu/downtrack.go))

向订阅者发送 RTP 数据的下行轨道：RTP 包写入、填充包、空帧生成、RTCP 处理、序列号管理、Pacer 集成、连接质量评分。

#### 4.5.3 WebRTCReceiver ([receiver.go](file:///d:/xuyong/source/AITest/livekit/livekit/pkg/sfu/receiver.go))

接收发布者媒体轨道，支持 Simulcast 多层接收、RED 冗余编码、连接质量统计。

#### 4.5.4 Buffer ([buffer/buffer.go](file:///d:/xuyong/source/AITest/livekit/livekit/pkg/sfu/buffer/buffer.go))

RTP 包缓冲：包接收与排序、TWCC 带宽估计、RTX 重传支持、丢包检测。

#### 4.5.5 StreamAllocator ([streamallocator/streamallocator.go](file:///d:/xuyong/source/AITest/livekit/livekit/pkg/sfu/streamallocator/streamallocator.go))

流分配器，在多个 Track 之间分配可用带宽。状态机: `STABLE ↔ DEFICIENT`。支持 Probe/Catchup/Boost 模式。

#### 4.5.6 BWE / Pacer

| 模块 | 说明 |
|------|------|
| `RemoteBWE` | 基于接收端的带宽估计 |
| `SendSideBWE` | 基于发送端的带宽估计 |
| `LeakyBucket` | 漏桶速率控制 |
| `PassThrough` | 直通模式 |

---

## 5. 核心数据流

### 5.1 参与者加入流程

```
Client                SignalServer         RoomManager          Room              ParticipantImpl
  |                        |                    |                  |                     |
  |--WebSocket Connect---->|                    |                  |                     |
  |                        |--HandleSession---->|                  |                     |
  |                        |                    |--GetOrCreateRoom->|                     |
  |                        |                    |                  |--NewParticipant--->|
  |                        |                    |                  |<--JoinGrant------   |
  |                        |                    |                  |--ICE Negotiation-->|
  |<--JoinResponse---------|<--...--------------|<--...-----------|<--...---------------|
  |<--ParticipantUpdate----|<--...--------------|<--...-----------|<--...---------------|
  |--Offer SDP------------>|--SetRemoteSDP----->|----------------->|                     |
  |<--Answer SDP-----------|<--...--------------|<--...-----------|<--...---------------|
  |--ICE Candidates------->|--AddICECandidate-->|----------------->|                     |
  |<--TrackPublished-------|<--...--------------|<--...-----------|<--...---------------|
```

### 5.2 媒体流转发流程

```
Publisher                    SFU                              Subscriber
   |                          |                                    |
   |--RTP Packets------------>|                                    |
   |                          |--Buffer.Receive()                  |
   |                          |--Forwarder (Layer Selection)       |
   |                          |--DownTrack.WriteRTP()              |
   |                          |--Pacer (Rate Control)              |
   |                          |--Sequence Number Rewrite           |
   |                          |--RTP Packets---------------------->|
   |<--RTCP (PLI/NACK/etc)----|--...-------------------------------|
   |                          |<--RTCP (RR/REMB)-------------------|
   |                          |--BWE → StreamAllocator             |
```

### 5.3 Agent 调度流程（AI Worker 视角）

见第 3.3-3.7 节。

---

## 6. 多节点分布式架构

```
                    ┌─────────────────────────┐
                    │        Redis             │
                    │  - Node Registry (Hash)  │
                    │  - Room→Node Map (Hash)  │
                    │  - Pub/Sub Channels      │
                    └──────────┬──────────────┘
                               │
           ┌───────────────────┼───────────────────┐
           │                   │                   │
    ┌──────▼──────┐    ┌──────▼──────┐    ┌──────▼──────┐
    │   Node A    │    │   Node B    │    │   Node C    │
    │ (Region:us) │    │ (Region:eu) │    │ (Region:as) │
    │             │    │             │    │             │
    │ Room1       │    │ Room2       │    │ Room3       │
    │ Agent Svc   │    │ Agent Svc   │    │ Agent Svc   │
    └─────────────┘    └─────────────┘    └─────────────┘
```

**Agent Worker 连接特点**: Worker 可以连接到任意节点，Job 通过 psrpc 从房间所在节点的 `agentClient` 路由到 Agent 所在节点的 `AgentService`，再分配给具体的 Worker。Worker 拿到 Token 后通过 SignalServer 加入房间，自动路由到正确的节点。

---

## 7. 关键设计决策

1. **SFU 架构（非 MCU）** — 选择性转发，降低服务器负载
2. **分层架构** — Service → Routing → RTC → SFU，职责清晰
3. **Google Wire 依赖注入** — 声明式构建依赖图
4. **Protobuf + psrpc** — 类型安全的 RPC 框架
5. **Redis Pub/Sub 分布式路由** — 跨节点消息路由
6. **Agent Worker 框架** — 允许 AI/LLM 程序作为程序化参与者接入音视频房间
7. **Worker 负载均衡** — 按可用容量加权随机选择
8. **Token 注入机制** — Worker 获得 JWT Token 后即可作为标准参与者加入房间
9. **Simulcast + Dynacast** — 动态广播控制
10. **自适应带宽管理** — GCC/TWCC + StreamAllocator
11. **参与者迁移** — 支持不中断连接的节点间迁移

---

## 8. 关键常量

| 参数 | 默认值 | 说明 |
|------|--------|------|
| `RegisterTimeout` | 10s | Worker 注册超时 |
| `AssignJobTimeout` | 10s | Job 分配超时 |
| `CurrentProtocol` | 1 | Worker 协议版本 |
| `EnabledCacheTTL` | 1min | Agent 可用性缓存 TTL |
| `PingIntervalSeconds` | 5s | Agent 心跳间隔 |
| `PingTimeoutSeconds` | 15s | Agent 心跳超时 |
| `DefaultTargetLoad` | 0.7 | Worker 目标负载水位 |
| `WorkerPool(50)` | 50 | Job 请求并发数 |
| `roomUpdateInterval` | 5s | 房间更新频率 |
| `subscriberUpdateInterval` | 3s | 订阅者更新频率 |
| `reconcileInterval` | 3s | 订阅对账间隔 |
| `tokenRefreshInterval` | 5min | Token 刷新间隔 |
| `keyFrameIntervalMin/Max` | 200ms/1000ms | 关键帧请求间隔 |
| `InitPacketBufferSizeVideo/Audio` | 300/70 | 包缓冲大小 |