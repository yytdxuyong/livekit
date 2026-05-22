# LiveKit SIP 设计文档

## 1. 概述

LiveKit 的 SIP（Session Initiation Protocol）功能允许传统的电话网络（PSTN）和 SIP 设备与 LiveKit 实时音视频房间互通。用户可以通过手机拨打号码加入 LiveKit 房间参与 AI Agent 对话，Agent 也可以主动外呼到目标电话号码。

SIP 功能的实现采用**两层架构**：LiveKit Server 负责调度、路由和配置管理，独立的 **SIP Bridge** 服务负责真正的 SIP 协议处理（INVITE、SDP、RTP 桥接）。

### 技术栈

| 类别 | 技术选型 |
|------|----------|
| 协议处理 | 外部 SIP Bridge 服务 |
| Server ↔ Bridge 通信 | psrpc（双向 RPC） |
| 配置存储 | Redis（Hash 结构） |
| API 层 | Twirp（Protobuf HTTP） |
| 权限控制 | JWT ClaimGrants（SIP.Admin / SIP.Call） |
| Agent 侧调用 | Python SDK `api.sip.create_sip_participant` / Go SDK |

---

## 2. 整体架构

```
                           ┌──────────────────────────────┐
                           │      PSTN / 电话网络          │
                           │  手机、座机、SIP 设备          │
                           └──────────────┬───────────────┘
                                          │ SIP (INVITE/RTP)
                                          ▼
                           ┌──────────────────────────────┐
                           │       SIP Bridge             │
                           │  - SIP 协议栈 (INVITE/SDP)    │
                           │  - RTP 媒体桥接               │
                           │  - 作为 Participant 加入房间  │
                           └──────┬───────────────┬───────┘
                                  │ psrpc RPC     │ WebRTC
                                  ▼               ▼
                    ┌─────────────────────┐  ┌──────────────┐
                    │   LiveKit Server    │  │ LiveKit Room │
                    │                     │  │              │
                    │  ┌───────────────┐  │  │ Agent + 用户 │
                    │  │ SIPService    │  │  │              │
                    │  │ Trunk/Dispatch│  │  └──────────────┘
                    │  │ 管理 API      │  │
                    │  ├───────────────┤  │
                    │  │ IOInfoService │  │
                    │  │ 呼入路由匹配   │  │
                    │  ├───────────────┤  │
                    │  │ SIPStore      │  │
                    │  │ (Redis)       │  │
                    │  └───────────────┘  │
                    └─────────────────────┘
```

---

## 3. 核心模块

### 3.1 SIPService — SIP 管理 API

