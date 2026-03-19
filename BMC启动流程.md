# 一、先给结论：这次过程总体上是什么情况

这次日志反映的是一个比较典型的 **BMC在线升级 + 升级完成后自动重启 + 从Primary镜像启动 + Linux和业务服务逐步恢复** 的过程。

整体结论：

1. **升级流程确实执行了**
   - 先停服务、保存配置、切根到RAM、写Flash、校验、保留配置、然后触发重启。
2. **重启后U-Boot和内核都正常起来了**
   - U-Boot识别到 AST2600，识别双镜像，最终选择 **Image 1 / Primary side** 启动。
   - Linux 5.4.278 成功启动，rootfs 正常挂载，`/sbin/init` 正常运行。
3. **系统最终基本起来了**
   - IPMI、redis、Event Service、Task Service、Redfish、网络、SSH、VNC、lighttpd 等都起来了。
   - 说明这次升级后 BMC 并没有“起不来”，而是整体恢复到可工作状态。
4. **但过程里有不少警告/错误**
   - 大部分是 **升级窗口期的预期现象**，例如服务被停掉后再访问 socket 失败、Redis 尚未起来、IPMI session 创建失败等。
   - 也有一些值得后续重点关注的问题，比如：
     - `Online Flash Reinitialize failed`
     - `fwimage1corrupted exists`
     - `ub_env` 读取环境失败
     - 某些 I2C/PECI/MCTP 设备初始化异常
     - Redfish resource/privilege 定义缺失告警
     - lighttpd 上传目录不存在
   - 这些不一定导致本次启动失败，但说明系统里有一些边角配置或兼容性问题。

------

# 二、整个过程按阶段拆解

------

## 阶段1：进入BMC固件升级流程

开头这一段：

```
Set watchdog 1 timeout 1200 seconds
BMC Firmware Update - preparing BMC upgrade
PREPARING FOR BMC FIRMWARE UPDATE...
Stopping fail-safe configuration Service...
Flushing IPMI configurations to INI files
Backuping the current configurations
```

### 这一步在干什么

这是升级程序开始执行的阶段，主要做了几件事：

1. **设置看门狗**
   - `Set watchdog 1 timeout 1200 seconds`
   - 防止升级过程中系统卡死，1200秒超时自动复位。
2. **进入升级准备态**
   - 提示 `preparing BMC upgrade`
3. **停止一部分业务服务**
   - fail-safe、NTP、DHCP monitor、VNC、SSH、TELNET、cron、IPMI stack、Redfish、redis 等都被依次停止。
   - 这是为了避免升级时还有业务进程占用文件系统、flash、配置文件。
4. **把运行态配置刷回磁盘**
   - `Flushing IPMI configurations to INI files`
   - 把内存中的配置落盘，避免升级后配置丢失。
5. **备份配置**
   - `Backuping the current configurations`
   - 为升级后恢复配置做准备。

### 这里看到的几个现象

例如：

```
/conf/localtime: Not a Regular File
/conf/if_inet6.txt: Not a Regular File
```

这个一般表示这些路径不是普通文件，可能是链接、特殊文件或者目录。
 **不一定是严重问题**，更像是备份脚本里的提示。

还有：

```
killall: can't kill pid 16496: No such process
killall: rsyncconf.sh: no process killed
killall: rsync: no process killed
kill: you need to specify whom to kill
```

这些大多是 **停服务脚本不够严谨**：

- 目标进程已经退出了
- 或压根没启动
- 或脚本传参为空

这类日志常见，但说明脚本质量一般。

------

## 阶段2：系统停业务、准备切根到RAM并执行刷写

核心日志：

```
FORCEKILL(8): Killing Processes ...
drop_caches: 3
Start unmounting /conf
Start unmounting /bkupconf
Start unmounting /extlog

Switching root to RAM.....
Root FS MTD Blk is /dev/mtdblock2
Copying /dev/mtdblock2 to /var/root.bin ...
Set /var/root.bin to loop device...
mount /dev/loop0 to /initrd
Copying /usr/local/www to /initrd...
Switched root successfully.....
```

### 这一步非常关键

这是升级里很典型的一步：**从原本Flash上的rootfs切换到RAM中的临时rootfs**。

为什么这么做？

