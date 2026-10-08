# 煤层气智能井场 MQTT 通信协议（对外通用版）

**文档版本**：V1.2\
**编制日期**：2026-10-08\
**适用范围**：射流泵井场、电潜泵井场

> **版本记录**：
>
> - V1.0（2026-09-21）：首次发布。基于井场通用数据部分编制，供接入方开发对接使用。
> - V1.2（2026-09-23）：对外发布前修订。V1.0 尚未实际部署，V1.2 为首个实际发布版本。

***

## 一、概述

### 1.1 文档目的

本文档定义煤层气生产现场与智慧井场平台之间的 MQTT 通信协议规范，包括 Topic 结构、数据格式、通信机制等，用于指导接入方的软件开发和平台接口对接。

接入方须按照本文档定义的 Topic 结构、数据格式与通信机制与平台对接，未定义的扩展需求须通过第七章所述协作流程提出。平台侧连接信息（服务器地址、端口、认证方式、网关与井位标识分配等）由平台方在开通接入时另行提供。本文档覆盖井场通用数据上报与控制指令交互，平台其他扩展能力不在本文档范围内。

### 1.2 适用井场类型

| 井场类型  | 举升设备          | 设备位置    |
| ----- | ------------- | ------- |
| 射流泵井场 | 地面柱塞泵 + 井下射流泵 | 地面 + 井下 |
| 电潜泵井场 | 井下电潜泵         | 井下      |

### 1.3 通信特性

| 特性    | 说明                       |
| ----- | ------------------------ |
| 通信协议  | MQTT 5.0 / MQTT 3.1.1    |
| QoS等级 | QoS 1（至少送达一次）           |
| 时间戳格式 | UNIX 毫秒时间戳（字符串）         |
| 数据编码  | UTF-8 / JSON             |
| 上报机制  | 轮询上报，默认间隔1s（可配置）        |

***

## 二、Topic 结构设计

### 2.1 Topic 命名规范

```
/cbm/{direction}/{gw_id}/{wf_id}/{well_id}/{category}/{subcategory}
对于网关级别的全局数据（如心跳），可省略{wf_id}/{well_id}层级
对于井场级数据（如汇管压力），可省略{well_id}层级
```

| 层级            | 说明          | 示例                    |
| ------------- | ----------- | --------------------- |
| /cbm          | 根路径，煤层气项目标识 | /cbm                  |
| {direction}   | 数据流向        | up / down             |
| {gw\_id}      | 网关唯一标识      | gw001                 |
| {wf\_id}      | 井组唯一标识      | wf001                 |
| {well\_id}    | 单井唯一标识      | xxx                   |
| {category}    | 数据类别        | prod / ctrl / algo 等  |
| {subcategory} | 数据子类别       | plunger / esp 等    |

### 2.2 数据流向定义

| 方向    | Topic前缀                | 说明                 |
| ----- | ---------------------- | ------------------ |
| 网关→平台 | /cbm/up/{gw\_id}/...   | 生产数据上报、算法结果上报、指令响应 |
| 平台→网关 | /cbm/down/{gw\_id}/... | 控制指令下发、连锁指令下发、算法参数下发 |

### 2.3 完整 Topic 树状结构

```
/cbm
├── up/{gw_id}                          # 网关上报数据
│   ├── sys
│   │   └── heartbeat                           # 网关心跳
│   └── {wf_id}
│       ├── prod                                 # 井场级生产数据
│       │   └── pipe_pres                        # 汇管压力数据（井场级）
│       ├── ack                                  # 指令响应（井场级）
│       │   └── algo/video                          # 算法配置确认回复
│       ├── algo                                 # 算法结果上报（井场级）
│       │   └── video                            # 视频识别结果
│       └── {well_id}
│           ├── prod                            # 生产数据上报
│           │   ├── plunger                        # 柱塞泵数据
│           │   ├── esp                            # 电潜泵数据
│           │   ├── flow_large                     # 流量计数据（大流量）
│           │   ├── flow_small                     # 流量计数据（小流量）
│           │   ├── gas_valve                      # 放气阀门数据（气体）
│           │   ├── output_flow                    # 产水流量计数据
│           │   ├── reinject                       # 回注水系统数据
│           │   ├── casing_pres                    # 套管压力数据
│           │   └── dh                             # 井下数据
│           ├── ack                            # 控制指令响应
│           │   ├── plunger                        # 柱塞泵指令确认
│           │   ├── esp                            # 电潜泵指令确认
│           │   ├── reinject                       # 回注水指令确认
│           │   ├── rein_valve                     # 注入阀门指令确认
│           │   ├── seq_ack                        # 连锁指令响应
│           │   └── gas_valve                      # 放气阀指令确认
│           ├── result                         # 执行结果回复
│           │   ├── plunger                        # 柱塞泵执行结果
│           │   ├── esp                            # 电潜泵执行结果
│           │   ├── reinject                       # 回注水执行结果
│           │   ├── rein_valve                     # 注入阀门执行结果
│           │   ├── seq_res                        # 连锁指令执行结果
│           │   └── gas_valve                      # 放气阀执行结果
│
└── down/{gw_id}                         # 平台下发数据
    └── {wf_id}
        ├── algo                          # 算法参数下发（井场级）
        │    └── video                       # 视频识别算法配置
        └── {well_id}
            ├── ctrl                          # 控制指令下发
            │   ├── plunger                       # 柱塞泵控制指令
            │   ├── esp                            # 电潜泵控制指令
            │   ├── reinject                       # 回注水控制指令
            │   ├── rein_valve                     # 井口注入阀门控制指令（柱塞泵往井口）
            │   └── gas_valve                      # 放气阀控制指令
            ├── algo                          # 算法结果下发
            │    └── target                       # 生产制度与目标
            └── seq                           # 连锁指令下发
```

