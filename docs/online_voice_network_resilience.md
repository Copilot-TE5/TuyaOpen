# TuyaOpen 在线语音网络韧性分析

> **文档版本**：v1.1  
> **适用项目**：[TuyaOpen](https://github.com/tuya/TuyaOpen)  
> **最后更新**：2026-03-12  
> **关联规格书**：[online_voice_service_spec.md](./online_voice_service_spec.md)

---

## 目录

1. [背景与目标](#1-背景与目标)
2. [系统架构概览](#2-系统架构概览)
3. [AI 客户端状态机](#3-ai-客户端状态机)
4. [网络断开处理](#4-网络断开处理)
5. [网络不稳定处理](#5-网络不稳定处理)
6. [心跳保活机制](#6-心跳保活机制)
7. [空闲检测与资源管理](#7-空闲检测与资源管理)
8. [用户体验提示通知](#8-用户体验提示通知)
9. [会话中断处理](#9-会话中断处理)
10. [音频缓冲保护](#10-音频缓冲保护)
11. [客制化扩展指南](#11-客制化扩展指南)
12. [总结与建议](#12-总结与建议)

---

## 1. 背景与目标

在线语音交互（语音识别 ASR、自然语言处理 NLU、语音合成 TTS）高度依赖稳定的网络连接。当网络出现波动或中断时，若处理不当将直接导致：

- 语音指令无响应，用户需反复唤醒
- 播放中断，TTS 音频被截断
- 系统卡死，无法自动恢复
- 不明确的错误状态，用户无从判断问题所在

本文档深入分析 TuyaOpen 项目中 **`src/tuya_ai_service/`** 模块针对以上问题的应对策略，并说明如何借助这些机制为用户提供更佳的使用体验。

---

## 2. 系统架构概览

```
┌──────────────────────────────────────────────────────────────────┐
│                          应用层 (apps/)                           │
│    app_chat_bot.c → ai_chat_main.c → ai_agent.c                  │
│                          ↑  alert_cb / event_cb                  │
├──────────────────────────────────────────────────────────────────┤
│                     AI Agent 层 (svc_ai_agent)                    │
│  tuya_ai_agent.c ─┬─ tuya_ai_input.c  (音频输入缓冲)             │
│                   └─ tuya_ai_output.c (结果输出与通知)            │
├──────────────────────────────────────────────────────────────────┤
│                     AI 基础层 (svc_ai_basic)                      │
│  tuya_ai_client.c  ← 状态机核心、心跳、重连                       │
│  tuya_ai_mqtt.c    ← MQTT 云端通信                                │
│  tuya_ai_http.c    ← TTS 音频 HTTP 下载                           │
├──────────────────────────────────────────────────────────────────┤
│                      网络适配层 (port/)                            │
│  tuya_svc_netmgr.c ← 网络状态查询（MQTT在线判断）                 │
│  tuya_svc_devos.c  ← 设备注册状态                                 │
└──────────────────────────────────────────────────────────────────┘
```

网络相关事件从底层沿箭头方向向上传递，最终通过音频提示（本地音频 / 云端 TTS / 自定义回调）告知用户。

---

## 3. AI 客户端状态机

> **核心文件**：`src/tuya_ai_service/svc_ai_basic/src/tuya_ai_client.c`

### 3.1 状态定义

```c
typedef enum {
    AI_STATE_IDLE,          // 空闲 / 等待网络就绪
    AI_STATE_SETUP,         // 请求云端服务配置（MQTT 握手前置）
    AI_STATE_CONNECT,       // 建立 TCP/TLS 连接
    AI_STATE_CLIENT_HELLO,  // 发送客户端握手包
    AI_STATE_AUTH_REQ,      // 发送鉴权请求（v1 协议专用）
    AI_STATE_AUTH_RESP,     // 等待鉴权响应
    AI_STATE_RUNNING,       // 正常运行，收发业务数据
    AI_STATE_END
} AI_CLIENT_STATE_E;
```

### 3.2 状态转移图

```
                    ┌─────────────────────────────────────────────┐
                    │     网络未就绪 / 时间未同步 → 循环等待        │
                    ▼                                             │
               ┌─────────┐  网络就绪 & 时间同步               │
               │  IDLE   │──────────────────────────┐          │
               └─────────┘                          ▼          │
                    ▲                          ┌─────────┐      │
                    │  MQTT离线                │  SETUP  │      │
                    │  ◄───────────────────────└─────────┘      │
                    │                          SETUP成功 ▼       │
                    │                        ┌──────────┐       │
                    │                        │ CONNECT  │       │
                    │   连接失败/超时          └──────────┘       │
                    │  ◄──────── sleep(1s) ── 连接失败           │
                    │                        CONNECT成功 ▼       │
                    │                     ┌──────────────────┐  │
                    │                     │  CLIENT_HELLO    │  │
                    │                     └──────────────────┘  │
                    │                     CLIENT_HELLO成功 ▼     │
                    │                     ┌──────────────────┐  │
                    │                     │    AUTH_RESP     │  │
                    │  AUTH失败/超时        └──────────────────┘  │
                    │  ◄──── sleep(1s) ── 鉴权失败               │
                    │                     AUTH成功 ▼             │
                    │                     ┌──────────────────┐  │
                    │                     │    RUNNING       │──┘
                    │  运行中出错 / 服务器  └──────────────────┘
                    │  关闭连接 / 心跳超时
                    └─────────────────────────────────────────────
```

### 3.3 线程主循环

```c
// tuya_ai_client.c：437-473 行
STATIC VOID __ai_client_thread_cb(void* args)
{
    OPERATE_RET rt = OPRT_OK;
    while (!ai_basic_client->terminate &&
           tal_thread_get_state(ai_basic_client->thread) == THREAD_STATE_RUNNING) {
        switch (ai_basic_client->state) {
        case AI_STATE_IDLE:    rt = __ai_idle();    break;
        case AI_STATE_SETUP:   rt = __ai_setup();   break;
        case AI_STATE_CONNECT: rt = __ai_connect(); break;
        // ... 其余状态
        case AI_STATE_RUNNING: rt = __ai_running(); break;
        default: break;
        }
        if (OPRT_OK != rt) {
            __ai_client_handle_err(rt);  // 统一错误处理入口
        }
    }
    __ai_client_free();
}
```

---

## 4. 网络断开处理

### 4.1 IDLE 状态的网络检测

```c
// tuya_ai_client.c：366-374 行
STATIC OPERATE_RET __ai_idle()
{
    // 双重检查：网络 MQTT 在线 + 时间同步完成
    if ((tuya_svc_netmgr_get_status() != NETWORK_STATUS_MQTT) ||
        (tal_time_check_time_sync() != OPRT_OK)) {
        return OPRT_COM_ERROR;  // 返回错误，线程等待后重试
    }
    __ai_client_set_state(AI_STATE_SETUP);
    return OPRT_OK;
}
```

**行为说明**：
- 断网期间，客户端持续停留在 `IDLE` 状态，**不会发起任何连接尝试**
- 网络恢复（MQTT 重新上线）后，状态机自动进入 `SETUP` 阶段，无需外部干预
- 同时要求 NTP 时间已同步，避免因时间戳错误导致鉴权失败

### 4.2 SETUP 阶段的断线检测

```c
// tuya_ai_client.c：376-394 行
STATIC OPERATE_RET __ai_setup()
{
    rt = tuya_ai_mq_ser_cfg_req();  // 通过 MQTT 请求服务器配置
    if (OPRT_OK != rt) {
        if (rt == OPRT_SVC_MQTT_GW_MQ_OFFLILNE) {
            __ai_client_set_state(AI_STATE_IDLE);  // MQTT 离线，退回 IDLE
        }
        return rt;
    }
    // ...
}
```

**行为说明**：若 MQTT 通道在 `SETUP` 过程中断开，立即退回 `IDLE` 等待，而非在应用层空转。

### 4.3 运行中断线处理

```c
// tuya_ai_client.c：193-195 行
} else if (ai_basic_client->state == AI_STATE_RUNNING) {
    PR_NOTICE("ai client running error %d, reconnect %d", rt, tal_net_get_errno());
    __ai_conn_close();  // 主动关闭连接并回到 IDLE
}

// tuya_ai_client.c：167-175 行
STATIC OPERATE_RET __ai_conn_close(VOID)
{
    ty_publish_event(EVENT_AI_CLIENT_CLOSE, NULL);  // 通知上层业务
    tuya_ai_client_stop_ping();                      // 停止心跳
    tuya_ai_basic_conn_close(AI_CODE_CLOSE_BY_CLIENT);
    __ai_client_set_state(AI_STATE_IDLE);            // 回到 IDLE 等待重连
    return rt;
}
```

**事件传播链**：

```
__ai_conn_close()
    └── ty_publish_event(EVENT_AI_CLIENT_CLOSE)
            └── tuya_ai_agent.c：__ai_session_closed_evt()
                    └── __ai_agent_del_sid()  // 清理会话资源
```

### 4.4 服务器主动关闭连接

```c
// tuya_ai_client.c：244-261 行
STATIC VOID __ai_handle_conn_close(char *data, uint32_t len)
{
    PR_NOTICE("recv conn close by server");
    tuya_ai_parse_conn_close(data + offset, attr_len);  // 解析关闭原因
    ty_publish_event(EVENT_AI_CLIENT_CLOSE, NULL);
    __ai_client_set_state(AI_STATE_IDLE);               // 退回 IDLE
}
```

服务器关闭连接包含原因码（如 Token 过期、配额耗尽等），解析后可供应用层决策。

---

## 5. 网络不稳定处理

### 5.1 指数退避重连

当网络抖动导致 `SETUP` 阶段持续失败时，采用**随机化指数退避（Randomized Exponential Backoff）**策略，避免大量设备同时重连造成云端冲击（雷暴效应）。

```c
// tuya_ai_client.c：40、560-562 行
#define AI_RECONN_TIME_NUM 7

AI_RECONN_TIME_T reconn[AI_RECONN_TIME_NUM] = {
    {5, 10},      // 第 1 次失败：随机等待  5 ~ 10  秒
    {10, 20},     // 第 2 次失败：随机等待 10 ~ 20  秒
    {20, 40},     // 第 3 次失败：随机等待 20 ~ 40  秒
    {40, 80},     // 第 4 次失败：随机等待 40 ~ 80  秒
    {80, 160},    // 第 5 次失败：随机等待 80 ~ 160 秒
    {160, 320},   // 第 6 次失败：随机等待 160 ~ 320 秒
    {320, 640}    // 第 7 次及以上：随机等待 320 ~ 640 秒（最大值）
};
```

**退避等待时间可视化**：

```
失败次数  最小等待  最大等待  典型场景
────────  ──────── ──────── ──────────────────────
  1 次       5s      10s    网络短暂抖动
  2 次      10s      20s    路由器重启
  3 次      20s      40s    运营商故障
  4 次      40s      80s    持续网络不稳
  5 次      80s     160s    长时间断网
  6 次     160s     320s
  7 次+    320s     640s    严重故障（约 10 分钟）
```

**重连逻辑代码**：

```c
// tuya_ai_client.c：177-199 行
STATIC VOID __ai_client_handle_err(OPERATE_RET rt)
{
    if (ai_basic_client->state == AI_STATE_SETUP) {
        uint32_t sleep_random = __ai_get_random_value(
            ai_basic_client->reconn[ai_basic_client->reconn_cnt].min,
            ai_basic_client->reconn[ai_basic_client->reconn_cnt].max
        );
        PR_NOTICE("connect to cloud failed, sleep %d s", sleep_random);
        tal_system_sleep(sleep_random * 1000);

        // 重连计数递增，上限为最高级别（不超出数组范围）
        uint32_t size = AI_RECONN_TIME_NUM - 1;
        if (ai_basic_client->reconn_cnt >= size) {
            ai_basic_client->reconn_cnt = size;  // 钳位到最大级别
        } else {
            ai_basic_client->reconn_cnt++;
        }
    } else if ((ai_basic_client->state == AI_STATE_CONNECT) ||
               (ai_basic_client->state == AI_STATE_AUTH_RESP)) {
        tal_system_sleep(1000);
        __ai_client_set_state(AI_STATE_SETUP);  // 退回 SETUP 重试
    } else if (ai_basic_client->state == AI_STATE_RUNNING) {
        __ai_conn_close();  // 运行中出错，触发关闭并回 IDLE
    } else {
        tal_system_sleep(1000);
    }
}
```

**重连计数重置**：连接成功进入 `CONNECT → CLIENT_HELLO` 时立即清零，确保下次抖动从最短等待时间重新开始：

```c
// tuya_ai_client.c：110 行
ai_basic_client->reconn_cnt = 0;  // 成功连接，重连计数归零
```

### 5.2 运行中短暂读取失败的容忍

```c
// tuya_ai_client.c：327-336 行
rt = tuya_ai_basic_pkt_read(&de_buf, &de_len, &frag);
if (OPRT_RESOURCE_NOT_READY == rt) {
    return OPRT_OK;  // 数据还未到达，静默等待，不触发重连
} else if ((OPRT_OK != rt) || (de_buf == NULL)) {
    if ((rt == -1) && (cnt <= 3)) {
        if (tuya_svc_netmgr_get_status() == NETWORK_STATUS_MQTT) {
            cnt++;
            tal_system_sleep(1000);
            return OPRT_OK;  // MQTT 还在线，最多容忍 3 次连续读取失败
        }
    }
    return rt;  // 确认异常，触发错误处理
}
```

**行为说明**：  
- `OPRT_RESOURCE_NOT_READY`：数据包尚未完整到达，属于正常情况，不触发重连  
- 错误码 `-1`（socket 错误）且 MQTT 仍在线：最多容忍 **3 次** 连续失败，给予网络自恢复机会  
- 超过 3 次或 MQTT 已离线：触发 `__ai_conn_close()` 进入重连流程

### 5.3 Token 到期前主动刷新

为防止 Token 过期导致连接突然中断，系统在 Token 到期前 **10 秒**主动发起刷新请求：

```c
// tuya_ai_client.c：208-219 行
STATIC VOID __ai_start_expire_tid()
{
    uint64_t expire = tuya_ai_mq_ser_cfg_get()->expire;
    uint64_t current = tal_time_get_posix();
    // 到期前 10 秒触发 __ai_conn_refresh
    tal_sw_timer_start(ai_basic_client->tid, (expire - current - 10) * 1000, TAL_TIMER_ONCE);
}

STATIC VOID __ai_conn_refresh(TIMER_ID timerID, void *pTimerArg)
{
    tuya_ai_basic_refresh_req();  // 发送连接刷新请求
}
```

刷新响应处理会更新 Token 并重启刷新计时器，实现无缝续期。

---

## 6. 心跳保活机制

> **作用**：检测 TCP 连接的"假活"（连接建立但数据无法传输）状态，这是网络不稳时最常见的隐患。

### 6.1 参数配置

```c
// tuya_ai_client.c：41-46 行
#ifndef AT_PING_TIMEOUT
#define AT_PING_TIMEOUT         6    // ping 等待 pong 超时时间（秒）
#endif
#ifndef AI_HEARTBEAT_INTERVAL
#define AI_HEARTBEAT_INTERVAL   120  // 心跳发送间隔（秒）
#endif
```

### 6.2 心跳发送

```c
// tuya_ai_client.c：493-501 行
STATIC VOID __ai_ping(VOID *data)
{
    // 先启动超时定时器（6 秒），再发送 ping
    tal_sw_timer_start(ai_basic_client->alive_timeout_timer,
                       AT_PING_TIMEOUT * 1000, TAL_TIMER_ONCE);
    rt = tuya_ai_basic_ping();
    if (OPRT_OK != rt) {
        PR_ERR("send ping to cloud failed, rt:%d", rt);
    }
}
```

### 6.3 超时处理

```c
// tuya_ai_client.c：503-515 行
STATIC VOID __ai_alive_timeout(TIMER_ID timer_id, VOID_T *data)
{
    PR_ERR("alive timeout");
    ai_basic_client->heartbeat_lost_cnt++;
    if (ai_basic_client->heartbeat_lost_cnt >= 3) {
        // 连续 3 次 ping 无响应 → 确认连接已断，主动关闭
        PR_ERR("ping lost >= 3, close tcp connection");
        __ai_conn_close();
    } else {
        // 继续尝试 ping（间隔 10ms），给予短暂恢复机会
        tal_workq_start_delayed(ai_basic_client->alive_work, 10, LOOP_ONCE);
    }
}
```

### 6.4 pong 响应处理

```c
// tuya_ai_client.c：263-268 行
STATIC VOID __ai_handle_pong(char *data, uint32_t len)
{
    tuya_ai_pong(data, len);
    tuya_ai_client_start_ping();  // 收到 pong，重新开始 120s 心跳计时
    PR_NOTICE("ai pong");
}
```

### 6.5 收到数据时重置心跳计时

```c
// tuya_ai_client.c：337 行（__ai_running 中）
__ai_stop_alive_time();  // 任何业务数据到达都重置心跳计时器
```

```c
// tuya_ai_client.c：201-206 行
STATIC VOID __ai_stop_alive_time()
{
    ai_basic_client->heartbeat_lost_cnt = 0;  // 丢包计数归零
    tal_sw_timer_stop(ai_basic_client->alive_timeout_timer);
}
```

### 6.6 心跳时序图

```
设备                                    云端
 │                                        │
 │──────── ping ──────────────────────►  │
 │                  6秒超时计时开始        │
 │  ◄───────────── pong ─────────────── │  → 正常：120s 后发下一次 ping
 │                                        │
 │──────── ping ──────────────────────►  │
 │                  6秒内无 pong          │
 │  heartbeat_lost_cnt = 1               │
 │  立即发下一次 ping (10ms 后)           │
 │──────── ping ──────────────────────►  │
 │  heartbeat_lost_cnt = 2               │
 │──────── ping ──────────────────────►  │
 │  heartbeat_lost_cnt = 3               │
 │  → __ai_conn_close() → IDLE → 重连   │
```

---

## 7. 空闲检测与资源管理

> **目的**：自动释放长时间空闲（无业务交互）的云端连接，节省云端资源配额。

### 7.1 常规空闲检测

鉴权成功后，每 **30 分钟**检查一次是否有业务数据往来：

```c
// tuya_ai_client.c：154-155 行（__ai_auth_resp 中）
ai_basic_client->recv_biz_pkt = FALSE;
tal_sw_timer_start(ai_basic_client->idle_check_timer, AI_IDLE_CHECK_TIME, TAL_TIMER_ONCE);
```

```c
// tuya_ai_client.c：284-313 行
STATIC VOID __ai_idle_check(TIMER_ID timer_id, VOID_T *data)
{
#if defined(AI_VERSION) && (0x02 == AI_VERSION)
    if (ai_basic_client->idle_check_enable) {
        // 空闲检测模式已激活
        if (!ai_basic_client->recv_biz_pkt) {
            __ai_conn_close();  // 30 分钟内无业务 → 断开连接
            return;
        } else {
            ai_basic_client->recv_biz_pkt = FALSE;  // 重置标志继续监测
        }
    } else {
        // 长时间运行检测（12~18 小时后激活空闲检测模式）
        uint32_t random_value = uni_random_range(6);
        uint32_t continue_run_time = (12 + random_value) * 60 * 60;
        if ((now_time - ai_basic_client->start_time) > continue_run_time) {
            ai_basic_client->idle_check_enable = TRUE;
            ai_basic_client->recv_biz_pkt = FALSE;
        }
    }
    tal_sw_timer_start(ai_basic_client->idle_check_timer, AI_IDLE_CHECK_TIME, TAL_TIMER_ONCE);
#endif
}
```

### 7.2 云端触发的延迟断开

云端可主动发送 `AI_PT_DELAY_DISCONNECT` 包，触发设备进入空闲检测模式：

```c
// tuya_ai_client.c：270-282 行
STATIC VOID __ai_delay_dis_req(VOID)
{
    PR_NOTICE("recv delay disconnect pkt");
    if (!ai_basic_client->idle_check_enable) {
        ai_basic_client->idle_check_enable = TRUE;
        ai_basic_client->recv_biz_pkt = FALSE;
        // 30 分钟后检查，若无业务则断开
        tal_sw_timer_start(ai_basic_client->idle_check_timer,
                           AI_IDLE_CHECK_TIME, TAL_TIMER_ONCE);
    }
}
```

**用户体验影响**：延迟断开不会立即影响用户体验，设备在 30 分钟静默期满后才会断开并进入重连流程，重连后可继续正常使用。

---

## 8. 用户体验提示通知

> **核心文件**：  
> - `src/tuya_ai_service/svc_ai_agent/include/tuya_ai_output.h`（类型定义）  
> - `src/tuya_ai_service/svc_ai_agent/src/tuya_ai_agent.c`（触发点）  
> - `apps/tuya.ai/ai_components/ai_audio/src/ai_audio_player.c`（播放实现）

### 8.1 网络相关提示类型

```c
// tuya_ai_output.h：30-60 行
typedef enum {
    AT_POWER_ON,            // 开机提示
    AT_NOT_ACTIVE,          // 未激活，请先配网
    AT_NETWORK_CFG,         // 进入配网状态，开始配网
    AT_NETWORK_CONNECTED,   // 网络连接成功        ← 重连成功时触发
    AT_NETWORK_FAIL,        // 网络连接失败，重试中  ← 会话创建失败时触发
    AT_NETWORK_DISCONNECT,  // 网络已断开           ← 可扩展触发
    AT_BATTERY_LOW,         // 低电量
    AT_PLEASE_AGAIN,        // 请重说（识别为空时）  ← 网络抖动导致 ASR 失败时触发
    // ...
} AI_ALERT_TYPE_E;
```

### 8.2 触发时机

| 提示类型 | 触发条件 | 触发位置 |
|----------|----------|----------|
| `AT_NETWORK_CONNECTED` | AI 客户端成功进入 RUNNING 状态 | `tuya_ai_agent.c`：`__ai_client_run_evt()` |
| `AT_NETWORK_FAIL` | 会话创建失败（连接异常） | `tuya_ai_agent.c`：`tuya_ai_agent_crt_session()` |
| `AT_NETWORK_CFG` | 设备状态变更为未注册 | `tuya_ai_agent.c`：`__ai_devos_state_evt()` |
| `AT_PLEASE_AGAIN` | ASR 识别结果为空 | `tuya_ai_agent.c`：`__ai_parse_asr()` |

```c
// tuya_ai_agent.c：959-968 行
STATIC OPERATE_RET __ai_client_run_evt(VOID_T *data)
{
    return tuya_ai_output_alert(AT_NETWORK_CONNECTED);  // 通知网络已连接
}

STATIC OPERATE_RET __ai_devos_state_evt(VOID *data)
{
    DEVOS_STATE_E state = (DEVOS_STATE_E)data;
    if (state == DEVOS_STATE_UNREGISTERED) {
        tuya_ai_output_alert(AT_NETWORK_CFG);  // 提示用户去配网
    }
    return OPRT_OK;
}
```

### 8.3 三种播放实现方式

#### 方式一：本地音频（离线可用，最可靠）

```c
// ai_audio_player.c（宏：AI_PLAYER_ALERT_SOURCE_LOCAL == 1）
case AI_AUDIO_ALERT_NETWORK_FAIL:
    audio_data = (uint8_t*)LOCAL_ALERT_SRC_NET_FAILED;
    audio_size = sizeof(LOCAL_ALERT_SRC_NET_FAILED);
    break;
case AI_AUDIO_ALERT_NETWORK_DISCONNECT:
    audio_data = (uint8_t*)LOCAL_ALERT_SRC_NET_DISCONNECT;
    audio_size = sizeof(LOCAL_ALERT_SRC_NET_DISCONNECT);
    break;
case AI_AUDIO_ALERT_NETWORK_CONNECTED:
    audio_data = (uint8_t*)LOCAL_ALERT_SRC_NET_CONNECTED;
    audio_size = sizeof(LOCAL_ALERT_SRC_NET_CONNECTED);
    break;
```

> ✅ **推荐用于网络断开场景**：网络已断开时无法请求云端 TTS，本地音频是唯一可靠的提示方式。

#### 方式二：云端 TTS（丰富表达，需网络）

```c
// ai_audio_player.c（宏：AI_AGENT_ENABLE_CLOUD_ALERT == 1）
OPERATE_RET __player_cloud_alert(AI_AUDIO_ALERT_TYPE_E type)
{
    rt = ai_agent_cloud_alert(type);
    if (rt != OPRT_OK) {
        // 云端 TTS 失败时降级为本地叮咚音
        TUYA_CALL_ERR_LOG(ai_audio_play_data(AI_AUDIO_CODEC_MP3,
                          (uint8_t*)media_src_dingdong,
                          sizeof(media_src_dingdong)));
    }
    return rt;
}
```

> ⚠️ **注意**：云端 TTS 提示本身依赖网络，对于网络断开场景会自动降级为叮咚音。

#### 方式三：自定义回调（灵活扩展）

```c
// ai_audio_player.c（宏：AI_PLAYER_ALERT_SOURCE_CUSTOM == 1）
if (__s_alert_custom_cb) {
    rt = __s_alert_custom_cb(type);  // 完全由应用层控制
}
```

### 8.4 应用层事件流转

```
tuya_ai_output_alert(AT_NETWORK_FAIL)
    └── ai_output_ctx.cbs.alert_cb(type)            // svc_ai_agent 层回调
            └── ai_agent.c: __ai_agent_alert_cb()   // ai_components 层
                    └── ai_user_event_notify(AI_USER_EVT_PLAY_ALERT, type)
                            └── ai_chat_main.c: AI_USER_EVT_PLAY_ALERT case
                                    └── ai_audio_player_alert(type)
                                            ├── __player_local_alert()   // 本地音频
                                            ├── __player_cloud_alert()   // 云端 TTS
                                            └── __s_alert_custom_cb()    // 自定义
```

---

## 9. 会话中断处理

### 9.1 对话中断事件

网络抖动可能导致上行音频部分丢失，云端识别结果不完整时发出 `AI_EVENT_CHAT_BREAK`：

```c
// tuya_ai_agent.c：140-157 行
STATIC OPERATE_RET __ai_event_cb(AI_EVENT_ATTR_T *event, AI_EVENT_HEAD_T *head, VOID *data)
{
    if ((head->type == AI_EVENT_CHAT_BREAK) || (head->type == AI_EVENT_SERVER_VAD)) {
        if (head->type == AI_EVENT_CHAT_BREAK && event->user_data && event->user_len > 0) {
            // 基于时间戳去重，避免重复中断
            char intr_time[INTTERUPT_TIME_MAX] = {0};
            OPERATE_RET rt = __parse_attr_time(event->user_data, event->user_len, intr_time);
            if (OPRT_OK == rt) {
                if ((ai_agent_ctx.last_intr_time[0] != '\0') &&
                    (strcmp(intr_time, ai_agent_ctx.last_intr_time) <= 0)) {
                    PR_DEBUG("Interrupt event ignored");
                    return OPRT_OK;  // 忽略重复或过时的中断事件
                }
                strncpy(ai_agent_ctx.last_intr_time, intr_time, INTTERUPT_TIME_MAX);
            }
        }
        tuya_ai_output_event(head->type, 0, event->event_id);  // 通知应用层打断播放
    }
    return OPRT_OK;
}
```

**去重机制意义**：网络抖动时可能收到多个重复的中断包，去重保护避免应用层被反复打断，改善用户体验。

### 9.2 ASR 识别失败提示

```c
// tuya_ai_agent.c（__ai_parse_asr 函数，约 418 行）
if (scode_len == 0) {
    tuya_ai_output_alert(AT_PLEASE_AGAIN);  // 识别结果为空，提示重说
}
```

> 当网络不稳导致音频数据丢包，服务端返回空 ASR 结果时，设备会播放"请重说"提示，引导用户重新操作。

---

## 10. 音频缓冲保护

> **核心文件**：`src/tuya_ai_service/svc_ai_agent/src/tuya_ai_input.c`

网络不稳时，上行音频发送速度低于采集速度，缓冲区保护防止内存溢出：

```c
// tuya_ai_input.c（定义）
#ifndef AI_INPUT_RINGBUF_SIZE
#define AI_INPUT_RINGBUF_SIZE (20 * 1024)  // 环形缓冲区：20KB
#endif
#ifndef AI_INPUT_BUF_SIZE
#define AI_INPUT_BUF_SIZE     (6 * 1024)   // 单次写入上限：6KB
#endif
```

```c
// tuya_ai_input.c（写入逻辑）
uint32_t free_size = tuya_ring_buff_free_size_get(ai_input_ctx.ringbuf);
if (free_size < (SIZEOF(AI_RINGBUF_HEAD_T) + head->len)) {
    tal_mutex_unlock(ai_input_ctx.mutex);
    return OPRT_RESOURCE_NOT_READY;  // 缓冲区满，丢弃当前帧（静默丢帧）
}
```

```c
// 写入失败的降级处理
EXIT:
    tuya_ring_buff_reset(ai_input_ctx.ringbuf);     // 重置缓冲区
    ai_input_ctx.state = AI_INPUT_STOP;             // 停止输入
    tal_mutex_unlock(ai_input_ctx.mutex);
    return OPRT_COM_ERROR;
```

**保护策略总结**：

| 情况 | 处理方式 | 用户感知 |
|------|----------|----------|
| 缓冲区满 | 静默丢弃当前帧 | 可能轻微失真，一般不感知 |
| 写入异常 | 重置缓冲区并停止 | 当次对话终止，可重新唤醒 |
| 数据长度超限 | 拒绝写入并报错 | 数据包异常，不影响正常使用 |

---

## 11. 客制化扩展指南

### 11.1 注册自定义网络状态提示

```c
// 在应用初始化时注册自定义 alert 回调
ai_audio_player_reg_alert_cb(my_alert_handler);

// 自定义处理函数
OPERATE_RET my_alert_handler(AI_AUDIO_ALERT_TYPE_E type)
{
    switch (type) {
    case AI_AUDIO_ALERT_NETWORK_FAIL:
        // 网络连接失败：亮红灯 + 播放本地提示音
        led_set_color(LED_RED);
        play_local_audio("net_fail.mp3");
        break;
    case AI_AUDIO_ALERT_NETWORK_CONNECTED:
        // 网络连接成功：亮绿灯
        led_set_color(LED_GREEN);
        break;
    case AI_AUDIO_ALERT_NETWORK_DISCONNECT:
        // 网络断开：亮黄灯
        led_set_color(LED_YELLOW);
        break;
    default:
        break;
    }
    return OPRT_OK;
}
```

### 11.2 调整心跳参数

在编译时通过宏定义修改（`CMakeLists.txt` 或 `app_default.config`）：

```cmake
# 缩短心跳间隔（用于网络质量较差的环境）
target_compile_definitions(app PRIVATE
    AI_HEARTBEAT_INTERVAL=60   # 60 秒发一次 ping（默认 120）
    AT_PING_TIMEOUT=10         # 10 秒等待 pong（默认 6）
)
```

### 11.3 监听网络相关事件

```c
// 订阅 AI 客户端连接/断开事件
ty_subscribe_event(EVENT_AI_CLIENT_RUN,   "my_app", on_ai_connected,    SUBSCRIBE_TYPE_NORMAL);
ty_subscribe_event(EVENT_AI_CLIENT_CLOSE, "my_app", on_ai_disconnected, SUBSCRIBE_TYPE_NORMAL);

OPERATE_RET on_ai_connected(VOID_T *data) {
    PR_NOTICE("AI service connected");
    // 可在此更新 UI、发送状态上报等
    return OPRT_OK;
}

OPERATE_RET on_ai_disconnected(VOID_T *data) {
    PR_NOTICE("AI service disconnected");
    // 可在此停止录音、更新显示等
    return OPRT_OK;
}
```

---

## 12. 总结与建议

### 12.1 机制汇总

| 场景 | 处理机制 | 恢复策略 | 用户感知 |
|------|----------|----------|----------|
| **网络断开** | IDLE 状态轮询等待 | 网络恢复后自动重连，无需干预 | 可通过 `AT_NETWORK_DISCONNECT` 提示 |
| **网络抖动（SETUP 失败）** | 7 级随机化指数退避 | 逐步拉长重试间隔，连接成功后归零 | 重连期间播放"网络连接中"提示 |
| **运行中 TCP 断链** | 容忍 3 次读取失败后关闭连接 | 回到 IDLE 重新建立连接 | 当次对话可能中断 |
| **心跳超时（TCP 假活）** | 连续 3 次 ping 无响应后强制断开 | 触发重连流程 | 无感知重连（若网络快速恢复） |
| **Token 到期** | 提前 10 秒主动刷新 | 后台续期，用户无感知 | 完全透明 |
| **ASR 识别失败** | 播放"请重说"提示 | 等待用户重新唤醒 | 听到"请重说"后重试 |
| **长时间空闲** | 30 分钟空闲检测 | 断开后用户下次唤醒时重连 | 首次唤醒可能略有延迟 |

### 12.2 用户体验优化建议

1. **使用本地音频作为网络提示的首选方式**  
   网络断开时云端 TTS 不可用，内置本地音频文件（`AI_PLAYER_ALERT_SOURCE_LOCAL`）可确保用户始终收到反馈。

2. **结合硬件状态指示（LED/屏幕）**  
   通过 `EVENT_AI_CLIENT_RUN` / `EVENT_AI_CLIENT_CLOSE` 事件驱动 LED 或显示屏状态更新，提供视觉反馈，降低用户困惑。

3. **弱网环境下考虑降低心跳间隔**  
   将 `AI_HEARTBEAT_INTERVAL` 从默认 120 秒缩短至 60 秒，可更快检测到"假活"连接，减少用户等待时间。

4. **避免在断网期间持续触发语音唤醒**  
   应用层可检测 `tuya_ai_client_is_ready()` 返回值，若返回 `FALSE` 则播放本地提示"网络连接中，请稍候"，避免用户反复唤醒无响应。

5. **为 `AT_NETWORK_FAIL` 和 `AT_NETWORK_DISCONNECT` 设置不同提示**  
   `FAIL` 表示"尝试中"，`DISCONNECT` 表示"已断开"，两者对用户的含义不同，应使用差异化的提示语或音效。

---

*本文档基于 TuyaOpen 源码分析，核心文件：*  
- `src/tuya_ai_service/svc_ai_basic/src/tuya_ai_client.c`  
- `src/tuya_ai_service/svc_ai_agent/src/tuya_ai_agent.c`  
- `src/tuya_ai_service/svc_ai_agent/src/tuya_ai_output.c`  
- `src/tuya_ai_service/svc_ai_agent/src/tuya_ai_input.c`  
- `apps/tuya.ai/ai_components/ai_agent/src/ai_agent.c`  
- `apps/tuya.ai/ai_components/ai_audio/src/ai_audio_player.c`

---

## 修訂記錄

| 版本 | 日期 | 說明 |
|------|------|------|
| v1.0 | 2026-03-10 | 初版：基於 TuyaOpen 源碼調查在線語音控制體驗，建立網路韌性分析文檔 |
| v1.1 | 2026-03-12 | 複審更新：確認所有機制描述與最新源碼一致，補充修訂記錄 |