因为当前系统就是从 Flash 上运行的，如果直接在“正在使用的rootfs”上刷写，容易出问题。
 所以做法是：

1. 停掉大部分服务
2. 卸载 `/conf`、`/bkupconf`、`/extlog`
3. 把 root 分区内容拷到 `/var/root.bin`
4. 映射成 loop 设备
5. 挂到 `/initrd`
6. **切换 root 到 RAM/临时环境**
7. 然后在这个临时环境里安全地刷写 SPI Flash

### 这说明什么

说明你们这个平台升级机制不是简单“写个镜像然后重启”，而是用了 **online flash + switch root to RAM** 的方案。这个是比较正规的。

------

## 阶段3：刷写和校验Flash

核心日志：

```
VerifyImage
FULL Firmware Upgrade
Error in getting the environment data
Flashing : Skipping flash to env section
DoFlash SUCCESS
FlashVerifying : Skipping verifying env section(f0000)
```

### 这一步说明

1. **执行的是整包升级**
   - `FULL Firmware Upgrade`
2. **镜像写入成功**
   - `DoFlash SUCCESS`
3. **env区没刷/没校验**
   - `Skipping flash to env section`
   - `Skipping verifying env section(f0000)`
4. **uboot环境读取失败**
   - `Error in getting the environment data`

### 这里最值得注意的点

```
ub_env.c:180 Error in getting the environment data
```

这说明升级工具在访问 **U-Boot environment 区** 时有问题。
 但它后面选择“跳过 env 段刷写和校验”，然后主流程继续走下去了。

#### 这代表什么

- **不是致命问题**，因为后面系统还是成功启动了。
- 但说明：
  - env区可能布局不标准
  - env读取接口有问题
  - 或者这个平台就是不打算在线升级env区

如果后面出现：

- 启动变量异常
- 启动镜像切换逻辑异常
- ABR/双镜像选择异常

这个点要重点回来看。

------

## 阶段4：升级后保留配置 / 后处理 / 触发重启

关键日志：

```
Entering Preserve Configuration ...
Error on copying updatefirmware.conf
Preserving the hpm image conf file
Preserving the MAC Backup file for eth0
FLSH_CMD_VERIFY_FLASH DoPostFlashActions
PostFlash Invoked RebootState: 1 flashedimage: 1
Failsafe boot error code fwimage1corrupted exists...
Online Flash Reinitialize failed
Flash All Complete!!
Restarting system...
```

### 这一步做了什么

1. **恢复/保留配置**
   - preserve config
   - 保留 HPM 相关配置
   - 保留 MAC 备份
2. **执行刷写后的动作**
   - `DoPostFlashActions`
3. **重启BMC**
   - `Restarting system`

### 这里几个异常点要分清

#### 1）`Error on copying updatefirmware.conf`

这个多半是配置文件复制失败。
 如果系统后面能起来，一般不是致命，但说明升级配置状态文件处理不完善。

#### 2）`Failsafe boot error code fwimage1corrupted exists...`

这个非常值得注意。

它看起来像是：

- 系统检测到某个“镜像1损坏”的 failsafe 标志文件/错误码存在
- 但实际后面 U-Boot 还是从 **Image 1** 启动成功了

这可能有几种情况：

- 老的错误标记没清掉
- 只是在升级后处理时做了一次检查，但不影响当前镜像
- 或者这个标记逻辑本身有误报

#### 3）`Online Flash Reinitialize failed`

这是这段里最像“真正异常”的一条。
 意思像是升级后尝试重新初始化在线刷写环境失败。

不过注意：

- 紧接着 `Flash All Complete!!`
- 然后系统执行了重启
- 最终又能起起来

所以它**不是致命失败**，更像是“升级收尾阶段的一个失败”，比如：

- 某个状态清理失败
- 某个flash子系统复位失败
- 某个后处理接口失败

这个值得你后面去代码里找 `Online Flash Reinitialize` 的实现位置。

------

## 阶段5：系统重启，进入U-Boot

从这里开始是新一轮启动：

```
reboot: Restarting system
U-Boot 2019.04
SOC: AST2600-A3
RST: WDT1 SOC
FMC 2nd Boot (ABR): Enable, Dual flashes, Source: Primary
Enable WDT2
```