***

## 三、公共数据结构

### 3.1 通用消息头

所有消息均包含以下公共字段（包括平台下发的指令，不包括心跳）。井场级数据（Topic 中无 `{well_id}` 层级，如汇管压力）的消息头省略 `well_id` 字段。

```json
{
  "header": {
    "ver": "1.2",                    // 消息版本号
    "msg_id": "uuid-string",           // 消息唯一标识，用于消息追踪和去重
    "ts": "1726992000000",             // 消息时间戳，UNIX毫秒时间戳（字符串）
    "gw_id": "GW001",                  // 网关ID
    "wf_id": "WF001",                  // 井组ID
    "well_id": "xxx",                  // 单井ID（井场级消息省略此字段）
    "well_type": "jet_pump",           // 井类型：jet_pump(射流泵井) / esp_pump(电潜泵井)
  }
}
```

> **ver 版本兼容约定**：版本前缀相同可向下兼容（如V1.1和V1.0），不同前缀则存在不兼容内容。

> **身份一致性规则**：消息 Topic 中的身份标识（`{gw_id}`/`{wf_id}`/`{well_id}`）必须与 header 中对应字段一致；接收方对身份不一致的消息记录后丢弃。

> **网关上行 msg_id 生成约定**：所有网关上行报文 header 均携带 msg_id。
> - 数据类上行（如 prod 生产数据、algo 算法结果）：网关自动生成，格式不做要求，推荐为 `"epoch毫秒-毫秒内自增序号"`（如 `"1758340800123-0"`）；保证有序号即可。
> - 响应类上行（ack/result/step_result）：回显下行 header.msg_id和cmd_id。

### 3.2 网关心跳（网关→平台）

**Topic**: `/cbm/up/{gw_id}/sys/heartbeat`

```json
{
  "heartbeat": {
    "gw_status": "online",              // 网关整体状态：online
    "sys_uptime": 864000,               // 网关系统运行时间，秒
    "ntp_synced": true                  // 是否已同步NTP时间
  }
}
```

平台超过2次未收到心跳则判定网关离线。

***

## 四、生产数据上报（网关→平台）

> **设定值字段说明**：数据结构中以 `_setpoint` / `_sp` 结尾的字段（如 `pump_pres_setpoint`、`valve_setpoint`、`speed_setpoint`、`valve_sp`）为底层设备当前生效设定值的回显，平台侧可将其与下发指令的目标值对比，判断指令是否成功下达。

> **异常表达说明**：数据异常通过 `status`（设备通信状态：online/offline）、`run_status`（工艺运行状态：running/stopped/fault）、`fault_code`（故障代码）表达。

### 4.1 泵设备数据

#### 4.1.1 射流泵井场 - 柱塞泵数据

**Topic**: `/cbm/up/{gw_id}/{wf_id}/{well_id}/prod/plunger`

