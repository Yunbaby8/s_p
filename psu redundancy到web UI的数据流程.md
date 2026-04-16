**1. 状态采集层**
最底下先去采 PSU 的真实状态，比如：

- 在不在位
- AC/DC 是否正常
- 有没有 fault
- 有没有 predictive fault

这些原始状态先被整理成“单颗 PSU 的状态 bit”。

**2. 项目监控层**
然后项目自己的监控逻辑会周期性跑一遍，把这些单颗 PSU 状态汇总起来。

在这里会做两件事：

- 先算每颗 PSU status sensor 的缓存值
- 再根据这些 PSU status 去算 `Redundancy_STAT` 这种聚合虚拟 sensor 的缓存值

也就是说，这一层已经把 `225` 该是什么值算出来了，只是先放在一份缓存里。

**3. 传感器主监控框架**
接下来系统的通用 sensor monitor 会轮询每颗 sensor。

对普通物理 sensor，它通常可以直接从底层读到值。  
但像 `Redundancy_STAT` 这种虚拟 logical sensor，本身没有真实硬件可读，所以它不能靠“直接读设备”拿值。

这类 sensor 正常要靠一个“钩子路径”来接管。

**4. 钩子函数层**
这里就是关键的一步。

钩子函数的作用是：

- 把项目监控层前面算好的缓存值
- 写回到系统真正维护的 sensor 运行时结构里

也就是把“项目缓存里的值”同步成“系统正式 sensor 值”。

如果这一步没挂上，前面虽然算出来了，后面系统还是读不到。

你这次最开始的问题，核心就出在这里：
- 虚拟 sensor 的值算出来了
- 但没有通过钩子函数灌回系统正式 sensor 数据区

所以后面全是 0。

**5. 共享内存 / 运行时 sensor 数据区**
钩子函数把值写回去之后，系统就把这颗 sensor 的正式读值保存在一块共享的运行时数据区里。

后续别的模块再查 sensor，不是去看项目缓存，而是看这里。

所以这一步相当于把“项目私有计算结果”变成“平台公共 sensor 状态”。

**6. IPMI 打包层**
再往上，IPMI 的配置/消息处理层会从这块运行时数据区里取值。

然后根据这颗 sensor 的类型，把读值打包成标准 IPMI 返回格式，比如：

- threshold sensor 怎么打
- discrete sensor 怎么打
- 哪些 bit 要保留
- 哪些保留位要补上

你这颗 `Redundancy_STAT` 属于离散 sensor，所以在这里会做离散状态的封装。

**7. libipmi / REST 输出层**
再上层会把这个已经打包好的 sensor 读值转成网页接口需要的结构，比如：

- `sensor_number`
- `reading`
- `sensor_state`
- `discrete_state`

然后通过 `/api/sensors` 或 `/api/detail_sensors_readings` 回给前端。

**8. 前端解析层**
最后网页前端拿到这几个字段后，再根据：

- sensor 编号
- reading 里的 bit
- 对应字库

把它翻译成页面上的文字。

比如你现在这颗 `225`，前端会按 bit 去解释：
- 哪个 bit 表示 54V redundancy
- 哪个 bit 表示 12V redundancy

再拼成最终显示文案。

**一句话把整条链路串起来**
原始 PSU 状态  
-> 项目监控逻辑算出虚拟 sensor 缓存值  
-> 钩子函数把缓存值同步到系统正式 sensor 数据区  
-> IPMI 层从正式数据区取值并打包  
-> REST 接口返回给网页  
-> 前端按 bit 和字库翻译显示

**你这次“数据链路没通”具体卡在哪**
卡在中间这一步：

- “项目监控逻辑已经算出缓存值”
- 但“钩子函数没有把它同步到系统正式 sensor 数据区”

所以后面的 IPMI、REST、前端看到的就一直是空值或默认值。

**1. 先有“平台定义表”**
系统一开始并不是直接知道 225 是什么，它要先靠几张定义表把传感器身份建立起来。

