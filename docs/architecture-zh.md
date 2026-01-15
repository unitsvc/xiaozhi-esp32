# 小智 ESP32 项目架构文档

中文

## 目录

- [项目概述](#项目概述)
- [整体架构](#整体架构)
- [核心技术栈](#核心技术栈)
- [系统架构层次](#系统架构层次)
- [核心模块详解](#核心模块详解)
  - [应用层 (Application Layer)](#应用层-application-layer)
  - [音频服务 (Audio Service)](#音频服务-audio-service)
  - [通信协议 (Protocol)](#通信协议-protocol)
  - [MCP 服务器 (MCP Server)](#mcp-服务器-mcp-server)
  - [显示系统 (Display System)](#显示系统-display-system)
  - [设备状态机 (State Machine)](#设备状态机-state-machine)
  - [硬件抽象层 (Board HAL)](#硬件抽象层-board-hal)
- [数据流程](#数据流程)
- [关键技术知识](#关键技术知识)
- [开发指南](#开发指南)

---

## 项目概述

**小智 ESP32** 是一个基于 ESP32 系列芯片（ESP32-C3、ESP32-S3、ESP32-P4）的开源 AI 语音聊天机器人项目。它通过集成：

- **离线语音唤醒** (ESP-SR)
- **流式音频处理** (ASR + LLM + TTS)
- **MCP 协议** (Model Context Protocol) 实现设备控制和云端能力扩展
- **多种通信方式** (WebSocket / MQTT+UDP)
- **多硬件平台支持** (70+ 开源硬件)

实现了一个完整的端到端 AI 语音交互系统。

---

## 整体架构

```
┌─────────────────────────────────────────────────────────────┐
│                        云端服务                              │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐      │
│  │   ASR 服务    │  │   LLM 服务    │  │   TTS 服务    │      │
│  │  (语音识别)   │  │  (大语言模型) │  │  (语音合成)   │      │
│  └──────────────┘  └──────────────┘  └──────────────┘      │
│           │                 │                 │              │
│           └─────────────────┴─────────────────┘              │
│                             │                                │
└─────────────────────────────┼────────────────────────────────┘
                              │
                    WebSocket / MQTT+UDP
                              │
┌─────────────────────────────┼────────────────────────────────┐
│                    ESP32 设备端                               │
│  ┌───────────────────────────────────────────────────────┐   │
│  │              Application (应用层)                      │   │
│  │         - 状态机管理                                   │   │
│  │         - 事件循环                                     │   │
│  │         - 任务调度                                     │   │
│  └───────────────────────────────────────────────────────┘   │
│                              │                                │
│  ┌──────────────┬────────────┴────────────┬──────────────┐   │
│  │              │                         │              │   │
│  │  Audio       │   Protocol             │   MCP        │   │
│  │  Service     │   (通信协议)            │   Server     │   │
│  │              │                         │              │   │
│  │  - 唤醒词     │   - WebSocket          │   - 工具注册  │   │
│  │  - 编解码     │   - MQTT+UDP           │   - 工具调用  │   │
│  │  - 音频流     │   - 数据传输            │   - 设备控制  │   │
│  └──────────────┴────────────┬────────────┴──────────────┘   │
│                              │                                │
│  ┌───────────────────────────┴────────────────────────────┐  │
│  │          Display & LED & Board HAL                     │  │
│  │    - OLED/LCD 显示                                      │  │
│  │    - LED 控制                                           │  │
│  │    - 按键/旋钮输入                                       │  │
│  │    - 电源管理                                           │  │
│  └────────────────────────────────────────────────────────┘  │
│                              │                                │
│  ┌───────────────────────────┴────────────────────────────┐  │
│  │              硬件驱动层                                 │  │
│  │   I2S  │  I2C  │  SPI  │  GPIO  │  WiFi  │  BLE       │  │
│  └────────────────────────────────────────────────────────┘  │
└───────────────────────────────────────────────────────────────┘
         │            │            │            │
    ┌────────┐  ┌────────┐  ┌────────┐  ┌────────┐
    │ 麦克风  │  │ 扬声器  │  │ 显示屏  │  │  按键  │
    └────────┘  └────────┘  └────────┘  └────────┘
```

---

## 核心技术栈

### 嵌入式开发框架
- **ESP-IDF 5.4+**: 乐鑫官方开发框架
- **FreeRTOS**: 实时操作系统，提供多任务调度
- **C++17**: 主要开发语言

### 音频处理
- **ESP-SR**: 乐鑫语音识别库，用于离线唤醒词检测
- **ESP-ADF (Audio Front-End)**: 音频前端处理
  - AEC (回声消除)
  - VAD (语音活动检测)
  - NS (噪声抑制)
- **Opus 编解码器**: 高质量低延迟音频编码
- **I2S 协议**: 数字音频接口

### 通信协议
- **WebSocket**: 全双工实时通信
- **MQTT + UDP**: 混合通信方案
  - MQTT: 控制消息
  - UDP: 音频流传输
- **JSON-RPC 2.0**: MCP 协议基础

### 显示与界面
- **LVGL (Light and Versatile Graphics Library)**: 嵌入式图形库
- **SSD1306 驱动**: OLED 显示屏
- **ST7789/ST7735/ILI9341**: LCD 驱动
- **QMI8658**: 六轴传感器（部分硬件）

### 网络连接
- **WiFi (ESP32 内置)**: 无线网络
- **BluFi**: 蓝牙配网
- **ML307 Cat.1 4G**: 移动网络模块（部分硬件）

### 开发工具
- **CMake**: 构建系统
- **Google C++ Style**: 代码规范

---

## 系统架构层次

项目采用分层架构设计：

```
┌──────────────────────────────────────────┐
│  应用层 (Application Layer)              │  ← 业务逻辑、状态管理
├──────────────────────────────────────────┤
│  服务层 (Service Layer)                  │  ← 音频、协议、MCP、显示
├──────────────────────────────────────────┤
│  硬件抽象层 (HAL)                         │  ← 板级支持包 (BSP)
├──────────────────────────────────────────┤
│  驱动层 (Driver Layer)                   │  ← I2S、I2C、SPI、GPIO
├──────────────────────────────────────────┤
│  操作系统层 (OS Layer)                   │  ← FreeRTOS
├──────────────────────────────────────────┤
│  硬件层 (Hardware)                       │  ← ESP32 芯片
└──────────────────────────────────────────┘
```

---

## 核心模块详解

### 应用层 (Application Layer)

**文件位置**: `main/application.cc`, `main/application.h`

**核心职责**:
- 系统初始化和启动
- 主事件循环 (`Run()`)
- 状态机管理
- 各模块协调

**关键组件**:

```cpp
class Application {
    DeviceStateMachine state_machine_;      // 设备状态机
    AudioService audio_service_;            // 音频服务
    std::unique_ptr<Protocol> protocol_;    // 通信协议
    std::unique_ptr<Ota> ota_;              // OTA 升级
    EventGroupHandle_t event_group_;        // FreeRTOS 事件组
};
```

**事件驱动架构**:

应用层使用 FreeRTOS 事件组实现事件驱动：

```cpp
// 主要事件定义
#define MAIN_EVENT_WAKE_WORD_DETECTED   (1 << 2)   // 唤醒词检测
#define MAIN_EVENT_NETWORK_CONNECTED    (1 << 7)   // 网络连接
#define MAIN_EVENT_NETWORK_DISCONNECTED (1 << 8)   // 网络断开
#define MAIN_EVENT_TOGGLE_CHAT          (1 << 9)   // 切换聊天状态
#define MAIN_EVENT_STATE_CHANGED        (1 << 12)  // 状态变更
```

**执行流程**:

```
app_main()
    ↓
Application::Initialize()
    ↓
Application::Run()  ← 主事件循环，永不返回
    ↓
事件等待 → 事件处理 → 状态转换 → 执行动作
```

---

### 音频服务 (Audio Service)

**文件位置**: `main/audio/audio_service.cc`, `main/audio/audio_service.h`

**架构详解**: 详见 `main/audio/README.md`

**核心功能**:
1. **音频采集**: 从麦克风读取 PCM 数据
2. **音频处理**: AEC、VAD、降噪
3. **编解码**: Opus 编码/解码
4. **音频播放**: 输出到扬声器
5. **唤醒词检测**: 离线语音唤醒

**多任务架构**:

```cpp
// 三个主要 FreeRTOS 任务
AudioInputTask    // 音频输入 - 从硬件读取
AudioOutputTask   // 音频输出 - 写入硬件
OpusCodecTask     // 编解码 - Opus 编解码
```

**上行音频流 (麦克风 → 云端)**:

```
麦克风 → I2S → AudioCodec
    ↓
AudioInputTask (读取原始 PCM)
    ↓
WakeWord / AudioProcessor (唤醒词检测 / AEC+VAD)
    ↓
audio_encode_queue_ (编码队列)
    ↓
OpusCodecTask (Opus 编码)
    ↓
audio_send_queue_ (发送队列)
    ↓
Protocol (网络发送)
```

**下行音频流 (云端 → 扬声器)**:

```
Protocol (网络接收)
    ↓
audio_decode_queue_ (解码队列)
    ↓
OpusCodecTask (Opus 解码)
    ↓
audio_playback_queue_ (播放队列)
    ↓
AudioOutputTask (写入 PCM)
    ↓
AudioCodec → I2S → 扬声器
```

**关键子模块**:

1. **AudioCodec** (`audio_codec.h`)
   - 硬件抽象层
   - 支持多种音频芯片 (ES8311, ES7210, ES7243E, ES8388, ES8156, AC101 等)

2. **AudioProcessor** (`audio_processor.h`)
   - AEC: 回声消除 (设备端或服务端)
   - VAD: 语音活动检测
   - NS: 噪声抑制

3. **WakeWord** (`wake_word.h`)
   - 基于 ESP-SR
   - 支持自定义唤醒词
   - MultiNet 模型

4. **OpusEncoder/Decoder** (`codecs/opus_wrapper.cc`)
   - OPUS 编码: 16kHz PCM → Opus 包
   - OPUS 解码: Opus 包 → 16/24kHz PCM

---

### 通信协议 (Protocol)

**文件位置**: `main/protocols/`

**支持的协议**:

#### 1. WebSocket 协议

**文件**: `websocket_protocol.cc`, `websocket_protocol.h`

**特点**:
- 全双工通信
- 基于 TCP
- 使用 ESP-IDF WebSocket 客户端
- 适合低延迟场景

**消息格式**:

```json
{
  "type": "audio|text|mcp|hello|...",
  "session_id": "...",
  "payload": { ... }
}
```

**详细文档**: `docs/websocket.md`

#### 2. MQTT + UDP 混合协议

**文件**: `mqtt_protocol.cc`, `mqtt_protocol.h`

**特点**:
- MQTT: 控制消息 (QoS 1)
- UDP: 音频流 (低延迟)
- 适合复杂网络环境

**MQTT 主题**:

```
iot/{device_id}/down    # 服务器下行
iot/{device_id}/up      # 设备上行
```

**UDP 音频包格式**:

```c
struct BinaryProtocol2 {
    uint16_t version;       // 协议版本
    uint16_t type;          // 0: OPUS, 1: JSON
    uint32_t reserved;
    uint32_t timestamp;     // 用于服务端 AEC
    uint32_t payload_size;
    uint8_t payload[];
};
```

**详细文档**: `docs/mqtt-udp.md`

#### 3. 协议抽象层

**基类**: `Protocol` (`protocol.h`)

```cpp
class Protocol {
    virtual bool Start() = 0;
    virtual bool OpenAudioChannel() = 0;
    virtual void CloseAudioChannel() = 0;
    virtual bool SendAudio(std::unique_ptr<AudioStreamPacket> packet) = 0;
    
    // 回调注册
    void OnIncomingAudio(callback);
    void OnIncomingJson(callback);
    void OnConnected(callback);
    void OnDisconnected(callback);
};
```

---

### MCP 服务器 (MCP Server)

**文件位置**: `main/mcp_server.cc`, `main/mcp_server.h`

**什么是 MCP**:

MCP (Model Context Protocol) 是一个标准化协议，允许：
- **LLM (大语言模型)** 调用设备端提供的工具 (Tools)
- **设备控制**: 音量、LED、GPIO、舵机等
- **信息查询**: 电量、温度、传感器数据等
- **功能扩展**: 拍照、录音、显示控制等

**协议基础**: JSON-RPC 2.0

**核心概念**:

1. **Tool (工具)**:
   - 设备端定义的可调用函数
   - 包含名称、描述、参数定义
   - LLM 可以调用这些工具

2. **Property (属性)**:
   - 工具的参数定义
   - 支持类型: boolean, integer, string
   - 可设置默认值、范围限制

**工具定义示例**:

```cpp
// 音量控制工具
PropertyList properties = {
    Property("volume", kPropertyTypeInteger, 50, 0, 100)
};

auto& mcp = McpServer::GetInstance();
mcp.AddTool("set_volume", "设置音量", properties,
    [](const PropertyList& props) -> ReturnValue {
        int volume = props["volume"].value<int>();
        Board::GetInstance().SetVolume(volume);
        return std::string("音量已设置为 " + std::to_string(volume));
    }
);
```

**常见设备端工具**:

| 工具名称 | 功能 | 参数 |
|---------|------|------|
| `set_volume` | 设置音量 | volume (0-100) |
| `set_led_color` | 设置 LED 颜色 | r, g, b (0-255) |
| `set_servo_angle` | 设置舵机角度 | angle (0-180) |
| `take_photo` | 拍摄照片 | - |
| `get_battery_level` | 获取电量 | - |

**MCP 消息流程**:

```
1. 设备连接 → 发送 hello (features.mcp: true)
                ↓
2. 服务器发送 initialize
                ↓
3. 设备响应服务器信息
                ↓
4. 服务器请求 tools/list
                ↓
5. 设备返回所有工具列表
                ↓
6. LLM 决定调用工具 → tools/call
                ↓
7. 设备执行工具 → 返回结果
```

**详细文档**: `docs/mcp-protocol.md`, `docs/mcp-usage.md`

---

### 显示系统 (Display System)

**文件位置**: `main/display/`

**支持的显示类型**:

1. **OLED 显示**: `oled_display.cc`
   - SSD1306 驱动
   - I2C 接口
   - 128x64 / 128x32 分辨率

2. **LCD 显示**: `lcd_display.cc`
   - ST7789, ST7735, ILI9341 等驱动
   - SPI 接口
   - 支持彩色显示

3. **LVGL 显示**: `lvgl_display/`
   - 基于 LVGL 图形库
   - 支持复杂 UI
   - 触摸屏支持

**显示抽象层**:

```cpp
class Display {
    virtual void ShowMessage(const char* message) = 0;
    virtual void ShowEmotion(const char* emotion) = 0;
    virtual void SetBrightness(int brightness) = 0;
};
```

**表情显示系统** (`emote_display.cc`):
- 支持自定义表情
- 基于资源文件 (Assets)
- 可在线生成: [xiaozhi-assets-generator](https://github.com/78/xiaozhi-assets-generator)

---

### 设备状态机 (State Machine)

**文件位置**: `main/device_state_machine.cc`, `main/device_state_machine.h`

**状态定义** (`device_state.h`):

```cpp
enum DeviceState {
    kDeviceStateUnknown,          // 未知状态
    kDeviceStateStarting,         // 启动中
    kDeviceStateWifiConfiguring,  // WiFi 配置中
    kDeviceStateIdle,             // 空闲
    kDeviceStateConnecting,       // 连接中
    kDeviceStateListening,        // 监听中 (录音)
    kDeviceStateSpeaking,         // 说话中 (播放)
    kDeviceStateUpgrading,        // 固件升级中
    kDeviceStateActivating,       // 激活中
    kDeviceStateAudioTesting,     // 音频测试
    kDeviceStateFatalError        // 致命错误
};
```

**状态转换规则**:

```
Starting → WifiConfiguring / Idle
         ↓
Idle ⇄ Connecting
         ↓
Idle ⇄ Listening ⇄ Speaking
         ↓
Idle → Upgrading
         ↓
Idle → Activating
```

**观察者模式**:

```cpp
state_machine_.AddStateChangeListener(
    [](DeviceState old_state, DeviceState new_state) {
        // 状态变更回调
    }
);
```

**线程安全**: 使用 `std::atomic` 和 `std::mutex` 保证状态访问安全

---

### 硬件抽象层 (Board HAL)

**文件位置**: `main/boards/`

**设计理念**:
- 为每个硬件平台创建独立的板级支持包 (BSP)
- 统一的硬件接口
- 70+ 硬件平台支持

**Board 基类**:

```cpp
class Board {
    static Board& GetInstance();
    
    virtual void Initialize() = 0;
    virtual AudioCodec* GetAudioCodec() = 0;
    virtual Display* GetDisplay() = 0;
    virtual int GetVolume() = 0;
    virtual void SetVolume(int volume) = 0;
    virtual void Reboot() = 0;
};
```

**常见板级组件** (`boards/common/`):

1. **WiFi 板卡**: `wifi_board.cc`
   - WiFi 连接管理
   - BluFi 配网

2. **4G 板卡**: `ml307_board.cc`
   - ML307 Cat.1 模块
   - AT 命令控制

3. **电池管理**:
   - `adc_battery_monitor.cc`: ADC 电压监测
   - `axp2101.cc`: AXP2101 电源管理芯片
   - `sy6970.cc`: SY6970 充电芯片

4. **输入设备**:
   - `button.cc`: 按键输入
   - `knob.cc`: 旋转编码器

5. **摄像头**: `esp32_camera.cc`
   - OV2640, OV5640 等
   - JPEG 编码
   - MCP 工具集成

**创建自定义板卡**: 参见 `docs/custom-board.md`

---

## 数据流程

### 完整语音交互流程

```
1. 【唤醒】
   用户说话 → 麦克风 → AudioInputTask
        ↓
   WakeWord 检测 → 匹配成功 → MAIN_EVENT_WAKE_WORD_DETECTED
        ↓
   Application 处理 → 状态切换到 Listening
        ↓
   播放提示音 → 开始录音

2. 【录音与上传】
   麦克风 → AudioProcessor (AEC+VAD)
        ↓
   PCM 数据 → Opus 编码
        ↓
   audio_send_queue_ → Protocol::SendAudio()
        ↓
   WebSocket/UDP → 云端 ASR

3. 【LLM 处理】
   云端: ASR 文本 → LLM → TTS 音频

4. 【播放响应】
   云端 TTS → Protocol::OnIncomingAudio()
        ↓
   audio_decode_queue_ → Opus 解码
        ↓
   audio_playback_queue_ → AudioOutputTask
        ↓
   扬声器播放
        ↓
   播放完成 → 状态切换回 Idle

5. 【MCP 工具调用】 (可选)
   LLM 决定调用工具 → MCP tools/call
        ↓
   McpServer::DoToolCall()
        ↓
   执行设备操作 (如设置 LED、拍照等)
        ↓
   返回结果给 LLM
```

### 网络连接流程

```
1. 设备启动
        ↓
2. WiFi 连接 / 4G 拨号
        ↓
3. DNS 解析服务器地址
        ↓
4. WebSocket 连接 / MQTT 连接
        ↓
5. 发送 hello 消息
        ↓
6. 接收服务器 hello (含 session_id, audio_params)
        ↓
7. MCP 初始化 (如果支持)
        ↓
8. MAIN_EVENT_NETWORK_CONNECTED
        ↓
9. 状态切换到 Idle → 等待唤醒
```

### OTA 升级流程

```
1. 服务器下发升级指令 (type: "upgrade")
        ↓
2. Application::UpgradeFirmware()
        ↓
3. 状态切换到 Upgrading
        ↓
4. Ota::Start() → 下载固件
        ↓
5. 校验固件
        ↓
6. 写入 OTA 分区
        ↓
7. 设置启动分区
        ↓
8. 重启设备
```

---

## 关键技术知识

### 1. FreeRTOS 任务管理

**任务优先级设计**:

```cpp
// 按优先级从高到低
AudioOutputTask      // 优先级最高 - 避免播放卡顿
AudioInputTask       // 次高优先级 - 避免录音丢帧
OpusCodecTask        // 中等优先级
NetworkTask          // 网络任务
MainTask             // 主事件循环
```

**任务间通信**:
- **Queue**: 音频数据队列
- **Event Group**: 事件通知
- **Mutex**: 资源互斥访问

### 2. ESP-IDF 特性

**NVS (Non-Volatile Storage)**:
- 存储 WiFi 配置
- 设备设置持久化

**Partition Table**:
```
factory:  出厂固件
ota_0:    OTA 分区 0
ota_1:    OTA 分区 1
spiffs:   文件系统 (Assets)
nvs:      NVS 存储
```

**SPIFFS 文件系统**:
- 存储字体、表情、唤醒词模型
- 支持在线更新

### 3. I2S 音频接口

**配置示例**:

```cpp
i2s_config_t i2s_config = {
    .mode = I2S_MODE_MASTER | I2S_MODE_TX | I2S_MODE_RX,
    .sample_rate = 16000,
    .bits_per_sample = I2S_BITS_PER_SAMPLE_16BIT,
    .channel_format = I2S_CHANNEL_FMT_ONLY_LEFT,
    .communication_format = I2S_COMM_FORMAT_STAND_I2S,
    // ...
};
```

**DMA 传输**: I2S 使用 DMA 实现零 CPU 开销的数据传输

### 4. Opus 音频编码

**优势**:
- 低延迟 (20-60ms)
- 高压缩比
- 适合语音和音乐

**参数**:
```cpp
opus_encoder_create(
    16000,              // 采样率
    1,                  // 单声道
    OPUS_APPLICATION_VOIP  // 语音优化
);
```

### 5. WebSocket vs MQTT+UDP

| 特性 | WebSocket | MQTT+UDP |
|-----|-----------|----------|
| 延迟 | 低 (TCP) | 极低 (UDP 音频) |
| 可靠性 | 高 | 中等 (UDP 可能丢包) |
| 复杂度 | 简单 | 较复杂 |
| 适用场景 | 通用 | 复杂网络环境 |

### 6. MCP 协议实现

**JSON-RPC 2.0 消息示例**:

```json
// Request
{
  "jsonrpc": "2.0",
  "method": "tools/call",
  "params": {
    "name": "set_volume",
    "arguments": { "volume": 80 }
  },
  "id": 1
}

// Response
{
  "jsonrpc": "2.0",
  "result": {
    "content": [
      { "type": "text", "text": "音量已设置为 80" }
    ],
    "isError": false
  },
  "id": 1
}
```

### 7. ESP-SR 语音识别

**MultiNet 模型**:
- 支持中文、英文、日文
- 离线运行
- 可自定义唤醒词

**唤醒词检测流程**:
```
音频流 → AFE 前端处理 → MultiNet 推理 → 唤醒词匹配
```

### 8. 电源管理

**低功耗策略**:
- 空闲时关闭 ADC/DAC (`AUDIO_POWER_TIMEOUT_MS`)
- Light Sleep 模式 (WiFi 保持连接)
- Deep Sleep 模式 (长时间不用)

**电池监测**:
- ADC 读取电池电压
- AXP2101/SY6970 芯片读取电量百分比

---

## 开发指南

### 1. 环境搭建

**必需工具**:
- VSCode / Cursor
- ESP-IDF 插件
- ESP-IDF 5.4+

**推荐系统**: Linux (编译速度快，驱动兼容性好)

**安装步骤**:

```bash
# 1. 克隆项目
git clone https://github.com/78/xiaozhi-esp32.git
cd xiaozhi-esp32

# 2. 安装 ESP-IDF (参考官方文档)
# https://docs.espressif.com/projects/esp-idf/zh_CN/latest/esp32/get-started/

# 3. 配置开发板
idf.py set-target esp32s3  # 或 esp32c3, esp32p4

# 4. 配置项目
idf.py menuconfig

# 5. 编译
idf.py build

# 6. 烧录
idf.py flash monitor
```

### 2. 选择开发板

**在 menuconfig 中**:
```
Component config → 
    Xiaozhi Configuration → 
        Board Selection → 
            [选择你的硬件]
```

**或使用预设配置**:
```bash
cp sdkconfig.defaults.esp32s3 sdkconfig.defaults
```

### 3. 添加新硬件支持

参考文档: `docs/custom-board.md`

**步骤**:
1. 创建 `main/boards/your-board/` 目录
2. 实现 `Board` 类
3. 定义 `AudioCodec`, `Display`, `LED` 等组件
4. 在 `Kconfig.projbuild` 添加配置选项

### 4. 代码风格

**遵循 Google C++ Style Guide**:
- 使用 4 空格缩进
- 类名: `PascalCase`
- 函数名: `PascalCase`
- 变量名: `snake_case_` (成员变量加下划线)
- 常量: `kConstantName`

**示例**:

```cpp
class MyClass {
public:
    void DoSomething();

private:
    int member_variable_;
    static constexpr int kMaxSize = 100;
};
```

### 5. 调试技巧

**日志级别**:

```cpp
ESP_LOGI(TAG, "Info message");
ESP_LOGW(TAG, "Warning message");
ESP_LOGE(TAG, "Error message");
ESP_LOGD(TAG, "Debug message");  // 需要在 menuconfig 启用
```

**监控串口**:

```bash
idf.py monitor
```

**分析内存**:

```bash
idf.py size
idf.py size-components
```

### 6. 常见问题

**Q1: 编译时提示缺少头文件**
- A: 运行 `idf.py reconfigure` 重新生成配置

**Q2: 音频有杂音或回声**
- A: 检查 AEC 配置，调整麦克风增益

**Q3: WiFi 连接失败**
- A: 检查 SSID/密码，尝试使用 BluFi 配网

**Q4: 设备无法唤醒**
- A: 检查唤醒词模型是否正确，调整检测阈值

**Q5: OTA 升级失败**
- A: 检查分区表配置，确保 OTA 分区足够大

### 7. 性能优化

**内存优化**:
- 使用 `std::unique_ptr` 避免内存泄漏
- 队列大小根据实际需求调整
- 及时释放不用的资源

**CPU 优化**:
- 音频处理使用硬件加速 (I2S DMA)
- 合理设置任务优先级
- 避免在高优先级任务中阻塞

**网络优化**:
- 使用 Opus 压缩减少带宽
- UDP 模式适合弱网环境
- 实现断线重连机制

---

## 参考资料

### 官方文档
- [ESP-IDF 编程指南](https://docs.espressif.com/projects/esp-idf/zh_CN/latest/esp32/index.html)
- [ESP-SR 语音识别](https://github.com/espressif/esp-sr)
- [LVGL 图形库](https://docs.lvgl.io/)

### 项目文档
- [自定义开发板指南](custom-board.md)
- [MCP 协议详解](mcp-protocol.md)
- [MCP 使用说明](mcp-usage.md)
- [WebSocket 协议](websocket.md)
- [MQTT+UDP 协议](mqtt-udp.md)

### 相关项目
- [Python 服务器](https://github.com/xinnan-tech/xiaozhi-esp32-server)
- [Java 服务器](https://github.com/joey-zhou/xiaozhi-esp32-server-java)
- [Golang 服务器](https://github.com/AnimeAIChat/xiaozhi-server-go)
- [自定义 Assets 生成器](https://github.com/78/xiaozhi-assets-generator)

### 社区
- QQ 群: 1011329060
- GitHub Issues: [提交问题](https://github.com/78/xiaozhi-esp32/issues)
- 飞书文档: [小智 AI 聊天机器人百科全书](https://ccnphfhqs21z.feishu.cn/wiki/F5krwD16viZoF0kKkvDcrZNYnhb)

---

## 贡献指南

欢迎贡献代码、文档或硬件支持！

**贡献方式**:
1. Fork 本项目
2. 创建特性分支 (`git checkout -b feature/AmazingFeature`)
3. 提交更改 (`git commit -m 'Add some AmazingFeature'`)
4. 推送到分支 (`git push origin feature/AmazingFeature`)
5. 提交 Pull Request

**代码审查要点**:
- 遵循 Google C++ Style
- 添加必要的注释
- 通过编译测试
- 不破坏现有功能

---

## 许可证

本项目采用 MIT 许可证，允许自由使用、修改和商业应用。

详见 [LICENSE](../LICENSE) 文件。

---

**最后更新**: 2026-01-15  
**版本**: v2.1.0  
**维护者**: 虾哥 (78)