```json
{
  "data": {
    "device_type": "plunger",          // 设备类型：plunger(柱塞泵)
    "status": "online",                // 设备状态：online / offline
    "run_status": "running",           // 运行状态：running / stopped / fault
    "ctrl_mode": "auto",               // 控制模式：auto / manual
    "prod_period": "drain_water",      // 生命周期，枚举见4.1.2节末
    "pump_temperature": 45.2,          // 泵头温度，℃
    "valve_feedback": 99.8,            // 高压阀门开度反馈 0-100%
    "inlet_pres": 2.5,                 // 井口注入压力，MPa
    "liquid_level": 10.1,              // 水箱液位，M
    "pump_pres_setpoint": 5.5,         // 泵压设定值，MPa（设备回显）
    "valve_setpoint": 99.8,            // 高压阀门开度设定值 0-100%（设备回显）
    "instant_flow": 15.6,              // 高压水瞬时流量，m³/h
    "total_flow": 2568.5,              // 高压水累计流量，m³
    "low_instant_flow": 15.6,          // 低压水瞬时流量，m³/h
    "low_total_flow": 2568.5,          // 低压水累计流量，m³
    "A_voltage": 380.5,                // A相电压，V
    "B_voltage": 380.5,                // B相电压，V
    "C_voltage": 380.5,                // C相电压，V
    "A_current": 15.6,                 // A相电流，A
    "B_current": 15.6,                 // B相电流，A
    "C_current": 15.6,                 // C相电流，A
    "total_energy": 1000.5,            // 累计耗电量，kWh
    "motor_freq": 8.5,                 // 变频器频率，Hz
    "motor_current": 50.0,             // 变频器电流，A
    "motor_freq_setpoint": 8.6,        // 频率设定值，Hz（设备回显）
    "pump_pres": 5.2,                  // 泵压，MPa
    "runtime_hours": 1256.5,           // 累计运行时间，小时
    "fault_code": 0,                   // 故障代码，无故障时为0
    "fault_description": null          // 故障描述，无故障时为null
  }
}
```

**故障代码表**：（科达变频器）

| 故障描述 | 故障代码 |
| ------- | ---- |
| 无故障 | 00 |
| 加速过电流 | 02 |
| 减速过电流 | 03 |
| 恒速过电流 | 04 |
| 加速过电压 | 05 |
| 减速过电压 | 06 |
| 恒速过电压 | 07 |
| 控制电源故障 | 08 |
| 欠压故障 | 09 |
| 变频器过载 | 10 |
| 电机过载 | 11 |
| 输入缺相 | 12 |
| 输出缺相 | 13 |
| 模块过热 | 14 |
| 外部设备故障 | 15 |
| 通讯故障 | 16 |
| 接触器故障 | 17 |
| 电流检测故障 | 18 |
| 电机调谐故障 | 19 |
| 编码器故障 | 20 |
| EEPROM读写故障 | 21 |
| 对地短路故障 | 23 |
| 累计运行时间到达故障 | 26 |
| 自定义1 | 27 |
| 自定义2 | 28 |
| 累计上电时间到达故障 | 29 |
| 掉载故障 | 30 |
| 运行时PID反馈丢失 | 31 |
| 逐波限流故障 | 40 |
| 运行时切换电机故障 | 41 |
| 速度偏差过大 | 42 |
| 电机过速度 | 43 |
| 电机过温故障 | 45 |
| 制动单元过载 | 61 |
| 制动单元短路 | 62 |

#### 4.1.2 电潜泵井场 - 井下电潜泵数据

**Topic**: `/cbm/up/{gw_id}/{wf_id}/{well_id}/prod/esp`

```json
{
  "data": {
    "device_type": "esp",              // 设备类型：esp(电潜泵)
    "status": "online",                // 设备状态：online / offline
    "run_status": "running",           // 运行状态：running / stopped / fault
    "ctrl_mode": "auto",               // 控制模式：auto / manual
    "prod_period": "drain_water",      // 生命周期，枚举见下表
    "speed": 45.0,                     // 泵实际转速，rpm（独立测量量）
    "speed_setpoint": 45.0,            // 转速设定值，rpm（设备回显）
    "freq": 23.5,                      // 电机工作频率，Hz（独立测量量）
    "A_current": 15.6,                 // A相电流，A
    "B_current": 15.6,                 // B相电流，A
    "C_current": 15.6,                 // C相电流，A
    "A_voltage": 380.5,                // A相电压，V
    "B_voltage": 380.5,                // B相电压，V
    "C_voltage": 380.5,                // C相电压，V
    "busbar_voltage": 380.5,           // 母线电压，V
    "motor_power": 35.6,               // 电机功率，kW
    "runtime_hours": 2568.5,           // 累计运行时间，小时
    "fault_code": 0,                   // 故障代码，无故障时为0
    "fault_description": null          // 故障描述，无故障时为null
  }
}
```

**prod_period（生命周期）枚举**：

| 值             | 说明   |
| ------------- | ---- |
| drain_water   | 排水期 |
| increase_prod | 提产期 |
| stable_prod   | 稳产期 |
| decrease_prod | 递减期 |

### 4.2 流量计数据

#### 4.2.1 大产气流量计

**Topic**: `/cbm/up/{gw_id}/{wf_id}/{well_id}/prod/flow_large`

