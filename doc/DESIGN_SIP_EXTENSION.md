# LiveKit SIP 分机路由设计文档

## 1. 概述

传统电话系统（PBX）中的"分机"概念——注册分机号、分机互拨、分机振铃组等——依赖于 SIP REGISTER 协议。LiveKit 明确**不支持 SIP REGISTER**，因此不存在传统意义上的分机系统。

LiveKit 采用完全不同的架构思路，通过 **Trunk + Dispatch Rule** 两层路由机制，结合 Agent 框架，实现了比传统分机更灵活的电话路由与 AI 交互能力。

### 与传统 PBX 概念对照

| 传统 PBX 概念 | LiveKit 对应方式 | 说明 |
|--------------|-----------------|------|
| **SIP 分机注册** (REGISTER) | ❌ 不支持 | SIP 话机不能向 LiveKit 注册 |
| **分机号** (如 101, 102) | 通过真实**电话号码**区分 | LiveKit 用 PSTN 号码作为"分机"标识 |
| **分机互拨** | **Room 内多参与者** 实现 | 多人在同一个 LiveKit Room 中通话 |
| **IVR 导航** | **Agent 语音交互** 实现 | Agent 引导用户，利用 LLM 理解意图并路由 |
| **ACD 排队** | **Dispatch Rule + Agent** 实现 | 来电通过规则分配到房间 |
| **分机组 / 技能组** | **多个 Dispatch Rule** 实现 | 不同号码 → 不同 Agent |

---

## 2. 架构对比

### 传统 PBX 架构

```
SIP 话机 ──REGISTER──> PBX Server
                         │
                         ├── 分机表: 101→Alice, 102→Bob
                         ├── IVR: 按键路由
                         ├── 队列: 排队→分机组
                         └── 分机互拨: 101 拨打 102
```

### LiveKit 架构

```
电话网络 (PSTN)
      │
      ▼
┌─────────────┐
│  SIP Bridge  │  ← SIP 协议处理（无 REGISTER）
└──────┬──────┘
       │ psrpc RPC
       ▼
┌──────────────────────────────────────────┐
│           LiveKit Server                 │
│                                          │
│  ┌─────────────────────────────────┐    │
│  │        SIP Trunk (运营商配置)     │    │
│  │  - 入站 Trunk: 号码 + 鉴权       │    │
│  │  - 出站 Trunk: 运营商 + 主叫号码  │    │
│  └──────────────┬──────────────────┘    │
│                 ▼                        │
│  ┌─────────────────────────────────┐    │
│  │     Dispatch Rule (路由规则)     │    │
│  │  - Direct:   全部进同一房间       │    │
│  │  - Individual: 每人独立房间      │    │
│  │  - 按号码分发: 多号码 → 多房间    │    │
│  └──────────────┬──────────────────┘    │
│                 ▼                        │
│  ┌─────────────────────────────────┐    │
│  │        LiveKit Room + Agent      │    │
│  │  - 一个房间 = 一个通话会话        │    │
│  │  - Agent = AI 语音助手           │    │
│  └─────────────────────────────────┘    │
└──────────────────────────────────────────┘
```

---

## 3. 核心概念：用"电话号码"替代"分机号"

在 LiveKit 中，每个电话号码天然就是"分机号"。你不需要在系统中维护分机注册表，只需要**购买或配置电话号码，并设置路由规则**。

### 3.1 号码 → 房间的映射

```
 拨入号码                       路由结果
──────────────────────────────────────────────
 +15105550123  ──→  customer-service  房间
 +15105550124  ──→  tech-support      房间
 +15105550999  ──→  sales             房间
 其他号码       ──→  default-room     房间
```

### 3.2 号码配置方式

#### 方式一：Trunk 级别绑定（推荐）

在创建 Inbound Trunk 时，将该 Trunk 与特定号码绑定：

```json
{
    "name": "customer-service-trunk",
    "numbers": ["+15105550123"],
    "inbound_addresses": ["203.0.113.0/24"],
    "inbound_username": "auth_user",
    "inbound_password": "auth_pass"
}
```

#### 方式二：Dispatch Rule 级别过滤

在 Dispatch Rule 上通过 `inbound_numbers` 字段进一步过滤：

```python
from livekit import api

lkapi = api.LiveKitAPI()

# 即使同一个 Trunk 有多个号码，也能分别路由
rule = api.CreateSIPDispatchRuleRequest(
    dispatch_rule=api.SIPDispatchRuleInfo(
        rule=api.SIPDispatchRule(
            dispatch_rule_direct=api.SIPDispatchRuleDirect(room_name="customer-service")
        ),
        name="customer-service-rule",
        inbound_numbers=["+15105550123"],   # 只匹配这个号码
        trunk_ids=["ST_xxxx"],
        room_config=api.RoomConfiguration(
            agents=[api.RoomAgentDispatch(agent_name="cs-agent")]
        )
    )
)
await lkapi.sip.create_sip_dispatch_rule(request=rule)
```

