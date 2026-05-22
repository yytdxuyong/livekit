# LiveKit 录音（Egress）系统设计文档

本文档详细介绍 LiveKit 录音系统（Egress/Recording）的架构设计、核心模块、数据流程和实现细节。

---

## 目录

1. [概述](#1-概述)
2. [整体架构](#2-整体架构)
3. [Egress 类型](#3-egress-类型)
4. [LiveKit Server 侧 — EgressService](#4-livekit-server-侧--egressservice)
5. [LiveKit Egress 独立服务](#5-livekit-egress-独立服务)
6. [自动录制 — AutoEgress](#6-自动录制--autoegress)
7. [Agent 侧录音 — RecorderIO](#7-agent-侧录音--recorderio)
8. [第三方集成示例 — Hamming Plugin](#8-第三方集成示例--hamming-plugin)
9. [Egress 状态机](#9-egress-状态机)
10. [权限控制](#10-权限控制)
11. [输出格式与存储](#11-输出格式与存储)
12. [与 SIP Bridge 的协作](#12-与-sip-bridge-的协作)
13. [历史演进](#13-历史演进)
14. [关键设计决策](#14-关键设计决策)

---

## 1. 概述

LiveKit 的录音系统（称为 **Egress**）负责将 LiveKit 房间中的音视频流录制为文件或推流到外部媒体服务。它是一个**独立于 LiveKit Server 的专门服务**，通过 psrpc 与 LiveKit Server 通信。

### 功能特性

- 录制整个房间的混合音视频
- 录制单个参与者的音视频
- 录制指定的轨道组合
- 自动录制（房间创建时配置，轨道发布时自动触发）
- 支持多种输出格式（MP4、OGG、WebM）
- 支持多种存储目标（S3、GCS、Azure Blob、本地文件系统）
- 支持 RTMP 推流
- 支持录制布局更新（动态调整画面布局）
- Agent 侧本地录制（用于通话报告）

### 历史演进

```
LiveKit v1.0.x:     RecordingService (旧版)
LiveKit v1.1.0+:    EgressService (替代 RecordingService)
LiveKit v1.3.0+:    独立 Egress 服务作为推荐方案
当前:               Egress 独立服务（LiveKit Cloud 和自托管均支持）
```

> 从 CHANGELOG 可以看到：`Removed deprecated RecordingService - Egress should be used instead`

---

## 2. 整体架构

### 2.1 系统架构

```
                  客户端/Agent 调用 API
                        │
                        ▼
              ┌──────────────────────┐
              │  LiveKit Server      │
              │  ┌────────────────┐  │
              │  │ EgressService  │  │  ← Twirp HTTP API
              │  │ pkg/service/   │  │
              │  │ egress.go      │  │
              │  └───────┬────────┘  │
              └──────────┼───────────┘
                         │ psrpc (Redis Pub/Sub)
              ┌──────────▼───────────┐
              │  LiveKit Egress      │  ← 独立服务
              │  github.com/         │
              │  livekit/egress      │
              │                      │
              │  ┌────────────────┐  │
              │  │ RoomComposite  │  │  ← 房间录制器
              │  │ Egress         │  │
              │  ├────────────────┤  │
              │  │ Participant    │  │  ← 参与者录制器
              │  │ Egress         │  │
              │  ├────────────────┤  │
              │  │ Track          │  │  ← 轨道录制器
              │  │ Egress         │  │
              │  └────────┬───────┘  │
              └───────────┼──────────┘
                          │
                 ┌────────▼────────┐
                 │  编码 + 封装     │
                 │  FFmpeg / GStreamer │
                 └────────┬────────┘
                          │
              ┌───────────┴────────────┐
              │                        │
         ┌────▼────┐  ┌─────▼─────┐  ┌─▼──────┐
         │ 本地文件 │  │  S3/GCS   │  │ RTMP   │
         │ 系统     │  │  Azure    │  │ 推流   │
         └─────────┘  └───────────┘  └────────┘
```

### 2.2 组件通信

```
┌──────────────────────────────────────────────────────────┐
│                    组件通信协议                             │
│                                                           │
│  客户端/Agent ── Twirp (HTTP/Protobuf) ──→ LiveKit Server │
│                                                           │
│  LiveKit Server ── psrpc (Redis Pub/Sub) ──→ Egress 服务  │
│                                                           │
│  Egress 服务 ── WebSocket (livekit-server-sdk) ──→ Room   │
│                    作为 Egress Participant 加入房间         │
│                                                           │
│  Egress 服务 ── 获取轨道数据 ──→ 混音/编码 ──→ 输出文件   │
│                                                           │
│  LiveKit Server ── Webhook ──→ 外部系统（通知录制状态）   │
└──────────────────────────────────────────────────────────┘
```

### 2.3 核心代码位置

| 模块 | 位置 | 语言 |
|------|------|------|
| EgressService (API 层) | `livekit/pkg/service/egress.go` | Go |
| EgressLauncher (内部启动器) | `livekit/pkg/service/egress.go` | Go |
| AutoEgress (自动录制) | `livekit/pkg/rtc/egress.go` | Go |
| Telemetry (录制事件) | `livekit/pkg/telemetry/telemetryservice.go` | Go |
| Egress ID 生成 | `livekit/pkg/service/egressid.go` | Go |
| IO Service (Egress 状态查询) | `livekit/pkg/service/ioservice.go` | Go |
| Egress 独立服务 | `github.com/livekit/egress`（独立仓库） | Go |
| Agent RecorderIO | `livekit-agents/livekit/agents/voice/recorder_io/recorder_io.py` | Python |

---

## 3. Egress 类型

LiveKit 支持 5 种 Egress 类型，覆盖不同的录制场景。

### 3.1 类型总览

| Egress 类型 | 录制范围 | 适用场景 | 底层实现 |
|------------|---------|---------|---------|
| **RoomCompositeEgress** | 整个房间的混合音视频 | 全会议录制、RTMP 推流 | Egress 服务加入房间，混合所有轨道 |
| **TrackCompositeEgress** | 指定音视频轨道的组合 | 选择性录制特定参与者 | 订阅指定 Track ID |
| **ParticipantEgress** | 单个参与者的音视频 | 单独录制某人 | 订阅参与者的所有轨道 |
| **TrackEgress** | 单条音视频轨道 | 轨道级录制 | 直接写入文件/推流 |
| **WebEgress** | 录制 Web 页面 | 网页录制 | 使用浏览器引擎渲染 |

### 3.2 RoomCompositeEgress

录制整个房间的音视频，Egress 服务作为参与者加入房间，订阅所有音视频轨道，进行混音后输出。

```go
// pkg/service/egress.go
func (s *EgressService) StartRoomCompositeEgress(ctx context.Context, req *livekit.RoomCompositeEgressRequest) {
    // 参数验证
    // 生成 Egress ID
    // 通过 psrpc 发送给 Egress 服务
    // 返回 EgressInfo
}

// RoomCompositeEgressRequest 关键字段:
type RoomCompositeEgressRequest struct {
    RoomName       string           // 房间名
    CustomBaseUrl  string           // 自定义模板 URL（用于布局）
    FileOutputs    []EncodedFileOutput    // 文件输出配置
    SegmentOutputs []SegmentOutput        // 分段输出
    StreamOutputs  []StreamOutput         // RTMP 推流
    AudioOnly     bool             // 仅音频
    VideoOnly     bool             // 仅视频
    Layout        string           // 布局模板
    AudioMixing   AudioMixing      // 音频混合模式
    Advanced      EncodingOptions  // 高级编码参数
    Preset        EgressPreset     // 预设配置
}
```

### 3.3 TrackCompositeEgress

录制指定音频轨道和视频轨道的组合。

```go
func (s *EgressService) StartTrackCompositeEgress(ctx, req *livekit.TrackCompositeEgressRequest) {
    // 类似 RoomComposite，但指定具体的 TrackID
}

type TrackCompositeEgressRequest struct {
    RoomName       string           // 房间名
    AudioTrackId   string           // 音频轨道 ID
    VideoTrackId   string           // 视频轨道 ID
    // ... 输出、编码配置同上
}
```

### 3.4 ParticipantEgress

录制单个参与者的音视频。

```go
func (s *EgressService) StartParticipantEgress(ctx, req *livekit.ParticipantEgressRequest) {
}

type ParticipantEgressRequest struct {
    RoomName       string           // 房间名
    Identity       string           // 参与者身份
    // ... 输出配置
}
```

### 3.5 TrackEgress

录制单条轨道，支持直接文件输出（无需编码封装）。

```go
func (s *EgressService) StartTrackEgress(ctx, req *livekit.TrackEgressRequest) {
}

type TrackEgressRequest struct {
    RoomName       string           // 房间名
    TrackId        string           // 轨道 ID
    Output         *TrackEgressRequest_File  // 或 WebSocket/RTMP
}
```

### 3.6 AudioMixing 模式

RoomCompositeEgress 中的音频混合有三种模式：

```go
AudioMixing:
  ┌─────────────────────────────────────────────────────┐
  │                                                      │
  │  AUDIO_MIXING_UNSPECIFIED  → 默认单声道混音           │
  │                                                      │
  │  DUAL_CHANNEL_AGENT       → 双通道                    │
  │     Channel 1: Agent 输出的音频（TTS）                  │
  │     Channel 2: 所有其他参与者 + 房间内其他音频          │
  │     ★ 适用于 AI 客服录音：区分 Agent 和用户             │
  │                                                      │
  │  DUAL_CHANNEL_MIXER      → 双通道                    │
  │     Channel 1: 发布者音频                              │
  │     Channel 2: 所有其他参与者                          │
  └─────────────────────────────────────────────────────┘
```

---

## 4. LiveKit Server 侧 — EgressService

### 4.1 结构定义

位置：`livekit/pkg/service/egress.go`

```go
type EgressService struct {
    launcher    rtc.EgressLauncher    // 实际启动 Egress 的接口（通过 psrpc 调 Egress 服务）
    client      rpc.EgressClient      // 与 Egress 服务的 RPC 客户端
    io          IOClient              // 持久化接口（存储 EgressInfo）
    roomService livekit.RoomService   // 房间服务（用于更新布局）
}

// EgressLauncher 接口
type EgressLauncher interface {
    StartEgress(context.Context, *rpc.StartEgressRequest) (*livekit.EgressInfo, error)
    StopEgress(context.Context, *livekit.StopEgressRequest) (*livekit.EgressInfo, error)
}
```

### 4.2 API 方法一览

| API | 路由 | 说明 | 权限 |
|-----|------|------|------|
| `StartRoomCompositeEgress` | POST | 录制整个房间 | RECORD |
| `StartTrackCompositeEgress` | POST | 录制指定轨道组合 | RECORD |
| `StartParticipantEgress` | POST | 录制单个参与者 | RECORD |
| `StartTrackEgress` | POST | 录制单条轨道 | RECORD |
| `StartWebEgress` | POST | 录制网页 | RECORD |
| `UpdateLayout` | POST | 更新房间录制布局 | RECORD |
| `UpdateStream` | POST | 更新 RTMP 推流地址 | RECORD |
| `ListEgress` | GET | 列出录制任务 | RECORD |
| `StopEgress` | POST | 停止录制 | RECORD |

### 4.3 启动流程

```go
func (s *EgressService) startEgress(ctx, req) (*livekit.EgressInfo, error) {
    // Step 1: 权限检查
    if err := EnsureRecordPermission(ctx); err != nil {
        return nil, twirpAuthError(err)
    }
    
    // Step 2: 检查 Egress 服务是否连接
    if s.launcher == nil {
        return nil, ErrEgressNotConnected
    }
    
    // Step 3: 通过 psrpc 调用 Egress 服务
    return s.launcher.StartEgress(ctx, req)
}
```

### 4.4 egressLauncher 内部逻辑

```go
func (s *egressLauncher) StartEgress(ctx, req) (*livekit.EgressInfo, error) {
    // 1. 生成 Egress ID（如果未提供）
    if req.EgressId == "" {
        req.EgressId = guid.New(utils.EgressPrefix) // "EG_..."
    }
    
    // 2. 解析房间名，获取 Room ID
    if req.RoomId == "" {
        roomName = extractRoomName(req)
        room, _ = s.store.LoadRoom(ctx, roomName, false)
        req.RoomId = room.Sid
    }
    
    // 3. 通过 psrpc 发送到 Egress 服务
    info, err = s.client.StartEgress(ctx, "", req)
    
    // 4. 持久化 EgressInfo
    s.io.CreateEgress(ctx, info)
    
    return info, nil
}
```

### 4.5 Egress ID 预生成

位置：`livekit/pkg/service/egressid.go`

```go
// Twirp 中间件：在请求到达 handler 之前预生成 Egress ID
// 这对于追踪 Egress 服务不可用时的错误非常关键
func TwirpEgressID() *twirp.ServerHooks {
    return &twirp.ServerHooks{
        RequestRouted: func(ctx context.Context) (context.Context, error) {
            if isStartEgressMethod(ctx) {
                egressID := guid.New(guid.EgressPrefix)
                ctx = WithEgressID(ctx, egressID)
                AppendLogFields(ctx, "egressID", egressID)
            }
            return ctx, nil
        },
    }
}
```

### 4.6 停止 Egress

```go
func (s *EgressService) StopEgress(ctx, req) (*livekit.EgressInfo, error) {
    // 1. 权限检查
    EnsureRecordPermission(ctx)
    
    // 2. 调用 Egress 服务停止
    info, err = s.launcher.StopEgress(ctx, req)
    
    // 3. 如果 psrpc 调用失败（如 Egress 服务已宕机），尝试从存储加载状态
    if err != nil {
        info, loadErr = s.io.GetEgress(ctx, &rpc.GetEgressRequest{EgressId: req.EgressId})
        // 根据状态判断是否可以重试
        switch info.Status {
        case EGRESS_STARTING, EGRESS_ACTIVE:
            return nil, err  // 真的错误
        default:
            return nil, FailedPrecondition  // 任务已结束
        }
    }
    return info, nil
}
```

### 4.7 布局更新

```go
func (s *EgressService) UpdateLayout(ctx, req *livekit.UpdateLayoutRequest) {
    // 1. 权限检查
    
    // 2. 从存储加载 EgressInfo
    info = s.io.GetEgress(ctx, req.EgressId)
    
    // 3. 将布局信息编码为 JSON metadata
    metadata = json.Marshal(&LayoutMetadata{Layout: req.Layout})
    
    // 4. 通过 RoomService 更新 Egress 参与者的 metadata
    s.roomService.UpdateParticipant(ctx, &livekit.UpdateParticipantRequest{
        Room:     info.RoomName,
        Identity: info.EgressId,  // Egress 参与者的 identity 就是 EgressID
        Metadata: string(metadata),
    })
}
```

---

## 5. LiveKit Egress 独立服务

### 5.1 服务定位

Egress 是一个**独立的 Go 服务**，仓库在 `github.com/livekit/egress`。它：

1. 通过 psrpc 接收 LiveKit Server 的录制请求
2. 作为 `PARTICIPANT_KIND_EGRESS` 类型的参与者加入房间
3. 订阅房间内的音视频轨道
4. 使用 FFmpeg 或 GStreamer 进行编码和封装
5. 输出到指定目标

### 5.2 参与者类型

```go
// Egress 参与者在 LiveKit Room 中的特殊处理
// pkg/rtc/room.go
func IsParticipantExemptFromTrackPermissionsRestrictions(p types.LocalParticipant) bool {
    return p.IsRecorder()  // Egress 参与者不受轨道权限限制
}
```

### 5.3 Egress 状态

```go
type EgressStatus int
const (
    EGRESS_STARTING     EgressStatus = 0  // 启动中
    EGRESS_ACTIVE       EgressStatus = 1  // 录制中
    EGRESS_PAUSED       EgressStatus = 2  // 已暂停
    EGRESS_COMPLETE     EgressStatus = 3  // 已完成
    EGRESS_FAILED       EgressStatus = 4  // 失败
    EGRESS_ABORTED      EgressStatus = 5  // 中止
    EGRESS_LIMIT_REACHED EgressStatus = 6 // 达到限制
)
```

### 5.4 输出类型

| 输出类型 | 说明 | 适用格式 |
|---------|------|---------|
| `EncodedFileOutput` | 编码后的文件输出 | MP4, OGG, WebM |
| `DirectFileOutput` | 直接文件输出（纯 WebRTC 轨道） | .ivf, .ogg |
| `SegmentOutput` | HLS 分段输出 | M3U8 + TS |
| `StreamOutput` | RTMP 推流 | RTMP(S) |

### 5.5 编码预设

```go
type EgressPreset int
const (
    PRESET_UNSPECIFIED  EgressPreset = 0  // 自定义编码参数
    PRESET_H264_720P_30 EgressPreset = 1  // 720p 30fps H.264
    PRESET_H264_1080P_30 EgressPreset = 2 // 1080p 30fps H.264
    PRESET_H264_1080P_60 EgressPreset = 3 // 1080p 60fps H.264
    PRESET_PORTRAIT_H264_720P_30 EgressPreset = 4  // 竖屏 720p
    PRESET_PORTRAIT_H264_1080P_30 EgressPreset = 5  // 竖屏 1080p
    PRESET_PORTRAIT_H264_1080P_60 EgressPreset = 6  // 竖屏 1080p 60fps
)
```

---

## 6. 自动录制 — AutoEgress

位置：`livekit/pkg/rtc/egress.go`

LiveKit 支持**自动录制**功能，在房间创建或参与者发布轨道时自动触发录制，无需手动调用 API。

### 6.1 自动参与者录制

```go
func StartParticipantEgress(ctx, launcher, ts, opts, identity, roomName, roomID) error {
    // 当参与者加入房间时自动触发
    req := &livekit.ParticipantEgressRequest{
        RoomName: roomName,
        Identity: identity,
        FileOutputs:    opts.FileOutputs,
        SegmentOutputs: opts.SegmentOutputs,
    }
    
    info, err := launcher.StartEgress(ctx, &rpc.StartEgressRequest{
        Request: &rpc.StartEgressRequest_Participant{
            Participant: req,
        },
        RoomId: roomID,
    })
}
```

### 6.2 自动轨道录制

```go
func StartTrackEgress(ctx, launcher, ts, opts, track, roomName, roomID) error {
    // 当参与者发布轨道时自动触发
    output := &livekit.DirectFileOutput{
        Filepath: getFilePath(opts.Filepath),  // 支持 {track_id} 占位符
    }
    
    // 支持 S3 / GCP / Azure 自动配置
    switch opts.Output.(type) {
    case *livekit.AutoTrackEgress_Azure:
        output.Output = &DirectFileOutput_Azure{...}
    case *livekit.AutoTrackEgress_Gcp:
        output.Output = &DirectFileOutput_Gcp{...}
    case *livekit.AutoTrackEgress_S3:
        output.Output = &DirectFileOutput_S3{...}
    }
    
    req := &livekit.TrackEgressRequest{
        RoomName: roomName,
        TrackId:  string(track.ID()),
        Output:   &TrackEgressRequest_File{File: output},
    }
    launcher.StartEgress(ctx, req)
}
```

### 6.3 文件路径模板

```go
func getFilePath(filepath string) string {
    // 如果路径不含扩展名，自动添加 {track_id}
    // "recordings/" → "recordings/{track_id}"
    // "output.mp4"  → "output-{track_id}.mp4"
    if filepath == "" || strings.HasSuffix(filepath, "/") || strings.Contains(filepath, "{track_id}") {
        return filepath
    }
    idx := strings.Index(filepath, ".")
    if idx == -1 {
        return fmt.Sprintf("%s-{track_id}", filepath)
    } else {
        return fmt.Sprintf("%s-%s%s", filepath[:idx], "{track_id}", filepath[idx:])
    }
}
```

---

## 7. Agent 侧录音 — RecorderIO

位置：`livekit-agents/livekit/agents/voice/recorder_io/recorder_io.py`

Agent 端有一套轻量级的录音机制，用于录制 Agent 自身的音频活动（如通话报告和调试）。

### 7.1 架构

```
Agent 音频流水线:
                    ┌──────────────────────┐
  用户音频 ─────────→│ RecorderAudioInput   │──→ STT → LLM
                    │   (录制输入)          │
                    └──────────────────────┘

  TTS 音频 ─────────→┌──────────────────────┐
                    │ RecorderAudioOutput  │──→ 播放给用户
                    │   (录制输出)          │
                    └──────────────────────┘
                              │
                              ▼
                    ┌──────────────────────┐
                    │    RecorderIO        │
                    │  ┌────────────────┐  │
                    │  │  输入缓冲区     │  │
                    │  │  输出缓冲区     │  │
                    │  └───────┬────────┘  │
                    │          │            │
                    │  ┌───────▼────────┐  │
                    │  │  编码线程       │  │  ← 独立线程
                    │  │  (PyAV/FFmpeg) │  │
                    │  └───────┬────────┘  │
                    └──────────┼──────────┘
                               │
                        ┌──────▼──────┐
                        │  输出文件    │
                        │  .ogg/.mp4  │
                        └─────────────┘
```

### 7.2 RecordingOptions

Agent 的录音选项在 `AgentSession.start()` 中配置：

```python
class RecordingOptions(TypedDict, total=False):
    audio: bool        # 录制会话音频（默认 True）
    traces: bool       # 导出 OpenTelemetry trace span（默认 True）
    logs: bool         # 导出 OpenTelemetry 日志（默认 True）
    transcript: bool   # 上传对话转录文本（chat history）（默认 True）
```

### 7.3 RecorderIO 核心实现

```python
class RecorderIO:
    def __init__(self, agent_session, sample_rate=48000):
        self._in_q: queue.Queue     # 输入音频缓冲区
        self._out_q: queue.Queue    # 输出音频缓冲区
        self._sample_rate = 48000
        self._output_path: Path | None = None
    
    async def start(self, *, output_path: str | Path):
        # 启动
        self._output_path = Path(output_path)
        # 启动前向任务（每 2.5s 将缓存写入队列）
        self._forward_atask = asyncio.create_task(self._forward_task())
        # 启动编码线程（PyAV 编码）
        thread = threading.Thread(target=self._encode_thread, daemon=True)
        thread.start()
    
    def record_input(self, audio_input) -> RecorderAudioInput:
        # 在音频输入流水线中插入录制节点
        self._in_record = RecorderAudioInput(recording_io=self, source=audio_input)
        return self._in_record
    
    def record_output(self, audio_output) -> RecorderAudioOutput:
        # 在音频输出流水线中插入录制节点
        self._out_record = RecorderAudioOutput(
            recording_io=self, audio_output=audio_output, write_fnc=self._write_cb
        )
        return self._out_record
```

### 7.4 编码线程

```python
def _encode_thread(self):
    # 使用 PyAV (FFmpeg 绑定) 进行编码
    container = av.open(str(self._output_path), mode='w')
    stream = container.add_stream('libopus', rate=self._sample_rate, channels=1)
    
    while True:
        in_buf = self._in_q.get()     # 从队列获取输入音频
        out_buf = self._out_q.get()   # 从队列获取输出音频
        if in_buf is None or out_buf is None:
            break  # 结束信号
        
        # 将 PCM16 编码为 Opus 并写入文件
        for frame in in_buf:
            # 转换为 PyAV AudioFrame
            # 写入 container
```

### 7.5 RecorderIO 与 Egress 的关系

```
Agent RecorderIO 与 Egress 服务的差异:

┌────────────────────────────────────────────────────────┐
│  Agent RecorderIO              Egress 服务              │
│  ──────────────                ──────────               │
│  录制范围: Agent 本地音视频     录制范围: 整个房间       │
│  音频源: 录音笔流水线内         音频源: 作为参与者进入房间 │
│  输出: 本地文件 (.ogg)         输出: S3/本地/RTMP       │
│  用途: 调试/通话报告            用途: 正式录制/存档      │
│  实现: Python + PyAV          实现: Go + FFmpeg        │
│  触发: 代码内配置              触发: API 或自动配置      │
└────────────────────────────────────────────────────────┘
```

---

## 8. 第三方集成示例 — Hamming Plugin

Hamming 是一个第三方通话分析平台，它通过 LiveKit Egress API 完成录音。

### 8.1 启动 Egress

```python
async def start_plugin_managed_room_composite_egress(self, room_name, filepath):
    # 调用 LiveKit API 启动房间录制
    async with livekit_api.LiveKitAPI() as lkapi:
        output = EncodedFileOutput(
            filepath=filepath,
            s3=_build_s3_upload_config(),
            file_type=_resolve_encoded_file_type(filepath),
        )
        request = RoomCompositeEgressRequest(
            room_name=room_name,
            audio_only=True,                              # 仅音频
            audio_mixing=DUAL_CHANNEL_AGENT,              # 双通道: Agent/用户分开
            file_outputs=[output],
        )
        response = await lkapi.egress.start_room_composite_egress(request)
        return response.egress_id
```

### 8.2 轮询完成状态

```python
async def _poll_for_plugin_managed_egress_result(self, egress_id, filepath):
    async with livekit_api.LiveKitAPI() as lkapi:
        for attempt in range(max_attempts):
            response = await lkapi.egress.list_egress(
                ListEgressRequest(room_name=room_name)
            )
            # 检查 egress 状态
            status_name = _get_egress_status_name(response, egress_id)
            if status_name == "EGRESS_COMPLETE":
                return _build_public_s3_url(filepath)
            elif status_name in {"EGRESS_FAILED", "EGRESS_ABORTED"}:
                return None
            await asyncio.sleep(poll_interval_seconds)
```

---

## 9. Egress 状态机

```
                    ┌──────────┐
                    │ STARTING │ ← 请求已接收，Egress 服务正在加入房间
                    └────┬─────┘
                         │ 成功加入房间
                         ▼
                    ┌──────────┐
                    │  ACTIVE  │ ← 录制中
                    └────┬─────┘
                         │
              ┌──────────┼──────────┐
              │          │          │
              ▼          ▼          ▼
        ┌─────────┐ ┌─────────┐ ┌─────────────┐
        │ PAUSED  │ │COMPLETE │ │   FAILED    │
        └────┬────┘ └─────────┘ └─────────────┘
             │ 恢复                    │
             ▼                         │
        ┌──────────┐                   │
        │  ACTIVE  │                   │
        └──────────┘                   │
                                       ▼
                                ┌─────────────┐
                                │   ABORTED   │
                                └─────────────┘
                                ┌───────────────┐
                                │LIMIT_REACHED  │ ← 文件大小/时长限制
                                └───────────────┘
```

### 状态说明

| 状态 | 触发条件 | 表示 |
|------|---------|------|
| STARTING | API 调用后 | Egress 正在初始化，加入房间 |
| ACTIVE | 已加入房间，开始接收轨道数据 | 录制进行中 |
| PAUSED | 调用 UpdateLayout 暂停 | 录制暂停（仅房间录制） |
| COMPLETE | 房间关闭或手动停止 | 正常结束 |
| FAILED | 编码错误、网络问题等 | 异常结束 |
| ABORTED | 手动强制停止 | 用户中止 |
| LIMIT_REACHED | 达到预设的文件大小或时长上限 | 限流结束 |

---

## 10. 权限控制

### 10.1 JWT Claim

```go
// 所有 Egress API 需要 RECORD 权限
func EnsureRecordPermission(ctx context.Context) error {
    // 检查 JWT Claim 中的 record 权限
    grants := GetGrants(ctx)
    if grants == nil || !grants.Record {
        return ErrPermissionDenied
    }
    return nil
}
```

### 10.2 权限级别

| 权限 | 可执行的操作 | 典型用户 |
|------|------------|---------|
| `record: true` | 所有 Egress API | 录制管理员、AI Agent |
| `room_admin: true` | 管理房间内的 Egress | 房间创建者 |
| 无权限 | 无法启动/停止 Egress | 普通参与者 |

### 10.3 Egress 参与者的特殊权限

```go
func IsParticipantExemptFromTrackPermissionsRestrictions(p types.LocalParticipant) bool {
    return p.IsRecorder()
    // Egress 参与者自动不受轨道权限限制
    // 即使房间配置了轨道权限，Egress 仍然可以录制所有轨道
}
```

---

## 11. 输出格式与存储

### 11.1 文件格式

| 格式 | 容器 | 视频编码 | 音频编码 | 适用场景 |
|------|------|---------|---------|---------|
| MP4 | .mp4 | H.264 | AAC/OPUS | 通用录制 |
| OGG | .ogg | — | OPUS | 纯音频录制 |
| WebM | .webm | VP8/VP9 | OPUS | 网页播放 |
| IVF | .ivf | VP8/VP9 | — | 纯视频 |

### 11.2 存储后端

```go
// EncodedFileOutput 支持的存储后端
type EncodedFileOutput struct {
    Filepath   string          // 文件路径
    FileType   FileType        // 文件类型（MP4/OGG/WEBM）
    S3         *S3Upload       // S3 配置
    GCP        *GCPUpload      // Google Cloud Storage
    Azure      *AzureBlobUpload // Azure Blob
    AliyunOSS  *AliyunOSSUpload // 阿里云 OSS
}

// DirectFileOutput 支持的存储后端（TrackEgress 专用）
type DirectFileOutput struct {
    Filepath   string
    S3         *S3Upload
    GCP        *GCPUpload
    Azure      *AzureBlobUpload
}
```

### 11.3 推流输出

```go
type StreamOutput struct {
    Protocol StreamProtocol  // RTMP / SRTP
    Urls     []string         // 推流 URL 列表
}
```

---

## 12. 与 SIP Bridge 的协作

### 12.1 录音链路

```
SIP 通话的完整录音链路:

PSTN 用户 ──SIP──→ SIP Bridge ──WebRTC──→ LiveKit Room
                                              │
                                         Egress 服务作为
                                         Participant 加入房间
                                              │
                                         录制混合后的音频流
                                              │
                                        ┌─────┴─────┐
                                        │  S3/本地   │
                                        │  文件系统  │
                                        └───────────┘
```

### 12.2 协作要点

1. **SIP Bridge 本身不处理录音** — 它只负责 SIP ↔ WebRTC 的桥接
2. **Egress 服务独立录制** — 作为专门参与者加入 Room，与 SIP 参与者平行
3. **DUAL_CHANNEL_AGENT 模式** — 对于 AI 客服场景，可以将 Agent TTS 和 PSTN 用户分别录在两个声道
4. **触发方式** — 通过 API 调用启动，或在 Room 配置中设置 AutoEgress

### 12.3 典型 SIP 录音请求示例

```python
# 使用 Python SDK 录制 SIP 通话
async with LiveKitAPI() as lkapi:
    request = RoomCompositeEgressRequest(
        room_name=room_name,
        audio_only=True,                           # 电话只需音频
        audio_mixing=DUAL_CHANNEL_AGENT,           # 区分 Agent 和电话用户
        file_outputs=[
            EncodedFileOutput(
                filepath=f"recordings/{call_id}.ogg",
                file_type=OGG,
                s3=S3Upload(
                    bucket="my-call-recordings",
                    region="us-east-1",
                ),
            )
        ],
    )
    info = await lkapi.egress.start_room_composite_egress(request)
```

---

## 13. 历史演进

### 13.1 版本时间线

| LiveKit Server 版本 | 变化 |
|--------------------|------|
| < 1.1.0 | 使用 `RecordingService` |
| 1.1.0 | 引入 `EgressService`，标记 `RecordingService` 为弃用 |
| 1.2.0 | 添加 AutoEgress 支持 |
| 1.3.0 | 移除弃用的 `RecordingService`，Egress 成为唯一方案 |
| 当前 | 独立 Egress 服务 + Cloud 集成 |

### 13.2 EgressService 替代 RecordingService 的原因

1. **更清晰的职责分离** — RecordingService 耦合到 LiveKit Server 内部，Egress 作为独立服务运行
2. **更好的扩展性** — 可以独立扩缩容 Egress 服务
3. **更强的编码能力** — 使用 FFmpeg/GStreamer 进行编码，支持更多格式
4. **支持推流** — RecordingService 不支持 RTMP 推流

---

## 14. 关键设计决策

1. **独立 Egress 服务** — 与 LiveKit Server 分离部署，通过 psrpc 通信，可独立扩缩容

2. **Egress 作为 Room Participant** — Egress 服务以 `PARTICIPANT_KIND_EGRESS` 身份加入 Room，复用 LiveKit 的 WebRTC 基础设施，无需额外协议转换

3. **Psrpc 通信** — 基于 Redis Pub/Sub 的异步 RPC，LiveKit Server 和 Egress 服务之间不直接耦合

4. **FFmpeg/GStreamer 编码** — 使用业界标准的编码工具，避免自研编码器的复杂性和质量风险

5. **Egress ID 预生成** — 在 Twirp 中间件中预生成 Egress ID，即使 Egress 服务不可用也能追踪到失败的请求

6. **AutoEgress 配置** — 通过 Room 级别的配置自动触发录制，无需客户端手动调用 API

7. **双通道音频混合** — `DUAL_CHANNEL_AGENT` 模式专为 AI 客服场景设计，将 Agent 和用户录在不同声道

8. **无权限限制** — Egress 参与者自动豁免于轨道权限限制，确保录制不会被房间配置阻塞

9. **布局动态更新** — 通过修改 Egress 参与者的 metadata 实现布局更新，无需重启录制

10. **Agent RecorderIO 作为补充** — Agent 侧的 RecorderIO 用于轻量级本地录制（调试/报告），与生产级的 Egress 服务形成互补