[`pkg/service/sip.go`](file:///d:/xuyong/source/AITest/livekit/livekit/pkg/service/sip.go)

`SIPService` 是 SIP 功能的对外 API 层，通过 Twirp（HTTP + Protobuf）暴露如下接口：

```go
type SIPService struct {
    conf        *config.SIPConfig      // SIP 配置
    nodeID      livekit.NodeID         // 当前节点 ID
    bus         psrpc.MessageBus       // 消息总线
    psrpcClient rpc.SIPClient          // 与 SIP Bridge 的 RPC 客户端
    store       SIPStore               // 持久化存储（Redis）
    roomService livekit.RoomService    // 房间服务
}
```

**API 接口一览**：

| 分类 | API | 说明 | 权限 |
|------|-----|------|------|
| **Trunk 管理** | `CreateSIPTrunk` | 创建旧版 Trunk（已弃用） | SIP.Admin |
| | `CreateSIPInboundTrunk` | 创建入站 Trunk | SIP.Admin |
| | `CreateSIPOutboundTrunk` | 创建出站 Trunk | SIP.Admin |
| | `GetSIPInboundTrunk` | 查询入站 Trunk | SIP.Admin |
| | `GetSIPOutboundTrunk` | 查询出站 Trunk | SIP.Admin |
| | `ListSIPInboundTrunk` | 列出所有入站 Trunk | SIP.Admin |
| | `ListSIPOutboundTrunk` | 列出所有出站 Trunk | SIP.Admin |
| | `UpdateSIPInboundTrunk` | 更新入站 Trunk | SIP.Admin |
| | `UpdateSIPOutboundTrunk` | 更新出站 Trunk | SIP.Admin |
| | `DeleteSIPTrunk` | 删除 Trunk | SIP.Admin |
| **Dispatch Rule 管理** | `CreateSIPDispatchRule` | 创建调度规则 | SIP.Admin |
| | `ListSIPDispatchRule` | 列出调度规则 | SIP.Admin |
| | `UpdateSIPDispatchRule` | 更新调度规则 | SIP.Admin |
| | `DeleteSIPDispatchRule` | 删除调度规则 | SIP.Admin |
| **通话操作** | `CreateSIPParticipant` | 外呼创建 SIP 参与者 | SIP.Call |
| | `TransferSIPParticipant` | 通话转移 | SIP.Call |

#### 3.1.1 SIP Trunk 类型

| 类型 | Proto 类型 | 用途 |
|------|-----------|------|
| `SIPTrunkInfo` | 旧版统一 Trunk | 已弃用，向前兼容 |
| `SIPInboundTrunkInfo` | 入站 Trunk | 接收来电：配置绑定号码、鉴权、地址 |
| `SIPOutboundTrunkInfo` | 出站 Trunk | 发起外呼：配置目标地址、主叫号码、鉴权 |

**SIPInboundTrunkInfo 关键字段**：

| 字段 | 说明 |
|------|------|
| `SipTrunkId` | Trunk 唯一标识（前缀 `ST_`） |
| `Name` | Trunk 名称 |
| `Numbers` | 绑定的电话号码列表，用于入站号码匹配 |
| `InboundAddresses` | 入站允许的 SIP 地址列表 |
| `InboundUsername` / `InboundPassword` | 入站鉴权凭据 |
| `Metadata` | 元数据 |

**SIPOutboundTrunkInfo 关键字段**：

| 字段 | 说明 |
|------|------|
| `SipTrunkId` | Trunk 唯一标识（前缀 `ST_`） |
| `Name` | Trunk 名称 |
| `Address` | 出站 SIP 服务器地址 |
| `Transport` | 传输协议（UDP/TCP/TLS） |
| `Numbers` | 出站主叫号码列表 |
| `Username` / `Password` | 出站鉴权凭据 |
| `FromHost` | 自定义 From 域名 |

#### 3.1.2 SIP Dispatch Rule

Dispatch Rule 定义了 SIP 来电应如何被路由处理：

```go
type SIPDispatchRuleInfo struct {
    SipDispatchRuleId string            // 规则 ID（前缀 SDR_）
    TrunkIds          []string          // 关联的 Trunk ID（空则匹配所有）
    Name              string            // 规则名称
    Metadata          string            // 元数据
    // 规则动作（oneof）
    Rule *SIPDispatchRule               // 具体路由动作
}
```

**路由动作类型（Dispatch Rule 的 oneof）**：

| 动作类型 | 说明 |
|----------|------|
| `DispatchRuleDirect` | 直接路由到指定房间 |
| `DispatchRuleIndividual` | 按主叫号码分发到对应 Agent |
| `DispatchRuleGroup` | 按组分发（轮询/随机） |
| `DispatchRuleCallee` | 按被叫号码分发到对应 Agent |

**DispatchRuleDirect 配置**：

| 字段 | 说明 |
|------|------|
| `RoomName` | 目标房间名 |
| `Pin` | 房间 PIN 码（可选） |

**DispatchResult（呼入路由结果）**：

| 结果 | 说明 |
|------|------|
| `ACCEPT` | 接受通话，创建房间/加入房间 |
| `REJECT` | 拒绝通话 |
| `DROP` | 直接丢弃（无 Dispatch Rule 匹配时） |
| `REQUEST_PIN` | 要求输入 PIN 码 |

---

### 3.2 IOInfoService — 呼入路由匹配

[`pkg/service/ioservice_sip.go`](file:///d:/xuyong/source/AITest/livekit/livekit/pkg/service/ioservice_sip.go)

`IOInfoService` 通过 psrpc 对外暴露以下 RPC，供 SIP Bridge 在收到来电时调用：

| RPC 方法 | 调用时机 | 说明 |
|----------|----------|------|
| `GetSIPTrunkAuthentication` | SIP INVITE 到达时 | 匹配 Trunk 并返回鉴权要求 |
| `EvaluateSIPDispatchRules` | 鉴权通过后 | 匹配 Dispatch Rule 决定路由 |
| `UpdateSIPCallState` | 通话状态变化时 | 更新通话状态（预留） |

#### 3.2.1 呼入路由匹配流程

```
SIP Bridge 收到来电 INVITE
        │
        ▼
① GetSIPTrunkAuthentication()
   │
   ├─ 从 INVITE 提取 SIPCall 信息（ToUser, FromUser, SourceIP）
   ├─ matchSIPTrunk() 按被叫号码匹配 Trunk
   │    └─ SelectSIPInboundTrunk() → 查询 Redis 匹配的 Trunk
   │    └─ sip.MatchTrunkIter() → 选择最匹配的 Trunk
   └─ 返回认证要求（Username/Password）
        │
        ▼  [SIP Bridge 完成 SIP 鉴权]
        │
② EvaluateSIPDispatchRules()
   │
   ├─ matchSIPTrunk() — 再次确认 Trunk（可能携带 trunkId）
   ├─ matchSIPDispatchRule() — 匹配 Dispatch Rule
   │    └─ SelectSIPDispatchRule() → 按 trunkId 查询 Dispatch Rule
   │    └─ sip.MatchDispatchRuleIter() → 按优先级选择最佳规则
   ├─ sip.EvaluateDispatchRule() — 执行规则逻辑
   └─ 返回 DispatchResult
        │
        ▼
   ┌──────────────────────────────────┐
   │ ACCEPT   → 创建/加入房间          │
   │ REJECT   → 拒绝呼叫               │
   │ DROP     → 无匹配规则，丢弃        │
   │ PIN      → 要求输入 PIN            │
   └──────────────────────────────────┘
```

#### 3.2.2 Trunk 匹配逻辑（SelectSIPInboundTrunk）

匹配优先级从高到低：
1. 精确匹配被叫号码（`Numbers` 字段完全匹配）
2. 匹配旧版 Trunk（`OutboundNumber` 匹配）
3. 匹配通用 Trunk（`Numbers` 为空 / 通配符）

测试用例验证了如下匹配顺序：

| 被叫号码 | 匹配的 Trunk（按优先级） |
|----------|------------------------|
| `"A"` | `old-A` → `old-any` → `any` |
| `"B1"` | `B` → `BC` → `old-any` → `any` |
| `"B2"` | `B` → `old-any` → `any` |
| `"wrong"` | `old-any` → `any` |

#### 3.2.3 Dispatch Rule 匹配逻辑（SelectSIPDispatchRule）

匹配优先级：
1. 精确匹配 `TrunkIds`（规则绑定的 Trunk ID 列表）
2. 匹配通配规则（`TrunkIds` 为空）

测试用例验证了如下匹配顺序：

| Trunk ID | 匹配的 Dispatch Rule（按优先级） |
|----------|--------------------------------|
| `"A"` | `any` |
| `"B1"` | `B` → `BC` → `any` |
| `"B2"` | `B` → `any` |
| `"wrong"` | `any` |

---

### 3.3 SIPStore — 持久化存储

[`pkg/service/redisstore_sip.go`](file:///d:/xuyong/source/AITest/livekit/livekit/pkg/service/redisstore_sip.go)

SIP 配置数据存储在 Redis 中，使用 Hash 结构组织：

| Redis Key | 说明 |
|-----------|------|
| `sip_inbound_trunk` | 入站 Trunk 配置（Hash: trunkID → Proto） |
| `sip_outbound_trunk` | 出站 Trunk 配置（Hash: trunkID → Proto） |
| `sip_trunk` | 旧版 Trunk 配置（Hash: trunkID → Proto，已弃用） |
| `sip_dispatch_rule` | Dispatch Rule 配置（Hash: ruleID → Proto） |

**向前兼容**：加载 Trunk 时会依次尝试新版（Inbound/Outbound）和旧版（Legacy），确保平滑迁移：

```go
func (s *RedisStore) LoadSIPInboundTrunk(ctx context.Context, id string) (*livekit.SIPInboundTrunkInfo, error) {
    // 1. 先尝试加载新版 InboundTrunk
    in, err := s.loadSIPInboundTrunk(ctx, id)
    if err == nil { return in, nil }
    // 2. 回退到旧版 LegacyTrunk → 转换
    tr, err := s.loadSIPLegacyTrunk(ctx, id)
    if err == nil { return tr.AsInbound(), nil }
    return nil, ErrSIPTrunkNotFound
}
```

**ID 前缀约定**：

| 前缀 | 实体类型 |
|------|----------|
| `ST_` | SIP Trunk |
| `SDR_` | SIP Dispatch Rule |

---

### 3.4 rpc.SIPClient — 与 SIP Bridge 的 RPC 通信

LiveKit Server 通过 `rpc.SIPClient`（psrpc 客户端）与 SIP Bridge 进行 RPC 通信：

| RPC 方法 | 方向 | 说明 |
|----------|------|------|
| `CreateSIPParticipant` | Server → Bridge | 发起外呼请求 |
| `TransferSIPParticipant` | Server → Bridge | 发起通话转移请求 |

**rpc.SIPClient 创建**（通过 Google Wire 注入）：

```go
func newSIPClient(p rpc.ClientParams) (rpc.SIPClient, error) {
    return rpc.NewSIPClientWithParams(rpc.ClientParams{
        Bus: p.Bus,
        ClientOptions: []psrpc.ClientOption{
            rpc.WithClientLogger(p.Logger),
            otelpsrpc.ClientOptions(otelpsrpc.Config{}),
        },
    })
}
```

---

### 3.5 权限控制

[`pkg/service/auth.go`](file:///d:/xuyong/source/AITest/livekit/livekit/pkg/service/auth.go#L206-L218)

SIP 操作需要特定的 JWT ClaimGrant 权限：

| 权限检查函数 | 所需 Claim | 适用操作 |
|-------------|-----------|---------|
| `EnsureSIPAdminPermission` | `claims.SIP.Admin == true` | Trunk 管理、Dispatch Rule 管理 |
| `EnsureSIPCallPermission` | `claims.SIP.Call == true` | 创建 SIP 参与者、通话转移 |

```go
func EnsureSIPAdminPermission(ctx context.Context) error {
    claims := GetGrants(ctx)
    if claims == nil || claims.SIP == nil || !claims.SIP.Admin {
        return ErrPermissionDenied
    }
    return nil
}

func EnsureSIPCallPermission(ctx context.Context) error {
    claims := GetGrants(ctx)
    if claims == nil || claims.SIP == nil || !claims.SIP.Call {
        return ErrPermissionDenied
    }
    return nil
}
```

---

## 4. 核心流程

### 4.1 呼入流程（外部电话 → LiveKit 房间）

```
                          PSTN/电话网络
                               │
                     SIP INVITE (含主叫/被叫号码)
                               ▼
                      ┌─────────────────┐
                      │   SIP Bridge    │
                      │  接收 INVITE     │
                      └────────┬────────┘
                               │ psrpc RPC
                               ▼
              ┌────────────────────────────────────┐
              │         LiveKit Server             │
              │                                    │
              │  Step 1: GetSIPTrunkAuthentication │
              │  ┌──────────────────────────────┐   │
              │  │ IOInfoService                │   │
              │  │  - matchSIPTrunk()           │   │
              │  │    按被叫号码查 Redis          │   │
              │  │    → 返回 Trunk 和鉴权要求     │   │
              │  └──────────────────────────────┘   │
              │                                    │
              │         [SIP Bridge 完成鉴权]       │
              │                                    │
              │  Step 2: EvaluateSIPDispatchRules  │
              │  ┌──────────────────────────────┐   │
              │  │ IOInfoService                │   │
              │  │  - matchSIPTrunk() 确认 Trunk │   │
              │  │  - matchSIPDispatchRule()    │   │
              │  │  - EvaluateDispatchRule()     │   │
              │  │    → 返回 DispatchResult      │   │
              │  │    → 包含 RoomName, Token 等   │   │
              │  └──────────────────────────────┘   │
              └──────────────────┬─────────────────┘
                                 │ DispatchResult
                                 ▼
                      ┌─────────────────┐
                      │   SIP Bridge    │
                      │  拿 Token 加入   │
                      │  LiveKit 房间    │
                      └────────┬────────┘
                               │ WebRTC (RTP 桥接)
                               ▼
                      ┌─────────────────┐
                      │  LiveKit Room   │
                      │                 │
                      │  ┌───────────┐  │
                      │  │ AI Agent  │  │
                      │  │  + SIP 用户│  │
                      │  └───────────┘  │
                      └─────────────────┘
```

**呼入关键点**：

1. **鉴权与路由分离**：SIP Bridge 先通过 `GetSIPTrunkAuthentication` 获取鉴权信息完成 SIP 层面的认证；认证通过后再通过 `EvaluateSIPDispatchRules` 决定业务路由
2. **两次 Trunk 匹配**：鉴权阶段和路由阶段各执行一次 `matchSIPTrunk`，但路由阶段可能携带 `trunkId` 加速查找
3. **DispatchResult 决定命运**：
   - `ACCEPT` → SIP Bridge 用返回的 Token 加入房间
   - `DROP` → 无匹配 Dispatch Rule，直接丢弃
   - `REJECT` / `REQUEST_PIN` → 相应处理

### 4.2 呼出流程（Agent → 外部电话）

```
 Python Agent (livekit-agents)
        │
        │ ctx.api.sip.create_sip_participant(
        │     room_name=ROOM_NAME,
        │     sip_trunk_id=SIP_TRUNK_ID,
        │     sip_call_to="+15105550123",
        │     participant_identity="phone_user",
        │     participant_name="Caller",
        │     wait_until_answered=True,    # 等待被叫应答
        │ )
        ▼
 LiveKit Server  HTTP API  /twirp
        │
        ▼
 SIPService.CreateSIPParticipant()
        │
        ├─ ① EnsureSIPCallPermission — 验证 SIP.Call 权限
        ├─ ② 加载 OutboundTrunk 配置（如果提供了 trunkId）
        ├─ ③ 加载目标地址（Trunk.FromHost 或请求中的 Trunk.FromHost）
        ├─ ④ 生成唯一 CallID
        │
        ├─ ⑤ rpc.NewCreateSIPParticipantRequest() 构建内部请求
        │     { projectID, callID, host, wsUrl, token,
        │       req (room_name, trunk_id, call_to,
        │            participant_identity, etc.),
        │       trunk (出站配置) }
        │
        └─ ⑥ 通过 psrpc 调用 SIP Bridge
               s.psrpcClient.CreateSIPParticipant(ctx, "", ireq)
               │
               timeout: 30s (默认) / 80s (waitUntilAnswered)
               ▼
        ┌─────────────────────────────────────────┐
        │             SIP Bridge                  │
        │                                         │
        │  - 使用 OutboundTrunk 配置发起 SIP INVITE│
        │  - 等待被叫应答（振铃 → 200 OK）         │
        │  - 使用 Token 作为 Participant 加入房间  │
        │  - 桥接 RTP ↔ WebRTC 媒体流             │
        └─────────────────┬───────────────────────┘
                          │ WebRTC
                          ▼
        ┌─────────────────────────────────────────┐
        │              LiveKit Room               │
        │  ┌───────────────┐ ┌──────────────────┐ │
        │  │ AI Agent      │ │ SIP Participant  │ │
        │  │ (发布/订阅)    │ │ (phone_user)     │ │
        │  └───────────────┘ └──────────────────┘ │
        └─────────────────────────────────────────┘
```

**外呼返回结果（SIPParticipantInfo）**：

| 字段 | 说明 |
|------|------|
| `ParticipantId` | SIP Bridge 生成的参与者 ID |
| `ParticipantIdentity` | 请求中指定的身份标识 |
| `RoomName` | 加入的房间名 |
| `SipCallId` | SIP 通话唯一 ID（可用于后续 Transfer） |

### 4.3 呼叫转移流程（Transfer）

```
 Python Agent
        │
        │ ctx.transfer_sip_participant(
        │     participant=sip_participant,
        │     transfer_to="+18003310500",
        │     play_dialtone=False,
        │ )
        ▼
 SIPService.TransferSIPParticipant()
        │
        ├─ ① 验证权限：EnsureSIPCallPermission + EnsureAdminPermission
        ├─ ② 通过 ParticipantIdentity 查找参与者信息
        ├─ ③ 从参与者 Attributes 中提取 SIPCallID
        │     (livekit.AttrSIPCallID)
        │
        └─ ④ 通过 psrpc 调用 SIP Bridge
               s.psrpcClient.TransferSIPParticipant(ctx, callID, ireq)
               │
               timeout: 默认 30s (ringingTimeout 可配置)
               ▼
        ┌──────────────────────────────────┐
        │          SIP Bridge              │
        │  - 发送 SIP REFER               │
        │  - 将当前通话转移到目标号码       │
        │  - 可选播放拨号音                │
        └──────────────────────────────────┘
```

**重要提醒**（来自代码注释）：
> 任何 SIP Transfer 的超时/取消都可能使 SIP Bridge 或 SIP REFER 交换处于"未知"状态。

---

## 5. Agent 侧 SIP API（Python SDK）

### 5.1 创建 SIP 参与者（外呼）

在 Agent 中通过 `JobContext` 发起外呼：

```python
# 方式一：通过 JobContext（推荐）
await ctx.create_sip_participant(
    participant_identity="phone_user",
    trunk_id="ST_xxxxxxxx",
    call_to="+15105550123",
    participant_name="Caller",
)

# 方式二：通过 API 直接调用
await ctx.api.sip.create_sip_participant(
    api.CreateSIPParticipantRequest(
        room_name=ctx.room.name,
        sip_trunk_id="ST_xxxxxxxx",
        sip_call_to="+15105550123",
        participant_identity="phone_user",
    )
)
```

**CreateSIPParticipantRequest 参数**：

| 参数 | 类型 | 说明 |
|------|------|------|
| `room_name` | `str` | 目标房间名 |
| `sip_trunk_id` | `str` | 出站 Trunk ID |
| `sip_call_to` | `str` | 目标号码（`+15105550123` 或 `sip:user@host`） |
| `participant_identity` | `str` | SIP 参与者身份标识 |
| `participant_name` | `str` | 参与者显示名称 |
| `wait_until_answered` | `bool` | 是否等待被叫应答（超时 80s） |

### 5.2 呼叫转移

```python
# 找出房间中的 SIP 参与者
participant = [
    p for p in ctx.room.remote_participants.values()
    if p.kind == rtc.ParticipantKind.PARTICIPANT_KIND_SIP
][0]

# 转移到指定号码
ctx.transfer_sip_participant(
    participant=participant,
    transfer_to="tel:+18003310500",
    play_dialtone=False,
)
```

`transfer_to` 格式：`"+12345555555"`（电话号码）或 `"sip:<user>@<host>"`（SIP URI）。

### 5.3 Warm Transfer 任务

[`beta/workflows/warm_transfer.py`](file:///d:/xuyong/source/AITest/livekit/livekit-agents/livekit-agents/livekit/agents/beta/workflows/warm_transfer.py)

`WarmTransferTask` 封装了温转移（Supervisor Escalation）的完整流程：

```python
result = await WarmTransferTask(
    target_phone_number=SUPERVISOR_PHONE_NUMBER,
    sip_trunk_id=SIP_TRUNK_ID,
    chat_ctx=self.chat_ctx,          # 传递对话历史给 Supervisor
    dtmf="1234#",
    ringing_timeout=15.0,
)
```

**内部流程**：
1. 拨打 Supervisor 号码（调用 `create_sip_participant`）
2. 可选发送 DTMF 拨号音（用于 IVR 导航）
3. 播放等待音乐（`BackgroundAudio`）
4. Supervisor 接听后，通过 `MoveParticipant` API 将其移入客户房间

---

## 6. SIP Bridge 通信协议

### 6.1 通信方式

LiveKit Server ↔ SIP Bridge 之间通过 **psrpc**（基于 Redis Pub/Sub 的 RPC 框架）进行双向通信。

### 6.2 请求方向

#### Server → Bridge（外呼/转移）

| RPC 方法 | 请求 | 响应 |
|----------|------|------|
| `CreateSIPParticipant` | `InternalCreateSIPParticipantRequest` | `InternalCreateSIPParticipantResponse` |
| `TransferSIPParticipant` | `InternalTransferSIPParticipantRequest` | `Empty` |

**InternalCreateSIPParticipantRequest 关键字段**：

| 字段 | 说明 |
|------|------|
| `SipCallId` | 唯一通话 ID |
| `ProjectId` | 项目 ID |
| `Address` | 目标 SIP 服务器地址 |
| `Number` | 主叫号码 |
| `Host` | 服务器地址 |
| `WsUrl` | WebSocket URL（用于加入房间） |
| `Token` | 加入房间的 JWT Token |
| `RoomName` | 房间名 |
| `ParticipantIdentity` | 参与者身份 |
| `CallTo` | 被叫号码 |
| `Headers` | 自定义 SIP Headers |
| `RingingTimeout` | 振铃超时 |

#### Bridge → Server（呼入路由）

| RPC 方法 | 请求 | 响应 |
|----------|------|------|
| `GetSIPTrunkAuthentication` | `GetSIPTrunkAuthenticationRequest` | `GetSIPTrunkAuthenticationResponse` |
| `EvaluateSIPDispatchRules` | `EvaluateSIPDispatchRulesRequest` | `EvaluateSIPDispatchRulesResponse` |
| `UpdateSIPCallState` | `UpdateSIPCallStateRequest` | `Empty` |

---

## 7. 关键数据模型

### 7.1 SIPCall

```go
type SIPCall struct {
    To        *SIPUri   // 被叫 URI（含 User, Host）
    From      *SIPUri   // 主叫 URI（含 User, Host）
    SourceIp  string    // 来源 IP 地址
}
```

### 7.2 参与者类型标识

SIP 参与者在 LiveKit 中通过特殊的 `ParticipantKind` 标识：

```python
rtc.ParticipantKind.PARTICIPANT_KIND_SIP
```

Agent 可以通过此标识过滤出 SIP 参与者：

```python
sip_participants = [
    p for p in ctx.room.remote_participants.values()
    if p.kind == rtc.ParticipantKind.PARTICIPANT_KIND_SIP
]
```

SIP 参与者还在其 Attributes 中携带 `livekit.AttrSIPCallID`，用于后续查找对应的 SIP 通话。

---

## 8. 关键常量

| 参数 | 默认值 | 说明 |
|------|--------|------|
| `CreateSIPParticipantTimeout` | 30s | 外呼超时（不等待应答） |
| `CreateSIPParticipantTimeoutAnswered` | 80s | 外呼超时（等待应答） |
| `TransferSIPParticipantTimeout` | 30s | 呼叫转移超时 |
| `SIPTrunkPrefix` | `ST_` | Trunk ID 前缀 |
| `SIPDispatchRulePrefix` | `SDR_` | Dispatch Rule ID 前缀 |
| Redis Key `sip_inbound_trunk` | — | 入站 Trunk 存储键 |
| Redis Key `sip_outbound_trunk` | — | 出站 Trunk 存储键 |
| Redis Key `sip_trunk` | — | 旧版 Trunk 存储键（已弃用） |
| Redis Key `sip_dispatch_rule` | — | Dispatch Rule 存储键 |

---

## 9. 典型场景

### 9.1 场景一：AI 语音助手接听电话

```
用户拨打 SIP 号码
  → SIP Bridge 收到 INVITE
  → Server: GetSIPTrunkAuthentication → 匹配 Trunk → 返回鉴权
  → Server: EvaluateSIPDispatchRules → 匹配 Dispatch Rule → ACCEPT
  → SIP Bridge 加入房间
  → AgentDispatch 触发 Agent 加入房间
  → Agent 与 SIP 用户在房间中对话
```

### 9.2 场景二：AI Agent 主动外呼

```
Agent 在已加入的房间中:
  → ctx.create_sip_participant(trunk_id, call_to, identity)
  → Server: CreateSIPParticipant → 构建请求 → psrpc 发送给 SIP Bridge
  → SIP Bridge 发起 SIP INVITE 到目标号码
  → 被叫振铃 → 应答 → SIP Bridge 加入房间
  → Agent 与电话用户在房间中开始对话
```

### 9.3 场景三：呼叫转移（Cold Transfer）

```
Agent 正在与 SIP 用户 A 通话:
  → Agent 调用 ctx.transfer_sip_participant(A, "B的号码")
  → Server: 从 A 的 Participant Attributes 提取 SipCallId
  → SIP Bridge 收到 TransferSIPParticipant → 发送 SIP REFER
  → A 的通话被转移到 B
  → Agent 可以关闭当前会话或继续处理其他事务
```

### 9.4 场景四：温转移（Warm Transfer / Supervisor Escalation）

```
Agent 正在与客户通话:
  → Agent 使用 WarmTransferTask
  → 拨打 Supervisor 号码（内部外呼）
  → 播放等待音乐给客户
  → Agent 向 Supervisor 简报情况
  → Supervisor 确认后，通过 MoveParticipant 移入客户房间
  → 三方在同一房间中通话
```

---

## 10. 配置与部署

### 10.1 Server 侧配置

SIP 依赖 Redis 存储，需在 `livekit.yaml` 中配置：

```yaml
redis:
  address: localhost:6379
```

SIPService 通过 Google Wire 依赖注入初始化：

```
SIPConfig → newSIPClient() → rpc.SIPClient
                            → NewSIPService(...)
                            → livekit.NewSIPServer(sipService) → 注册 HTTP 路由
```

### 10.2 错误码

[`pkg/service/errors.go`](file:///d:/xuyong/source/AITest/livekit/livekit/pkg/service/errors.go#L45-L48)

| 错误 | 说明 |
|------|------|
| `ErrSIPNotConnected` | Redis 未连接，SIP 不可用 |
| `ErrSIPTrunkNotFound` | 请求的 Trunk 不存在 |
| `ErrSIPDispatchRuleNotFound` | 请求的 Dispatch Rule 不存在 |
| `ErrSIPParticipantNotFound` | 请求的 SIP 参与者不存在 |

---

## 11. 关键设计决策

1. **两层架构** — LiveKit Server 负责调度路由，SIP Bridge 负责协议处理，各司其职
2. **psrpc 双向通信** — 使用 Redis Pub/Sub 作为 RPC 传输，支持多节点部署
3. **Trunk + Dispatch Rule 两级路由** — Trunk 管理 SIP 层面的配置（号码绑定、鉴权），Dispatch Rule 管理业务层面的路由（创建哪个房间、派发哪个 Agent）
4. **Redis 持久化** — SIP 配置存储在 Redis Hash 中，支持分布式多节点共享
5. **SIP 参与者标识** — 通过 `PARTICIPANT_KIND_SIP` 和 `AttrSIPCallID` Attribute 将 SIP 通话与 LiveKit 参与者关联
6. **向前兼容** — 存储层支持旧版 Trunk 格式的自动转换和加载
7. **超时可配置** — 外呼和转移超时均可配置，适应不同网络环境
8. **JWT 精细权限** — SIP.Admin 控制管理操作，SIP.Call 控制通话操作