```json
{
  "data": {
      "device_type": "flow_large",     // 大产气流量计
      "status": "online",              // 仪表状态：online / offline
      "instant_flow": 1250.5,          // 瞬时流量，m³/h
      "total_flow": 45625.8,           // 累计流量，m³
      "temperature": 25.6,             // 温度，℃
      "pres": 0.35                     // 压力，MPa
    }
}
```

#### 4.2.2 小产气流量计

**Topic**: `/cbm/up/{gw_id}/{wf_id}/{well_id}/prod/flow_small`

```json
{
  "data": {
    "device_type": "flow_small",     // 小产气流量计
    "status": "offline",
    "instant_flow": 0,
    "total_flow": 12580.2,
    "temperature": 0,
    "pres": 0
    }
}
```

#### 4.2.3 产水流量计

**Topic**: `/cbm/up/{gw_id}/{wf_id}/{well_id}/prod/output_flow`

```json
{
  "data": {
      "device_type": "output_flow",     // 产水流量计
      "status": "online",
      "instant_flow": 0.85,             // 瞬时流量，m³/h
      "total_flow": 1256.8,             // 累计流量，m³
      "last_prod": 15.2,                // 上一天累积量，m³（每日08:00计算此前24h值）
      "daily_prod": 8.6                 // 日产量，m³（自最近08:00日切点由累计流量估算）
    }
}
```

### 4.3 回注水系统数据（电潜泵版本）

**Topic**: `/cbm/up/{gw_id}/{wf_id}/{well_id}/prod/reinject`

```json
{
  "data": {
    "device_type": "reinject",         // 设备类型：reinject(回注水系统)
    "status": "online",
    "run_status": "running",           // 运行状态：running / stopped / fault
    "ctrl_mode": "auto",               // 控制模式：auto / manual
    "speed": 1450.0,                   // 转速，rpm
    "speed_setpoint": 1500,            // 转速设定值，rpm（设备回显）
    "motor_current": 8.5,              // 电机电流，A
    "motor_voltage": 380.0,            // 电机电压，V
    "motor_power": 4.5,                // 电机功率，kW
    "instant_flow": 15.6,              // 瞬时回注流量，m³/h
    "total_flow": 2568.5,              // 累计回注流量，m³
    "runtime_hours": 856.5,            // 累计运行时间，小时
    "fault_code": 0,                   // 故障代码
    "fault_description": null          // 故障描述
  }
}
```

### 4.4 压力数据

#### 4.4.1 套压数据

**Topic**: `/cbm/up/{gw_id}/{wf_id}/{well_id}/prod/casing_pres`

```json
{
  "data": {
    "device_type": "casing_pres",      // 套压
    "status": "online",
    "pres": 1.25                       // 压力值，MPa
  }
}
```

#### 4.4.2 井下压力计数据

**Topic**: `/cbm/up/{gw_id}/{wf_id}/{well_id}/prod/dh`

```json
{
  "data": {
    "device_type": "dh",             // 井下压力计
    "status": "online",
    "pres": 8.5,                     // 井底流压，MPa
    "temperature": 65.2,             // 井底温度，℃
    "submergence": 20.1              // 沉没度，m
  }
}
```

#### 4.4.3 汇管压力数据（井场级）

**Topic**: `/cbm/up/{gw_id}/{wf_id}/prod/pipe_pres`

汇管为井场级共享设施，Topic 挂井场（`{wf_id}`）层级，消息头省略 `well_id` 字段。

```json
{
  "data": {
    "device_type": "pipe_pres",      // 汇管压力
    "status": "online",
    "pres": 1.25                     // 压力值，MPa
  }
}
```

#### 4.4.4 气阀数据

##### 4.4.4.1 放气阀数据

**Topic**: `/cbm/up/{gw_id}/{wf_id}/{well_id}/prod/gas_valve`

```json
{
  "data": {
    "device_type": "gas_valve",        // 放气阀
    "status": "online",
    "valve_fb": 99.9,                  // 放气阀开度反馈，0-100%
    "valve_sp": 100,                   // 放气阀开度设定值，0-100%（设备回显）
    "full_open": 0,                    // 放气阀门开到位
    "full_close": 1                    // 放气阀门关到位
  }
}
```
***

## 五、控制指令（平台→网关）

### 5.1 控制指令下发

#### 5.1.1 指令消息结构

```json
{
  "cmd": {
    "cmd_id": "CMD20240115001",        // 指令唯一标识
    "action": "set_pres",              // 动作类型
    "value": 6.0,                      // 动作参数（可选，仅带参数动作携带）
    "timeout": 15                      // 网关侧执行超时时间，秒
  }
}
```

