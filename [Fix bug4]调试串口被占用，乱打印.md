**问题分析记录**

**1. 问题标题**
U7920I6 的 PSU present detect 在 Polaris debug 场景下触发 GPIO/pinctrl 资源冲突，导致串口持续刷内核 log。

**2. 现象描述**

- 串口持续反复打印类似日志：

```
aspeed-g6-pinctrl ... pin M24 already requested by 1e650010.mdio; cannot claim for 1e780000.gpio:0 status -22 
```

- 日志频率高，基本淹没正常串口调试信息。
- 现象不是一次性报错，而是会持续重复。
- 从日志形式看，属于某条 GPIO 路径不断尝试申请一个已经被其他功能占用的 pin。

**3. 影响范围**

- 直接影响：串口调试体验，难以查看真实业务日志。
- 间接影响：PSU present / status 检测路径可能持续异常重试。
- 潜在影响：PSU 状态、告警、SEL、冗余判断都可能被污染，尤其是 PSU6~12 这些逻辑位点。

**4. 变更背景**

- 目标代码平台是 Sirius。
- 当时为了验证 Sirius 的 PSU present detect 逻辑，借用了 Polaris 的硬件。
- 因此引入了以 ERICZHOU... 开头的调试宏，把 Sirius 的 PSU 逻辑改成“在 Polaris 上可运行”的一套路径。
- 这类改动不只是业务逻辑判断，还会影响底层监控资源的选择。

**5. 初步结论**
这次问题的根因不是“PSU present 逻辑主动去改 pinmux”，而是：

- U7920I6 仍按 12 个 PSU 运行。
- 但 ERICZHOU_SIRIUS_PSU_DEBUG_ON_POLARIS 分支下，PSU 监控表只初始化了前 5 个 PSU。
- 后 7 个 PSU 条目被 C 自动补零。
- 补零后的监控项会被解释成 MonMethod = GPIO、GPIONum = 0 = GPIOA0。
- GPIOA0 对应 AST2600 的物理 pin M24。
- M24 又已经被 1e650010.mdio 对应的 MDIO3 功能占用。
- 所以每次 PSU 轮询访问到这些条目时，都会触发一次 GPIO claim 失败并刷 log。

**6. 代码链路分析**

上层触发链路：

- PLR_DetectPSUType() 会遍历所有 PSU：
  - PLR_PSU.c (line 322)
- 遍历条件直接使用 PLR_PSU_MAX_NUM：

```
for( psuIndex = 0; psuIndex < PLR_PSU_MAX_NUM; psuIndex++ ) 
```

present 检测链路：

- PLR_GetPSUPresentStatus() 读取 g_PLR_PSUPresentMonMethod_t[PSUID]
  - PLR_PMBUS.c (line 223)
- PLR_GetPSUTotalStatus() 根据 MonMethod 分支
  - PLR_PMBUS.c (line 147)
- 当 MonMethod == PLR_PSU_INFO_MON_GPIO 时，会走：

```
ret = PLR_GPIOOperate( PSUMonMethod.GPIONum, PLR_GET_GPIO_DATA, status ); 
```

- PLR_PMBUS.c (line 161)

GPIO 落地链路：

- PLR_GPIOOperate() 把 GPIO 编号原样传给 HAL
  - PLR_GPIO.c (line 38)
- HAL 最终调用 get_gpio_data(u16GPIONumber)
  - PLR_AMI_API.c (line 172)

也就是说，只要监控表里出现 GPIONum = 0，最终就真的会去访问 GPIO 0。

**7. 根因分层**

业务层：

- PSU present detect 业务目标本身没有直接证据表明有逻辑错误。
- 真正异常的是它在 debug 场景下被导向了错误的底层资源。

配置层：

- PLR_PSU_MAX_NUM 是 12，但 debug 分支只初始化了 5 个 PSU 的监控表。
- 这是这次问题最核心的直接根因。

平台层：

- Sirius 逻辑跑在 Polaris 硬件上，本身就需要平台适配。
- ERICZHOU... 宏把路径切到了 Polaris 资源，但没有把 12 个逻辑 PSU 的边界处理完整。

底层资源层：

- 后 7 个 PSU 默认掉到了 GPIOA0。
- GPIOA0 对应 M24。
- M24 已被 MDIO3 占用，所以 GPIO claim 失败。

**8. 关键证据**

证据 1：平台仍然定义为 12 个 PSU

- U7920I6_Sirius_PSU_Table_Info.h (line 55)

```
#define PLR_PSU_MAX_NUM 12 
```

证据 2：Polaris debug 分支只初始化了前 5 项

- g_PLR_PSUDeviceInfo_t
  - U7920I6_Sirius_PSU_Table_Init.h (line 23)
- g_PLR_PSUACOKMonMethod_t
  - U7920I6_Sirius_PSU_Table_Init.h (line 73)
- g_PLR_PSUDCOKMonMethod_t
  - U7920I6_Sirius_PSU_Table_Init.h (line 108)
- g_PLR_PSUPresentMonMethod_t
  - U7920I6_Sirius_PSU_Table_Init.h (line 143)

证据 3：未初始化条目会被补成全 0，而全 0 对应 GPIOA0

- 结构体字段顺序：
  - PLR_PSU_Def.h (line 28)
- 枚举 0 = PLR_PSU_INFO_MON_GPIO
  - PLR_PSU_Def.h (line 57)