这里最关键的是几类表：

- sensor number table
  定义每个逻辑 sensor 编号代表什么。
  例如 225 代表 Redundancy_STAT。
- PSU table
  定义每个 PSU index 对应哪条 I2C bus、哪个地址、哪个 CPLD 寄存器/bit、极性是什么。
  也就是“去哪里读这个 PSU 的 present / AC / DC / fault”。
- sensor hook table
  定义哪些 sensor 需要挂 pre/post monitor hook。
  这对虚拟 sensor 很关键，因为它不是直接从硬件读出来的。
- SDR / sensor record
  定义这颗 sensor 对外在 IPMI 世界里是什么类型、允许哪些离散 bit、读值怎么被解释。

也就是说：
**number table 决定它是谁，PSU table 决定底层怎么读，hook table 决定怎么接入 monitor，SDR 决定上层怎么解释。**

**2. 底层先按 PSU table 去采单颗 PSU 状态**
到了真正取值时，系统不会直接去读 Redundancy_STAT 这种虚拟 sensor，而是先读单颗 PSU 的原始状态。

这一步会参考 PSU 相关 table 里的配置，去拿：

- 是否 present
- AC 是否正常
- 输入是否丢失
- 是否 fault
- 是否 predictive fault

这些状态最后会被整理成一份“单颗 PSU 的状态 bit”，通常会放进类似这种结构里：

- g_PLR_PSUStatus_t[i].IsFault

这里的 i 是 PSU index，不是 sensor number。

所以这一步结束后，系统已经知道：

- PSU0 好不好
- PSU1 好不好
- …
- PSU11 好不好

或者在 Polaris debug 模式下，实际上只会反复读 0~3 这几个实际 PSU index。

**3. 项目监控层再用这些单颗 PSU 状态算“虚拟 sensor”**
接着项目监控逻辑会周期性运行，把单颗 PSU 状态汇总成更高层的逻辑 sensor。

Redundancy_STAT 就是典型的虚拟/聚合 sensor。

它不是自己去读硬件，而是根据前面那张 PSU 状态表去统计：

- 54V 有多少颗 good
- 12V 有多少颗 good

这里“good”不是简单地说这个 sensor 页面看起来正常，而是要同时满足业务条件，比如：

- present
- 没有 failure
- 没有 input lost

然后再按规则决定：

- fully redundant
- degraded
- lost

最后把这个结果编码成一个 bitfield，先放到项目自己的缓存里。

也就是说，这时 225 的值已经算出来了，但它**还只是项目层的缓存值**，还没变成系统正式 sensor 值。

**4. monitor 框架轮询 sensor 时，要靠 hook 把缓存值灌回去**
系统的通用 sensor monitor 在轮询时，会去处理每颗 sensor。

对于普通物理 sensor：

- 它可以直接通过 HAL/device logic 读到值

但对于 225 这种虚拟 sensor：

- 它本身没有真实硬件
- 它的 device logic 可能是空的
- 所以它必须依赖 hook

这时候 hook table 就起作用了。

如果 hook table 里注册了这颗 sensor：

- monitor 跑到它时，就会调用对应的 post-monitor hook
- hook 再把“项目缓存值”复制到“系统正式 sensor 结构”

如果没注册：

- monitor 虽然知道有这颗 sensor
- 但不会有人把缓存值同步进去
- 那系统正式 sensor 里就还是 0

你这次第一个大问题就卡在这里：
**缓存里有值，但因为 hook 没挂上，正式 sensor 数据区还是空的。**

**5. 正式 sensor 数据区里除了 reading，还有一堆“解释规则”**
一旦 hook 把值同步进去了，这颗 sensor 在系统运行时结构里就会同时带着几类信息：

- SensorReading
- EventTypeCode
- SettableThreshMask
- EventFlags
- 其他 sensor 属性

这里最关键的是：