**超时判定**：`timeout` 为平台指定的网关侧执行超时时间（秒）。网关在该时间内未执行完毕，上报 `execute_status = timeout`；若因网关离线等异常导致平台长时间未收到 result，由平台按兜底策略判定指令失败。

| 参数      | 类型   | 必填 | 说明                        |
| ------- | ---- | -- | ------------------------- |
| cmd_id  | 字符串  | 是  | 指令唯一ID                    |
| action  | 字符串  | 是  | 该指令所对应动作                  |
| value   | 数值   | 否  | 动作参数，仅带参数动作携带            |
| timeout | 整数   | 是  | 网关侧执行超时时间，秒               |

#### 5.1.2 单指令类型

##### 射流泵井场 - 柱塞泵控制

**Topic**: `/cbm/down/{gw_id}/{wf_id}/{well_id}/ctrl/plunger`

| 动作          | 说明       | 参数                     |
| ----------- | -------- | ---------------------- |
| start       | 启动       | 无参数                   |
| stop        | 停机       | 无参数                   |
| set_pres    | 设定泵压     | {"value": 5.5} 目标泵压，MPa |
| set_freq    | 设定频率     | {"value": 30} 目标频率，Hz  |
| clear_alarm | 消除变频器报警  | 无参数                   |

**示例 - 启动**：

```json
{
  "cmd": {
    "cmd_id": "CMD20240115006",
    "action": "start",
    "timeout": 15
  }
}
```

**示例 - 设定泵压**：

```json
{
  "cmd": {
    "cmd_id": "CMD20240115001",
    "action": "set_pres",
    "value": 6.0,
    "timeout": 5
  }
}
```

##### 电潜泵井场 - 电潜泵控制

**Topic**: `/cbm/down/{gw_id}/{wf_id}/{well_id}/ctrl/esp`

| 动作      | 说明   | 参数                        |
| ------- | ---- | ------------------------- |
| start   | 启动   | 无参数                     |
| stop    | 停机   | 无参数                     |
| set_speed | 设定转速 | {"value": 45.0} 目标转速，rpm |
| reverse | 反转启动 | {"value": 60} 反转速度，rpm |

**示例 - 设定转速**：

```json
{
  "cmd": {
    "cmd_id": "CMD20240115003",
    "action": "set_speed",
    "value": 45.0,
    "timeout": 5
  }
}
```

##### 电潜泵井场 - 回注水系统控制

**Topic**: `/cbm/down/{gw_id}/{wf_id}/{well_id}/ctrl/reinject`

| 动作      | 说明   | 参数                         |
| ------- | ---- | -------------------------- |
| start   | 启动   | 无参数                      |
| stop    | 停机   | 无参数                      |
| set_speed | 设定转速 | {"value": 1500} 目标转速，rpm |

```json
{
  "cmd": {
    "cmd_id": "CMD20240115002",
    "action": "set_speed",
    "value": 1500,
    "timeout": 5
  }
}
```

##### 射流泵井场 - 注入阀门设置

**Topic**: `/cbm/down/{gw_id}/{wf_id}/{well_id}/ctrl/rein_valve`

| 动作       | 说明   | 参数                      |
| -------- | ---- | ----------------------- |
| set_valve | 设定开度 | {"value": 100} 目标开度（%） |

```json
{
  "cmd": {
    "cmd_id": "CMD20240115004",
    "action": "set_valve",
    "value": 100,
    "timeout": 5
  }
}
```

##### 井场 - 放气阀门设置

**Topic**: `/cbm/down/{gw_id}/{wf_id}/{well_id}/ctrl/gas_valve`

| 动作       | 说明   | 参数                      |
| -------- | ---- | ----------------------- |
| set_valve | 设定开度 | {"value": 100} 目标开度（%） |


#### 5.1.3 连锁指令

> **设计说明**：
>
> - **用户自定义连锁**：平台编排好每个步骤依次下发
> - **通道定位**：seq 为增强型连锁控制通道，动作词汇与 ctrl 单指令通道不要求一一对应，由网关侧解析执行；平台侧负责编排，保证用户使用体验的一致性。

##### 用户自定义连锁指令（平台编排下发）

**Topic**: `/cbm/down/{gw_id}/{wf_id}/{well_id}/seq`

**示例 - 自定义连锁控制**：

```json
{
  "cmd": {
    "cmd_id": "CMD20240115005",
    "parameters": {
      "steps": [
        {"action": "stop", "parameters": {}, "step_timeout": 15},
        {"action": "wait", "parameters": {"time": 50}, "step_timeout": 60},
        {"action": "start", "parameters": {"value": 5.0}, "step_timeout": 20}
      ]
    }
  }
}
```