---

## 4. Dispatch Rule 路由模式详解

### 4.1 Direct — 固定房间（类会议室）

所有来电者进入**同一个房间**，适合客服中心等场景：

```
来电者 A ──┐
来电者 B ──┼──> [support-room] <── Agent (客服 AI)
来电者 C ──┘
```

**配置示例（JSON）**：

```json
{
    "rule": {
        "dispatchRuleDirect": {
            "roomName": "support-room",
            "pin": "1234"
        }
    },
    "trunk_ids": ["ST_xxx"],
    "name": "all-to-one-room",
    "room_config": {
        "agents": [{
            "agent_name": "support-agent",
            "metadata": "customer support"
        }]
    }
}
```

**适用场景**：
- 客服热线（所有客服在同一"虚拟工位"）
- 会议桥（多人在同一房间通话）
- 广播室（一个 Agent 对多个来电者）

### 4.2 Individual — 独立房间（类独立分机）

每个来电者进入**独立房间**，相当于每人一个"专属分机"：

```
来电者 张三 (+1510xxx)  ──> [call-张三-abc123] <── Agent 实例 A
来电者 李四 (+1510yyy)  ──> [call-李四-def456] <── Agent 实例 B
来电者 王五 (+1510zzz)  ──> [call-王五-ghi789] <── Agent 实例 C
```

**配置示例（Python）**：

```python
rule = api.SIPDispatchRule(
    dispatch_rule_individual=api.SIPDispatchRuleIndividual(
        room_prefix="call-",    # 房间名前缀
    )
)

request = api.CreateSIPDispatchRuleRequest(
    dispatch_rule=api.SIPDispatchRuleInfo(
        rule=rule,
        name="one-agent-per-caller",
        trunk_ids=["ST_xxx"],
        room_config=api.RoomConfiguration(
            agents=[api.RoomAgentDispatch(agent_name="inbound-agent")]
        )
    )
)

dispatch = await lkapi.sip.create_sip_dispatch_rule(request=request)
```

**房间命名规则**：

| 场景 | 房间名示例 |
|------|-----------|
| 前缀 `call-`，号码 +15105550123 | `call-+15105550123_7a3f` |
| 前缀空，号码 +15105550123 | `+15105550123_7a3f` |
| 前缀 `ivr-`，号码 +15105550123 | `ivr-+15105550123_b2e1` |

每个房间名后附加随机后缀，避免同一号码重复拨入时的冲突。

**适用场景**：
- AI 语音助手（每个用户独立会话）
- 预订系统（每人独立下单流程）
- 个人专属客服

### 4.3 按被叫号码多路线（类分机组）

同一个 Trunk 上的不同号码路由到不同 Agent：

```python
# 号码 +15105550123 → 客服 Agent
rule_cs = api.CreateSIPDispatchRuleRequest(
    dispatch_rule=api.SIPDispatchRuleInfo(
        rule=api.SIPDispatchRule(
            dispatch_rule_direct=api.SIPDispatchRuleDirect(room_name="customer-service")
        ),
        name="cs-route",
        inbound_numbers=["+15105550123"],
        trunk_ids=["ST_xxx"],
        room_config=api.RoomConfiguration(
            agents=[api.RoomAgentDispatch(agent_name="cs-agent")]
        )
    )
)

# 号码 +15105550124 → 技术支持 Agent
rule_tech = api.CreateSIPDispatchRuleRequest(
    dispatch_rule=api.SIPDispatchRuleInfo(
        rule=api.SIPDispatchRule(
            dispatch_rule_direct=api.SIPDispatchRuleDirect(room_name="tech-support")
        ),
        name="tech-route",
        inbound_numbers=["+15105550124"],
        trunk_ids=["ST_xxx"],
        room_config=api.RoomConfiguration(
            agents=[api.RoomAgentDispatch(agent_name="tech-agent")]
        )
    )
)

await lkapi.sip.create_sip_dispatch_rule(request=rule_cs)
await lkapi.sip.create_sip_dispatch_rule(request=rule_tech)
```

**适用场景**：
- 多部门电话系统（不同号码接入不同部门）
- 多语言线路（英语号码 → 英语 Agent，中文号码 → 中文 Agent）

### 4.4 路由模式总结

| 路由模式 | 房间策略 | 房间名 | Agent 实例数 | 典型场景 |
|----------|---------|--------|-------------|---------|
| **Direct** | 所有人同一房间 | 固定房间名 | 1 个 Agent 服务所有人 | 客服中心、会议室 |
| **Individual** | 每人独立房间 | 前缀 + 号码 + 后缀 | 每人 1 个 Agent | AI 个人助手 |
| **多号码路由** | 按号码分发 | 固定/独立（按规则） | 每个号码对应一组 Agent | 多部门电话系统 |

