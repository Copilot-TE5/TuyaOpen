# TuyaOpen 在线语音服务产品规格书

> **文档版本**：v1.1  
> **适用项目**：[TuyaOpen](https://github.com/tuya/TuyaOpen)  
> **最后更新**：2026-03-10

---

## 目录

1. [产品概述](#1-产品概述)
2. [在线语音服务架构](#2-在线语音服务架构)
3. [核心功能模块](#3-核心功能模块)
   - 3.1 [语音活动检测（VAD）](#31-语音活动检测vad)
   - 3.2 [关键词唤醒（KWS）](#32-关键词唤醒kws)
   - 3.3 [语音识别（ASR）](#33-语音识别asr)
   - 3.4 [自然语言处理（NLU/NLP）](#34-自然语言处理nlunlp)
   - 3.5 [语音合成（TTS）](#35-语音合成tts)
   - 3.6 [AI 对话代理（AI Agent）](#36-ai-对话代理ai-agent)
4. [交互触发模式](#4-交互触发模式)
5. [音频编解码支持](#5-音频编解码支持)
6. [通信协议与安全](#6-通信协议与安全)
7. [应用程序与示例](#7-应用程序与示例)
8. [支持的硬件平台](#8-支持的硬件平台)
9. [快速上手指南](#9-快速上手指南)
10. [API 接口概览](#10-api-接口概览)
11. [配置参数说明](#11-配置参数说明)
12. [系统限制与注意事项](#12-系统限制与注意事项)

---

## 1. 产品概述

TuyaOpen 是涂鸦智能（Tuya）推出的开源跨平台物联网 AI SDK，通过连接**涂鸦云（Tuya Cloud）**与 AI 服务，为开发者提供完整的**在线语音交互**能力，包括：

- 🎤 **实时语音识别（ASR）**：将用户语音上传云端，完成准确转写
- 💬 **自然语言理解（NLU）**：理解用户意图，结合大语言模型生成回复
- 🔊 **语音合成（TTS）**：将 AI 回复内容转换为自然语音播放
- 🔍 **本地唤醒词检测（KWS）**：无需联网即可响应唤醒词
- 📊 **语音活动检测（VAD）**：自动检测用户说话时机

整个语音处理链路从设备端采集音频，经由涂鸦云完成 AI 处理，再将结果以 TTS 语音形式反馈给用户，形成完整的**端云一体化语音交互闭环**。

### 服务定位

| 维度 | 说明 |
|------|------|
| **目标用户** | 嵌入式开发者、IoT 产品厂商、AI 硬件创客 |
| **核心价值** | 零门槛集成云端 AI 语音服务，快速实现智能硬件产品落地 |
| **云端依托** | 涂鸦 AI 云平台（tuya.ai），无需对接第三方 ASR/TTS 服务商 |
| **开发模式** | 开源 SDK + 授权码（License），即插即用 |

---

## 2. 在线语音服务架构

```
┌─────────────────────────────────────────────────────────────────┐
│                         设备端（本地）                           │
│                                                                  │
│  ┌──────────┐   ┌──────────┐   ┌──────────┐   ┌────────────┐   │
│  │  麦克风   │→  │  VAD     │→  │  KWS     │→  │ 音频编码器 │   │
│  │  采集     │   │ 语音检测 │   │ 唤醒词   │   │(Opus/Speex)│   │
│  └──────────┘   └──────────┘   └──────────┘   └─────┬──────┘   │
│                                                       │          │
│  ┌──────────┐   ┌──────────┐   ┌──────────┐          │          │
│  │  扬声器   │←  │  解码器  │←  │  TTS下载 │          │          │
│  │  播放     │   │(MP3/Opus)│   │ (HTTP)   │          │          │
│  └──────────┘   └──────────┘   └──────────┘          │          │
└───────────────────────────────────────────────────────┼──────────┘
                                                        │ MQTT/TCP/UDP
                                                        ↓
┌─────────────────────────────────────────────────────────────────┐
│                       涂鸦 AI 云端                               │
│                                                                  │
│  ┌──────────┐   ┌──────────┐   ┌──────────┐   ┌────────────┐   │
│  │  ASR     │→  │  NLU     │→  │  LLM     │→  │  TTS       │   │
│  │ 语音识别 │   │ 语义理解 │   │ 大语言模型│   │  语音合成  │   │
│  └──────────┘   └──────────┘   └──────────┘   └────────────┘   │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### 服务层次结构

| 层次 | 模块 | 说明 |
|------|------|------|
| **应用层** | `apps/tuya.ai/` | 完整应用示例（聊天机器人等） |
| **AI 音频层** | `ai_components/ai_audio/` | 音频输入/输出封装接口 |
| **AI 代理层** | `svc_ai_agent/` | 会话管理、事件处理核心框架 |
| **AI 基础层** | `svc_ai_basic/` | 协议定义、MQTT/HTTP 通信 |
| **编解码层** | `svc_ai_codec/` | Opus/Speex 音频编码器 |
| **硬件适配层** | `tools/porting/adapter/` | VAD、KWS 硬件抽象接口 |

---

## 3. 核心功能模块

### 3.1 语音活动检测（VAD）

VAD（Voice Activity Detection）用于实时检测音频流中是否存在人声，避免将静音片段上传云端，节省带宽并提升响应速度。

**功能特性：**
- 实时检测，低延迟（10ms 帧间隔）
- 可配置语音最小持续时长和静音最小持续时长
- 支持灵敏度调节（scale 参数）
- 检测结果通过事件 `EVENT.VAD` 通知上层应用

**配置参数：**

| 参数 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `sample_rate` | uint32_t | 16000 | 采样率（Hz） |
| `channel_num` | uint8_t | 1 | 声道数（1=单声道） |
| `speech_min_ms` | int | 300 | 语音最小持续时长（ms），低于此值不触发 |
| `noise_min_ms` | int | 500 | 静音补偿时长（ms），语音结束后继续录制的时长 |
| `frame_duration_ms` | int | 10 | 每帧时长（ms） |
| `scale` | float | 1.0 | 灵敏度调节（0.5=更敏感，2.0=更迟钝） |

**VAD 状态：**

| 状态值 | 说明 |
|--------|------|
| `TKL_VAD_STATUS_NONE` (0) | 无语音 / 静音 |
| `TKL_VAD_STATUS_SPEECH` (1) | 检测到语音 |

**核心 API：**
```c
OPERATE_RET tkl_vad_init(TKL_VAD_CONFIG_T *config);   // 初始化
OPERATE_RET tkl_vad_start(void);                       // 启动检测
OPERATE_RET tkl_vad_feed(uint8_t *data, uint32_t len); // 喂入音频帧
TKL_VAD_STATUS_T tkl_vad_get_status(void);             // 获取当前状态
OPERATE_RET tkl_vad_stop(void);                        // 停止检测
OPERATE_RET tkl_vad_deinit(void);                      // 释放资源
```

**示例程序：** `examples/multimedia/audio_vad/`

---

### 3.2 关键词唤醒（KWS）

KWS（Keyword Spotting）在本地完成唤醒词识别，无需网络连接，功耗极低，响应及时。检测到唤醒词后，系统进入语音交互状态，建立与云端的 AI 对话会话。

**支持的唤醒词列表：**

| 枚举值 | 唤醒词 | 语言 | 适用场景 |
|--------|--------|------|----------|
| `TKL_KWS_WAKEUP_NIHAO_TUYA` | "你好图雅" | 中文 | 涂鸦品牌设备 |
| `TKL_KWS_WAKEUP_NIHAO_XIAOZHI` | "你好小智" | 中文 | 涂鸦小智助手 |
| `TKL_KWS_WAKEUP_HEY_TUYA` | "Hey Tuya" | 英文 | 英文环境设备 |
| `TKL_KWS_WAKEUP_SMARTLIFE` | "Smart Life" | 英文 | Smart Life 品牌 |
| `TKL_KWS_WAKEUP_ZHINENGGUANJIA` | "智能管家" | 中文 | 智慧家居场景 |
| `TKL_KWS_WAKEUP_XIAOZHI_TONGXUE` | "小智同学" | 中文 | 教育/学习场景 |
| `TKL_KWS_WAKEUP_XIAOZHI_GUANJIA` | "小智管家" | 中文 | 家居管家场景 |
| `TKL_KWS_WAKEUP_XIAOAI_XIAOAI` | "小爱小爱" | 中文 | 小米生态设备 |
| `TKL_KWS_WAKEUP_XIAODU_XIAODU` | "小度小度" | 中文 | 百度小度设备 |

**核心 API：**
```c
OPERATE_RET tkl_kws_init(void);                              // 初始化唤醒引擎
OPERATE_RET tkl_kws_reg_wakeup_cb(TKL_KWS_WAKEUP_CB cb);  // 注册唤醒回调
OPERATE_RET tkl_kws_enable(void);                            // 启用唤醒检测
OPERATE_RET tkl_kws_disable(void);                           // 禁用唤醒检测
OPERATE_RET tkl_kws_deinit(void);                            // 释放资源
```

**示例程序：** `examples/multimedia/audio_kws/`

---

### 3.3 语音识别（ASR）

ASR（Automatic Speech Recognition）语音识别完全在涂鸦云端完成，设备只需采集音频并通过加密协议上传，无需本地安装任何 ASR 引擎。

**服务特性：**
- ☁️ **云端处理**：识别准确率高，支持持续更新模型
- 🔒 **加密传输**：默认使用 GCM 加密（AI_PACKET_SL4）
- 📡 **流式传输**：支持边说边识别，低延迟响应
- 🌐 **多语言支持**：支持中文、英文等多语言识别（取决于云端配置）
- 📦 **分片上传**：大音频文件支持 START/ING/END 分片传输

**音频上传格式：**

| 参数 | 推荐值 | 说明 |
|------|--------|------|
| 编码格式 | Opus | 高质量低码率，推荐首选 |
| 备选格式 | Speex / PCM | 根据硬件能力选择 |
| 采样率 | 16000 Hz | 标准语音识别采样率 |
| 声道数 | 1（单声道） | 语音识别标准配置 |
| 比特深度 | 16 bit | 标准精度 |

**ASR 结果回调：**
ASR 识别结果以文本形式通过 `text_cb` 回调函数返回，类型为 `AI_TEXT_ASR`（0x00）。

---

### 3.4 自然语言处理（NLU/NLP）

语义理解和回复生成完全由涂鸦云端的大语言模型（LLM）完成，开发者无需配置任何 AI 模型。

**服务特性：**
- 🧠 **大语言模型驱动**：连接涂鸦 AI 云端的 LLM 服务
- 🎭 **可定制 AI 角色**：支持在 APP 端实时切换 AI 实体角色
- 💾 **对话上下文管理**：支持多轮对话，记忆上下文
- 🔧 **MCP 工具调用**：支持 Model Context Protocol（MCP），AI 可调用本地工具
- 🌊 **流式文本输出**：支持文字流式（Streaming）输出，实时显示

**输出文本类型：**

| 类型标识 | 值 | 说明 |
|----------|----|------|
| `AI_TEXT_ASR` | 0x00 | 语音识别文字结果 |
| `AI_TEXT_NLG` | 0x01 | 自然语言生成回复 |
| `AI_TEXT_SKILL` | 0x02 | 技能指令响应 |
| `AI_TEXT_OTHER` | 0x03 | 其他文本 |
| `AI_TEXT_CLOUD_EVENT` | 0x04 | 云端事件消息 |

**MCP 工具调用支持：**
```c
// 注册 MCP 工具调用回调
OPERATE_RET tuya_ai_agent_mcp_set_cb(TY_AI_MCP_CB cb, void *user_data);
// 响应 MCP 工具调用结果
OPERATE_RET tuya_ai_agent_mcp_response(char *message);
```

---

### 3.5 语音合成（TTS）

TTS（Text-to-Speech）服务由涂鸦云端完成合成，设备通过 HTTP 下载合成后的音频文件并本地播放，支持流式播放（边下边播）。

**TTS 服务特性：**
- 🌐 **云端合成**：自然流畅的语音质量
- 📻 **流式播放**：支持 START/DATA/STOP/ABORT 状态机，低延迟启播
- 🎵 **背景音乐混合**：支持 TTS 与背景音乐同时播放
- 📢 **系统提示音**：内置多种场景提示音（开机、断网、低电量等）
- 🔊 **音量控制**：支持 0-100 音量调节

**TTS 音频格式配置：**

| 参数 | 默认值 | 可选值 | 说明 |
|------|--------|--------|------|
| 编码格式 | `mp3` | `mp3` / `opus` | 下载音频格式 |
| 比特率 | 16000 bps | 16000 / 48000 / 64000 | 音频质量 |
| 采样率 | 16000 Hz | 16000 / 24000 | 合成采样率 |

**TTS 配置示例：**
```c
AI_AGENT_TTS_CFG_T tts_cfg = {
    .bit_rate    = 16000,   // 比特率
    .sample_rate = 16000,   // 采样率
    .format      = "opus"   // 推荐使用 opus 获得更好质量
};
tuya_ai_agent_set_tts_cfg(&tts_cfg);
```

**内置系统提示音类型：**

| 提示音 | 枚举值 | 触发场景 |
|--------|--------|----------|
| 开机提示 | `AI_AUDIO_ALERT_POWER_ON` | 设备启动时 |
| 未配网提示 | `AI_AUDIO_ALERT_NOT_ACTIVE` | 设备未连接网络 |
| 配网中 | `AI_AUDIO_ALERT_NETWORK_CFG` | 正在配网 |
| 连网成功 | `AI_AUDIO_ALERT_NETWORK_CONNECTED` | WiFi 连接成功 |
| 连网失败 | `AI_AUDIO_ALERT_NETWORK_FAIL` | 网络连接失败 |
| 网络断开 | `AI_AUDIO_ALERT_NETWORK_DISCONNECT` | 网络断开时 |
| 低电量 | `AI_AUDIO_ALERT_BATTERY_LOW` | 电量低时 |
| 请再说 | `AI_AUDIO_ALERT_PLEASE_AGAIN` | 未识别到语音 |
| 长按说话 | `AI_AUDIO_ALERT_LONG_KEY_TALK` | 长按按键模式提示 |
| 按键说话 | `AI_AUDIO_ALERT_KEY_TALK` | 按键触发模式提示 |
| 唤醒说话 | `AI_AUDIO_ALERT_WAKEUP_TALK` | 唤醒词触发后提示 |
| 随机对话 | `AI_AUDIO_ALERT_RANDOM_TALK` | 随机对话触发提示 |
| 唤醒响应 | `AI_AUDIO_ALERT_WAKEUP` | 唤醒词识别成功响应 |

**核心 API：**
```c
// 播放 TTS URL（从云端下载音频）
OPERATE_RET ai_audio_play_tts_url(AI_AUDIO_PLAY_TTS_T *playtts, bool is_loop);

// 播放 TTS 流数据（流式接收并播放）
OPERATE_RET ai_audio_play_tts_stream(AI_AUDIO_PLAYER_TTS_STATE_E state,
                                     AI_AUDIO_CODEC_E codec,
                                     char *data, int len);

// 播放系统提示音
OPERATE_RET ai_audio_player_alert(AI_AUDIO_ALERT_TYPE_E type);

// 音量控制（0-100）
OPERATE_RET ai_audio_player_set_vol(int vol);
OPERATE_RET ai_audio_player_get_vol(int *vol);

// 播放音乐（支持播放列表）
OPERATE_RET ai_audio_play_music(AI_AUDIO_MUSIC_T *music);
```

**示例程序：** `examples/multimedia/audio_player/tts/`

---

### 3.6 AI 对话代理（AI Agent）

AI Agent 是整个在线语音服务的核心框架，负责管理与涂鸦云的连接、会话生命周期和数据流转。

**功能特性：**
- 🔗 **连接管理**：自动维护与涂鸦 AI 云的 MQTT 长连接
- 📋 **会话管理**：支持多会话并发，独立会话隔离
- 🔄 **事件驱动**：完整的事件生命周期（START → PROC → END）
- 🌊 **流式通信**：音频/文本双向流式传输
- 🛡️ **加密通信**：默认 GCM 加密，支持 RSA 密钥交换
- 📊 **服务端 VAD**：可选启用云端语音活动检测

**会话生命周期：**
```
tuya_ai_agent_init()          // 初始化 AI Agent
    ↓
tuya_ai_agent_crt_session()   // 创建对话会话
    ↓
tuya_ai_output_register_cbs() // 注册结果回调
    ↓
tuya_ai_input_start()         // 启动音频输入
    ↓
tuya_ai_audio_input()         // 持续上传音频帧
    ↓
[接收 ASR 结果 → text_cb]
[接收 TTS 音频 → media_data_cb]
    ↓
tuya_ai_input_stop()          // 停止音频输入
    ↓
tuya_ai_agent_del_session()   // 关闭会话
    ↓
tuya_ai_agent_deinit()        // 释放资源
```

**输出回调函数签名：**
```c
typedef struct {
    // 事件回调（开始/结束/中断等）
    OPERATE_RET (*event_cb)(AI_EVENT_TYPE etype, AI_PACKET_PT ptype, AI_EVENT_ID eid);
    // 媒体属性回调（音视频参数）
    OPERATE_RET (*media_attr_cb)(AI_BIZ_ATTR_INFO_T *attr);
    // 媒体数据回调（TTS 音频数据）
    OPERATE_RET (*media_data_cb)(AI_PACKET_PT type, char *data, uint32_t len, uint32_t total_len);
    // 文本回调（ASR/NLG 文字结果）
    OPERATE_RET (*text_cb)(AI_TEXT_TYPE_E type, cJSON *root, bool eof);
    // 提示音回调（系统事件提示）
    OPERATE_RET (*alert_cb)(AI_ALERT_TYPE_E type);
} AI_OUTPUT_CBS_T;
```

**多媒体输入接口（多模态支持）：**
```c
// 上传音频数据（语音对话）
OPERATE_RET tuya_ai_audio_input(uint64_t timestamp, uint64_t pts,
                                uint8_t *data, uint32_t len, uint32_t total_len);

// 上传视频帧（视觉 AI）
OPERATE_RET tuya_ai_video_input(uint64_t timestamp, uint64_t pts,
                                uint8_t *data, uint32_t len, uint32_t total_len);

// 上传图片（图像识别）
OPERATE_RET tuya_ai_image_input(uint64_t timestamp, uint8_t *data,
                               uint32_t len, uint32_t total_len);

// 上传文本（文字对话）
OPERATE_RET tuya_ai_text_input(uint8_t *data, uint32_t len, uint32_t total_len);

// 上传文件（文件处理）
OPERATE_RET tuya_ai_file_input(uint8_t *data, uint32_t len, uint32_t total_len);
```

---

## 4. 交互触发模式

TuyaOpen 支持 4 种语音交互触发模式，适配不同产品形态：

### 4.1 唤醒词模式（Wakeup Mode）

**注册方式：** `ai_mode_wakeup_register()`

**工作流程：**
```
待机（VAD静音检测）
    ↓
KWS 检测到唤醒词（"你好图雅"等）
    ↓
播放唤醒响应提示音
    ↓
VAD 自动检测用户说话
    ↓
上传语音到云端 ASR
    ↓
播放 TTS 回复
    ↓
回到待机
```

**适用产品：** 智能音箱、家居中控、桌面助手等免手触产品

### 4.2 按住模式（Hold Mode）

**注册方式：** `ai_mode_hold_register()`

**工作流程：**
```
用户按下按键
    ↓
开始录音并上传
    ↓
用户松开按键
    ↓
发送录音结束信号
    ↓
等待并播放 TTS 回复
```

**适用产品：** 对讲机、手持设备、需防误触的产品

### 4.3 自由对话模式（Free Mode）

**注册方式：** `ai_mode_free_register()`

**工作流程：**
```
常驻 VAD 检测
    ↓
检测到语音自动开始上传
    ↓
VAD 检测到静音自动结束
    ↓
播放 TTS 回复
    ↓
继续 VAD 检测（持续对话）
```

**适用产品：** 儿童故事机、陪伴机器人、情感交互设备

### 4.4 单次模式（Oneshot Mode）

**注册方式：** `ai_mode_oneshot_register()`

**工作流程：**
```
按键触发单次录音
    ↓
录音完成后发送到云端
    ↓
播放回复并结束本次交互
```

**适用产品：** 语音查询终端、工业控制设备

### 触发模式对比

| 模式 | 触发方式 | 结束方式 | 推荐场景 |
|------|----------|----------|----------|
| 唤醒词模式 | KWS 唤醒词 | VAD 检测静音 | 智能音箱、无按键设备 |
| 按住模式 | 按下按键 | 松开按键 | 手持设备、精准控制 |
| 自由模式 | VAD 自动检测 | VAD 自动检测 | 陪伴机器人、持续对话 |
| 单次模式 | 按键单次触发 | 录音完成 | 查询终端、单指令设备 |

---

## 5. 音频编解码支持

### 5.1 上行音频编码（设备→云端）

| 编码格式 | 标识符 | 特点 | 推荐程度 |
|----------|--------|------|----------|
| **Opus** | `AUDIO_CODEC_OPUS` (111) | 高质量、低延迟、低码率，开源标准 | ⭐⭐⭐ 强烈推荐 |
| **Speex** | `AUDIO_CODEC_SPEEX` (108) | 专为语音设计，轻量级 | ⭐⭐ 推荐 |
| **PCM** | `AUDIO_CODEC_PCM` (101) | 无压缩，高质量但带宽消耗大 | ⭐ 备选 |
| **G.711 μ-law** | `AUDIO_CODEC_G711U` (105) | 电话质量语音 | ⭐ 特殊需求 |
| **G.711 A-law** | `AUDIO_CODEC_G711A` (106) | 电话质量语音（欧标） | ⭐ 特殊需求 |
| **G.726** | `AUDIO_CODEC_G726` (107) | ADPCM 变体 | ⭐ 特殊需求 |
| **G.722** | `AUDIO_CODEC_G722` (110) | 宽带语音编码 | ⭐ 特殊需求 |

### 5.2 下行音频解码（云端→设备，TTS）

| 解码格式 | 说明 |
|----------|------|
| **MP3** | 默认 TTS 格式，兼容性好 |
| **Opus** | 高质量 TTS，推荐 |
| **OggOpus** | Ogg 容器 + Opus 内容 |
| **WAV** | PCM 未压缩格式 |

### 5.3 视频编码支持（多模态）

| 编码格式 | 标识符 | 说明 |
|----------|--------|------|
| H.264 | `VIDEO_CODEC_H264` | 主流视频编码 |
| H.265 | `VIDEO_CODEC_H265` | 高效视频编码 |
| MJPEG | `VIDEO_CODEC_MJPEG` | 逐帧 JPEG |
| YUV420 | `VIDEO_CODEC_YUV420` | 原始 YUV 格式 |

### 5.4 支持的图片格式（多模态）

| 格式 | 标识符 | 说明 |
|------|--------|------|
| JPEG | `IMAGE_FORMAT_JPEG` | 标准压缩图片 |
| PNG | `IMAGE_FORMAT_PNG` | 无损压缩图片 |

### 5.5 编码器配置参数

```c
// Opus 编码器配置（推荐参数）
TUYA_AI_ENCODER_INFO_T encoder_info = {
    .encode_type     = AUDIO_CODEC_OPUS,  // 使用 Opus 编码
    .sample_rate     = 16000,             // 采样率 16kHz
    .channels        = 1,                 // 单声道
    .bits_per_sample = 16,               // 16bit 深度
    .frame_size      = 320,              // 帧大小（样本数，对应 20ms）
};
```

---

## 6. 通信协议与安全

### 6.1 网络通信协议

| 协议 | 用途 | 说明 |
|------|------|------|
| **MQTT** | 主通道（获取 AI 服务器配置、上传音频） | 涂鸦云 MQTT Broker |
| **TCP/UDP** | 实时音频流传输（高性能通道） | 直连 AI 服务器 |
| **HTTP/HTTPS** | TTS 音频文件下载 | CDN 加速下载 |
| **WebSocket** | 备用实时通道 | 可选 |

### 6.2 安全加密级别

| 安全级别 | 标识符 | 加密算法 | 说明 |
|----------|--------|----------|------|
| 0 | `AI_PACKET_SL0` | 无加密 | 测试用，不建议生产使用 |
| 2 | `AI_PACKET_SL2` | ChaCha20 | 流加密 |
| 3 | `AI_PACKET_SL3` | AES-CBC | 块加密 |
| **4（默认）** | `AI_PACKET_SL4` | **AES-GCM** | **推荐，默认配置** |
| 6 | `AI_PACKET_RSA` | RSA | 非对称加密（密钥交换） |

所有生产设备默认使用 **AES-GCM（AI_PACKET_SL4）** 加密，确保音频数据传输安全。

### 6.3 协议数据包类型

| 包类型 | 标识符 | 值 | 说明 |
|--------|--------|-----|------|
| 音频数据 | `AI_PT_AUDIO` | 31 | 语音数据包 |
| 视频数据 | `AI_PT_VIDEO` | 30 | 视频数据包 |
| 图片数据 | `AI_PT_IMAGE` | 32 | 图片数据包 |
| 文件数据 | `AI_PT_FILE` | 33 | 文件数据包 |
| 文本数据 | `AI_PT_TEXT` | 34 | 文本数据包 |
| 事件 | `AI_PT_EVENT` | 35 | 控制事件 |
| 会话创建 | `AI_PT_SESSION_NEW` | 7 | 新建会话 |
| 会话关闭 | `AI_PT_SESSION_CLOSE` | 8 | 关闭会话 |

### 6.4 服务状态码

| 状态码 | 含义 |
|--------|------|
| 200 | 成功 |
| 400 | 请求错误 |
| 401 | 未认证（License 无效） |
| 404 | 资源未找到 |
| 408 | 请求超时 |
| 500 | 服务器内部错误 |
| 504 | 网关超时 |
| 601 | 客户端主动关闭 |
| 602 | 连接复用关闭 |
| 603 | IO 错误关闭 |
| 604 | 心跳超时关闭 |
| 605 | 会话过期关闭 |

---

## 7. 应用程序与示例

### 7.1 your_chat_bot（AI 智能聊天机器人）

**路径：** `apps/tuya.ai/your_chat_bot/`  
**功能：** 完整的 AI 语音对话应用，支持语音交互、屏幕显示和 APP 远程查看。

**核心功能：**
- ✅ AI 智能语音对话（ASR + NLU + LLM + TTS 完整链路）
- ✅ 按键唤醒 / 语音唤醒，支持轮流对话
- ✅ 支持语音打断（需硬件回声消除支持）
- ✅ LCD 屏幕实时显示聊天内容（微信气泡样式）
- ✅ 涂鸦 APP 端实时查看和切换 AI 角色
- ✅ 快速蓝牙配网
- ✅ 表情动画显示

**支持的硬件：**

| 开发板 | 芯片 | 特性 |
|--------|------|------|
| TUYA T5AI_Board | T5 | 3.5" LCD、摄像头、全功能 |
| TUYA T5AI_EVB | T5 | 评估板 |
| DNESP32S3 | ESP32-S3 | ESP 系列代表 |
| ESP32S3_BREAD_COMPACT_WIFI | ESP32-S3 | 紧凑型开发板 |
| RaspberryPi | - | 树莓派（Linux 环境） |

### 7.2 audio_vad 示例

**路径：** `examples/multimedia/audio_vad/`  
**功能：** 演示 VAD 语音活动检测的完整使用方法。  
**适用场景：** 学习 VAD 配置和事件处理，构建自定义语音触发逻辑。

### 7.3 audio_kws 示例

**路径：** `examples/multimedia/audio_kws/`  
**功能：** 演示本地唤醒词"你好图雅"的识别流程。  
**适用场景：** 学习 KWS 唤醒词集成，实现免触式交互。

### 7.4 audio_player/tts 示例

**路径：** `examples/multimedia/audio_player/tts/`  
**功能：** 演示从 URL 下载并播放 TTS 语音。  
**适用场景：** 学习 TTS 播放接口，集成语音播报功能。

### 7.5 audio_recorder 示例

**路径：** `examples/multimedia/audio_recorder/`  
**功能：** 演示音频录制和 WAV 文件编码。  
**适用场景：** 学习音频采集，实现本地录音功能。

### 7.6 其他 AI 应用

| 应用 | 路径 | 说明 |
|------|------|------|
| 机器狗控制 | `apps/tuya.ai/your_robot_dog/` | 语音控制机器狗运动 |
| 表情桌面 | `apps/tuya.ai/your_desk_emoji/` | AI 驱动表情显示 |
| OTTO 机器人 | `apps/tuya.ai/your_otto_robot/` | 语音控制仿人机器人 |
| 串口聊天机器人 | `apps/tuya.ai/your_serial_chat_bot/` | 通过串口与 AI 交互 |
| 双目情绪 | `apps/tuya.ai/duo_eye_mood/` | AI 情感表达眼睛 |

---

## 8. 支持的硬件平台

### 8.1 支持的芯片/模组

| 芯片/模组 | 类型 | WiFi | 蓝牙 | 音频编解码 | 备注 |
|-----------|------|------|------|------------|------|
| **T5 / T5-E1** | MCU | ✅ | ✅ | 外置（ES8388等） | 涂鸦自研，推荐 |
| **ESP32-S3** | MCU | ✅ | ✅ | 外置 | Espressif，广泛支持 |
| **T2 / T3** | MCU | ✅ | ✅ | 外置 | 涂鸦 WiFi MCU |
| **Raspberry Pi** | Linux SBC | ✅ | ✅ | 内置/外置 | Linux 平台 |
| **BK7231N** | MCU | ✅ | ✅ | 外置 | Beken 平台 |
| **LN882H** | MCU | ✅ | ✅ | 外置 | Lightning Semi |

### 8.2 支持的音频编解码芯片

| 芯片 | 接口 | 特性 | 推荐场景 |
|------|------|------|----------|
| **ES8388** | I2S + I2C | 集成 ADC/DAC，高保真 | T5AI 标准配置，强烈推荐 |
| **ES8389** | I2S + I2C | ES8388 升级版 | 新款开发板 |
| **WM8311** | I2S + I2C | 模拟音频输出 | 低成本方案 |
| **无编解码器** | 直接 PCM | 纯软件处理 | Linux 平台 |

### 8.3 硬件要求

| 功能 | 最低要求 |
|------|----------|
| **麦克风** | 单声道麦克风（KWS/VAD/ASR 需要） |
| **扬声器** | 单声道扬声器（TTS 播放） |
| **网络** | WiFi 802.11b/g/n |
| **Flash** | ≥ 4MB（完整功能） |
| **RAM** | ≥ 512KB（基础功能） |
| **CPU** | ≥ 160MHz（Opus 编码推荐） |

---

## 9. 快速上手指南

### 9.1 前置准备

**步骤 1：获取涂鸦云账号和 PID**
1. 在 [https://iot.tuya.com](https://iot.tuya.com) 注册涂鸦开发者账号
2. 创建产品，获取产品 PID（`TUYA_PRODUCT_ID`）

**步骤 2：获取 TuyaOpen 专用授权码**

> 注意：必须使用 TuyaOpen 专用授权码，普通涂鸦授权码无法连接 AI 服务。

获取方式：
- **方式 1**：购买已烧录 TuyaOpen 授权码的模组
- **方式 2**：在 [涂鸦平台购买页](https://platform.tuya.com/purchase/index?type=6) 购买
- **方式 3**：在 [淘宝购买](https://item.taobao.com/item.htm?id=911596682625)
- **免费领取**：在 GitHub 上 Star [TuyaOpen 仓库](https://github.com/tuya/TuyaOpen) 后发邮件申请（限 500 个）

**步骤 3：配置授权码**

编辑 `apps/tuya.ai/your_chat_bot/include/tuya_config.h`：
```c
#define TUYA_PRODUCT_ID  "your_product_id_here"
#define TUYA_DEVICE_UUID "uuidxxxxxxxxxxxxxxxx"
#define TUYA_DEVICE_AUTHKEY "keyxxxxxxxxxxxxxxxxxxxxxxxxxxxxx"
```

### 9.2 编译和烧录

```bash
# 1. 初始化环境
cd /path/to/TuyaOpen && . ./export.sh

# 2. 进入应用目录
cd apps/tuya.ai/your_chat_bot

# 3. 选择开发板
tos config_choice

# 4. 编译
tos build

# 5. 烧录
tos flash
```

### 9.3 配网和使用

1. 下载**涂鸦智能 APP**（支持 iOS / Android）
2. 点击添加设备，扫描 BLE 信号配网
3. 配网完成后，设备自动连接涂鸦 AI 云
4. 对设备说唤醒词"**你好图雅**"开始语音对话
5. 可在 APP 端实时查看聊天记录并切换 AI 角色

### 9.4 环境初始化代码示例

```c
#include "tuya_ai_agent.h"
#include "ai_audio_input.h"
#include "ai_audio_player.h"

// 1. 初始化 AI Agent
AI_AGENT_CFG_T ai_cfg = {
    .codec_enable = TRUE,
    .tts_cfg = {
        .bit_rate    = 16000,
        .sample_rate = 16000,
        .format      = "opus"
    },
    .enable_mcp = TRUE
};
tuya_ai_agent_init(&ai_cfg);

// 2. 初始化音频播放器
ai_audio_player_init();
ai_audio_player_set_vol(70);  // 设置音量为 70%

// 3. 初始化音频输入
AI_AUDIO_INPUT_CFG_T input_cfg = {
    .vad_mode      = AI_AUDIO_VAD_AUTO,    // 自动 VAD
    .vad_off_ms    = 800,                  // VAD 静音补偿 800ms
    .vad_active_ms = 300,                  // 语音活动阈值 300ms
    .slice_ms      = 20,                   // 20ms 分片
    .output_cb     = my_audio_output_cb    // 音频数据回调
};
ai_audio_input_init(&input_cfg);

// 4. 注册输出回调
AI_OUTPUT_CBS_T output_cbs = {
    .event_cb      = on_event,
    .text_cb       = on_text,
    .media_data_cb = on_media_data,
    .alert_cb      = on_alert
};
tuya_ai_output_register_cbs("", &output_cbs, AI_OUTPUT_CBS_MODE_NORMAL);
```

---

## 10. API 接口概览

### 10.1 AI Agent 核心接口

| 函数 | 说明 |
|------|------|
| `tuya_ai_agent_init(cfg)` | 初始化 AI Agent |
| `tuya_ai_agent_deinit()` | 释放 AI Agent |
| `tuya_ai_agent_crt_session(scode, ...)` | 创建对话会话 |
| `tuya_ai_agent_del_session(scode)` | 删除对话会话 |
| `tuya_ai_agent_get_session(scode)` | 获取会话信息 |
| `tuya_ai_agent_is_ready()` | 检查 Agent 是否就绪 |
| `tuya_ai_agent_set_tts_cfg(cfg)` | 设置 TTS 配置 |
| `tuya_ai_agent_server_vad_ctrl(flag)` | 控制服务端 VAD |
| `tuya_ai_agent_mcp_set_cb(cb, data)` | 注册 MCP 工具调用回调 |
| `tuya_ai_agent_mcp_response(msg)` | 响应 MCP 工具调用 |

### 10.2 音频输入接口

| 函数 | 说明 |
|------|------|
| `tuya_ai_audio_input(ts, pts, data, len, total)` | 上传音频帧 |
| `tuya_ai_audio_input_direct(...)` | 直接上传（不经过编码器） |
| `tuya_ai_input_start(force)` | 启动输入会话 |
| `tuya_ai_input_stop()` | 停止输入会话 |
| `tuya_ai_input_get_state()` | 获取输入状态 |
| `tuya_ai_input_alert(type, cb)` | 发送警报事件 |

### 10.3 音频输出接口

| 函数 | 说明 |
|------|------|
| `ai_audio_player_init()` | 初始化播放器 |
| `ai_audio_play_tts_url(playtts, loop)` | 播放 TTS（URL 方式） |
| `ai_audio_play_tts_stream(state, codec, data, len)` | 播放 TTS（流式） |
| `ai_audio_play_music(music)` | 播放音乐 |
| `ai_audio_player_alert(type)` | 播放系统提示音 |
| `ai_audio_player_stop(type)` | 停止播放 |
| `ai_audio_player_set_vol(vol)` | 设置音量（0-100） |
| `ai_audio_player_get_vol(&vol)` | 获取当前音量 |
| `ai_audio_player_is_playing()` | 查询播放状态 |

### 10.4 VAD 接口

| 函数 | 说明 |
|------|------|
| `tkl_vad_init(config)` | 初始化 VAD |
| `tkl_vad_start()` | 启动 VAD 检测 |
| `tkl_vad_feed(data, len)` | 喂入音频数据 |
| `tkl_vad_get_status()` | 获取 VAD 状态 |
| `tkl_vad_stop()` | 停止 VAD |
| `tkl_vad_deinit()` | 释放 VAD |

### 10.5 KWS 接口

| 函数 | 说明 |
|------|------|
| `tkl_kws_init()` | 初始化 KWS 引擎 |
| `tkl_kws_reg_wakeup_cb(cb)` | 注册唤醒回调 |
| `tkl_kws_enable()` | 启用唤醒检测 |
| `tkl_kws_disable()` | 禁用唤醒检测 |
| `tkl_kws_deinit()` | 释放 KWS 引擎 |

---

## 11. 配置参数说明

### 11.1 Kconfig 编译选项

| 配置项 | 说明 | 默认值 |
|--------|------|--------|
| `CONFIG_ENABLE_COMP_AI_AUDIO_CODEC_OPUS` | 启用 Opus 编码器 | 开启 |
| `CONFIG_ENABLE_COMP_AI_AUDIO_CODEC_SPEEX` | 启用 Speex 编码器 | 可选 |
| `CONFIG_ENABLE_COMP_AI_VIDEO` | 启用视频多模态 | 可选 |
| `CONFIG_ENABLE_AI_UI_TEXT_STREAMING` | 启用文本流式显示 | 开启 |
| `CONFIG_AI_INPUT_STACK_SIZE` | 输入任务栈大小 | 28672 |
| `CONFIG_ENABLE_JOYINSIDE` | 启用聚内 AI | 可选 |

### 11.2 TTS 配置

```c
AI_AGENT_TTS_CFG_T tts_cfg = {
    .bit_rate    = 16000,  // 推荐 16000，高质量可用 48000
    .sample_rate = 16000,  // 推荐 16000Hz
    .format      = "opus"  // 推荐 opus，兼容性好用 mp3
};
```

### 11.3 音频输入配置

```c
AI_AUDIO_INPUT_CFG_T input_cfg = {
    .vad_mode      = AI_AUDIO_VAD_AUTO,   // 自动 VAD（或 AI_AUDIO_VAD_MANUAL 按键触发）
    .vad_off_ms    = 800,                 // 静音补偿时长（ms），建议 500-1000
    .vad_active_ms = 300,                 // 语音活动判定阈值（ms），建议 200-500
    .slice_ms      = 20,                  // 音频切片时长（ms）
    .output_cb     = audio_output_cb      // 音频数据输出回调
};
```

### 11.4 VAD 配置

```c
TKL_VAD_CONFIG_T vad_config = {
    .sample_rate      = 16000,   // 采样率（Hz）
    .channel_num      = 1,       // 声道数
    .speech_min_ms    = 300,     // 最短语音持续时长（ms）
    .noise_min_ms     = 500,     // 最短静音持续时长（ms）
    .frame_duration_ms = 10,     // 每帧时长（ms）
    .scale            = 1.0      // 灵敏度（0.5 更敏感，2.0 更迟钝）
};
```

---

## 12. 系统限制与注意事项

### 12.1 授权与认证

- ⚠️ **必须使用 TuyaOpen 专用授权码**，使用普通涂鸦 IoT 授权码将无法连接 AI 服务
- ⚠️ 每个授权码对应唯一设备，不可共享
- ⚠️ 授权码包含 UUID（20位）和 AUTHKEY（32位）两个字段

### 12.2 网络要求

- 设备必须能访问涂鸦云服务器（国内/海外服务器自动选择）
- 语音交互期间网络带宽建议 ≥ 100 Kbps（上行）
- 服务器地址和端口通过 MQTT 动态下发，无需硬编码

### 12.3 音频要求

- KWS 唤醒词检测和 VAD 需要麦克风硬件支持
- 语音打断功能需要**回声消除（AEC）**硬件支持，否则扬声器音频会触发 VAD
- 推荐使用 ES8388 音频编解码芯片获得最佳效果

### 12.4 云服务依赖

- 在线语音服务（ASR/NLU/TTS）完全依赖涂鸦 AI 云端，无法离线使用
- 仅 VAD 和 KWS 为本地功能，可在无网络状态下运行
- 云端 AI 模型和能力由涂鸦持续更新，开发者无需维护

### 12.5 已知限制

| 限制项 | 说明 | 建议 |
|--------|------|------|
| 离线 ASR | 不支持本地语音识别 | 确保网络稳定 |
| 多语言 TTS | TTS 语言取决于云端配置 | 在涂鸦开发者平台配置 |
| 并发会话 | 最大并发会话数受云端限制 | 单设备通常 1-2 个会话 |
| 音频延迟 | 端到端延迟约 500ms-2s | 取决于网络质量和 ASR 处理速度 |

---

## 附录

### A. 相关文档

| 文档 | 说明 |
|------|------|
| [在线语音网络韧性分析](./online_voice_network_resilience.md) | 深入分析在线语音服务的网络异常处理、断线重连策略及韧性设计 |
| [网络状态统整机制详解](./network_status_management.md) | TuyaOpen SDK 多网络连接状态统一管理架构与实现细节 |

### B. 相关资源链接

| 资源 | 链接 |
|------|------|
| TuyaOpen GitHub | [https://github.com/tuya/TuyaOpen](https://github.com/tuya/TuyaOpen) |
| 涂鸦开发者平台 | [https://iot.tuya.com](https://iot.tuya.com) |
| T5AI 开发板文档 | [https://developer.tuya.com/en/docs/iot-device-dev/T5-E1-IPEX-development-board](https://developer.tuya.com/en/docs/iot-device-dev/T5-E1-IPEX-development-board?id=Ke9xehig1cabj) |
| TuyaOpen 授权码购买 | [https://platform.tuya.com/purchase/index?type=6](https://platform.tuya.com/purchase/index?type=6) |
| T5AI_EVB 开源硬件 | [https://oshwhub.com/flyingcys/t5ai_evb](https://oshwhub.com/flyingcys/t5ai_evb) |

### B. 术语表

| 术语 | 全称 | 说明 |
|------|------|------|
| VAD | Voice Activity Detection | 语音活动检测 |
| KWS | Keyword Spotting | 关键词识别/唤醒词检测 |
| ASR | Automatic Speech Recognition | 自动语音识别 |
| TTS | Text-to-Speech | 文字转语音合成 |
| NLU | Natural Language Understanding | 自然语言理解 |
| NLG | Natural Language Generation | 自然语言生成 |
| LLM | Large Language Model | 大语言模型 |
| AEC | Acoustic Echo Cancellation | 回声消除 |
| MCP | Model Context Protocol | 模型上下文协议 |
| MQTT | Message Queuing Telemetry Transport | 消息队列遥测传输协议 |
| Opus | - | 高效音频编码标准（RFC 6716） |
| Speex | - | 专为语音设计的开源编解码器 |

### C. 版本历史

| 版本 | 日期 | 说明 |
|------|------|------|
| v1.1 | 2026-03-12 | 新增相关文档交叉引用（网络韧性分析、网络状态管理） |
| v1.0 | 2026-03-10 | 初始版本，完整覆盖在线语音服务规格 |

---

*本文档基于 TuyaOpen 项目代码分析整理，如有疑问请参考源代码或联系涂鸦技术支持。*