### 这个阶段说明

1. **确实发生了系统重启**
2. **复位原因是 WDT1 SOC**
   - `RST: WDT1 SOC`
   - 说明看门狗或者重启流程最终通过WDT触发了SoC reset
   - 在升级重启场景下这是正常的，不一定表示异常。
3. **平台信息**
   - AST2600-A3
   - ABR（Automatic Boot Recovery）开启
   - 双Flash/双镜像机制开启
   - 当前源是 Primary

### 这是你后面分析双镜像启动很重要的一段

```
FMC 2nd Boot (ABR): Enable, Dual flashes, Source: Primary
```

说明这台BMC有比较典型的 **A/B镜像或主备镜像** 机制。

------

## 阶段6：U-Boot检查双镜像，决定启动哪一个

关键日志：

```
Image 1 ... version: 98.00.00
Image 2 ... version: 2.00.01
Image to be booted is 1
Booting from Primary side
Bootargs = [root=/dev/mtdblock2 ro ip=none console=ttyS4,115200 rootfstype=squashfs imagebooted=1]
```

### 这段非常关键

U-Boot检查了两套镜像：

- **Image 1**
  - version 98.00.00
- **Image 2**
  - version 2.00.01

然后决定：

- **要启动的是 Image 1**
- 从 **Primary side** 启动
- rootfs 是 `/dev/mtdblock2`
- 文件系统是 `squashfs`
- `imagebooted=1`

### 说明什么

1. 升级后系统最终选择了 **Image 1** 启动。
2. 如果你的本意是“把新镜像刷到Image 1，再切到Image 1”，那这和预期一致。
3. 如果你本来想刷 Image 2 再切换，那这里就说明切换没发生。

### 小提示

这里版本号非常不对称：

- Image 1 = `98.00.00`
- Image 2 = `2.00.01`

这很像：

- 一个是厂内版本编号体系
- 一个是常规产品版本号体系
- 或者升级包里版本管理不统一

后面你最好核实一下版本规则，否则做升级策略分析时很容易搞混。

------

## 阶段7：Linux内核启动

从这里开始进入Linux内核：

```
Starting kernel ...
[0.000000] Booting Linux on physical CPU 0xf00
[0.000000] Linux version 5.4.278-ami
...
[4.462782] VFS: Mounted root (squashfs filesystem) readonly on device 31:2.
[4.537436] Run /sbin/init as init process
```

### 这一段总体评价

**内核启动是成功的，而且比较完整。**

你可以把这段概括成：

1. **CPU/内存初始化完成**
2. **设备树加载正常**
3. **保留内存区域建立成功**
4. **串口控制台正常**
5. **SPI NOR Flash识别成功**
6. **MTD分区识别成功**
7. **网络驱动初始化成功**
8. **USB / I2C / I3C / eMMC / RTC / PECI / JTAG 等大量设备驱动加载**
9. **squashfs rootfs 挂载成功**
10. **切入 `/sbin/init`**

### 里面几个值得关注的点

#### 正常点

- `OF: fdt: Machine model: AST2600 EVB`
- `VFS: Mounted root (squashfs filesystem) readonly`
- `Run /sbin/init as init process`

说明：

- dtb OK
- rootfs OK
- init OK

#### 轻微异常/警告点

##### 1）`spi-nor spi1.0: unrecognized JEDEC id bytes: ff ff ff ff ff ff`

第二个SPI设备探测失败。

但因为系统仍然正常识别出拼接的Flash和分区，所以这不一定是主问题，可能是：

- 第二颗Flash未接
- 某片选未启用
- 某个备用通道本来就空着

##### 2）大量 `there is no bus-mode property. use bye-mode as default.`

这个很像驱动打印写得不规范（甚至 `bye-mode` 像拼写错误）。
 表示I2C总线没有配置 `bus-mode` 属性，于是用了默认模式。
 **一般不致命**，但说明设备树配置不够完整。

##### 3）`peci ... Failed to register peci client`

PECI客户端没注册成功。
 如果你的平台有 Intel CPU 温度/遥测需求，这个可能影响 CPU PECI相关功能。

##### 4）`aspeed_crypto ... cannot find sec node`

后面又显示加密加速器成功注册，所以更像某个可选 sec node 不存在，不算大问题。