| 动作             | 说明              | 参数                                                                 |
| -------------- | --------------- | ------------------------------------------------------------------ |
| start          | 启动              | parameters: {"value": 10}, "step_timeout": 20 启动速度、启动超时时间（秒）            |
| stop           | 停机              | parameters:{},"step_timeout": 15 停机超时时间（秒）                            |
| wait           | 等待              | parameters: {"time": 100}, "step_timeout": time\*1.1+5 等待时间（秒）、超时时间=等待时间×1.1+5（秒） |
| set            | 设定              | parameters: {"value": 100}, "step_timeout": 15 目标值（区分正负，正值为正转转速，负值为反转转速，正值时不能设置负数，负值时不能设置正数）、设定超时时间（秒） |
| accel          | 调速              | parameters: {"value": 10}, "step_timeout": 15 增量值（区分正负，当前值+增量值，负值为减）、调速超时时间（秒） |
| set_valve      | 设定放气阀门开度        | parameters: {"value": 100}, "step_timeout": 15 目标开度（0-100%）、设定超时时间（秒）    |
| start_reinject | 回注水泵启动（仅限电潜泵）   | parameters: {"value": 10}, "step_timeout": 10 启动速度                      |
| stop_reinject  | 回注水泵停止（仅限电潜泵）   | parameters:{},"step_timeout": 15 停机超时时间（秒）                            |
| set_reinject   | 设定回注水泵转速（仅限电潜泵） | parameters: {"value": 1500}, "step_timeout": 15 目标转速，rpm                 |

连锁指令失败或者超时立即停止，后续步骤不执行，平台不重发连锁指令。

### 5.1.4 人工智能识别算法配置参数（平台→网关）
**Topic**: `/cbm/down/{gw_id}/{wf_id}/algo/video`
视频为井场级共享设施，Topic 挂井场（`{wf_id}`）层级，消息头省略 `well_id` 字段。{AlgorithmNo}算法编号 1：人车识别；2：液位识别。
**示例 - 配置参数，没有配置的可以不用下发**：
| 动作      | 说明   | 参数                        |
| ------- | ---- | ------------------------- |
| enabletimedupload   | 开启定时上传图片   | {"value": true}   true：开启：false：关闭|
| timeduploadinterval | 定时间隔          | {"value": 300}   ms|
| alarminterval       | 报警间隔          | {"value": 300}   ms|


```json
{
  "cmd": {
    "cmd_id": "CMD20240115005",
    "AlgorithmNo":1,
    "enabletimedupload": true,
    "timeduploadinterval": 600000,
    "alarminterval": 60000
  }
}
```

### 5.2 指令确认回复（网关→平台）

**Topic**: `/cbm/up/{gw_id}/{wf_id}/{well_id}/ack/{device}`

```json
{
  "ack": {
    "cmd_id": "CMD20240115002",           // 对应的指令ID
    "ack_status": "accepted",             // 接受状态
    "ack_reason": null                    // 接受原因，成功为null
  }
}
```

**ack topic 的 {device} 后缀约定**：

| 下行路径                                                              | ack topic 后缀 |
| ----------------------------------------------------------------- | ----------- |
| ctrl/{设备}（plunger / esp / reinject / rein_valve / gas_valve） | ack/{设备}    |
| seq（连锁指令）                                                        | ack/seq_ack |

**cmd_id 取值约定**：ctrl/ 与 seq/ 下行报文均在 cmd 节点携带 cmd_id，网关在 ack 与 result 中原样回显。
**msg_id 取值约定**：网关在 header 中原样回显。

**确认状态枚举**：

| 值        | 说明                    |
| -------- | --------------------- |
| accepted | 指令已接受，开始执行            |
| rejected | 指令被拒绝（参数错误、设备故障、指令互斥） |
| partial  | 指令已部分执行响应（预留，当前对外协议范围内无适用指令） |

```txt
ack_reason
{
  null,(成功)
  "设备离线，无法执行",
  "指令互斥，无法执行",
  "参数错误"
  ……
}
```
### 5.2.1 人工智能识别算法配置指令确认回复（网关→平台）
**Topic**: `/cbm/up/{gw_id}/{wf_id}/ack/algo/video`

```json
{
  "ack": {
    "cmd_id": "CMD20240115002",           // 对应的指令ID
    "AlgorithmNo":1,
    "ack_status": "accepted",             // 接受状态
    "ack_reason": null                    // 接受原因，成功为null
  }
}
```

### 5.3 指令执行结果（网关→平台）

#### 5.3.1 普通指令执行结果

**Topic**: `/cbm/up/{gw_id}/{wf_id}/{well_id}/result/{device}`

