# PSTN 手机号码呼入 LiveKit 系统完整流程

本文档详细介绍 PSTN（Public Switched Telephone Network，公共交换电话网络）手机号码如何拨打到 LiveKit 系统，并与 AI Agent 进行交互的完整流程。

---

## 目录

1. [整体架构](#整体架构)
2. [前置配置](#前置配置)
   1. [SIP Bridge 部署与配置（重要）](#1-sip-bridge-部署与配置重要)
   2. [SIP Inbound Trunk（入站中继）](#2-sip-inbound-trunk入站中继)
   3. [SIP Dispatch Rule（调度规则）](#3-sip-dispatch-rule调度规则)
3. [详细呼入流程](#详细呼入流程)
4. [Agent 侧实现示例](#agent-侧实现示例)
5. [关键数据结构](#关键数据结构)
6. [典型业务场景](#典型业务场景)
7. [总结](#总结)

---

## 整体架构

LiveKit 的 SIP 功能采用**两层架构**：

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
               ┌─────────────────────────────────────────┐
               │         LiveKit Server                  │
               │  ┌───────────────────────────────┐     │
               │  │ IOInfoService (呼入路由匹配)   │     │
               │  └───────────────────────────────┘     │
               │  ┌───────────────────────────────┐     │
               │  │ SIPService (管理 API)        │     │
               │  └───────────────────────────────┘     │
               │  ┌───────────────────────────────┐     │
               │  │ SIPStore (Redis 存储配置)     │     │
               │  └───────────────────────────────┘     │
               └──────────────────┬──────────────────────┘
                                  │
                                  ▼
                         ┌─────────────────┐
                         │  LiveKit Room   │
                         │                 │
                         │  ┌───────────┐ │
                         │  │ AI Agent  │ │
                         │  │  + SIP 用户│ │
                         │  └───────────┘ │
                         └─────────────────┘
```

### 核心组件

| 组件 | 职责 | 位置 |
|------|------|------|
| **PSTN/电话网络** | 传统电话网络，承载手机/座机通话 | 外部 |
| **SIP Bridge** | SIP 协议处理，RTP 媒体桥接，作为 Participant 加入房间 | 独立服务 |
| **LiveKit Server** | 调度路由，配置管理，API 暴露 | 核心服务 |
| **LiveKit Room** | 实时音视频房间，Agent 和 SIP 用户在此交互 | 核心服务 |
| **AI Agent** | 与 SIP 用户对话的智能代理 | livekit-agents |

### 技术栈

| 类别 | 技术选型 |
|------|----------|
| 协议处理 | 外部 SIP Bridge 服务 |
| Server ↔ Bridge 通信 | psrpc（双向 RPC，基于 Redis Pub/Sub） |
| 配置存储 | Redis（Hash 结构） |
| API 层 | Twirp（Protobuf HTTP） |
| 权限控制 | JWT ClaimGrants（SIP.Admin / SIP.Call） |
| Agent 侧调用 | Python SDK |

---

## 前置配置

在接收来电前，需要先配置几个关键组件。

### 1. SIP Bridge 部署与配置（重要）

**SIP Bridge 是一个独立的服务**，负责处理 SIP 协议和 RTP 媒体桥接，它需要单独部署和配置。SIP Bridge 的源代码和部署文档在独立的仓库：https://github.com/livekit/sip

**SIP Bridge 的关键配置点**：

| 配置项 | 说明 | 配置位置 |
|--------|------|---------|
| **SIP 监听地址/端口** | SIP Bridge 监听 SIP 请求的地址和端口，默认 SIP 端口是 **5060**（UDP/TCP），安全 SIP 是 **5061**（TLS） | SIP Bridge 配置文件 |
| **媒体（RTP）端口范围** | 用于 RTP 媒体流的端口范围，默认通常是 10000-20000 | SIP Bridge 配置文件 |
| **Redis 地址** | 连接到 LiveKit Server 使用的 Redis，用于 psrpc 通信 | SIP Bridge 配置文件 |
| **LiveKit WebSocket URL** | 用于加入 LiveKit 房间 | SIP Bridge 配置文件 |
| **LiveKit API Key/Secret** | 用于生成加入房间的 Token | SIP Bridge 配置文件 |

**PSTN/SIP 运营商侧配置**：

要让 PSTN 手机号码能打到 LiveKit 系统，需要在您的 SIP 运营商或 ITSP（Internet Telephony Service Provider）处配置：

| 配置项 | 说明 |
|--------|------|
| **SIP 中继（Trunk）** | 配置运营商的 SIP 中继指向您的 **SIP Bridge 的公网 IP 地址和端口**（默认 5060） |
| **电话号码（DID）** | 向运营商申请电话号码（Direct Inward Dialing），这个号码就是用户拨打的号码 |
| **鉴权配置** | 配置 SIP 鉴权的用户名和密码，与 SIP Inbound Trunk 中的 `InboundUsername`/`InboundPassword` 一致 |

**网络要求**：
- SIP Bridge 需要有公网 IP 或通过端口转发暴露 SIP（5060）和 RTP 端口范围
- 如果使用 UDP，需要开放相应的 UDP 端口
- 建议配置防火墙规则，只允许来自运营商 SIP 服务器的请求

### 2. SIP Inbound Trunk（入站中继）

**作用**：配置 SIP 层面的参数，包括绑定的电话号码、鉴权信息等。

**关键字段**：

| 字段 | 说明 |
|------|------|
| `SipTrunkId` | Trunk 唯一标识（前缀 `ST_`） |
| `Name` | Trunk 名称 |
| `Numbers` | 绑定的电话号码列表，用于入站号码匹配 |
| `InboundAddresses` | 入站允许的 SIP 地址列表 |
| `InboundUsername` / `InboundPassword` | 入站鉴权凭据 |
| `Metadata` | 元数据 |

### 3. SIP Dispatch Rule（调度规则）

**作用**：定义 SIP 来电应如何被路由处理。

**关键字段**：

| 字段 | 说明 |
|------|------|
| `SipDispatchRuleId` | 规则 ID（前缀 SDR_） |
| `TrunkIds` | 关联的 Trunk ID（空则匹配所有） |
| `Name` | 规则名称 |
| `Metadata` | 元数据 |
| `Rule` | 具体路由动作（oneof） |

**路由动作类型**：

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

### 配置示例（概念）

```yaml
# SIP Inbound Trunk
sip_inbound_trunk:
  sip_trunk_id: ST_abc123
  name: customer-support-trunk
  numbers:
    - +12345556666
  inbound_addresses:
    - sip.example.com
  inbound_username: livekit
  inbound_password: secret123

# SIP Dispatch Rule
sip_dispatch_rule:
  sip_dispatch_rule_id: SDR_def456
  trunk_ids:
    - ST_abc123
  name: support-queue-rule
  rule:
    dispatch_rule_direct:
      room_name: support-queue
```

---

## 详细呼入流程

让我们从用户拨打手机号码开始，一步步走完整个流程。

### 步骤 1：用户拨打手机号码

```
用户手机 (+19995550000)
      ↓ 拨打号码 (+12345556666)
PSTN/电话网络
      ↓ SIP INVITE
SIP Bridge 收到呼叫
```

SIP INVITE 消息包含：
- To URI（被叫号码）
- From URI（主叫号码）
- SDP（会话描述协议，包含媒体能力）
- 其他 SIP Header

---

### 步骤 2：SIP Bridge 与 LiveKit Server 通信 — 鉴权阶段

SIP Bridge 收到 INVITE 后，首先向 LiveKit Server 查询鉴权信息。

```go
// pkg/service/ioservice_sip.go

func (s *IOInfoService) GetSIPTrunkAuthentication(ctx context.Context, req *rpc.GetSIPTrunkAuthenticationRequest) (*rpc.GetSIPTrunkAuthenticationResponse, error) {
    // 从请求中提取 SIPCall 信息
    call := req.SIPCall()
    
    log := logger.GetLogger().WithValues(
        "toUser", call.To.User, 
        "fromUser", call.From.User, 
        "src", call.SourceIp
    )
    
    // 匹配 Trunk
    trunk, err := s.matchSIPTrunk(ctx, "", call)
    if err != nil {
        return nil, err
    }
    
    log.Debugw("SIP trunk matched for auth", "sipTrunk", trunk.SipTrunkId)
    
    // 返回认证要求（Username/Password）
    return sip.InboundTrunkAuthPrompt(trunk)
}
```

#### 2.1 Trunk 匹配逻辑

```go
func (s *IOInfoService) matchSIPTrunk(ctx context.Context, trunkID string, call *rpc.SIPCall) (*livekit.SIPInboundTrunkInfo, error) {
    // 如果提供了 trunkId，先尝试直接加载（优化）
    if trunkID != "" {
        if tr, err := s.ss.LoadSIPInboundTrunk(ctx, trunkID); err == nil {
            tr, err = sip.MatchTrunkIter(iters.Slice([]*livekit.SIPInboundTrunkInfo{tr}), call)
            if err == nil {
                return tr, nil
            }
        }
    }
    
    // 否则，列出可能的 Trunk 并匹配
    it := s.SelectSIPInboundTrunk(ctx, call.To.User)
    return sip.MatchTrunkIter(it, call)
}

func (s *IOInfoService) SelectSIPInboundTrunk(ctx context.Context, called string) iters.Iter[*livekit.SIPInboundTrunkInfo {
    // 查询 Redis，按被叫号码筛选 Trunk
    it := livekit.ListPageIter(s.ss.ListSIPInboundTrunk, &livekit.ListSIPInboundTrunkRequest{
        Numbers: []string{called},
    })
    return iters.PagesAsIter(ctx, it)
}
```

#### 2.2 Trunk 匹配优先级

从高到低：
1. **精确匹配**被叫号码（Numbers 字段完全匹配）
2. **匹配旧版 Trunk**（OutboundNumber 匹配，向前兼容）
3. **匹配通用 Trunk**（Numbers 为空 / 通配符）

---

### 步骤 3：SIP Bridge 完成 SIP 鉴权

SIP Bridge 使用 Trunk 返回的鉴权信息完成 SIP 层面的认证：
- 如果 Trunk 配置了 `InboundUsername` 和 `InboundPassword`，则 SIP Bridge 会发起 SIP 认证挑战（401/407）
- 待 SIP 客户端（或 SIP 运营商）携带正确的鉴权信息后继续

---

### 步骤 4：SIP Bridge 与 LiveKit Server 通信 — 路由阶段

鉴权通过后，SIP Bridge 再次向 LiveKit Server 查询路由规则，决定如何处理这个呼叫。

```go
func (s *IOInfoService) EvaluateSIPDispatchRules(ctx context.Context, req *rpc.EvaluateSIPDispatchRulesRequest) (*rpc.EvaluateSIPDispatchRulesResponse, error) {
    call := req.SIPCall()
    log := logger.GetLogger()
    log = log.WithValues("toUser", call.To.User, "fromUser", call.From.User, "src", call.SourceIp)
    
    // 验证 Source IP 格式（如果有）
    if call.SourceIp != "" {
        _, err := netip.ParseAddr(call.SourceIp)
        if err != nil {
            log.Errorw("cannot parse source IP", err)
            return nil, twirp.WrapError(twirp.NewError(twirp.InvalidArgument, err.Error()), err)
        }
    }
    
    // 再次确认 Trunk（可能携带 trunkId）
    trunk, err := s.matchSIPTrunk(ctx, req.SipTrunkId, call)
    if err != nil {
        return nil, err
    }
    trunkID := ""
    if trunk != nil {
        trunkID = trunk.SipTrunkId
        log.Debugw("SIP trunk matched")
    } else {
        log.Debugw("No SIP trunk matched")
    }
    
    // 匹配 Dispatch Rule
    best, err := s.matchSIPDispatchRule(ctx, trunk, req)
    if err != nil {
        // 没有匹配的 Dispatch Rule
        if e := (*sip.ErrNoDispatchMatched)(nil); errors.As(err, &e) {
            return &rpc.EvaluateSIPDispatchRulesResponse{
                SipTrunkId: trunkID,
                Result:     rpc.SIPDispatchResult_DROP, // 直接丢弃
            }, nil
        }
        return nil, err
    }
    
    log.Debugw("SIP dispatch rule matched", "sipRule", best.SipDispatchRuleId)
    
    // 执行规则逻辑
    resp, err := sip.EvaluateDispatchRule("", trunk, best, req)
    if err != nil {
        return nil, err
    }
    
    resp.Upgrade()
    resp.SipTrunkId = trunkID
    return resp, nil
}
```

#### 4.1 Dispatch Rule 匹配逻辑

```go
func (s *IOInfoService) matchSIPDispatchRule(ctx context.Context, trunk *livekit.SIPInboundTrunkInfo, req *rpc.EvaluateSIPDispatchRulesRequest) (*livekit.SIPDispatchRuleInfo, error) {
    var trunkID string
    if trunk != nil {
        trunkID = trunk.SipTrunkId
    }
    
    // 列出可能的 Dispatch Rule
    it := s.SelectSIPDispatchRule(ctx, trunkID)
    
    // 选择最佳匹配规则
    return sip.MatchDispatchRuleIter(trunk, it, req)
}

func (s *IOInfoService) SelectSIPDispatchRule(ctx context.Context, trunkID string) iters.Iter[*livekit.SIPDispatchRuleInfo {
    var trunkIDs []string
    if trunkID != "" {
        trunkIDs = []string{trunkID}
    }
    
    // 查询 Redis，按 trunkId 筛选 Dispatch Rule
    it := livekit.ListPageIter(s.ss.ListSIPDispatchRule, &livekit.ListSIPDispatchRuleRequest{
        TrunkIds: trunkIDs,
    })
    return iters.PagesAsIter(ctx, it)
}
```

#### 4.2 Dispatch Rule 匹配优先级

从高到低：
1. **精确匹配 TrunkIds**（规则绑定的 Trunk ID 列表）
2. **匹配通配规则**（TrunkIds 为空）

#### 4.3 DispatchResult（呼入路由结果）

| 结果 | 说明 |
|------|------|
| `ACCEPT` | 接受通话，创建房间/加入房间 |
| `REJECT` | 拒绝通话 |
| `DROP` | 直接丢弃（无 Dispatch Rule 匹配时） |
| `REQUEST_PIN` | 要求输入 PIN 码 |

如果返回 `ACCEPT`，响应中会包含：
- `RoomName`：目标房间名
- `Token`：加入房间的 JWT Token
- `WsUrl`：LiveKit WebSocket URL
- 其他必要信息

---

### 步骤 5：SIP Bridge 加入房间

SIP Bridge 根据返回的 `DispatchResult（ACCEPT）`：

1. 使用返回的 JWT Token
2. 作为 `PARTICIPANT_KIND_SIP` 类型的参与者加入房间
3. 桥接 RTP 媒体流 ↔ WebRTC
4. 发布音频轨道（从 SIP 到 LiveKit）
5. 订阅音频轨道（从 LiveKit 到 SIP）

**SIP Participant 标识**：
- `ParticipantKind`：`PARTICIPANT_KIND_SIP`
- `Attributes`：携带 `livekit.AttrSIPCallID` 存储 SIP Call ID

---

### 步骤 6：Agent Dispatch 触发 Agent 加入

当 SIP 参与者加入房间时，LiveKit 的 Agent Dispatch 机制被触发：

1. 根据 Dispatch Rule 的配置
2. 自动拉起相应的 Agent 加入同一房间
3. Agent 准备就绪，开始监听房间事件

---

### 步骤 7：Agent 与 SIP 用户对话

现在 Agent 和 SIP 用户在同一个 LiveKit Room 中：

1. Agent 订阅 SIP 用户的音频轨道
2. SIP 用户的语音被 STT（Speech-to-Text）转成文本
3. 文本被送到 LLM（Large Language Model）处理
4. LLM 生成响应文本
5. 响应文本被 TTS（Text-to-Speech）转成语音
6. 语音通过 Agent 的音频轨道发布给 SIP 用户
7. 两者开始双向对话！

---

## Agent 侧实现示例

以下是 `livekit-agents/examples/telephony/basic_dtmf_agent.py` 中的实现，展示了如何处理电话用户、支持 DTMF（按键）输入。

```python
import logging
import os

from dotenv import load_dotenv

from livekit.agents import (
    Agent,
    AgentSession,
    JobContext,
    MetricsCollectedEvent,
    cli,
    inference,
    metrics,
)
from livekit.agents.beta.workflows.dtmf_inputs import (
    GetDtmfTask,
)
from livekit.agents.llm.tool_context import ToolError, function_tool
from livekit.agents.voice.events import RunContext
from livekit.agents.worker import AgentServer
from livekit.plugins import silero
from livekit.plugins.turn_detector.multilingual import MultilingualModel

logger = logging.getLogger("dtmf-agent")

load_dotenv()

DTMF_AGENT_DISPATCH_NAME = os.getenv("DTMF_AGENT_DISPATCH_NAME", "my-telephony-agent")

server = AgentServer()


class DtmfAgent(Agent):
    def __init__(self) -> None:
        super().__init__(
            instructions=(
                "你是 Horizon Wireless 的自动客服，正在通过电话与用户交谈。"
                "请以简洁友好的问候开场，提到 Horizon Wireless。"
                "说明你会先通过 ask_for_phone_number 工具确认用户的账号电话号码。"
                "电话号码确认后，引导用户通过 ask_for_service_options 工具选择服务选项。"
                "保持专业、简洁，平滑地在步骤之间过渡。"
            ),
        )

        self.phone_number: str | None = None

    async def on_enter(self) -> None:
        # 当 Agent 加入时，先问候用户
        self.session.generate_reply(
            instructions=(
                "以 Horizon Wireless 虚拟助手的身份问候来电者，简要描述你的角色，"
                "并告诉用户你现在将收集他们的 10 位账号。"
            )
        )

    @function_tool
    async def ask_for_service_options(self, context: RunContext) -> str:
        """询问用户选择哪种电话服务。"""

        if self.phone_number is None:
            raise ToolError(
                "尚未提供电话号码，您应该先通过 ask_for_phone_number 工具询问"
            )

        while True:
            try:
                result = await GetDtmfTask(
                    num_digits=1,
                    chat_ctx=self.chat_ctx.copy(
                        exclude_instructions=True,
                        exclude_function_call=True,
                        exclude_handoff=True,
                        exclude_config_update=True,
                    ),
                    extra_instructions=(
                        "告诉用户可以选择三种 Horizon Wireless 服务之一："
                        "按 1 听取当前套餐详情，按 2 启用国际数据漫游，"
                        "或按 3 探索升级选项。请用户输入单个数字并稍等回应。"
                    ),
                    repeat_instructions=2,
                )
            except ToolError as e:
                await self.session.generate_reply(instructions=e.message, allow_interruptions=False)
                continue

            if result.user_input == "1":
                return "您当前的套餐是每月 $100"
            elif result.user_input == "2":
                return "国际数据漫游已启用"
            elif result.user_input == "3":
                return "您的新套餐是每月 $150"

            await self.session.generate_reply(
                instructions=(
                    "抱歉未识别该选项，请用户输入 1 查询套餐详情、"
                    "2 启用国际漫游或 3 升级套餐。"
                ),
                allow_interruptions=False,
            )

    @function_tool
    async def ask_for_phone_number(self, context: RunContext) -> str:
        """询问用户提供电话号码。"""
        while True:
            try:
                result = await GetDtmfTask(
                    num_digits=10,
                    chat_ctx=self.chat_ctx.copy(
                        exclude_instructions=True,
                        exclude_function_call=True,
                        exclude_handoff=True,
                        exclude_config_update=True,
                    ),
                    ask_for_confirmation=True,
                    extra_instructions=(
                        "告诉用户你将记录他们的 10 位账号，可以说出或拨入。"
                        "提供示例，如 415 555 0199，提醒用户信息会安全存储，"
                        "然后捕获数字。分组回读号码以确认，并邀请用户确认或重新输入。"
                    ),
                    repeat_instructions=2,
                )
            except ToolError as e:
                await self.session.generate_reply(instructions=e.message, allow_interruptions=False)
                continue

            break

        self.phone_number = result.user_input
        return f"用户的电话号码是 {result.user_input}"


@server.rtc_session(agent_name=DTMF_AGENT_DISPATCH_NAME)
async def entrypoint(ctx: JobContext) -> None:
    # 每个日志条目都会包含这些字段
    ctx.log_context_fields = {
        "room": ctx.room.name,
    }

    # 配置 AgentSession
    session: AgentSession = AgentSession(
        vad=silero.VAD.load(),  # 语音活动检测
        llm=inference.LLM("openai/gpt-4.1-mini"),  # 大语言模型
        stt=inference.STT("deepgram/nova-3"),  # 语音转文本
        tts=inference.TTS("inworld/inworld-tts-1"),  # 文本转语音
        turn_detection=MultilingualModel(),  # 多语言轮次检测
    )

    @session.on("metrics_collected")
    def _on_metrics_collected(ev: MetricsCollectedEvent) -> None:
        metrics.log_metrics(ev.metrics)

    async def log_usage() -> None:
        logger.info(f"Usage: {session.usage}")

    # 会话结束时的回调
    ctx.add_shutdown_callback(log_usage)

    # 启动会话
    await session.start(
        agent=DtmfAgent(),
        room=ctx.room,
    )


if __name__ == "__main__":
    cli.run_app(server)
```

### Agent 识别 SIP 参与者

Agent 可以通过以下方式识别和筛选 SIP 参与者：

```python
# 方式一：通过 ParticipantKind 筛选
sip_participants = [
    p for p in ctx.room.remote_participants.values()
    if p.kind == rtc.ParticipantKind.PARTICIPANT_KIND_SIP
]

# 方式二：通过 Attributes 检查（更可靠）
sip_participants = []
for p in ctx.room.remote_participants.values():
    if livekit.AttrSIPCallID in p.attributes:
        sip_participants.append(p)

# 获取 SIP Call ID（用于后续操作，如呼叫转移）
if sip_participants:
    sip_call_id = sip_participants[0].attributes.get(livekit.AttrSIPCallID)
    logger.info(f"SIP Call ID: {sip_call_id}")
```

---

## 关键数据结构

### 1. SIPCall

```go
type SIPCall struct {
    To        *SIPUri   // 被叫 URI（含 User, Host）
    From      *SIPUri   // 主叫 URI（含 User, Host）
    SourceIp  string    // 来源 IP 地址
}

type SIPUri struct {
    User  string
    Host  string
    Raw   string
}
```

### 2. SIPInboundTrunkInfo

```go
type SIPInboundTrunkInfo struct {
    SipTrunkId       string
    Name             string
    Numbers          []string
    InboundAddresses []string
    InboundUsername  string
    InboundPassword  string
    Metadata         string
}
```

### 3. SIPDispatchRuleInfo

```go
type SIPDispatchRuleInfo struct {
    SipDispatchRuleId string
    TrunkIds          []string
    Name              string
    Metadata          string
    Rule              *SIPDispatchRule
}

// DispatchRule 的 oneof 类型
type SIPDispatchRule struct {
    DispatchRuleDirect     *DispatchRuleDirect
    DispatchRuleIndividual *DispatchRuleIndividual
    DispatchRuleGroup      *DispatchRuleGroup
    DispatchRuleCallee     *DispatchRuleCallee
}

type DispatchRuleDirect struct {
    RoomName string
    Pin      string
}
```

### 4. Redis 存储键

| Redis Key | 说明 |
|-----------|------|
| `sip_inbound_trunk` | 入站 Trunk 配置（Hash: trunkID → Proto） |
| `sip_outbound_trunk` | 出站 Trunk 配置（Hash: trunkID → Proto） |
| `sip_trunk` | 旧版 Trunk 配置（Hash: trunkID → Proto，已弃用） |
| `sip_dispatch_rule` | Dispatch Rule 配置（Hash: ruleID → Proto） |

### 5. ID 前缀约定

| 前缀 | 实体类型 |
|------|----------|
| `ST_` | SIP Trunk |
| `SDR_` | SIP Dispatch Rule |

---

## 典型业务场景

### 场景一：AI 语音助手接听电话

```
用户拨打客服电话 (+12345556666)
  ↓
SIP Bridge 收到 INVITE
  ↓
Server: GetSIPTrunkAuthentication → 匹配 Trunk → 返回鉴权
  ↓
Server: EvaluateSIPDispatchRules → 匹配 Dispatch Rule → ACCEPT
  ↓
SIP Bridge 加入房间（"support-queue-1"）
  ↓
AgentDispatch 触发 Agent 加入房间
  ↓
Agent 与 SIP 用户在房间中对话
  ↓
Agent 使用 LLM 理解用户需求
  ↓
Agent 调用工具（查询订单、预约等）
  ↓
Agent 通过 TTS 播放语音回复
```

---

## 关键模块代码参考

### 1. SIPService（管理 API）

位置：`pkg/service/sip.go`

负责管理 Trunk、Dispatch Rule，处理外呼请求。

```go
type SIPService struct {
    conf        *config.SIPConfig
    nodeID      livekit.NodeID
    bus         psrpc.MessageBus
    psrpcClient rpc.SIPClient
    store       SIPStore
    roomService livekit.RoomService
}
```

主要 API 方法：
- `CreateSIPInboundTrunk` / `CreateSIPOutboundTrunk`
- `UpdateSIPInboundTrunk` / `UpdateSIPOutboundTrunk`
- `GetSIPInboundTrunk` / `GetSIPOutboundTrunk`
- `ListSIPInboundTrunk` / `ListSIPOutboundTrunk`
- `DeleteSIPTrunk`
- `CreateSIPDispatchRule` / `UpdateSIPDispatchRule` / `DeleteSIPDispatchRule`
- `ListSIPDispatchRule`
- `CreateSIPParticipant`（外呼）
- `TransferSIPParticipant`（呼叫转移）

### 2. IOInfoService（呼入路由匹配）

位置：`pkg/service/ioservice_sip.go`

通过 psrpc 对外暴露 RPC，供 SIP Bridge 在收到来电时调用。

主要 RPC 方法：
- `GetSIPTrunkAuthentication`：鉴权阶段调用
- `EvaluateSIPDispatchRules`：路由阶段调用
- `UpdateSIPCallState`：通话状态更新（预留）

### 3. SIPStore（持久化存储）

位置：`pkg/service/redisstore_sip.go`

SIP 配置数据存储在 Redis 中，使用 Hash 结构组织。

```go
type RedisSIPStore struct {
    rc redis.Client
}
```

---

## 权限控制

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

## 关键设计决策

1. **两层架构** — LiveKit Server 负责调度路由，SIP Bridge 负责协议处理，各司其职
2. **psrpc 双向通信** — 使用 Redis Pub/Sub 作为 RPC 传输，支持多节点部署
3. **Trunk + Dispatch Rule 两级路由** — Trunk 管理 SIP 层面的配置（号码绑定、鉴权），Dispatch Rule 管理业务层面的路由（创建哪个房间、派发哪个 Agent）
4. **Redis 持久化** — SIP 配置存储在 Redis Hash 中，支持分布式多节点共享
5. **鉴权与路由分离** — SIP Bridge 先通过 `GetSIPTrunkAuthentication` 获取鉴权信息完成 SIP 层面的认证；认证通过后再通过 `EvaluateSIPDispatchRules` 决定业务路由
6. **两次 Trunk 匹配** — 鉴权阶段和路由阶段各执行一次 `matchSIPTrunk`，但路由阶段可能携带 `trunkId` 加速查找
7. **SIP 参与者标识** — 通过 `PARTICIPANT_KIND_SIP` 和 `AttrSIPCallID` Attribute 将 SIP 通话与 LiveKit 参与者关联
8. **向前兼容** — 存储层支持旧版 Trunk 格式的自动转换和加载
9. **超时可配置** — 外呼和转移超时均可配置，适应不同网络环境
10. **JWT 精细权限** — SIP.Admin 控制管理操作，SIP.Call 控制通话操作

---

## 总结

### PSTN 手机号码呼入 LiveKit 并与 Agent 交互的完整链路：

1. **配置层**：SIP Inbound Trunk（绑定号码、鉴权）+ SIP Dispatch Rule（路由规则）
2. **协议层**：SIP Bridge 处理 SIP INVITE/RTP
3. **路由层**：LiveKit Server 匹配 Trunk 和 Dispatch Rule，返回 Token
4. **会话层**：SIP Bridge 作为 Participant 加入房间
5. **应用层**：Agent Dispatch 触发 Agent 加入，开始对话

### 关键端口配置总结：

| 端口/端口范围 | 用途 | 配置位置 |
|--------------|------|---------|
| **5060** (UDP/TCP) | SIP 标准监听端口，用于接收 SIP INVITE 等请求 | SIP Bridge 配置文件 |
| **5061** (TLS) | 安全 SIP 监听端口（可选） | SIP Bridge 配置文件 |
| **10000-20000** (UDP) | RTP 媒体流端口范围，用于传输语音数据 | SIP Bridge 配置文件 |
| Redis 端口 (默认 6379) | psrpc 通信，用于 LiveKit Server 与 SIP Bridge 之间的 RPC | LiveKit Server 和 SIP Bridge 配置文件 |
| LiveKit WebSocket (默认 7881) | SIP Bridge 加入房间的连接 | LiveKit Server 和 SIP Bridge 配置文件 |

### 网络配置要点：

1. **SIP Bridge 需要暴露在公网**，至少需要公网 IP 或端口转发
2. **防火墙需要开放**：
   - SIP 端口（5060 UDP/TCP）
   - RTP 端口范围（10000-20000 UDP）
3. **PSTN/SIP 运营商需要配置**：
   - 将 SIP 中继指向 SIP Bridge 的公网 IP:5060
   - 申请 DID 电话号码
   - 配置 SIP 鉴权

所有配置数据存储在 Redis 中，通过 psrpc 实现 LiveKit Server 与 SIP Bridge 的双向通信。