------

## 阶段8：init脚本和分区挂载

关键日志：

```
INIT: version 2.96 booting
/etc/rcS.d/S03emmcpart.sh
Starting eMMC partitioning...
Partition setup completed successfully
Mounted /dev/mmcblk0p1 to /extlog
Mounted /dev/mmcblk0p5 to /mnt/sdmmc0p5
Mounted /dev/mmcblk0p6 to /bkupconf
```

### 这段说明

系统采用传统 `init + rcS` 脚本方式，而不是 systemd。

这里做了什么：

1. 检查 eMMC 分区布局
2. 确认 p1/p2/p3/p5/p6 都存在且格式化正确
3. 挂载：
   - `/extlog`
   - `/bkupconf`
   - 其他 `/mnt/sdmmc*`

### 这一步很重要

因为你前面升级阶段有：

- preserve config
- extlog
- backup conf

所以这里这些分区重新挂起来，意味着：

- 日志分区
- 配置备份分区
- 升级缓存分区

都逐步恢复了。

------

## 阶段9：核心业务服务启动

这是业务栈恢复的关键阶段。

------

### 9.1 IPMI栈启动

```
Starting IPMI Stack: IPMIMain
Crashdump semaphore created
Init I2c switch device...
```

说明：

- IPMI主栈启动了
- crashdump同步对象建立了
- I2C switch开始初始化

但这里有问题：

```
i2c i2c-0: delete_device: Can't find device in list
Invalid 7-bit I2C address 0x00
set hostname redis connect error
FRU Device Address cannot be found for FRU ID b
Redis connect fail, error: No such file or directory
```

### 这些怎么理解

这几条很典型，别一看到 error 就慌：

#### `Redis connect fail`

这是因为 **此时 redis 还没启动**，后面 redis 会起来。
 所以这属于 **启动时序问题**，不是根因问题。

#### `Invalid 7-bit I2C address 0x00`

这个比较值得关注。0x00 不是合法普通I2C从地址。
 说明：

- 某段脚本/配置传了非法地址
- 或FRU / mux / CPLD配置表有空值/默认值没替换

#### `FRU Device Address cannot be found`

说明某个 FRU ID 到 I2C地址映射没找到。
 这往往和FRU表、板级配置、CPLD/I2C拓扑有关。

------

### 9.2 Redis、Event、Task、Redfish起来

```
Starting redis-server
Starting Event Service
Starting Task Service
Starting Redfish Server
```

这表示：

- Redis socket 恢复了
- Event service恢复
- Task service恢复
- Redfish server恢复

### 这一步说明系统已经从“内核和底层起来”进入到“管理功能上线”。

------

### 9.3 Redfish启动中的告警

有很多类似：

```
require oem... failed
warning: can not find privilege definition ...
warning: can not find entity definition ...
```

### 这类日志说明什么

这是 **Redfish资源建模/权限映射定义不完整**。

意思大致是：

- 某些 OEM 资源模块没找到
- 某些 URI 对应的 handler/resource/entity/privilege 定义缺失

#### 影响

- 不一定影响基础Redfish访问
- 但某些特定OEM接口可能不可用
- 某些菜单/API可能会404、405或者权限判定异常

#### 结论

这不是“系统起不来”的问题，属于 **Redfish模型配置完整性问题**。

------

### 9.4 MCTP / I2C slave / 设备初始化异常

有这些：

```
i2c-slave-mqueue ... not supported by adapter
MCTP_ERROR: Unable to Prepare the Device(s) for ARP
The i2c-0 file cannot found
```

### 这说明

MCTP over I2C 这块并不完全顺。

可能原因：

- 某个adapter不支持slave-mqueue
- 板级I2C拓扑和软件假设不一致
- 某些总线编号变化了
- 某些MCTP设备不支持ARP

如果你们后面要做：

- PLDM
- MCTP over SMBus/I2C
- Host/BMC消息通道

这块要重点排。

------

### 9.5 网络起来

关键日志：

```
Interface eth0 is up
ftgmac100 ... eth0: Link is Up - 1Gbps/Full
DNS Registering ... 10.10.43.141
dhcp6c...
Synchronize clock Success
```

### 说明

网络最终恢复正常了：

