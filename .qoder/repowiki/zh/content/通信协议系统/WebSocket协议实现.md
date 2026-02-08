# WebSocket协议实现

<cite>
**本文引用的文件**
- [websocket_protocol.h](file://main/protocols/websocket_protocol.h)
- [websocket_protocol.cc](file://main/protocols/websocket_protocol.cc)
- [protocol.h](file://main/protocols/protocol.h)
- [websocket.md](file://docs/websocket.md)
- [settings.h](file://main/settings.h)
- [system_info.h](file://main/system_info.h)
- [application.h](file://main/application.h)
- [audio_service.h](file://main/audio/audio_service.h)
- [audio_service.cc](file://main/audio/audio_service.cc)
</cite>

## 目录
1. [引言](#引言)
2. [项目结构](#项目结构)
3. [核心组件](#核心组件)
4. [架构总览](#架构总览)
5. [详细组件分析](#详细组件分析)
6. [依赖关系分析](#依赖关系分析)
7. [性能考虑](#性能考虑)
8. [故障排除指南](#故障排除指南)
9. [结论](#结论)
10. [附录](#附录)

## 引言
本文件面向实时语音交互场景，系统化梳理该项目中基于ESP-IDF的WebSocket协议实现，覆盖连接建立、握手流程、双向通信机制、帧格式与数据传输、与HTTP的差异与优势、连接生命周期管理、消息格式设计（含音频与控制命令）、性能优化策略以及调试与排障方法。文档严格依据仓库源码与配套文档进行归纳总结，帮助开发者快速理解与扩展WebSocket通信子系统。

## 项目结构
WebSocket协议实现位于主程序模块的协议层，围绕统一的Protocol抽象接口展开，WebSocket实现类负责与服务器建立持久连接、发送/接收JSON控制消息与二进制音频帧，并维护会话状态与错误处理。

```mermaid
graph TB
subgraph "协议层"
P["Protocol 抽象接口<br/>protocol.h"]
WS["WebsocketProtocol 实现<br/>websocket_protocol.cc/.h"]
end
subgraph "应用层"
APP["Application 应用入口<br/>application.h"]
SET["Settings 配置存储<br/>settings.h"]
SYS["SystemInfo 系统信息<br/>system_info.h"]
end
subgraph "音频层"
ASH["音频服务常量与宏<br/>audio_service.h"]
ASC["音频服务实现<br/>audio_service.cc"]
end
DOC["WebSocket协议文档<br/>docs/websocket.md"]
APP --> P
P --> WS
WS --> SET
WS --> SYS
WS --> ASH
WS --> ASC
DOC -. 对照说明 .-> WS
```

**图表来源**
- [protocol.h](file://main/protocols/protocol.h#L44-L95)
- [websocket_protocol.h](file://main/protocols/websocket_protocol.h#L13-L32)
- [websocket_protocol.cc](file://main/protocols/websocket_protocol.cc#L23-L200)
- [application.h](file://main/application.h#L32-L88)
- [settings.h](file://main/settings.h#L7-L26)
- [system_info.h](file://main/system_info.h#L9-L19)
- [audio_service.h](file://main/audio/audio_service.h#L36-L40)
- [audio_service.cc](file://main/audio/audio_service.cc#L37-L38)

**章节来源**
- [websocket_protocol.h](file://main/protocols/websocket_protocol.h#L1-L35)
- [websocket_protocol.cc](file://main/protocols/websocket_protocol.cc#L1-L254)
- [protocol.h](file://main/protocols/protocol.h#L1-L99)
- [websocket.md](file://docs/websocket.md#L1-L496)
- [application.h](file://main/application.h#L1-L91)
- [settings.h](file://main/settings.h#L1-L29)
- [system_info.h](file://main/system_info.h#L1-L22)
- [audio_service.h](file://main/audio/audio_service.h#L36-L40)
- [audio_service.cc](file://main/audio/audio_service.cc#L37-L38)

## 核心组件
- Protocol 抽象接口：定义统一的协议行为契约，包括音频通道打开/关闭、文本与音频消息发送、回调注册、会话参数与错误处理等。
- WebsocketProtocol：具体实现，负责WebSocket连接、握手、二进制与文本消息收发、版本协商、会话参数对齐、错误与超时处理。
- Application：应用主控，调度事件、状态机转换、音频服务集成、协议实例管理。
- Settings/SystemInfo：配置读取与系统信息（MAC、UUID等）提供，用于握手头部与鉴权。
- 音频服务：提供OPUS编解码、帧时长宏、队列长度等参数，支撑WebSocket音频帧的编码/解码与传输。

**章节来源**
- [protocol.h](file://main/protocols/protocol.h#L44-L95)
- [websocket_protocol.cc](file://main/protocols/websocket_protocol.cc#L23-L200)
- [application.h](file://main/application.h#L32-L88)
- [settings.h](file://main/settings.h#L7-L26)
- [system_info.h](file://main/system_info.h#L9-L19)
- [audio_service.h](file://main/audio/audio_service.h#L36-L40)

## 架构总览
WebSocket协议在整体系统中的位置如下：

```mermaid
graph TB
Client["设备端 Application<br/>application.h"]
Proto["Protocol 抽象<br/>protocol.h"]
WS["WebsocketProtocol<br/>websocket_protocol.cc/.h"]
Net["网络栈/WebSocket客户端<br/>esp-idf"]
Server["服务器端"]
Audio["音频服务与编解码<br/>audio_service.*"]
Client --> Proto
Proto --> WS
WS --> Net
WS <- --> Server
WS --> Audio
```

**图表来源**
- [application.h](file://main/application.h#L63-L88)
- [protocol.h](file://main/protocols/protocol.h#L44-L95)
- [websocket_protocol.cc](file://main/protocols/websocket_protocol.cc#L82-L200)
- [audio_service.h](file://main/audio/audio_service.h#L36-L40)

## 详细组件分析

### WebsocketProtocol 类与继承关系
WebsocketProtocol继承自Protocol，实现音频通道的打开/关闭、音频与文本消息发送、回调注册、握手与版本协商等。

```mermaid
classDiagram
class Protocol {
+Start() bool
+OpenAudioChannel() bool
+CloseAudioChannel() void
+IsAudioChannelOpened() bool
+SendAudio(packet) bool
+SendText(text) bool
+OnIncomingAudio(cb)
+OnIncomingJson(cb)
+OnAudioChannelOpened(cb)
+OnAudioChannelClosed(cb)
+OnNetworkError(cb)
+OnConnected(cb)
+OnDisconnected(cb)
-server_sample_rate_ int
-server_frame_duration_ int
-session_id_ string
-last_incoming_time_ time_point
}
class WebsocketProtocol {
+WebsocketProtocol()
+~WebsocketProtocol()
+Start() bool
+OpenAudioChannel() bool
+CloseAudioChannel() void
+IsAudioChannelOpened() bool
+SendAudio(packet) bool
+SendText(text) bool
-ParseServerHello(root) void
-GetHelloMessage() string
-event_group_handle_ EventGroupHandle_t
-websocket_ unique_ptr<WebSocket>
-version_ int
}
Protocol <|-- WebsocketProtocol
```

**图表来源**
- [protocol.h](file://main/protocols/protocol.h#L44-L95)
- [websocket_protocol.h](file://main/protocols/websocket_protocol.h#L13-L32)

**章节来源**
- [websocket_protocol.h](file://main/protocols/websocket_protocol.h#L13-L32)
- [websocket_protocol.cc](file://main/protocols/websocket_protocol.cc#L15-L21)
- [protocol.h](file://main/protocols/protocol.h#L44-L95)

### 连接建立与握手流程
- 条件连接：音频通道按需建立，避免不必要的网络占用。
- 头部设置：Authorization（支持Bearer前缀）、Protocol-Version、Device-Id（MAC）、Client-Id（UUID）。
- 连接与握手：Connect成功后发送“hello”消息，等待服务器返回相同类型的“hello”，并校验transport字段；收到后设置事件标志，表示通道就绪。
- 超时与错误：握手超时（默认10秒）或连接失败触发网络错误回调。

```mermaid
sequenceDiagram
participant App as "Application"
participant WS as "WebsocketProtocol"
participant Net as "WebSocket客户端"
participant Srv as "服务器"
App->>WS : 调用 OpenAudioChannel()
WS->>Net : 创建WebSocket并设置Headers
WS->>Net : Connect(url)
Net-->>WS : 连接成功/失败
alt 连接成功
WS->>Net : 发送 "hello"(JSON)
Net->>Srv : 握手请求
Srv-->>Net : "hello"(JSON, transport=websocket)
Net-->>WS : OnData(JSON)
WS->>WS : 解析并校验transport
WS->>WS : 设置事件标志(握手完成)
WS-->>App : 触发通道打开回调
else 连接失败/超时
WS-->>App : 触发网络错误回调
end
```

**图表来源**
- [websocket_protocol.cc](file://main/protocols/websocket_protocol.cc#L82-L200)
- [websocket.md](file://docs/websocket.md#L16-L80)

**章节来源**
- [websocket_protocol.cc](file://main/protocols/websocket_protocol.cc#L82-L200)
- [websocket.md](file://docs/websocket.md#L16-L80)

### 双向通信机制与消息格式
- 文本帧（JSON）：用于控制消息，如“hello”、“listen”、“abort”、“mcp”、“stt”、“tts”、“system”等。
- 二进制帧（Opus音频）：设备侧发送录音编码帧；服务器侧下发TTS合成音频帧。
- 版本协商：通过配置项选择二进制协议版本（1/2/3），影响二进制帧的头部与字段布局。
- 会话参数对齐：服务器返回的audio_params用于对齐采样率与帧时长，WebSocket实现会记录并用于后续音频处理。

```mermaid
flowchart TD
Start(["接收回调 OnData"]) --> CheckBin{"是否二进制帧?"}
CheckBin --> |是| VerSel{"协议版本?"}
VerSel --> |版本2| Parse2["解析 BinaryProtocol2<br/>提取timestamp/payload"]
VerSel --> |版本3| Parse3["解析 BinaryProtocol3<br/>提取payload"]
VerSel --> |默认/版本1| Raw["原始Opus payload"]
Parse2 --> EmitA["派发 AudioStreamPacket"]
Parse3 --> EmitA
Raw --> EmitA
CheckBin --> |否| ParseTxt["cJSON解析JSON"]
ParseTxt --> Type{"type 字段"}
Type --> |"hello"| ParseHello["解析会话参数/transport"]
Type --> |"stt/tts/mcp/system"| Dispatch["分发到业务回调"]
ParseHello --> EmitReady["设置握手完成事件"]
EmitA --> End(["结束"])
Dispatch --> End
EmitReady --> End
```

**图表来源**
- [websocket_protocol.cc](file://main/protocols/websocket_protocol.cc#L111-L165)
- [websocket_protocol.cc](file://main/protocols/websocket_protocol.cc#L227-L253)
- [protocol.h](file://main/protocols/protocol.h#L17-L31)

**章节来源**
- [websocket_protocol.cc](file://main/protocols/websocket_protocol.cc#L111-L165)
- [websocket_protocol.cc](file://main/protocols/websocket_protocol.cc#L227-L253)
- [protocol.h](file://main/protocols/protocol.h#L17-L31)
- [websocket.md](file://docs/websocket.md#L128-L293)

### 帧格式与数据传输
- 二进制协议版本1：直接发送Opus音频数据。
- 二进制协议版本2：带版本、类型、保留、时间戳、payload_size与payload的紧凑结构。
- 二进制协议版本3：简化版头部，包含类型、保留、payload_size与payload。
- 文本协议：以JSON为主，通过“type”字段区分业务类型，支持features、transport、audio_params等字段。

```mermaid
erDiagram
BINARY_V2 {
uint16 version
uint16 type
uint32 reserved
uint32 timestamp
uint32 payload_size
}
BINARY_V3 {
uint8 type
uint8 reserved
uint16 payload_size
}
AUDIO_PACKET {
int sample_rate
int frame_duration
uint32 timestamp
bytes payload
}
BINARY_V2 ||--o{ AUDIO_PACKET : "版本2封装"
BINARY_V3 ||--o{ AUDIO_PACKET : "版本3封装"
```

**图表来源**
- [protocol.h](file://main/protocols/protocol.h#L17-L31)
- [websocket_protocol.cc](file://main/protocols/websocket_protocol.cc#L33-L57)

**章节来源**
- [protocol.h](file://main/protocols/protocol.h#L17-L31)
- [websocket_protocol.cc](file://main/protocols/websocket_protocol.cc#L33-L57)
- [websocket.md](file://docs/websocket.md#L95-L126)

### 连接生命周期管理
- 建立：按需打开音频通道，设置头部，发起连接。
- 就绪：收到服务器“hello”且transport匹配，通道打开。
- 传输：音频二进制帧与JSON控制帧双向流动。
- 断开：服务器断开或主动关闭，触发通道关闭回调，回到空闲态。
- 超时：握手超时触发网络错误。

```mermaid
stateDiagram-v2
[*] --> 未连接
未连接 --> 连接中 : 打开音频通道
连接中 --> 已就绪 : 收到服务器hello
已就绪 --> 传输中 : 开始发送/接收
传输中 --> 已就绪 : 服务器断开(重连逻辑)
传输中 --> 未连接 : 主动关闭
已就绪 --> 未连接 : 主动关闭
```

**图表来源**
- [websocket_protocol.cc](file://main/protocols/websocket_protocol.cc#L167-L172)
- [websocket_protocol.cc](file://main/protocols/websocket_protocol.cc#L187-L199)
- [websocket.md](file://docs/websocket.md#L308-L366)

**章节来源**
- [websocket_protocol.cc](file://main/protocols/websocket_protocol.cc#L167-L172)
- [websocket_protocol.cc](file://main/protocols/websocket_protocol.cc#L187-L199)
- [websocket.md](file://docs/websocket.md#L308-L366)

### WebSocket与HTTP的区别与优势
- HTTP为请求-响应模型，适合一次性数据交换；WebSocket为全双工长连接，适合低延迟实时交互（如语音通话、TTS播放、MCP控制）。
- WebSocket在本项目中用于：
  - 实时音频上/下行（Opus编码）
  - 控制消息（STT/TTS/MCP/System）
  - 会话状态与参数对齐
- 优势体现在：减少连接建立开销、降低时延、支持双向推送。

**章节来源**
- [websocket.md](file://docs/websocket.md#L1-L80)

### 消息格式设计（音频与控制命令）
- 设备→服务器：hello（携带features、transport、audio_params）、listen（start/stop/detect）、abort、wake word detected、mcp等。
- 服务器→设备：hello（transport、audio_params、session_id）、stt、llm、tts（start/stop/sentence_start）、mcp、system、custom等。
- 二进制音频帧：Opus编码，按协议版本封装元数据。

**章节来源**
- [websocket.md](file://docs/websocket.md#L128-L293)
- [websocket_protocol.cc](file://main/protocols/websocket_protocol.cc#L202-L225)

## 依赖关系分析
- WebsocketProtocol依赖Protocol接口以统一行为；依赖Settings读取URL、Token、协议版本；依赖SystemInfo提供Device-Id与Client-Id；依赖音频服务常量与实现以对齐采样率与帧时长。
- 与Application耦合体现在事件调度与状态机驱动，WebSocket仅在需要音频通道时才建立连接，降低资源消耗。

```mermaid
graph LR
WS["WebsocketProtocol"] --> PH["protocol.h"]
WS --> SH["settings.h"]
WS --> SI["system_info.h"]
WS --> AH["audio_service.h"]
WS --> AC["audio_service.cc"]
APP["Application"] --> PH
APP --> WS
```

**图表来源**
- [websocket_protocol.cc](file://main/protocols/websocket_protocol.cc#L82-L110)
- [application.h](file://main/application.h#L63-L88)
- [settings.h](file://main/settings.h#L7-L26)
- [system_info.h](file://main/system_info.h#L9-L19)
- [audio_service.h](file://main/audio/audio_service.h#L36-L40)
- [audio_service.cc](file://main/audio/audio_service.cc#L37-L38)

**章节来源**
- [websocket_protocol.cc](file://main/protocols/websocket_protocol.cc#L82-L110)
- [application.h](file://main/application.h#L63-L88)

## 性能考虑
- 帧时长与队列：OPUS_FRAME_DURATION_MS为60ms，音频服务中定义了最大解码/发送队列长度，避免内存峰值与延迟累积。
- 二进制协议版本：版本2携带时间戳，有利于服务器端AEC；版本3更轻量；根据服务器能力选择合适版本。
- 采样率对齐：服务器下行音频采样率可能与设备不同，解码后重采样以适配设备播放链路。
- 按需连接：仅在需要音频通道时建立WebSocket，减少CPU与网络占用。
- 日志与错误：通过事件组与回调机制及时反馈错误，避免阻塞主循环。

**章节来源**
- [audio_service.h](file://main/audio/audio_service.h#L36-L40)
- [websocket_protocol.cc](file://main/protocols/websocket_protocol.cc#L33-L57)
- [websocket.md](file://docs/websocket.md#L390-L391)

## 故障排除指南
- 连接失败/超时：检查URL、Token、网络连通性；确认服务器返回的hello中transport字段与客户端一致；查看日志中的错误码。
- 服务器断开：检查鉴权（Authorization）、协议版本一致性、会话ID有效性；确认服务器端策略。
- 音频无声/失真：核对服务器audio_params与设备端采样率/帧时长是否一致；确认Opus编码参数；检查二进制协议版本。
- 超时与错误回调：WebSocket实现内置超时与错误设置，结合应用层回调定位问题。

**章节来源**
- [websocket_protocol.cc](file://main/protocols/websocket_protocol.cc#L167-L172)
- [websocket_protocol.cc](file://main/protocols/websocket_protocol.cc#L187-L193)
- [websocket.md](file://docs/websocket.md#L369-L379)

## 结论
该WebSocket协议实现以Protocol抽象为核心，围绕ESP-IDF的WebSocket客户端构建了低延迟、可扩展的实时通信通道。通过“hello”握手、二进制音频帧与JSON控制消息的组合，满足语音识别、TTS播放、MCP控制等多场景需求。配合按需连接、版本协商与参数对齐机制，在资源受限的嵌入式平台上实现了稳定高效的实时交互体验。

## 附录
- 配置键：websocket.url、websocket.token、websocket.version。
- 关键宏：OPUS_FRAME_DURATION_MS=60，用于统一音频帧时长。
- 文档对照：docs/websocket.md提供了消息示例、状态流转与注意事项，建议与源码配合阅读。

**章节来源**
- [websocket.md](file://docs/websocket.md#L82-L91)
- [audio_service.h](file://main/audio/audio_service.h#L36-L40)
- [websocket.md](file://docs/websocket.md#L409-L483)