- SensorReading
  当前真实值，比如你后面看到的 0x0108
- EventTypeCode
  它是 threshold sensor 还是 discrete sensor，决定后面怎么打包
- SettableThreshMask
  哪些 bit 是这颗 sensor 允许上报/解释的合法位

这个 SettableThreshMask 不是随便来的，它本质上来自 **SDR 定义的离散读值 mask**。

也就是说：
**真正的 bit 过滤规则，是 SDR 决定的。**

**6. IPMI 打包时会按 SDR mask 做过滤**
接下来 IPMI/AMI 这一层会从正式 sensor 数据区取值，然后根据 sensor 类型打包。

对于离散 sensor，逻辑大概是：

- 取 SensorReading 的低 8 bit
- 和 SettableThreshMask 的低 8 bit 做 &
- 结果放进 ComparisonStatus

再：

- 取 SensorReading 的高 8 bit
- 和 SettableThreshMask 的高 8 bit 做 &
- 结果放进 OptionalStatus

然后还会补一个离散 sensor 的保留位：

- OptionalStatus |= 0x80

所以这里发生了两件事：

1. **掩码过滤**
   只有 SDR 允许的 bit 能留下来。
   不在 mask 里的 bit 会被清掉。
2. **标准保留位补位**
   离散 sensor 的某些保留位会被强制置 1。

这就是为什么你之前会看到：

- 业务值是 0x0108
- 最后前端拿到的是 0x8108

其中：

- 0x0108 是你的业务 bit
- 0x8000 是离散 sensor 保留位带上去的

**7. 这个 mask 到底根据什么来的**
对你这颗 225，掩码规则来自它自己的 SDR 记录。

SDR 里会定义这颗 sensor 的：

- sensor type
- event/reading type
- assertion/deassertion mask
- discrete reading mask / settable mask

这决定了：

- 哪些 bit 是合法状态位
- 上层应该把它当成哪种离散 sensor 来解释

所以不是前端随便定义几个 bit 就完了，必须是：

- 你项目层算出来的 bit
- SDR 允许的 bit
- IPMI 打包层保留下来的 bit

这三层都一致，最终显示才会对。

**8. REST / web 层只看打包后的结果**
再往上走，REST 层把 IPMI 打包后的 sensor 信息转成网页接口字段，比如：

- sensor_number
- reading
- sensor_state
- discrete_state

这时候网页已经不再看你项目层缓存了，它只看这里返回的值。

所以你前面抓到：

- reading = 33032
- 也就是 0x8108

说明链路已经完整打通到这里。

**9. 前端最后按 sensor 编号和 bit 做翻译**
最后前端拿到 reading 后，会根据：

- sensor_number == 225
- 读值里的 bit
- 对应字库

去翻译成页面上的文案。

比如现在这颗 225，前端 special case 解析的是：

- BIT3
- BIT8

于是显示成：

- 54V PSU: 非冗余，资源缺失
- 12V PSU: 非冗余，资源缺失

**10. 你这次整个问题为什么会出现**
把整条链路串起来，其实你这次卡了两个点：

第一阶段卡的是：

- 项目监控层已经算出 225 的缓存值
- 但 hook 没把它同步进正式 sensor 数据区
- 所以后面的 IPMI / REST / 前端都只看到 0

第二阶段卡的是：

- 在 Polaris 上做 Sirius 验证时，单颗 PSU status 的取值索引做了复用
- 但 redundancy 聚合统计一开始没有完全按同样的复用语义去算
- 所以前后看起来不一致

**一句话版全链路**
平台先靠 number table / PSU table / hook table / SDR table 定义 sensor 身份和取值规则；底层按 PSU table 采单颗 PSU 状态，项目监控层把它们汇总成虚拟 sensor 缓存值，hook 再把缓存值同步进正式 sensor 数据区，IPMI 层按 SDR 的 mask 过滤并打包成离散读值，REST 回给前端，前端再按 sensor 编号和 bit 文案显示。