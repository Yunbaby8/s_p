PSU:PMbus I2C 底层实现

**业务层**：我要读 PSU 哪个状态

**协议层**：PMBus 命令码、byte/word/block、PEC

**传输层**：I2C write/read transaction

**系统层**：open、lock、ioctl、close

### 1. 分层图

```text
业务/语义层
  PLR_GetPMBUSStatusInput()
    选择 PSU，判断 present，读取并返回 STATUS_INPUT 状态 byte

PMBus 命令包装层
  PMBus_ReadStatusINPUT()
    固定 PMBus command = STATUS_INPUT

PMBus 协议层
  PMBus_I2CRead()
    查 PMBus 命令表，决定 byte/word/block 长度，处理 PEC，组织 PMBus read

I2C 传输层
  i2c_writeread()
    打开 /dev/i2c-X，加锁，发起一次 write-read

I2C message 组织层
  internal_writeread()
    把 write/read buffer 组织成 I2C message 数组

系统/驱动接口层
  internal_multimsg()
    转成 Linux I2C_RDWR ioctl，真正交给驱动
```

------

### 2. 逐个函数总结

#### PLR_GetPMBUSStatusInput()

- 所属层

业务/语义层。

它已经不是单纯读 I2C，而是在表达：

```text
读取某个 PSU 的 PMBus STATUS_INPUT 状态。
```

- 关键动作

```text
1. 根据 PSUID 判断 PSU 是否 present。
2. absent 就不继续读 PMBus。
3. 从 PSU table 取 BusNum / SlaveAddr。
4. 拼 "/dev/i2c-X"。
5. 调 PMBus_ReadStatusINPUT()。
6. 把 ReadBuf[0] 作为 STATUS_INPUT 输出。
```

它负责“决定读哪个 PSU”和“把返回值当成输入状态”。

- 关键参数

```text
PSUID:
  输入，上层传入，表示 PSU 编号。

Status:
  输出，返回 STATUS_INPUT 的 1 byte。

BusNum / SlaveAddr:
  全局表项里的配置值。

ReadBuf[0]:
  局部 buffer 的第一个 byte，最终成为 *Status。
```

- 参数来源

```text
PSUID:
  上层传入，例如 sensor monitor 根据 SensorNum 算出来。

BusNum:
  g_PLR_PSUDeviceInfo_t[PSUID].BusNum。

SlaveAddr:
  g_PLR_PSUDeviceInfo_t[PSUID].SlaveAddr。

PresentStatus:
  PLR_GetPSUPresentStatus()。
  它可能来自 GPIO / CPLD / 9555，不一定来自 PMBus。
```

- 易错点

```text
1. 这个函数已经做业务判断。
   如果 PSU 被判 absent，不会发 PMBus read。

2. Present 状态不是 STATUS_INPUT。
   Present 可能来自板级 GPIO/CPLD/9555。

3. 这里的 SlaveAddr 没有 >>1。
   说明这条路径期望 g_PLR_PSUDeviceInfo_t[] 里存的是 7-bit 地址。

4. 它只取 ReadBuf[0]。
   所以它默认 STATUS_INPUT 是 byte 类型。
```

------

#### PMBus_ReadStatusINPUT()

- 所属层

PMBus 命令包装层。

- 关键动作

```text
1. 接收 i2c_dev、slave、read_buf。
2. 固定 command code = STATUS_INPUT。
3. 调 PMBus_I2CRead()。
```

它只负责“这次读的是哪个 PMBus command”。

- 关键参数

```text
i2c_dev:
  输入，"/dev/i2c-X"。

slave:
  输入，PSU I2C slave address。

read_buf:
  输出，接收 PMBus 返回数据。

STATUS_INPUT:
  固定 command code，值是 0x7C。
```

- 参数来源

```text
i2c_dev:
  上层 PLR_GetPMBUSStatusInput() 拼出来。

slave:
  上层从 g_PLR_PSUDeviceInfo_t[] 取出。

STATUS_INPUT:
  libpmb.h 里的 PMBus command enum。
```

- 易错点

```text
1. 它不是业务解释层。
   不判断 54V/12V，不判断 vendor，不解释 bit。

2. 它不决定读 1 byte 还是 2 bytes。
   长度由 PMBus_I2CRead() 查命令表决定。

3. 如果不同 PSU 对 STATUS_INPUT bit 定义不同，问题不在这个函数解决。
```

------

#### PMBus_I2CRead()

- 所属层

PMBus 协议层。

- 关键动作

```text
1. 初始化 PMBus command table。
2. 必要时读取 CAPABILITY 检测 PEC。
3. 根据 cmdCode 查命令属性。
4. 检查 command 是否允许 read。
5. 根据 dataBytes 决定 readlen。
6. 组织 writebuf。
7. 调 i2c_writeread()。
8. 如启用 PEC，校验 PEC。
9. 把内部 readbuf 拷贝给调用者 buf。
```

