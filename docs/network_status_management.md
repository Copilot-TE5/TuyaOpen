# TuyaOpen 网络状态统整机制详解

本文档深入分析 TuyaOpen SDK 如何统一管理多种网络连接的状态，涵盖架构设计、核心数据结构、状态机流程以及应用层集成方式，并附带完整代码示例。

---

## 目录

1. [整体架构](#1-整体架构)
2. [核心数据结构与状态定义](#2-核心数据结构与状态定义)
3. [网络管理器初始化](#3-网络管理器初始化)
4. [连接注册与优先级管理](#4-连接注册与优先级管理)
5. [状态事件回调机制](#5-状态事件回调机制)
6. [各连接类型的内部状态机](#6-各连接类型的内部状态机)
7. [事件发布与应用层订阅](#7-事件发布与应用层订阅)
8. [应用层查询与配置接口](#8-应用层查询与配置接口)
9. [完整数据流程图](#9-完整数据流程图)
10. [关键文件索引](#10-关键文件索引)

---

## 1. 整体架构

TuyaOpen 采用**分层、事件驱动**的架构来统整网络状态：

```
┌─────────────────────────────────────────────┐
│            应用层 / IoT 云服务               │
│  tuya_iot.c — 订阅网络事件，触发云端重连     │
└───────────────────┬─────────────────────────┘
                    │  事件发布订阅（tal_event）
┌───────────────────▼─────────────────────────┐
│          网络管理器 (netmgr)                 │
│  src/tuya_cloud_service/netmgr/netmgr.c     │
│  • 维护全局唯一网络状态                      │
│  • 优先级仲裁，选出活跃连接                  │
│  • 发布 EVENT_LINK_STATUS_CHG / TYPE_CHG    │
└──────┬────────────┬──────────────┬──────────┘
       │            │              │
┌──────▼──┐   ┌────▼─────┐  ┌────▼──────┐
│  WiFi   │   │  Wired   │  │ Cellular  │
│netconn_ │   │ netconn_ │  │ netconn_  │
│wifi.c   │   │ wired.c  │  │cellular.c │
│  pri=1  │   │  pri=2   │  │  pri=0    │
└──────┬──┘   └────┬─────┘  └────┬──────┘
       │            │              │
┌──────▼────────────▼──────────────▼──────────┐
│              TAL 网络抽象层                   │
│  tal_wifi / tal_wired / tal_cellular         │
│  src/tal_network/                           │
└─────────────────────────────────────────────┘
```

核心设计原则：
- **单一真相来源**：全局 `s_netmgr` 结构体持有当前的活跃连接类型与链路状态。
- **优先级仲裁**：多种连接并存时，数值最小的 `pri` 字段对应最高优先级。
- **统一回调接口**：所有底层连接均向上汇报同一个回调 `__netmgr_event_cb`。
- **发布-订阅解耦**：状态变更通过 `tal_event` 系统向上层发布，应用无需轮询。

---

## 2. 核心数据结构与状态定义

### 2.1 网络连接类型枚举

```c
// 文件: src/tuya_cloud_service/netmgr/netmgr.h

/** 网络连接类型，支持按位组合 */
typedef enum {
    NETCONN_AUTO     = 1 << 0,  // 自动选择（使用当前活跃连接）
    NETCONN_WIFI     = 1 << 1,  // Wi-Fi 无线连接
    NETCONN_WIRED    = 1 << 2,  // 有线以太网
    NETCONN_CELLULAR = 1 << 3,  // 蜂窝网络（4G/LTE）
} netmgr_type_e;

/** 类型转字符串宏，便于日志打印 */
#define NETMGR_TYPE_TO_STR(type)                             \
    ((type) == NETCONN_WIFI       ? "wifi"                   \
     : (type) == NETCONN_WIRED    ? "wired"                  \
     : (type) == NETCONN_CELLULAR ? "cellular"               \
     : (type) == NETCONN_AUTO     ? "auto"                   \
                                  : "unknown")
```

### 2.2 网络链路状态枚举

```c
// 文件: src/tuya_cloud_service/netmgr/netmgr.h

/** 网络链路状态 */
typedef enum {
    NETMGR_LINK_DOWN,      // 网络已断开
    NETMGR_LINK_UP,        // 网络已连接
    NETMGR_LINK_UP_SWITH,  // 网络已连接，但活跃连接类型发生切换
} netmgr_status_e;

/** 状态转字符串宏 */
#define NETMGR_STATUS_TO_STR(status)                                \
    ((status) == NETMGR_LINK_DOWN       ? "link_down"               \
     : (status) == NETMGR_LINK_UP       ? "link_up"                 \
     : (status) == NETMGR_LINK_UP_SWITH ? "link_up_switch"          \
                                        : "unknown")
```

### 2.3 连接配置命令类型

```c
// 文件: src/tuya_cloud_service/netmgr/netmgr.h

/** 用于 netmgr_conn_set / netmgr_conn_get 的命令类型 */
typedef enum {
    NETCONN_CMD_PRI,           // 读/写 优先级 (int)
    NETCONN_CMD_IP,            // 读/写 IP 配置 (NW_IP_S)
    NETCONN_CMD_MAC,           // 读/写 MAC 地址 (NW_MAC_S)
    NETCONN_CMD_STATUS,        // 读   当前链路状态 (netmgr_status_e)
    NETCONN_CMD_SSID_PSWD,     // 写   WiFi SSID 与密码 (netconn_wifi_info_t)
    NETCONN_CMD_COUNTRYCODE,   // 写   国家代码字符串，如 "CN"/"US"/"EU"
    NETCONN_CMD_NETCFG,        // 写   网络配置参数 (netconn_wifi_netcfg_t)
    NETCONN_CMD_SET_STATUS_CB, // 写   自定义状态回调（替换默认行为）
    NETCONN_CMD_CLOSE,         // 写   关闭连接
    NETCONN_CMD_RESET,         // 写   重置连接
} netmgr_conn_config_type_e;
```

### 2.4 连接基础结构（多态接口）

所有连接类型（WiFi、有线、蜂窝）都将 `netmgr_conn_base_t` 作为结构体的**第一个成员**，从而实现 C 语言多态。

```c
// 文件: src/tuya_cloud_service/netmgr/netmgr.h

/**
 * @brief 网络连接基础接口，所有具体连接类型的公共部分
 */
typedef struct netmgr_conn_base {
    uint8_t pri;                    // 优先级（数值越小，优先级越高）
    netmgr_type_e type;             // 连接类型
    netmgr_status_e status;         // 当前链路状态
    TAL_NETWORK_CARD_TYPE_E card_type; // 底层网络卡类型

    /* 虚函数表：由各具体连接类型实现 */
    OPERATE_RET (*open)(void *config);
    OPERATE_RET (*close)(void);
    OPERATE_RET (*set)(netmgr_conn_config_type_e cmd, void *param);
    OPERATE_RET (*get)(netmgr_conn_config_type_e cmd, void *param);
    void (*event_cb)(netmgr_type_e type, netmgr_status_e event); // 注入的统一回调

    struct netmgr_conn_base *next;  // 链表指针，用于按优先级排序
} netmgr_conn_base_t;
```

### 2.5 全局网络管理器状态

```c
// 文件: src/tuya_cloud_service/netmgr/netmgr.c

/** 全局网络管理器状态（单例） */
typedef struct {
    MUTEX_HANDLE lock;        // 互斥锁，保护并发访问
    BOOL_T inited;            // 是否已初始化

    netmgr_type_e type;       // 本次初始化时启用的网络类型集合
    netmgr_type_e active;     // 当前正在使用的活跃连接类型
    netmgr_status_e status;   // 当前全局链路状态

    netmgr_conn_base_t *conn; // 按优先级排序的连接链表头指针
} netmgr_t;

static netmgr_t s_netmgr = {0};  // 唯一实例
```

---

## 3. 网络管理器初始化

`netmgr_init()` 是整个系统的入口，负责初始化各连接类型并构建优先级链表。

```c
// 文件: src/tuya_cloud_service/netmgr/netmgr.c

OPERATE_RET netmgr_init(netmgr_type_e type)
{
    OPERATE_RET rt = OPRT_OK;

    // 1. 初始化底层网络卡驱动
    TUYA_CALL_ERR_RETURN(tal_network_card_init());

    // 2. 创建互斥锁，设置初始状态
    TUYA_CALL_ERR_RETURN(tal_mutex_create_init(&s_netmgr.lock));
    s_netmgr.status = NETMGR_LINK_DOWN;
    s_netmgr.type = type;

    // 3. 根据编译宏按需注册各连接类型
    //    注：注册顺序不影响最终优先级，链表插入时会按 pri 排序

#ifdef ENABLE_WIRED
    if (type & NETCONN_WIRED) {
        __netmgr_conn_register(NETCONN_WIRED, (netmgr_conn_base_t *)&s_netmgr_wired);
    }
#endif

#ifdef ENABLE_CELLULAR
    if (type & NETCONN_CELLULAR) {
        __netmgr_conn_register(NETCONN_CELLULAR, (netmgr_conn_base_t *)&s_netmgr_cellular);
    }
#endif

#ifdef ENABLE_WIFI
    if (type & NETCONN_WIFI) {
        __netmgr_conn_register(NETCONN_WIFI, (netmgr_conn_base_t *)&s_netmgr_wifi);
    }
#endif

    // 4. 获取当前活跃连接
    s_netmgr.active = __get_active_conn();
    if (s_netmgr.active == NETCONN_AUTO) {
        PR_ERR("No connection available, please check your configuration");
        return OPRT_INVALID_PARM;
    }

    s_netmgr.inited = TRUE;

    // 5. 启动 LAN 初始化定时器（蜂窝网络不需要 LAN）
#if !defined(ENABLE_CELLULAR) || (ENABLE_CELLULAR == 0)
    tal_sw_timer_create(__tuya_lan_init_tm_cb, NULL, &sg_lan_init_timer);
    tal_sw_timer_start(sg_lan_init_timer, 500, TAL_TIMER_CYCLE);
#endif

    return rt;
}
```

**典型调用示例（应用层）：**

```c
// 同时启用 WiFi 和有线网络
netmgr_init(NETCONN_WIFI | NETCONN_WIRED);

// 仅启用 WiFi
netmgr_init(NETCONN_WIFI);

// 仅启用蜂窝网络
netmgr_init(NETCONN_CELLULAR);
```

---

## 4. 连接注册与优先级管理

### 4.1 优先级配置

各连接类型在其静态实例中预设 `pri` 字段。注册时，`__netmgr_conn_register()` 使用条件 `cur_conn->pri < conn->pri` 进行排序，这会创建一个**降序链表**（`pri` 值较大的节点排在链表前端），而 `__get_active_conn()` 返回链表中**第一个**处于 UP 状态的节点。因此，**`pri` 值越大，有效优先级越高**：

| 连接类型 | `pri` 值 | 有效优先级 | 说明 |
|---------|---------|---------|------|
| `NETCONN_WIRED` | `2` | **最高**（链表首位，优先选用） | 有线以太网稳定且免费，优先使用 |
| `NETCONN_WIFI` | `1` | 中等 | WiFi 次选 |
| `NETCONN_CELLULAR` | `0` | 最低（链表末位，兜底使用） | 蜂窝流量费用高，作为最后备用 |

```c
// 文件: src/tuya_cloud_service/netmgr/netconn_wifi.c
netmgr_conn_wifi_t s_netmgr_wifi = {
    .base = {
        .pri = 1,                    // WiFi 优先级
        .type = NETCONN_WIFI,
        .status = NETMGR_LINK_DOWN,
        .card_type = TAL_NET_TYPE_PLATFORM,
        .open  = netconn_wifi_open,
        .close = netconn_wifi_close,
        .get   = netconn_wifi_get,
        .set   = netconn_wifi_set,
    },
    .ccode = {"CN"},
    .conn = {
        .table_size = NETCONN_WIFI_CONN_TABLE,
        .table = {1, 3, 5, 10, 15, 20}, // 重连延迟表（秒）
    },
};

// 文件: src/tuya_cloud_service/netmgr/netconn_wired.c
netmgr_conn_wired_t s_netmgr_wired = {
    .base = {
        .pri = 2,                    // 有线优先级
        .type = NETCONN_WIRED,
        .status = NETMGR_LINK_DOWN,
        .card_type = TAL_NET_TYPE_POSIX,
        .open  = netconn_wired_open,
        .close = netconn_wired_close,
        .get   = netconn_wired_get,
        .set   = netconn_wired_set,
    },
};
```

### 4.2 链表插入算法

`__netmgr_conn_register()` 将新连接按优先级插入到有序单链表中：

```c
// 文件: src/tuya_cloud_service/netmgr/netmgr.c

OPERATE_RET __netmgr_conn_register(netmgr_type_e type, netmgr_conn_base_t *conn)
{
    OPERATE_RET rt = OPRT_OK;

    if (NULL == conn) {
        return OPRT_INVALID_PARM;
    }

    // 注入统一的上报回调
    conn->event_cb = __netmgr_event_cb;

    // 防止重复注册
    netmgr_conn_base_t *cur_conn = s_netmgr.conn;
    while (cur_conn) {
        if (type == cur_conn->type) {
            PR_DEBUG("netmgr [%s] already registered", NETMGR_TYPE_TO_STR(type));
            return OPRT_INVALID_PARM;
        }
        cur_conn = cur_conn->next;
    }

    // 空链表，直接插入
    if (NULL == s_netmgr.conn) {
        s_netmgr.conn = conn;
        conn->next = NULL;
        goto __EXIT;
    }

    // 遍历链表，找到第一个 pri 值小于新节点的位置并将新节点插入其前
    // 算法以降序排列链表（pri 大的在前）: wired(2) → wifi(1) → cellular(0)
    // 有效优先级: wired(最高) → wifi → cellular(最低，兜底)
    netmgr_conn_base_t *prev_conn = NULL;
    cur_conn = s_netmgr.conn;
    while (cur_conn) {
        if (cur_conn->pri < conn->pri) {
            if (prev_conn == NULL) {
                s_netmgr.conn = conn;   // 插入链表头
                conn->next = cur_conn;
            } else {
                prev_conn->next = conn; // 插入链表中间
                conn->next = cur_conn;
            }
            break;
        }
        prev_conn = cur_conn;
        cur_conn = cur_conn->next;
    }

    // 未找到更低优先级的节点，追加到链表尾部
    if (cur_conn == NULL && prev_conn != NULL) {
        prev_conn->next = conn;
        conn->next = NULL;
    }

__EXIT:
    // 调用连接的 open 回调进行底层初始化
    if (NULL != conn->open) {
        rt = conn->open(NULL);
    }
    return rt;
}
```

### 4.3 活跃连接仲裁

每当网络状态变化时，通过遍历有序链表，返回**第一个状态为 `NETMGR_LINK_UP` 的连接**（即优先级最高且已连接的）：

```c
// 文件: src/tuya_cloud_service/netmgr/netmgr.c

static netmgr_type_e __get_active_conn()
{
    netmgr_type_e active_type = NETCONN_AUTO;
    netmgr_conn_base_t *cur_conn = s_netmgr.conn;

    if (NULL == cur_conn) {
        PR_ERR("no connection registered");
        return NETCONN_AUTO;
    }

    // 默认使用链表头（优先级最高的连接）
    active_type = cur_conn->type;

    while (cur_conn) {
        netmgr_status_e netmgr_status = NETMGR_LINK_DOWN;
        cur_conn->get(NETCONN_CMD_STATUS, &netmgr_status);

        if (netmgr_status == NETMGR_LINK_UP) {
            // 返回最高优先级且已连接的连接
            PR_TRACE("netmgr active connection [%s]", NETMGR_TYPE_TO_STR(cur_conn->type));
            active_type = cur_conn->type;
            break;
        }
        cur_conn = cur_conn->next;
    }

    return active_type;
}
```

---

## 5. 状态事件回调机制

`__netmgr_event_cb` 是所有底层连接的统一上报入口，负责对比新旧状态并决定发布哪些事件：

```c
// 文件: src/tuya_cloud_service/netmgr/netmgr.c

/**
 * @brief 统一的网络事件回调（由各底层连接在状态变化时调用）
 *
 * @param type   发生变化的连接类型
 * @param status 该连接的新状态（本函数内不直接使用，而是重新查询活跃连接）
 */
static void __netmgr_event_cb(netmgr_type_e type, netmgr_status_e status)
{
    // 参数 status 仅供调试，实际以重新查询的结果为准
    (void)status;

    // 只处理当前已启用的连接类型上报的事件
    if (!(s_netmgr.type & type)) {
        return;
    }

    // 1. 重新仲裁：查询当前应该使用哪个连接及其状态
    netmgr_type_e active_conn = __get_active_conn();
    netmgr_status_e active_status = NETMGR_LINK_DOWN;
    __get_netmgr_status(active_conn, &active_status);

    // 2. 对比旧状态，按需发布事件
    if (active_status != s_netmgr.status && active_conn != s_netmgr.active) {
        // 场景 A：链路状态 + 活跃连接类型 同时改变
        PR_DEBUG("conn type changed [%s]->[%s], status %d->%d",
                 NETMGR_TYPE_TO_STR(s_netmgr.active), NETMGR_TYPE_TO_STR(active_conn),
                 s_netmgr.status, active_status);

        s_netmgr.status = active_status;
        s_netmgr.active = active_conn;

        // 切换底层网络卡
        netmgr_conn_base_t *p_conn = __get_conn_by_type(active_conn);
        tal_network_card_set_active(p_conn->card_type);

        // 同时发布两个事件
        tal_event_publish(EVENT_LINK_TYPE_CHG,   (void *)&s_netmgr.active);
        tal_event_publish(EVENT_LINK_STATUS_CHG, (void *)&s_netmgr.status);

    } else if (active_status != s_netmgr.status) {
        // 场景 B：仅链路状态改变（如 WiFi 断开后重连，连接类型未变）
        PR_DEBUG("status changed [%s]->[%s]",
                 NETMGR_STATUS_TO_STR(s_netmgr.status),
                 NETMGR_STATUS_TO_STR(active_status));

        s_netmgr.status = active_status;
        tal_event_publish(EVENT_LINK_STATUS_CHG, (void *)&s_netmgr.status);

    } else if (active_conn != s_netmgr.active) {
        // 场景 C：仅活跃连接类型改变（状态均为 UP，但切换了连接介质）
        PR_DEBUG("conn type changed [%s]->[%s]",
                 NETMGR_TYPE_TO_STR(s_netmgr.active),
                 NETMGR_TYPE_TO_STR(active_conn));

        s_netmgr.active = active_conn;

        // 切换底层网络卡
        netmgr_conn_base_t *p_conn = __get_conn_by_type(active_conn);
        tal_network_card_set_active(p_conn->card_type);

        tal_event_publish(EVENT_LINK_TYPE_CHG, (void *)&s_netmgr.active);
    }
    // 场景 D：状态和连接均未改变，不发布任何事件（避免冗余通知）
}
```

---

## 6. 各连接类型的内部状态机

### 6.1 WiFi 连接状态机

WiFi 连接实现了带指数退避的自动重连机制：

```c
// 文件: src/tuya_cloud_service/netmgr/netconn_wifi.h

/** WiFi 重连阶段状态 */
typedef enum {
    NETCONN_WIFI_CONN_REDAY,   // 初始/准备状态，可以发起连接
    NETCONN_WIFI_CONN_CHECK,   // 已发起连接，等待结果（有超时保护）
    NETCONN_WIFI_CONN_LINKUP,  // 连接成功，链路已建立
    NETCONN_WIFI_CONN_WAIT,    // 连接失败，等待重连延迟计时
    NETCONN_WIFI_CONN_STOP,    // 主动断开，停止自动重连
} netconn_wifi_conn_status_e;

/** WiFi 连接事件 */
typedef enum {
    NETCONN_NETCFG_START = 1,  // 开始网络配置
    NETCONN_NETCFG_DONE,       // 网络配置完成
    NETCONN_NETCFG_TIMEOUT,    // 网络配置超时
    NETCONN_WIFI_CONN_START,   // 开始连接
    NETCONN_WIFI_CONN,         // 连接成功
    NETCONN_WIFI_CONN_FAILED,  // 连接失败
    NETCONN_WIFI_DISCONN,      // 断开连接
} netconn_wifi_event_e;
```

**重连延迟表**（秒）：

```c
// 文件: src/tuya_cloud_service/netmgr/netconn_wifi.c

netmgr_conn_wifi_t s_netmgr_wifi = {
    .conn = {
        .table_size = NETCONN_WIFI_CONN_TABLE,  // 6 个阶梯
        .table = {1, 3, 5, 10, 15, 20},         // 每次失败后延迟递增（秒）
    },
};
```

**底层 WiFi 事件处理**（负责更新状态并向上通知）：

```c
// 文件: src/tuya_cloud_service/netmgr/netconn_wifi.c

static void __netconn_wifi_event(WF_EVENT_E event, void *arg)
{
    netmgr_conn_wifi_t *wifi = &s_netmgr_wifi;

    tal_sw_timer_stop(wifi->conn.timer);

    if (event == WFE_CONNECTED) {
        // WiFi 连接成功
        wifi->conn.count = 0;
        wifi->conn.stat = NETCONN_WIFI_CONN_LINKUP;
        wifi->base.status = NETMGR_LINK_UP;

    } else {
        // WiFi 连接失败或断开，触发自动重连逻辑
        if (NETCONN_WIFI_CONN_CHECK == wifi->conn.stat ||
            NETCONN_WIFI_CONN_WAIT  == wifi->conn.stat) {
            // 按延迟表递增等待时间后重连
            tal_sw_timer_start(wifi->conn.timer,
                               wifi->conn.table[wifi->conn.count] * 1000,
                               TAL_TIMER_ONCE);
            if (wifi->conn.count < wifi->conn.table_size - 1) {
                wifi->conn.count++;
            }
            wifi->conn.stat = NETCONN_WIFI_CONN_WAIT;

        } else if (NETCONN_WIFI_CONN_LINKUP == wifi->conn.stat) {
            // 从已连接状态断开，立即尝试重连
            wifi->conn.stat = NETCONN_WIFI_CONN_REDAY;
            __netconn_wifi_connect();
        }

        wifi->base.status = NETMGR_LINK_DOWN;
    }

    // 通知上层网络管理器
    if (wifi->base.event_cb) {
        wifi->base.event_cb(NETCONN_WIFI, wifi->base.status);
    }
}
```

**WiFi 重连状态机流程：**

```
[REDAY] ──发起连接──▶ [CHECK] ──超时/失败──▶ [WAIT] ──定时器到期──▶ [CHECK]
                         │                                              ▲
                     连接成功                                      (延迟递增循环)
                         │
                         ▼
                     [LINKUP] ──断开──▶ [REDAY] ──立即重连──▶ [CHECK]

[STOP]: 主动断开，不再自动重连
```

### 6.2 有线网络状态处理

有线网络无需重连机制，直接将底层物理链路状态映射为管理器状态：

```c
// 文件: src/tuya_cloud_service/netmgr/netconn_wired.c

static void __netconn_wired_event(WIRED_STAT_E event)
{
    netmgr_conn_wired_t *netmgr_wired = &s_netmgr_wired;

    PR_NOTICE("wired status changed to %d, old stat: %d",
              event, netmgr_wired->base.status);

    // 直接将物理链路状态映射到管理器状态
    netmgr_wired->base.status = (event == TKL_WIRED_LINK_UP)
                                 ? NETMGR_LINK_UP
                                 : NETMGR_LINK_DOWN;

    // 通知上层网络管理器
    if (netmgr_wired->base.event_cb) {
        netmgr_wired->base.event_cb(NETCONN_WIRED, netmgr_wired->base.status);
    }
}

OPERATE_RET netconn_wired_open(void *config)
{
    OPERATE_RET rt = OPRT_OK;
    netmgr_conn_wired_t *netmgr_wired = &s_netmgr_wired;

    netmgr_wired->base.status = NETMGR_LINK_DOWN;
    // 注册底层有线网络状态变化回调
    TUYA_CALL_ERR_RETURN(tal_wired_set_status_cb(__netconn_wired_event));

    return rt;
}
```

---

## 7. 事件发布与应用层订阅

### 7.1 事件定义

```c
// 文件: src/tal_system/include/tal_event_info.h

/** 网络链路状态改变事件（UP/DOWN 切换）*/
#define EVENT_LINK_STATUS_CHG   "link.status"

/** 网络活跃连接类型改变事件（WiFi ↔ Wired 切换）*/
#define EVENT_LINK_TYPE_CHG     "link.type"

/** 设备激活信息就绪事件（首次配网时使用）*/
#define EVENT_LINK_ACTIVATE     "link.activate"
```

### 7.2 IoT 云服务层订阅示例

```c
// 文件: src/tuya_cloud_service/cloud/tuya_iot.c

/**
 * @brief 网络连接类型切换时的处理回调
 *
 * 当活跃连接从 WiFi 切换到有线（或反之）时被调用，
 * 需要重新建立 MQTT 云连接。
 */
static OPERATE_RET __tuya_iot_link_type_change_cb(void *data)
{
    OPERATE_RET rt = OPRT_OK;
    netmgr_type_e netmgr_type = (netmgr_type_e)data;

    PR_DEBUG("netmgr_type: %s", NETMGR_TYPE_TO_STR(netmgr_type));

    tuya_iot_client_t *p_client = tuya_iot_client_get();
    if (p_client) {
        PR_NOTICE("Tuya iot client reconnect");
        tuya_iot_reconnect(p_client);  // 触发云端重连
    }

    return rt;
}

/* --- 在 IoT 状态机 STATE_START 阶段订阅事件 --- */
case STATE_START:
    // 订阅连接类型切换事件（持久订阅）
    TUYA_CALL_ERR_LOG(
        tal_event_subscribe(EVENT_LINK_TYPE_CHG, "iot",
                            __tuya_iot_link_type_change_cb,
                            SUBSCRIBE_TYPE_NORMAL)
    );
    // 订阅激活事件（一次性订阅，收到后自动注销）
    tal_event_subscribe(EVENT_LINK_ACTIVATE, "iot",
                        tuya_iot_token_activate_evt,
                        SUBSCRIBE_TYPE_ONETIME);
    break;

/* --- 在 STATE_STOP 阶段注销订阅 --- */
case STATE_STOP:
    tal_event_unsubscribe(EVENT_LINK_TYPE_CHG, "iot",
                          __tuya_iot_link_type_change_cb);
    tal_event_unsubscribe(EVENT_LINK_ACTIVATE, "iot",
                          tuya_iot_token_activate_evt);
    break;
```

### 7.3 自定义应用订阅示例

开发者可以在自己的应用中监听网络状态事件：

```c
#include "tal_event.h"
#include "tal_event_info.h"
#include "netmgr.h"

/**
 * @brief 监听网络链路状态变化（上线/下线）
 */
static OPERATE_RET my_link_status_cb(void *data)
{
    netmgr_status_e status = *(netmgr_status_e *)data;

    if (status == NETMGR_LINK_UP) {
        // 网络已连接，可以开始业务逻辑
        PR_NOTICE("Network is UP, starting business logic...");
    } else {
        // 网络已断开，暂停依赖网络的操作
        PR_NOTICE("Network is DOWN, pausing network operations...");
    }

    return OPRT_OK;
}

/**
 * @brief 监听活跃连接类型切换
 */
static OPERATE_RET my_link_type_cb(void *data)
{
    netmgr_type_e type = (netmgr_type_e)data;
    PR_NOTICE("Active connection switched to: %s", NETMGR_TYPE_TO_STR(type));

    return OPRT_OK;
}

void my_app_init(void)
{
    // 订阅链路状态事件（持久）
    tal_event_subscribe(EVENT_LINK_STATUS_CHG, "my_app",
                        my_link_status_cb,
                        SUBSCRIBE_TYPE_NORMAL);

    // 订阅链路类型切换事件（持久）
    tal_event_subscribe(EVENT_LINK_TYPE_CHG, "my_app",
                        my_link_type_cb,
                        SUBSCRIBE_TYPE_NORMAL);
}
```

---

## 8. 应用层查询与配置接口

### 8.1 公开 API 汇总

```c
// 文件: src/tuya_cloud_service/netmgr/netmgr.h

/**
 * @brief 初始化网络管理器
 * @param type  要启用的网络连接类型（支持按位 OR 组合）
 * @return OPERATE_RET  OPRT_OK 表示成功
 */
OPERATE_RET netmgr_init(netmgr_type_e type);

/**
 * @brief 设置网络连接属性
 * @param type  目标连接类型（NETCONN_AUTO 使用当前活跃连接）
 * @param cmd   命令类型（netmgr_conn_config_type_e）
 * @param param 命令参数指针
 * @return OPERATE_RET
 */
OPERATE_RET netmgr_conn_set(netmgr_type_e type,
                             netmgr_conn_config_type_e cmd,
                             void *param);

/**
 * @brief 获取网络连接属性
 * @param type  目标连接类型（NETCONN_AUTO 使用当前活跃连接）
 * @param cmd   命令类型（netmgr_conn_config_type_e）
 * @param param 输出参数指针
 * @return OPERATE_RET
 */
OPERATE_RET netmgr_conn_get(netmgr_type_e type,
                             netmgr_conn_config_type_e cmd,
                             void *param);
```

### 8.2 常用使用场景

```c
// --- 查询当前网络状态 ---
netmgr_status_e status;
netmgr_conn_get(NETCONN_AUTO, NETCONN_CMD_STATUS, &status);
PR_INFO("Current status: %s", NETMGR_STATUS_TO_STR(status));

// --- 查询当前 IP 地址 ---
NW_IP_S ip_info;
netmgr_conn_get(NETCONN_AUTO, NETCONN_CMD_IP, &ip_info);
PR_INFO("IP: %s", ip_info.ip);

// --- 配置 WiFi 凭据并连接 ---
netconn_wifi_info_t wifi_info = {0};
strncpy(wifi_info.ssid, "MySSID",   sizeof(wifi_info.ssid) - 1);
strncpy(wifi_info.pswd, "MyPasswd", sizeof(wifi_info.pswd) - 1);
netmgr_conn_set(NETCONN_WIFI, NETCONN_CMD_SSID_PSWD, &wifi_info);

// --- 断开 WiFi 连接 ---
netmgr_conn_set(NETCONN_WIFI, NETCONN_CMD_CLOSE, NULL);

// --- 设置 WiFi 国家代码 ---
netmgr_conn_set(NETCONN_WIFI, NETCONN_CMD_COUNTRYCODE, "US");
```

---

## 9. 完整数据流程图

```
设备启动
    │
    ▼
netmgr_init(NETCONN_WIFI | NETCONN_WIRED)
    │
    ├── __netmgr_conn_register(WIRED, pri=2)
    │       └── netconn_wired_open()
    │               └── tal_wired_set_status_cb(__netconn_wired_event)
    │
    └── __netmgr_conn_register(WIFI, pri=1)
            └── netconn_wifi_open()
                    └── tal_wifi_init / 注册 __netconn_wifi_event
    │
    ▼
链表结构：wired(pri=2) → wifi(pri=1)
（pri 大的在链表前端；所有节点的 event_cb 均指向 __netmgr_event_cb）

    ─────── 运行时：WiFi 断开事件 ───────

底层驱动
    │ WFE_DISCONNECTED
    ▼
__netconn_wifi_event()
    │ 更新 wifi.base.status = NETMGR_LINK_DOWN
    │ 按延迟表启动重连定时器
    ▼
wifi.base.event_cb(NETCONN_WIFI, NETMGR_LINK_DOWN)
    │ (即 __netmgr_event_cb)
    ▼
__get_active_conn()
    │ 遍历链表：wifi=DOWN, wired=?
    │ 若 wired=UP  → 返回 NETCONN_WIRED
    │ 若 wired=DOWN → 返回 NETCONN_WIFI（默认链表头）
    ▼
比较新旧状态
    │
    ├── [active 从 WIFI → WIRED 且 status UP→UP]
    │       → tal_event_publish(EVENT_LINK_TYPE_CHG, NETCONN_WIRED)
    │
    ├── [active 不变，status UP → DOWN]
    │       → tal_event_publish(EVENT_LINK_STATUS_CHG, NETMGR_LINK_DOWN)
    │
    └── [active 从 WIFI → WIRED，status UP → DOWN]
            → tal_event_publish(EVENT_LINK_TYPE_CHG, ...)
            → tal_event_publish(EVENT_LINK_STATUS_CHG, ...)

    ▼
应用层订阅回调被触发
    ├── __tuya_iot_link_type_change_cb() → tuya_iot_reconnect()
    └── my_link_status_cb() → 业务逻辑处理
```

---

## 10. 关键文件索引

| 文件路径 | 说明 |
|---------|------|
| `src/tuya_cloud_service/netmgr/netmgr.h` | 网络管理器公开接口与核心数据结构定义 |
| `src/tuya_cloud_service/netmgr/netmgr.c` | 网络管理器实现（优先级仲裁、事件回调、状态机）|
| `src/tuya_cloud_service/netmgr/netconn_wifi.h` | WiFi 连接头文件（状态枚举、结构体定义）|
| `src/tuya_cloud_service/netmgr/netconn_wifi.c` | WiFi 连接实现（重连状态机、事件处理）|
| `src/tuya_cloud_service/netmgr/netconn_wired.h` | 有线网络连接头文件 |
| `src/tuya_cloud_service/netmgr/netconn_wired.c` | 有线网络连接实现 |
| `src/tuya_cloud_service/netmgr/netconn_cellular.h` | 蜂窝网络连接头文件 |
| `src/tuya_cloud_service/netmgr/netconn_cellular.c` | 蜂窝网络连接实现 |
| `src/tal_system/include/tal_event_info.h` | 事件名称常量定义 |
| `src/tal_network/include/tal_network_register.h` | 底层网络卡注册接口 |
| `src/tuya_cloud_service/cloud/tuya_iot.c` | IoT 云服务层（事件订阅示例）|