```json
{
  "result": {
    "cmd_id": "CMD20240115002",        // 对应的指令ID
    "execute_status": "success",       // 执行状态
    "elapsed_time": 4.5,               // 执行耗时，秒
    "fail_reason": null                // 执行失败原因，成功为null
  }
}
```

**执行状态枚举**：

| 值       | 说明   |
| ------- | ---- |
| success | 执行成功 |
| failed  | 执行失败 |
| timeout | 执行超时 |

| 参数              | 类型   | 默认范围 | 说明          |
| --------------- | ---- | ---- | ----------- |
| execute_status  | 字符串  | 无    | 指令执行的状态     |
| elapsed_time    | 浮点数  | 无    | 对应指令执行耗时    |
| fail_reason     | 字符串  | 无    | 指令执行失败的原因     |

#### 5.3.2 连锁指令执行结果

**Topic**: `/cbm/up/{gw_id}/{wf_id}/{well_id}/result/seq_res`

```json
{
  "step_result": {
    "cmd_id": "CMD20240115005",        // 对应的指令ID
    "step_index": 1,                   // 当前步骤索引，从1开始
    "fail_reason": null,               // 执行失败原因，成功为null
    "step_status": "success",          // 执行状态
    "start_timestamp": "1726992000000",  // 该步骤开始执行时间戳，UNIX毫秒（字符串）
    "end_timestamp": "1726992000000",    // 该步骤结束执行时间戳，UNIX毫秒（字符串）
    "step_time": 4.5,                  // 该步骤执行耗时，秒
    "total_time": 100.5                // 总执行耗时，秒
  }
}
```

连锁指令的步骤总数由平台侧编排管理（用户经平台下发），平台依据 `cmd_id` 关联步骤序列，网关无需上报总步骤数。

| step_status | 执行状态 |
| ---- | ---- |
| success | 执行成功 |
| failed | 执行失败 |
| timeout | 执行超时 |
| running | 执行中 |

***

## 六、算法结果上报（网关→平台）

### 6.1 生产制度与目标（平台→网关）

**Topic**: `/cbm/down/{gw_id}/{wf_id}/{well_id}/algo/target`

```json
{
  "prod_target": {
    "effective_date": "2024-01-15",            // 生效日期
    "prod_period": "drain_water",              // 生命周期
    "targets": {
      "pres_loop": {
        "target_pres": 1.85,                   // 目标井下流压，MPa
      },
      "casing_loop": {
        "target_casing_pres": 1.85,            // 目标套压，MPa
        "target_gas_increase": 200,             // 提产目标，m³ 
      },
      "gas_loop": {
        "target_gas_prod": 120.0,             // 产气量目标，m³/h
      }
    }
  }
}
```

**生产制度枚举**：

| 场景                   | 下发内容   |
| -------------------- | ---- |
| 排水期 |  drain_water |
| 提产期 | increase_prod  |
| 稳产期 |  stable_prod |
| 递减期 |  decrease_prod |

### 6.2 人工智能识别算法上报
**Topic**: `/cbm/up/{gw_id}/{wf_id}/algo/video`
视频为井场级共享设施，Topic 挂井场（`{wf_id}`）层级，消息头省略 `well_id` 字段。

```json
{
  "data": {
    "device_type": "video",      // 视频识别
    "AlgorithmNo":1,             // 识别算法 1：人车识别；2：液位识别；
    "isalarm": true,          //是否报警 类型：BOOL true：报警 false：未报警（上传采集图片）
    "detection":  [{                  //识别目标信息集合 根据以下参数可以在图片中标定出目标
          "box": [0,1002,197,1221],     //识别目标在图片图片中的坐标top, left, right, bottom
          "score": "0.56",              //识别目标置信度
          "class": "0",                 //识别目标类型的ID序号
          }],
    "alarm_objects":["person"],       //识别目标名称集合
    "confidence":["0.56"],            //识别目标置信度集合
    "img":"",                         //当前图片 base64图片编码
    "alarm_content":"识别到目标报警"   //报警内容
  }
}
```
***

## 七、扩展机制（经协议仓库协作流程协商确认后发布）

### 7.1 扩展协作流程

协议扩展（新增设备、新增参数、新增算法、枚举值调整等）通过协议 GitHub 仓库以协作流程进行：

1. 外部接入单位通过 **Issue** 提交扩展需求或问题反馈（按模板分类：协议纠错 / 新增字段 / 新增设备 / 疑问咨询）
2. 需求受理后，接入单位通过 **fork + Pull Request** 提交修改，协议文档与 JSON Schema 须同步修改
3. 维护者对 PR 进行技术审查
4. 审查通过后由协议负责人合入，并统一发布新版本
5. 所有变更通过 Release 版本化发布，接入方订阅 Release 获取更新