对于这条路线的 `STATUS_INPUT`：

```text
cmdCode  = 0x7C
dataType = PMB_BYTE
writebuf = [0x7C]
writelen = 1
readlen  = 1
```

如果 PEC 开启：

```text
readlen = 2
第 1 byte 是 STATUS_INPUT 数据
第 2 byte 是 PEC
```

- 关键参数

```text
i2c_dev:
  输入，上层传入。

slave:
  输入，I2C slave address。

cmdCode:
  输入，本路线是 STATUS_INPUT。

buf:
  输出为主。
  byte/word read 时是输出。
  block/raw read 时 buf[0] 还会作为输入长度。

writebuf:
  局部，普通 read 下就是 [cmdCode]。

readlen:
  局部，由 command table 和 PEC 状态决定。

PEC_Verify:
  全局状态。
```

- 参数来源

```text
cmdCode:
  PMBus_ReadStatusINPUT() 传入。

dataBytes:
  g_PMBusCmds[cmdCode] 查表得到。

readlen:
  dataBytes 推导；PEC 开启则 +1。

PEC_Verify:
  全局状态，由 PECEnable() 根据 CAPABILITY 设置，也可能被其他调用影响。
```

- 易错点

```text
1. PEC_Verify 是全局，不是 per PSU/per slave/per bus。
   这是危险设计。

2. block/raw read 的 buf[0] 是输入长度。
   byte/word read 的 buf 是输出。
   同一个参数语义随 command 类型变化。

3. 如果 cmdCode 查不到，代码没有立即失败，存在防御缺口。

4. 这个函数只组织 PMBus read，不解释 STATUS_INPUT bit。

5. 它不设置 PAGE。
   如果 command 依赖 PAGE，默认 page 固定这个前提很危险。
```

------

#### i2c_writeread()

- 所属层

I2C 传输层。

- 关键动作

```text
1. 打开 i2c_dev。
2. 获取 I2C access lock。
3. 调 internal_writeread()。
4. 释放 lock。
5. 关闭 fd。
```

它负责“发起一次 I2C write-read”，但还没有直接组 Linux `i2c_msg`。

- 关键参数

```text
i2c_dev:
  输入，"/dev/i2c-X"。

slave:
  输入，传给 Linux I2C message 的地址。

write_data:
  输入，本路线是 [0x7C]。

read_data:
  输出，读回数据写到这里。

write_count:
  输入，本路线是 1。

read_count:
  输入，本路线是 1，PEC 开启时是 2。
```

- 参数来源

```text
i2c_dev/slave:
  上层一路传下来。

write_data/write_count/read_count:
  PMBus_I2CRead() 组织出来。
```

- 易错点

```text
1. 它不关心 slave 是 7-bit 还是 8-bit。
   上层传错，它照发。

2. 它不处理 PEC。
   绕过 PMBus_I2CRead 直接调用 I2C read/write，就没有 PEC 逻辑。

3. 它不懂 PMBus command。
   I2C 成功不代表 PMBus 语义正确。
```

------

#### internal_writeread()

- 所属层

I2C message 组织层。

- 关键动作

```text
1. 如果 write_count > 0，创建一个 WRITE message。
2. 如果 read_count > 0，创建一个 READ message。
3. 把 message 数组交给 internal_multimsg()。
```

对 PMBus read 来说，它组织出来的是：

```text
message[0]:
  WRITE slave, data=[cmdCode]

message[1]:
  READ slave, len=read_count
```

- 关键参数

```text
slave:
  输入，I2C address。

write_data:
  输入，本路线是 command code buffer。

read_data:
  输出，挂到 READ message 上，之后由 driver 填充。

write_count/read_count:
  输入，决定 message 是否存在和长度。

i2cdev:
  输入，已经打开的 fd。
```

- 参数来源

```text
slave/write_data/read_data/count:
  来自 i2c_writeread()。

i2cdev:
  i2c_writeread() open 得到。
```

- 易错点

```text
1. read_count 如果上层算错，这里不会校验 PMBus 合理性。

2. write_count/read_count 为 0 会影响 message 数。
   PMBus read 正常应该是 write_count=1, read_count>0。

3. 这里仍然不懂 PMBus，只是在组织 I2C message。
```

------

#### internal_multimsg()

- 所属层

系统/驱动接口层。

- 关键动作

```text
1. 创建 i2c_rdwr_ioctl_data。
2. 把内部 i2c_message 转成 Linux struct i2c_msg。
3. 设置 addr / len / buf / flags。
4. READ message 设置 I2C_M_RD | I2C_M_NOSTART。
5. WRITE message flags = 0。
6. 调 ioctl(I2C_RDWR)。
```

这是这条路线真正进入 Linux I2C driver 的位置。

- 关键参数