- eth0 up
- 千兆链路建立
- IPv4地址拿到了（10.10.43.141）
- DNS注册尝试了
- 时钟同步成功

虽然中间有：

```
Error: argument "eth0" is wrong: table id value is invalid
RTNETLINK answers: File exists
Timeout.....
```

这些像是网络脚本幂等性差或高级路由规则重复配置，
 但因为最终 link up + 地址正常，所以问题不大。

------

### 9.6 SSH / VNC / lighttpd / Web恢复

后面这些：

```
Starting OpenBSD Secure Shell server: sshd.
Starting AMI VNC Server
Starting VNC Server
Starting lighttpd
WEB is enabled and port numbers are NON-SSL:0x50 SSL:0x1bb
```

说明：

- SSH起来了
- VNC/视频重定向起来了
- lighttpd起来了
- Web端口开了（0x50=80, 0x1bb=443）

这说明BMC管理面基本恢复完成。

------

## 阶段10：启动完成

日志后部：

```
Starting Boot Complete.
login:
success getting user info
```

说明BMC已经到了可登录和可服务状态。

------

# 三、你可以把这个过程总结成一条“标准时序线”

下面这段你可以直接抄进笔记。

------

## BMC在线升级 + 重启启动过程总结

### 1. 升级准备阶段

- 设置看门狗超时
- 停止fail-safe、NTP、DHCP、VNC、SSH、TELNET、cron、IPMI、Redfish、redis等业务服务
- 将IPMI运行配置刷回INI
- 备份当前配置

### 2. 升级执行阶段

- 卸载 `/conf`、`/bkupconf`、`/extlog`
- 清理缓存并杀掉剩余进程
- 将当前rootfs复制到RAM并切换root到临时环境
- 验证升级镜像
- 执行整包刷写（FULL Firmware Upgrade）
- 跳过env分区刷写与校验
- 保留/恢复配置

### 3. 升级后处理阶段

- 执行PostFlash动作
- 发现 `fwimage1corrupted` 标记
- `Online Flash Reinitialize failed`
- 升级程序宣布 `Flash All Complete`
- 触发BMC重启

### 4. U-Boot启动阶段

- AST2600-A3上电复位，复位原因为 `WDT1 SOC`
- ABR双镜像机制开启，当前从Primary侧启动
- 检查Image1和Image2版本与CRC
- 最终选择 **Image 1** 启动
- 传递bootargs：`root=/dev/mtdblock2 ro rootfstype=squashfs imagebooted=1`

### 5. Linux内核启动阶段

- Linux 5.4.278-ami 启动
- 设备树、保留内存、串口、SPI NOR、MTD分区、I2C/USB/RTC/eMMC等驱动初始化
- 根文件系统 `squashfs` 成功挂载
- 启动 `/sbin/init`

### 6. 用户态初始化阶段

- rcS脚本检查并挂载eMMC分区
- 恢复 `/extlog`、`/bkupconf` 等分区
- 启动IPMI栈、Execution Daemon、Process Manager
- 后续启动 redis、Event Service、Task Service、Sync Agent、Redfish、MCTP、网络、UART日志、Crashdump、NTP、SSH、VNC、lighttpd 等服务

### 7. 启动完成阶段

- eth0链路建立并获取网络配置
- Redfish / Web / SSH 管理面恢复
- 系统进入可登录可管理状态

------

# 四、这份日志里“正常的错误”和“值得深挖的错误”要分开

------

## A. 属于升级窗口期的正常/可接受现象

这些大多是因为服务被停掉了，或者启动顺序未完成：

- `Failed to create IPMI Session`
- `Connection refused`
- `redis connect fail`
- `could not connect to /run/redis/redis.sock`
- `No such process killed`
- `RTNETLINK answers: File exists`
- DNS 注册 timeout
- `lighttpd upload-dirs doesn't exist`

这些一般不是根因，更多是：

- 服务还没起来
- 脚本重复执行
- 停服务后还有程序在访问旧socket
- 某些目录没准备好

------

## B. 建议你后续重点跟的异常点

### 1）`Online Flash Reinitialize failed`

优先级高。
 去代码里找：

- `Online Flash Reinitialize`
- `DoPostFlashActions`
- `main.c:1389`

你要搞清楚：