---

## 5. Agent 自动派发

Dispatch Rule 的 `room_config.agents` 字段可以配置当房间创建时自动派发 Agent：

```python
room_config = api.RoomConfiguration(
    agents=[
        api.RoomAgentDispatch(
            agent_name="inbound-agent",   # Worker 注册的 Agent 名
            metadata="dispatch metadata", # 传递给 Agent 的元数据
        )
    ]
)
```

**工作原理**：

```
来电到达
  │
  ├─ Dispatch Rule 决定创建/进入房间
  │
  ├─ Room 创建时，根据 room_config.agents 创建 AgentDispatch
  │
  ├─ AgentDispatch → AgentService → Worker 分配 → Agent
  │
  └─ Agent 使用 JobContext.connect() 加入房间
```

这相当于传统 PBX 中的"分机振铃组"功能，区别在于振铃的不是物理话机，而是 AI Agent 进程。

---

## 6. 完整配置示例

### 6.1 场景：多部门 AI 客服中心

假设你有一个公司，需要以下电话路由：

| 号码 | 用途 | 路由方式 | Agent |
|------|------|---------|-------|
| `+15105550100` | 总机 | Direct → `main-lobby` | `reception-agent` |
| `+15105550101` | 销售部 | Individual → `sales-*` | `sales-agent` |
| `+15105550102` | 技术支持 | Direct → `tech-support` | `tech-agent` |

**步骤 1：配置 SIP Trunk**（一次性配置，使用 livekit-cli 或 SDK）：

```bash
livekit-cli create-sip-inbound-trunk --request <<EOF
{
    "trunk": {
        "name": "company-main-trunk",
        "numbers": ["+15105550100", "+15105550101", "+15105550102"],
        "inbound_addresses": ["203.0.113.0/24"],
        "inbound_username": "company_auth",
        "inbound_password": "strong_password"
    }
}
EOF
```

```bash
livekit-cli create-sip-outbound-trunk --request <<EOF
{
    "trunk": {
        "name": "company-outbound-trunk",
        "address": "sip.twilio.com",
        "numbers": ["+15105550100"],
        "transport": "TLS",
        "username": "outbound_auth",
        "password": "outbound_password"
    }
}
EOF
```

**步骤 2：配置 Dispatch Rules**（Python SDK）：

```python
from livekit import api

lkapi = api.LiveKitAPI()
TRUNK_ID = "ST_xxxxxxxx"

# 总机：一个房间，接待所有来电
rule_reception = api.CreateSIPDispatchRuleRequest(
    dispatch_rule=api.SIPDispatchRuleInfo(
        rule=api.SIPDispatchRule(
            dispatch_rule_direct=api.SIPDispatchRuleDirect(room_name="main-lobby")
        ),
        name="reception-route",
        inbound_numbers=["+15105550100"],
        trunk_ids=[TRUNK_ID],
        room_config=api.RoomConfiguration(
            agents=[api.RoomAgentDispatch(agent_name="reception-agent")]
        )
    )
)

# 销售部：每人独立房间
rule_sales = api.CreateSIPDispatchRuleRequest(
    dispatch_rule=api.SIPDispatchRuleInfo(
        rule=api.SIPDispatchRule(
            dispatch_rule_individual=api.SIPDispatchRuleIndividual(room_prefix="sales-")
        ),
        name="sales-route",
        inbound_numbers=["+15105550101"],
        trunk_ids=[TRUNK_ID],
        room_config=api.RoomConfiguration(
            agents=[api.RoomAgentDispatch(agent_name="sales-agent")]
        )
    )
)

# 技术支持：一个房间
rule_tech = api.CreateSIPDispatchRuleRequest(
    dispatch_rule=api.SIPDispatchRuleInfo(
        rule=api.SIPDispatchRule(
            dispatch_rule_direct=api.SIPDispatchRuleDirect(room_name="tech-support")
        ),
        name="tech-route",
        inbound_numbers=["+15105550102"],
        trunk_ids=[TRUNK_ID],
        room_config=api.RoomConfiguration(
            agents=[api.RoomAgentDispatch(agent_name="tech-agent")]
        )
    )
)

for rule in [rule_reception, rule_sales, rule_tech]:
    await lkapi.sip.create_sip_dispatch_rule(request=rule)

await lkapi.aclose()
```

**步骤 3：编写 Agent 入口**：