```text
messages:
  输入/输出。
  READ message 的 data buffer 会被填充。

msgcount:
  输入，PMBus read 通常是 2。

i2cfd:
  输入，/dev/i2c-X 的 fd。

do_wait:
  输入，是否等待 bus ready。
```

- 参数来源

```text
messages:
  internal_writeread() 组织。

msgcount:
  internal_writeread() 根据 write_count/read_count 计算。

i2cfd:
  i2c_writeread() 打开的 fd。
```

- 易错点

```text
1. 这是纯驱动接口层。
   它不理解 PMBus，不理解 STATUS_INPUT。

2. READ flags 用了 I2C_M_RD | I2C_M_NOSTART。
   如果某些 adapter/device 对 repeated-start / no-start 行为敏感，这里是排查点。

3. ioctl 成功只代表 transaction 完成。
   不代表数据语义正确。
```

------

### 3. 整条 PMBus read 路线的高风险点

```text
1. slave 地址 7-bit / 8-bit
   这条 STATUS_INPUT 路线里，g_PLR_PSUDeviceInfo_t[PSUID].SlaveAddr 直接传到底层。
   因此它应当是 7-bit。
   但其他路径会对 SlaveAddr >> 1，说明项目里并非所有 SlaveAddr 字段语义一致。

2. STATUS_INPUT 是 PMBus register，Present 不一定是
   PLR_GetPMBUSStatusInput() 读 PMBus STATUS_INPUT 前先查 Present。
   Present 可能来自 GPIO/CPLD/9555。
   调试时不要把 Present 和 PMBus STATUS_INPUT 混成一个来源。

3. 54V / 12V / vendor-specific bit
   同一个 PMBus command 或 bit 在不同 PSU 类型下不一定语义一致。
   S6210A 的 AC status 已经按 PSU type 分支解释，这说明不能默认所有 PSU bit 定义一致。

4. global type 和 per-PSU type 混用
   项目有 g_PLR_PSUTypeList[]，也有 g_PLR_PSUType。
   如果业务判断用了 global type，混插场景会有误解释风险。

5. PEC 是全局状态
   PMBus_I2CRead() 的 PEC_Verify 是 libpmb.c 文件级全局。
   它不区分 bus/slave/PSU。
   这是高风险设计。

6. block read 的 buf[0]
   在 byte/word read 里 buf 是输出。
   在 block/raw read 里 buf[0] 是输入长度，读完又变成输出内容。
   接口语义不统一，容易踩坑。

7. PAGE 隐含状态
   PMBus_I2CRead() 不自动设置 PAGE。
   如果某个 PSU command 依赖 page，默认 page 固定就是危险前提。

8. 传输成功不等于语义正确
   internal_multimsg() 成功只说明 I2C_RDWR 成功。
   地址、command、readlen、PEC、page、vendor bit 都可能仍然错。

9. 绕过 PMBus_I2CRead 的路径没有 PEC/table 保护
   PLR_GetPMBUSReading() 这种 raw I2C read 自己指定 command/readlen。
   它不会查 PMBus 命令表，也不会处理 PEC。
```

------

### 4. Checklist

```text
1. 先判层级：
   当前代码是在业务解释、PMBus 协议、I2C message，还是 ioctl 层？

2. 查地址：
   SlaveAddr 来源是什么？当前是 7-bit 还是 8-bit？有没有重复 >>1？

3. 查 command：
   command code 是 wrapper 固定、业务层传入，还是 table 查出？

4. 查长度：
   readlen 来自 PMBus table、ReadLen 参数、buf[0]，还是 PEC +1？

5. 查状态来源：
   这个状态来自 PMBus register，还是 GPIO/CPLD/9555？

6. 查语义解释：
   返回 byte 在哪里被解释？是否区分 54V/12V/vendor？

7. 查全局状态：
   PEC_Verify、g_PLR_PSUType、g_PLR_PSUStatus_t 是否影响当前 PSU？

8. 查 block 特殊用法：
   如果是 block/raw，调用前 buf[0] 是否正确设置？

9. 查 PAGE：
   command 是否依赖 PAGE？调用前有没有设置？

10. 查绕路：
   是否绕过 PMBus_I2CRead 直接调用 i2c_writeread/HAL_I2C_RW？
   如果是，就不要期待 PEC 和 PMBus command table 保护。
```

最简理解：这 6 个函数不是“同一层的调用链”，而是从语义一路降到驱动接口。`PLR_GetPMBUSStatusInput()` 决定业务要读什么，`PMBus_ReadStatusINPUT()` 固定 PMBus command，`PMBus_I2CRead()` 组织协议，`i2c_writeread()` 管设备和锁，`internal_writeread()` 组织 message，`internal_multimsg()` 才真正把 transaction 交给 Linux I2C driver。这一版 收尾的那个最简理解我现在感觉和我想让codex做的事情稍微有点不符了，是不是应该改改，符合程度呢