- 它失败到底影响什么
- 是不是每次升级后都会报
- 是否影响下一次升级或镜像状态切换

------

### 2）`fwimage1corrupted exists`

优先级高。
 要确认：

- 这个标志文件/状态从哪来的
- 为什么镜像1实际能正常启动，但还存在corrupted标志
- 是否应该在升级成功后被清理

------

### 3）`Error in getting the environment data`

优先级中高。
 说明uboot env读取有问题。
 需要查：

- env区地址和大小是否正确
- 在线升级工具和uboot env布局是否一致
- 当前平台是否故意不使用env区

------

### 4）I2C/FRU异常

包括：

- `Invalid 7-bit I2C address 0x00`
- `FRU Device Address cannot be found`
- `The i2c-0 file cannot found`

优先级中。
 这块会影响：

- FRU读取
- 传感器
- 板卡识别
- MCTP/I2C通信

------

### 5）PECI / MCTP异常

包括：

- `Failed to register peci client`
- `i2c_slave_register: not supported by adapter`
- `MCTP_ERROR... non ARPable Devices`

优先级看业务。
 如果你们产品依赖Host/BMC管理消息或CPU遥测，这块不能放着不管。

------

### 6）Redfish resource/privilege 缺失

优先级中。
 如果你后面做Web/Redfish，就要查：

- privilege定义文件
- entity映射
- OEM resource模块加载路径

------

# 五、从这份日志可以反推出你们平台的一些架构特点

这部分也很有用，你以后看日志会更快。

------

## 1. BMC平台

- SoC：AST2600-A3
- Bootloader：U-Boot 2019.04
- Kernel：Linux 5.4.278-ami
- 用户态：SysV init 风格（不是systemd）

## 2. 存储布局

- SPI NOR 上有双镜像布局
- 至少有：
  - conf
  - root
  - www
- A/B或主备镜像并存
- eMMC用于 extlog / bkupconf / 其他持久化数据

## 3. 文件系统

- rootfs：squashfs，只读
- 可写配置和日志在 eMMC 分区上

## 4. 升级机制

- 在线升级时先切根到RAM
- 再刷SPI Flash
- 保留配置后重启

## 5. 服务栈

- IPMI、Redis、Redfish、Task、Event、MCTP、SSH、VNC、lighttpd、SNMP、LLDP 等都在BMC侧运行

------

# 六、给你一版“笔记版总结”

这个版本最适合你存到自己的md里。

------

## BMC烧录与重启过程笔记

### 升级阶段

1. 设置WDT防卡死
2. 停止主要业务服务
3. 刷写运行配置到INI
4. 备份 `/conf`
5. 卸载 `/conf` `/bkupconf` `/extlog`
6. 切root到RAM临时系统
7. 执行整包升级（FULL Firmware Upgrade）
8. 跳过env区刷写/校验
9. preserve配置
10. PostFlash后重启BMC

### 重启阶段

1. WDT1触发SoC复位
2. U-Boot启动
3. ABR双镜像检测
4. 检查Image1/Image2版本和CRC
5. 最终从Primary / Image1启动

### 内核阶段

1. 内核5.4.278启动
2. SPI NOR/MTD分区识别成功
3. I2C/USB/eMMC/RTC/网络等驱动初始化
4. rootfs=squashfs 成功挂载
5. `/sbin/init` 启动

### 用户态阶段

1. eMMC分区检查并挂载
2. 启动IPMI栈、procmanager、execution daemon
3. 启动redis / event / task / redfish
4. 启动MCTP、网络、UART日志、crashdump、NTP
5. 启动SSH、VNC、Web(lighttpd)
6. 系统进入runlevel 3，启动完成

### 关注异常

- Online Flash Reinitialize failed
- fwimage1corrupted exists
- Error in getting the environment data
- Invalid 7-bit I2C address 0x00
- FRU Device Address cannot be found
- PECI/MCTP初始化异常
- Redfish部分资源/权限定义缺失

### 总结

本次升级和重启整体成功，BMC最终正常启动并恢复网络、Redfish、SSH、Web等管理功能；日志中的大部分报错属于升级窗口期或服务时序问题，但Flash后处理、env区、I2C/FRU、MCTP/PECI和Redfish配置完整性仍需后续排查。