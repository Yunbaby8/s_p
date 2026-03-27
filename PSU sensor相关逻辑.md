



# PSU status sensor

核心函数：

- PLR_GetPSUPresentStatus (line 210) 作用：对单个 PSU 返回 PRESENT / ABSENT
- PLR_GetPSUTotalStatus (line 147) 作用：按配置从 GPIO / CPLD / 9555 真正把底层状态读出来
- PLR_PSUSensorMonitor 中 case PLR_SENSOR_PSU0_STATUS ~ PLR_SENSOR_PSU4_STATUS (line 2755) 作用：基于 Present / AC / Fault / Predictive 生成 PSU 状态位图，并写入 g_PLR_PSUStatus_t[i].IsFault

# PSU Fan speed sensor

核心函数：

- PLR_PSUSensorMonitor 中 case PLR_SENSOR_PSU0_FAN_SPEED ~ PLR_SENSOR_PSU4_FAN_SPEED (line 2126) 作用：这是 PSU 风扇转速 sensor 的主逻辑，先检查在位，再读 PMBus，再做转换和缓存
- PLR_GetPMBUSReading (line 415) 作用：真正从 PSU PMBus 寄存器读原始值
- 补一个底层转换： PLR_PMBUSConvertLinear (line 2170) 作用：把 PMBus 线性格式转换成可用读数

#  **PSU** **redundancy sensor**

核心函数：

- PLR_PSUSensorMonitor 中 case PLR_SENSOR_REDUNDANT_AC (line 2867) 作用：统计 AC_A_GoodCount / AC_B_GoodCount，得出 0xAF
- PLR_PSUSensorMonitor 中 case PLR_SENSOR_REDUNDANT_PSU (line 2905) 作用：统计 good PSU 数量，得出 0xB4

# Pin and Pout

核心函数：

- PLR_Project_SensorMonitor.c (line 2013) 这是 PIN/POUT 的主流程函数所在位置。它负责识别当前 sensor 是 PIN 还是 POUT，确定 PSU 编号，检查在位状态，读取 PMBus 值，做转换和缓存写入。
- PLR_GetPMBUSReading (line 415) 这是底层真正去 PSU 读 PIN/POUT 寄存器原始值的核心函数。
- PLR_PostMonitorSensor_PSU (line 442) 这是把前面算好的缓存值写到共享内存、让上层真正读到的核心函数。

# PSU在位逻辑

```mermaid
flowchart TD
    A["进入 PSUx_STATUS sensor 处理逻辑"] --> B["根据 SensorNum 计算 PsuIndex"]
    B --> C["调用 PLR_GetPSUPresentStatus(PsuIndex, &PresentStatus)"]

    C --> D{"PSUID 是否越界?"}
    D -- 是 --> E["返回失败"]
    D -- 否 --> F["从 g_PLR_PSUPresentMonMethod_t[PSUID] 取得 PSUMonMethod"]

    F --> G["调用 PLR_GetPSUTotalStatus(PSUMonMethod, &status)"]

    G --> H{"MonMethod 类型"}
    H -- GPIO --> I["调用 PLR_GPIOOperate 读取 GPIO 电平"]
    H -- CPLD --> J["调用 PLR_RWCPLDRegister 读取 CPLD 寄存器"]
    H -- 9555 --> K["通过 I2C 读取 9555 寄存器"]
    H -- 其他 --> L["返回失败"]

    I --> M["得到原始 status"]
    J --> N["status = data_out & RegBit"]
    K --> O["status = readbuf[0] & Channel"]

    N --> M
    O --> M

    M --> P{"Polarity 是否为 HIGH?"}
    P -- 是 --> Q{"status 是否非 0?"}
    Q -- 是 --> R["PresentStatus = PLR_PSU_PRESENT"]
    Q -- 否 --> S["PresentStatus = PLR_PSU_ABSENT"]

    P -- 否 --> T{"status 是否非 0?"}
    T -- 是 --> U["PresentStatus = PLR_PSU_ABSENT"]
    T -- 否 --> V["PresentStatus = PLR_PSU_PRESENT"]

    R --> W{"PresentStatus 是否为 PRESENT?"}
    S --> W
    U --> W
    V --> W

    W -- 是 --> X["PSUStatus |= BIT0"]
    W -- 否 --> Y["PSUStatus = 0"]

    X --> Z["写入 g_CachedSensorInfo[SensorNum]"]
    Y --> Z
```



# psu fan speed

````mermaid