### 7.2 新增设备

当需要新增设备时，按以下步骤扩展：

1. **新增Topic**：在对应类别下新增子Topic
   ```
   /cbm/up/{gw_id}/{wf_id}/{well_id}/prod/{new_device}
   ```
2. **定义数据结构**：在JSON中新增设备数据字段
3. **更新文档**：在本文档中补充新增设备的点位定义

### 7.3 新增参数

当现有设备需要新增监测参数时：

1. 在对应设备的数据结构中新增字段
2. 更新文档说明

### 7.4 新增算法

当需要新增算法时：

1. **新增Topic**：
   ```
   /cbm/up/{gw_id}/{wf_id}/{well_id}/algo/{new_algo}
   ```
2. **定义数据结构**：参考现有算法结果格式
3. **更新文档**：补充算法类型枚举

***

## 八、附录

### 8.1 缩写词汇编码

| 缩写       | 说明       | 全称                   |
| -------- | -------- | -------------------- |
| gw       | 网关       | gateway              |
| wf       | 井组       | wellfield            |
| prod     | 生产数据     | production           |
| ctrl     | 控制指令     | control              |
| algo     | 算法数据     | algorithm            |
| flow     | 流量计      | flowmeter            |
| pres     | 压力       | pressure             |
| ts       | 时间戳      | timestamp            |
| dh       | 井下数据     | downhole             |
| reinject | 回注水系统数据  | reinjection          |
| cmd      | 控制指令     | command              |
| seq      | 连锁指令     | sequence             |
| res      | 结果       | result               |
| ack      | 确认       | acknowledgment       |

### 8.2 计量单位

| 参数类型      | 单位     | 说明      |
| --------- | ------ | ------- |
| 气流量（瞬时）   | m³/h   | 立方米/小时  |
| 气流量（累计）   | m³     | 立方米     |
| 水流量（瞬时）   | m³/h   | 立方米/小时  |
| 水流量（累计）   | m³     | 立方米    |
| 压力            | MPa    | 兆帕      |
| 温度            | ℃     | 摄氏度     |
| 频率            | Hz     | 赫兹      |
| 转速            | rpm    | 转/分钟    |
| 电流            | A      | 安培      |
| 电压            | V      | 伏特      |
| 功率            | kW     | 千瓦      |
| 电量            | kWh    | 千瓦时     |
| 阀门开度        | %      | 百分比     |

### 8.3 V1.0 → V1.2 修订明细

| #  | 变更内容                                             | 类别   |
| -- | ------------------------------------------------ | ---- |
| 1  | 删除 header.data_quality 字段及数据质量枚举               | 不兼容  |
| 2  | header.well_type 枚举值 esp → esp_pump（与设备子类别消歧）  | 不兼容  |
| 3  | 时间戳统一为 UNIX 毫秒字符串（step_result 的 start/end_timestamp 由数字改为字符串） | 不兼容  |
| 4  | plunger.power → total_energy（累计耗电量，kWh）        | 不兼容  |
| 5  | 放气阀 Topic 命名统一：gas_small → gas_valve            | 不兼容  |
| 6  | pipe_pres 汇管压力 Topic 由单井层级移至井场层级，消息头省略 well_id | 不兼容  |
| 7  | output_flow 单位由 L/min、L 统一为 m³/h、m³             | 不兼容  |
| 8  | ctrl 动作命名统一 set_xxx：esp/reinject 的 set → set_speed，rein_valve/gas_valve 的 set → set_valve | 不兼容  |
| 9  | reverse 反转启动删除 time（持续时长）参数                    | 不兼容  |
| 10 | start / stop / clear_alarm 无参数化（不再携带 value:1）    | 兼容   |
| 11 | timeout 语义明确为网关侧执行超时，网关超时上报 timeout          | 澄清   |
| 12 | 明确 \*\_setpoint 字段为底层设备设定值回显                  | 澄清   |
| 13 | 明确 last_prod / daily_prod 统计口径（08:00 日切）       | 澄清   |
| 14 | esp 的 speed（泵实际转速）与 freq（电机工作频率）为两个独立测量量 | 澄清   |
| 15 | reinject 流量字段补标单位 m³/h、m³                      | 澄清   |
| 16 | 新增身份一致性规则（Topic 与 header 身份不一致的消息记录后丢弃）      | 规则新增 |
| 17 | 扩展机制改为 GitHub 协作流程（Issue + fork/PR + 维护者审查）  | 流程变更 |

> V1.0 发布后尚未实际部署，V1.2 为首个实际发布版本；已基于 V1.0 开发的接入方请按本表调整。

***