```python
# reception-agent
@server.rtc_session(agent_name="reception-agent")
async def reception_entry(ctx: JobContext):
    await ctx.connect()
    session = AgentSession()
    await session.start(
        agent=Agent(
            instructions="你是公司总机接待。根据用户需求，将其转接到销售部或技术支持。",
            llm=openai.LLM(model="gpt-4o"),
            tts=openai.TTS(),
            stt=deepgram.STT(),
        ),
        room=ctx.room,
    )

# sales-agent
@server.rtc_session(agent_name="sales-agent")
async def sales_entry(ctx: JobContext):
    await ctx.connect()
    session = AgentSession()
    await session.start(
        agent=Agent(
            instructions="你是销售顾问，帮助用户了解产品并完成购买。",
            llm=openai.LLM(model="gpt-4o"),
            tts=openai.TTS(),
            stt=deepgram.STT(),
        ),
        room=ctx.room,
    )
```

---

## 7. 高级场景

### 7.1 Agent 驱动的来电转接（Warm Transfer）

当接待 Agent 需要将用户转给销售 Agent 时，使用 `WarmTransferTask`：

```python
from livekit.agents.beta.workflows import WarmTransferTask

# 通过 SIP 外呼拨打内部"销售分机号码"
result = await WarmTransferTask(
    sip_call_to="+15105550101",      # 销售的 SIP 号码
    sip_trunk_id="ST_xxxxxxxx",       # 出站 Trunk ID
    chat_ctx=self.chat_ctx,           # 传递对话上下文
    instructions="客户需要购买产品X，请继续服务。",
)
```

**流程**：

```
客户拨打总机 +15105550100
  └─ 进入 main-lobby 房间
       └─ reception-agent 接待
            └─ 判断需要销售 → 调用 WarmTransferTask
                 └─ 外呼 +15105550101 → 进入 sales-* 房间
                      └─ sales-agent 接听
                           └─ MoveParticipant → 三方在同一房间
```

### 7.2 Agent 主动外呼（类"分机外呼"）

Agent 在处理过程中需要主动联系外部人员：

```python
@function_tool
async def call_expert(context: RunContext, phone_number: str, reason: str) -> str:
    """外呼专家"""
    await context.session.userdata["ctx"].create_sip_participant(
        trunk_id="ST_xxxxxxxx",
        call_to=phone_number,
        participant_identity="expert_" + phone_number,
    )
    return f"已拨打专家号码 {phone_number}，原因：{reason}"
```

### 7.3 通话转 IVR（Cold Transfer）

将当前 SIP 通话转移到另一个号码（如真人客服）：

```python
# 在 Agent 中找出 SIP 参与者
sip_participants = [
    p for p in ctx.room.remote_participants.values()
    if p.kind == rtc.ParticipantKind.PARTICIPANT_KIND_SIP
]

if sip_participants:
    await ctx.transfer_sip_participant(
        participant=sip_participants[0],
        transfer_to="+18005551234",   # 真人客服号码
        play_dialtone=True,
    )
```

---

## 8. 与传统 PBX 的功能映射

| 传统 PBX 功能 | LiveKit 实现方式 | 复杂度 |
|-------------|-----------------|--------|
| 分机注册 | ❌ 不支持（需外挂 PBX） | — |
| 分机互拨 | Room 内多人通话 / Warm Transfer | 中 |
| IVR 语音导航 | Agent LLM 自然语言理解 | 低 |
| ACD 排队 | Dispatch Rule 独立房间（天然隔离） | 低 |
| 来电显示 | SIP 参与者属性自动携带 | 自动 |
| DTMF 按键 | Agent 框架原生支持 `send_dtmf` + `DTMFInputs` | 低 |
| 通话录音 | `AgentSession` 的 RecordingOptions | 低 |
| 呼叫转移 (Blind) | `TransferSIPParticipant` | 低 |
| 呼叫转移 (Attended) | `WarmTransferTask` | 中 |
| 保持/等待音乐 | `BackgroundAudio` 播放内置音效 | 低 |
| 答录机检测 | `AMD` (Answering Machine Detection) | 低 |
| 通话统计 | `MetricsCollectedEvent` + Prometheus | 低 |

---

## 9. 关键设计决策

1. **无分机注册** — LiveKit 不实现 SIP REGISTER，不维护内部分机表，降低了系统复杂度，适合 AI-first 场景
2. **号码即身份** — 用真实 PSTN 电话号码区分用户，不需要额外的内部分机号体系
3. **Room 即会话** — 一个 Room = 一次通话/一个业务会话，"分机互拨"变成 Room 内的多人通话
4. **Agent 即话务员** — AI Agent 取代真人座席，Dispatch Rule 自动派发 Agent 实例
5. **路由灵活** — 通过 Trunk + Dispatch Rule 两层路由，实现比传统分机更复杂的路由逻辑
6. **可编程转移** — 通过 LLM 理解用户意图后自动进行 Warm Transfer 或 Cold Transfer，无需 DTMF 按键导航