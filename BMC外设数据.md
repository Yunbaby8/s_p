

```
计算机存储体系

        CPU
         │
     ┌───┴────┐
     │        │
   RAM     非易失存储
           │
   ┌───────┼────────┐
   │       │        │
 Flash   eMMC     硬盘
（SPI）           （SSD/HDD）
```

| 项目           | SPI Flash（你现在的 mtd） | eMMC（mmcblk） | 硬盘（SSD/NVMe） |
| -------------- | ------------------------- | -------------- | ---------------- |
| 容量           | MB级（16~256MB）          | GB级（8~64GB） | TB级             |
| 速度           | 慢                        | 中等           | 很快             |
| 写入次数       | 少（寿命有限）            | 中等           | 高（有磨损均衡） |
| 接口           | SPI                       | MMC            | PCIe / SATA      |
| 是否带控制器   | ❌                         | ✔              | ✔（更强）        |
| 是否适合频繁写 | ❌                         | ✔              | ✔✔               |
| 文件系统       | 简单/只读                 | 完整支持       | 完整支持         |
| 典型用途       | 固件                      | 配置/日志      | 数据/业务        |

![image-20260320105319108](C:\Users\01260065\AppData\Roaming\Typora\typora-user-images\image-20260320105319108.png)



Flash内存大小：

```
0x08000000 = 128MB
```

### 🟡 mtd0

```
08000000 → 整个Flash
```

👉 整盘映射（总空间）

------

### 🟢 第一套系统（A镜像）

```
mtd1 → conf
mtd2 → root
mtd3 → www
```

------

### 🔵 第二套系统（B镜像）

```
mtd4 → conf
mtd5 → root
mtd6 → www
```

![image-20260320110823586](C:\Users\01260065\AppData\Roaming\Typora\typora-user-images\image-20260320110823586.png)

```
mmcblk0    7.3G
```

👉 ** eMMC = 8GB（实际显示约 7.3GB）**