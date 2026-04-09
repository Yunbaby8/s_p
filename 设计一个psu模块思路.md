1. 如果让我设计 PSU 模块，我会怎么分层
   - hw access 层：只负责拿硬件数据，GPIO、CPLD、9555、PMBus 都在这一层。
   - per-PSU 状态层：把硬件原始值整理成“每颗 PSU 当前状态”。
   - 整机汇总层：从所有 PSU 状态里算 redundancy、good count、mode summary。
   - sensor/export 层：把最终状态转成 sensor、Redfish、OEM command 可直接用的数据。
2. 每层为什么这样分
   - hw access 单独分出来，是为了把 I2C/GPIO 细节关在最底下，上层不直接碰总线。
   - per-PSU 状态层 单独放，是为了让 present、AC、input、fault 这些只有一份真值，不要每个上层各算一遍。
   - 整机汇总层 单独放，是为了让 redundancy 只基于一套最终 PSU 状态来算，不要 sensor、command、Redfish 各算各的。
   - sensor/export 单独放，是为了让对外展示和内部状态解耦，上层只读缓存，不重新读硬件。
3. 每层主要维护哪些数据
   - hw access 层
     维护静态配置表里定义的 bus、slave、gpio、channel、polarity，用局部变量保存本次读到的寄存器值。
   - per-PSU 状态层
     维护每颗 PSU 的 present、ac、inputA/inputB、fault、predictive、current mode、valid、last_update、重试计数、最终 status bit。
   - 低频设备信息
     维护每颗 PSU 的 vendor/type/model/mfr id，更新频率低，和运行态分开。
   - 整机汇总层
     维护 present_count、healthy_count、redundancy、degraded/lost、整机 active/standby 摘要。
   - sensor/export 层
     维护给上层直接读的最终缓存，比如每个 sensor 的离散 bit、Redfish 字段、命令返回值。
4. 这些数据分别应该落在哪类数据结构里
   - 静态配置表
     放成 const 全局 table，按 PSU index 一项一项配置。
     这里放 bus、slave、present 监控方式、AC 监控方式、极性、是否双 AC。
   - 每颗 PSU 一份的运行时状态结构体
     放成“结构体 + 全局数组”，按 PSU index 管。
     这是主状态存放地。
   - 每颗 PSU 一份的低频设备信息结构体
     也放成“结构体 + 全局数组”。
     这里专门放 vendor/type/model，不和 present/ac/fault 混。
   - 整机级别的汇总结构体
     放成一个全局结构体。
     这里放 redundancy、healthy count、summary。
   - 对外共享缓存
     放成共享缓存区或全局导出缓存。
     这里只放最终结果，不放原始寄存器值。
   - 线程循环里的局部变量
     放本轮读取的 read_buf、status_word、mfr_specific、临时解析结果。
     这些不要进全局。
5. 数据从哪来，到哪去
   - present
     从 GPIO/CPLD/9555 拿。
     先放线程局部变量，再写回“每颗 PSU 运行时状态数组”。
     后面 sensor、redundancy、mode 策略都读这个数组。
   - AC
     从 ACOK pin 或 PMBus STATUS_MFR_SPECIFIC 拿。
     原始 byte 只放局部变量。
     解析完写回“每颗 PSU 运行时状态数组”的 ac 和 valid。
   - input A/B
     从 STATUS_INPUT / STATUS_INPUT_B 拿。
     原始寄存器只在本轮局部变量里。
     解析后的 A/B 错误状态写进“每颗 PSU 运行时状态数组”。
   - vendor/type
     从 MFR_MODEL / MFR_ID 拿。
     写进“每颗 PSU 低频设备信息数组”。
     AC 解析、特殊 bit 解释都读这个数组。
   - 最终 PSU status bit
     从 per-PSU 运行时状态统一拼出来。
     写进 per-PSU 状态数组，同时写一份到 sensor/export 缓存。
   - redundancy
     不直接读硬件。
     由汇总模块从“所有 PSU 的最终状态数组”统一计算，写进“整机汇总结构体”，再导出给 sensor/export。
6. 哪些地方要循环
   - 主轮询线程
     周期扫 present、AC、input、fault、predictive，更新每颗 PSU 运行时状态数组。
   - 低频探测线程或低频任务
     插拔后或低频刷新 vendor/type/model。
   - 模式同步线程
     周期检查目标 mode 和 PSU 当前 mode 是否一致，不一致就写 PMBus。
   - 整机汇总
     最好跟主轮询同周期，紧跟在 per-PSU 状态更新之后算一次。
7. 哪些地方要缓存
   - 必须缓存
     present、ac、inputA/inputB、fault、predictive、最终 status bit、vendor/type、current mode、redundancy summary。
   - 不必长期缓存
     单次 PMBus 原始 byte、临时读 buffer、单次转换过程里的中间 bit。
   - 应该给别的模块直接读的
     per-PSU 最终状态数组、整机汇总结构体、sensor/export 最终缓存。
   - 不该共享的
     I2C 临时读数、vendor 私有寄存器细节、某轮局部解析过程。
8. 哪些地方最容易设计乱，应该怎么收口
   - sensor 层直接读 I2C
     容易乱。sensor/export 层只读缓存，不直接碰硬件。
   - 多个上层各自判断 AC/fault/redundancy
     容易乱。AC/fault 在 per-PSU 状态层统一解释，redundancy 在整机汇总层统一计算。
   - vendor 差异散在各处
     容易乱。vendor 差异只收在“raw -> per-PSU 状态”的解析阶段。
   - 原始值和最终 bit 混着放
     容易乱。原始寄存器只放局部变量，运行态结构里放解析后的统一字段和最终 bit。
   - 把整机状态塞进单颗 PSU 结构
     容易乱。redundancy summary 一定单独放在整机汇总结构体里。
9. 用 present / AC / redundancy 举 2~3 条很简化的数据链路例子
   - present
     GPIO/CPLD -> 线程局部变量 -> 每颗 PSU 运行时状态数组里的 present -> 拼出 PSU status 的 BIT0 -> 写入 sensor/export 缓存 -> 整机汇总读这个数组算 present_count。
   - AC
     STATUS_MFR_SPECIFIC -> 线程局部变量 -> 按 vendor/type 解析 -> 写入每颗 PSU 运行时状态数组里的 ac -> 再转成最终 status 的 BIT3 -> redundancy 汇总只看最终状态数组，不再重新读 PMBus。
   - redundancy
     汇总模块扫描“每颗 PSU 最终状态数组” -> 判断哪些 PSU 同时满足 present=1 且 fault/ac_lost=0 -> 写入整机汇总结构体里的 redundancy 和 healthy_count -> PLR_SENSOR_REDUNDANT_PSU、Redfish、OEM command 都只读这份整机汇总结果。

如果要一句话收口，就是：

- 硬件原始值只在轮询线程本轮局部变量里停留
- 每颗 PSU 的统一结果放“每 PSU 一份的全局状态数组”
- 整机结论放“整机一份的汇总结构体”
- 对外接口只读最终缓存，不自己再去读硬件、也不自己再算一套逻辑