flowchart TD
    A["监控线程周期扫描 PSU Fan Speed sensor"] --> B["根据 sensor number 映射到对应 PSU"]
    B --> C["设置风扇转速寄存器与缩放参数"]

    C --> D{"是否处于 PSU 升级 / Flash 模式?"}
    D -- 是 --> E["状态置为 BLOCK<br/>停止本轮读取"]
    D -- 否 --> F["检查 PSU 是否在位"]

    F --> G{"PSU 是否在位?"}
    G -- 否 --> H["状态置为 NOT PRESENT<br/>停止本轮读取"]
    G -- 是 --> I["进入底层读取流程"]

    I --> J{"底层检查 AC 是否正常?"}
    J -- 否 --> K["读取失败<br/>停止本轮读取"]
    J -- 是 --> L["通过 I2C 向 PSU 发送 PMBus 风扇读取命令"]

    L --> M{"I2C / PMBus 读取是否成功?"}
    M -- 否 --> N["状态置为 READING ERROR<br/>清空防抖历史<br/>停止本轮读取"]
    M -- 是 --> O["得到 2 字节原始风扇数据"]

    O --> P["将 PMBus Linear 数据转换为实际转速值 x100"]
    P --> Q["按传感器线性模型缩放为 8-bit raw reading"]
    Q --> R["进行 anti-shake 防抖处理"]

    R --> R1{"anti-shake 返回值是否为 0xFF?"}
    R1 -- 是 --> R2["状态置为 READING ERROR<br/>停止本轮读取"]
    R1 -- 否 --> S["写入 sensor 缓存<br/>读数 / 状态 / 原始信息"]

    S --> T["Post hook 回填给 IPMI / SDR"]
    T --> U{"最终状态"}
    U -- 正常 --> V["对外显示有效读数"]
    U -- 不在位 --> W["对外显示 Disabled"]
    U -- 读取失败 --> X["对外显示 Unavailable"]
    U -- BLOCK --> X

    E --> Y["结束"]
    H --> Y
    K --> Y
    N --> Y
    R2 --> Y
    V --> Y
    W --> Y
    X --> Y
```
````



# **PSU redundancy sensor**



```mermaid
flowchart TD
    A["监控线程周期扫描 sensor"] --> B["进入 PSU 相关监控流程"]
    B --> C["从 CPLD / 9555 / PMBus 获取每个 PSU 的 Present / AC / Fault / Predictive 等信息"]

    C --> D{"是否处于 PSU 升级 / Flash 模式?"}
    D -- 是 --> D1["对应 PSU 类 sensor: 状态置为 BLOCK"]
    D -- 否 --> E{"底层读取是否成功?"}

    E -- 否 --> E1["对应 PSU 类 sensor: 状态置为 READING_ERR"]
    E -- 是 --> F["生成每个 PSUx_STATUS 位图"]

    F --> G["写入每个 PSU 的状态表"]
    F --> H["单 PSU 状态类 sensor 写入缓存"]

    H --> H1{"单 PSU sensor 对应 PSU 是否不在位?"}
    H1 -- 是 --> H2["单 PSU sensor: 状态置为 NOT_PRESENT"]
    H1 -- 否 --> H3["单 PSU sensor: 状态置为 OK"]

    G --> I["基于每个 PSU 的状态表汇总 REDUNDANT_AC (0xAF)"]
    G --> J["基于每个 PSU 的状态表汇总 REDUNDANT_PSU (0xB4)"]

    I --> K{"满足 AC 输入冗余条件?"}
    K -- 是 --> K1["0xAF = BIT0"]
    K -- 否 --> K2["0xAF = BIT1"]

    J --> L{"满足 PSU 数量冗余条件?"}
    L -- 是 --> L1["0xB4 = BIT0"]
    L -- 否 --> L2["0xB4 = BIT1"]

    K1 --> M["写入 0xAF 缓存, 状态 = OK"]
    K2 --> M
    L1 --> N["写入 0xB4 缓存, 状态 = OK"]
    L2 --> N

    M --> O["后处理阶段拷入共享内存"]
    N --> O
    H2 --> O
    H3 --> O
    D1 --> O
    E1 --> O

    O --> P["IPMI / SEL / 上层接口从共享内存读取"]

    P --> Q{"redundancy sensor 对外解释"}
    Q -- BIT0 --> R["Fully Redundant"]
    Q -- BIT1 --> S["Redundancy Lost"]

    O --> T["BLOCK / READING_ERR 可对外表现为 Unavailable"]
    O --> U["NOT_PRESENT 可对外表现为 Disabled"]

```

# Pin and Pout

```mermaid
flowchart TD
    A["监控线程周期扫描 PSU 功率类 sensor"] --> B["识别当前是 PSU PIN 还是 PSU POUT"]

    B --> C["根据 sensor 类型确定目标 PSU 和待读取的功率寄存器"]
    C --> D{"是否处于 PSU 升级 / Flash 模式?"}

    D -- 是 --> D1["该 sensor 状态置为 BLOCK / 不可用"]
    D -- 否 --> E["检查目标 PSU 是否在位"]

    E --> F{"PSU 是否在位?"}
    F -- 否 --> F1["该 sensor 状态置为 NOT_PRESENT"]
    F -- 是 --> G["通过 PMBus 读取原始功率值"]

    G --> H{"读取是否成功?"}
    H -- 否 --> H1["该 sensor 状态置为 READING_ERR"]
    H -- 是 --> I["将 PMBus 原始值转换为实际功率值"]

    I --> J["对读数做缩放 / 单位换算 / 抗抖处理"]
    J --> K["写入该 sensor 的缓存值和状态"]

    K --> L{"是否为 PSU PIN?"}
    L -- 是 --> L1["同步更新每个 PSU 的输入功率表，供总功耗统计复用"]
    L -- 否 --> L2["仅保留该 PSU 的输出功率读数"]

    L1 --> M["后处理阶段将缓存值拷入共享内存"]
    L2 --> M
    D1 --> M
    F1 --> M
    H1 --> M

    M --> N["IPMI / SEL / 上层接口从共享内存读取该 sensor"]

    N --> O{"最终对外显示"}
    O -- PIN --> P["显示 PSU 输入功率"]
    O -- POUT --> Q["显示 PSU 输出功率"]

```