- GPIO 编号 0 = GPIOA0
  - PLR_GPIO_Map.h (line 16)

证据 4：GPIOA0 对应 M24

- pinctrl-aspeed-g6.c (line 62)

```
#define M24 0 PIN_DECL_2(M24, GPIOA0, MDC3, SCL11); 
```

证据 5：1e650010.mdio 绑定的是 MDIO3 组，而 MDIO3 使用 M24/M25

- SoC DTS:
  - aspeed-g6.dtsi (line 311)
- pinctrl 组定义：
  - aspeed-g6-pinctrl.dtsi (line 465)
- pinctrl 源码：
  - pinctrl-aspeed-g6.c (line 67)

证据 6：日志里的资源所有者完全对得上

- 1e650010.mdio 是 MDIO 控制器 owner。
- 1e780000.gpio:0 是 GPIO 控制器想申请 gpio 0。
- gpio 0 -> GPIOA0 -> M24，而 M24 已经被 MDIO 占用。

**9. 现象为什么会持续刷屏**

- PLR_DetectPSUType() 不是只跑一次，而是会不断轮询。
- 只要循环仍遍历到 PSU6~12，就会继续访问那些补零条目。
- 每访问一次，都会尝试把 GPIOA0 作为 GPIO claim 一次。
- 每 claim 一次，内核就报一次 pinctrl 冲突。
- 所以日志会持续刷屏，而不是偶发。

**10. 最终根因总结**
这次 bug 的根因可以完整表述为：

为了在 Polaris 上验证 Sirius 的 PSU present detect，代码启用了 ERICZHOU... 调试宏，把 U7920I6 的 PSU 监控资源切到了 Polaris 调试分支。但该分支只初始化了前 5 个 PSU 的监控表，而系统仍按 PLR_PSU_MAX_NUM = 12 去轮询全部 PSU。结果后 7 个 PSU 条目被 C 自动补零，最终被解释成 MonMethod = GPIO、GPIONum = 0 = GPIOA0。GPIOA0 对应 AST2600 的物理 pin M24，而 M24 已经被 1e650010.mdio 对应的 MDIO 功能占用，因此每次轮询都会触发一次 GPIO claim 失败，最终在串口上持续刷出 pinctrl 冲突日志。

**11. 解决方案**

临时止血：

- 在 Polaris debug 路径里，对 PSUID >= 5 直接返回 ABSENT。
- 或者只让检测循环在 debug 场景遍历前 5 个 PSU。
- 这能最快停止 GPIOA0 的错误访问。

正式修复：

- 不建议直接把 PLR_PSU_MAX_NUM 永久改成 5，因为 12 是 U7920I6 的真实平台定义。
- 推荐在 Polaris debug 分支中显式处理 PSU6~12，不要让它们继续依赖隐式零初始化。
- 最稳妥的方向是：让这些条目在 debug 场景中显式无效，或显式不参与检测。

不推荐方案：

- 把 PSU6~12 永久复用到前 5 个真实底层资源。
- 这样能压住 GPIOA0 冲突，但会把多个逻辑 PSU 映射到同一真实资源，后续可能引发状态镜像、重复告警、冗余判断失真。

**12. 风险评估**

- 如果只做止血，不修根因，后续换路径或扩功能时仍可能复发。
- 如果直接改 PLR_PSU_MAX_NUM = 5，会影响真实 Sirius 的 12 PSU 语义，可能波及 sensor、SNMP、IPMI、表项定义。
- 如果用“复用前 5 个底层资源”的方式长期保留，问题会从“GPIO claim 冲突”变成“多个 PSU 共享同一资源”的逻辑污染。

**13. 验证方法**

- 在 PLR_GetPSUTotalStatus() 临时打印 PSUID / MonMethod / GPIONum / Bus / SlaveAddr。
- 重点观察 PSUID = 5~11 是否出现：
  - MonMethod = 0
  - GPIONum = 0
- 做 A/B 验证：
  - 让 PSUID >= 5 直接返回 ABSENT
  - 看 M24 / gpio:0 log 是否立即消失
- 如果日志消失，就基本坐实这条根因链。
- 验证完成后再检查：
  - 串口是否恢复可用
  - present/status 是否正常
  - 是否引入新的告警或状态重复

**14. 下次遇到同类问题的排查方法**

先看日志：

- 把 pin 名、gpio 编号、owner 抄完整。
- 这次的三个关键字就是：M24、gpio:0、1e650010.mdio。

先反推资源：

- 查 pinctrl 源码，看 pin -> GPIO/功能 映射。
- 查 GPIO map，看 gpio 编号 -> GPIOA0/B0... 映射。
- 查 DTS，看 owner 为什么会占住这个 pin。

再找上层访问者：

- 找谁在调用 PLR_GPIOOperate()。
- 找谁在周期性轮询这个状态。
- 找最终使用的表项来自哪一张初始化表。

重点检查表驱动问题：

- 数组大小和平台定义是否一致。
- 初始化项是否写满。
- 未初始化项默认值会落到哪里。
- 0 是否恰好是一个合法且危险的资源值。

宏影响审计：

- 查看 debug 宏改了哪些表。
- 确认是否存在“上层按 12 跑、下层只配 5”的不一致。
- 多个宏如果成组出现，不要只看一个。

最小 A/B 验证：

- 优先切断可疑 index 或可疑分支。
- 不要一开始就大改平台基础定义。
- 先证明链路，再决定最终修法。