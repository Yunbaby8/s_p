

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

| 项目           | SPI Flash（你现在的 mtd） | eMMC（mmcblk）                                               | 硬盘（SSD/NVMe） |
| -------------- | ------------------------- | ------------------------------------------------------------ | ---------------- |
| 容量           | MB级（16~256MB）          | GB级（8~64GB）                                               | TB级             |
| 速度           | 慢                        | 中等                                                         | 很快             |
| 写入次数       | 少（寿命有限）            | 中等                                                         | 高（有磨损均衡） |
| 接口           | SPI                       | MMC                                                          | PCIe / SATA      |
| 是否带控制器   | ❌                         | ✔                                                            | ✔（更强）        |
| 是否适合频繁写 | ❌                         | ✔                                                            | ✔✔               |
| 文件系统       | 简单/只读                 | 完整支持                                                     | 完整支持         |
| 典型用途       | 固件                      | flowchart TD    A["监控线程周期扫描 PSU 功率类 sensor"] --> B["识别当前是 PSU PIN 还是 PSU POUT"]​    B --> C["根据 sensor 类型确定目标 PSU 和待读取的功率寄存器"]    C --> D{"是否处于 PSU 升级 / Flash 模式?"}​    D -- 是 --> D1["该 sensor 状态置为 BLOCK / 不可用"]    D -- 否 --> E["检查目标 PSU 是否在位"]​    E --> F{"PSU 是否在位?"}    F -- 否 --> F1["该 sensor 状态置为 NOT_PRESENT"]    F -- 是 --> G["通过 PMBus 读取原始功率值"]​    G --> H{"读取是否成功?"}    H -- 否 --> H1["该 sensor 状态置为 READING_ERR"]    H -- 是 --> I["将 PMBus 原始值转换为实际功率值"]​    I --> J["对读数做缩放 / 单位换算 / 抗抖处理"]    J --> K["写入该 sensor 的缓存值和状态"]​    K --> L{"是否为 PSU PIN?"}    L -- 是 --> L1["同步更新每个 PSU 的输入功率表，供总功耗统计复用"]    L -- 否 --> L2["仅保留该 PSU 的输出功率读数"]​    L1 --> M["后处理阶段将缓存值拷入共享内存"]    L2 --> M    D1 --> M    F1 --> M    H1 --> M​    M --> N["IPMI / SEL / 上层接口从共享内存读取该 sensor"]​    N --> O{"最终对外显示"}    O -- PIN --> P["显示 PSU 输入功率"]    O -- POUT --> Q["显示 PSU 输出功率"]​mermaid#mermaidChart3{font-family:sans-serif;font-size:16px;fill:#333;}@keyframes edge-animation-frame{from{stroke-dashoffset:0;}}@keyframes dash{to{stroke-dashoffset:0;}}#mermaidChart3 .edge-animation-slow{stroke-dasharray:9,5!important;stroke-dashoffset:900;animation:dash 50s linear infinite;stroke-linecap:round;}#mermaidChart3 .edge-animation-fast{stroke-dasharray:9,5!important;stroke-dashoffset:900;animation:dash 20s linear infinite;stroke-linecap:round;}#mermaidChart3 .error-icon{fill:#552222;}#mermaidChart3 .error-text{fill:#552222;stroke:#552222;}#mermaidChart3 .edge-thickness-normal{stroke-width:1px;}#mermaidChart3 .edge-thickness-thick{stroke-width:3.5px;}#mermaidChart3 .edge-pattern-solid{stroke-dasharray:0;}#mermaidChart3 .edge-thickness-invisible{stroke-width:0;fill:none;}#mermaidChart3 .edge-pattern-dashed{stroke-dasharray:3;}#mermaidChart3 .edge-pattern-dotted{stroke-dasharray:2;}#mermaidChart3 .marker{fill:#333333;stroke:#333333;}#mermaidChart3 .marker.cross{stroke:#333333;}#mermaidChart3 svg{font-family:sans-serif;font-size:16px;}#mermaidChart3 p{margin:0;}#mermaidChart3 .label{font-family:sans-serif;color:#333;}#mermaidChart3 .cluster-label text{fill:#333;}#mermaidChart3 .cluster-label span{color:#333;}#mermaidChart3 .cluster-label span p{background-color:transparent;}#mermaidChart3 .label text,#mermaidChart3 span{fill:#333;color:#333;}#mermaidChart3 .node rect,#mermaidChart3 .node circle,#mermaidChart3 .node ellipse,#mermaidChart3 .node polygon,#mermaidChart3 .node path{fill:#ECECFF;stroke:#9370DB;stroke-width:1px;}#mermaidChart3 .rough-node .label text,#mermaidChart3 .node .label text,#mermaidChart3 .image-shape .label,#mermaidChart3 .icon-shape .label{text-anchor:middle;}#mermaidChart3 .node .katex path{fill:#000;stroke:#000;stroke-width:1px;}#mermaidChart3 .rough-node .label,#mermaidChart3 .node .label,#mermaidChart3 .image-shape .label,#mermaidChart3 .icon-shape .label{text-align:center;}#mermaidChart3 .node.clickable{cursor:pointer;}#mermaidChart3 .root .anchor path{fill:#333333!important;stroke-width:0;stroke:#333333;}#mermaidChart3 .arrowheadPath{fill:#333333;}#mermaidChart3 .edgePath .path{stroke:#333333;stroke-width:2.0px;}#mermaidChart3 .flowchart-link{stroke:#333333;fill:none;}#mermaidChart3 .edgeLabel{background-color:rgba(232,232,232, 0.8);text-align:center;}#mermaidChart3 .edgeLabel p{background-color:rgba(232,232,232, 0.8);}#mermaidChart3 .edgeLabel rect{opacity:0.5;background-color:rgba(232,232,232, 0.8);fill:rgba(232,232,232, 0.8);}#mermaidChart3 .labelBkg{background-color:rgba(232, 232, 232, 0.5);}#mermaidChart3 .cluster rect{fill:#ffffde;stroke:#aaaa33;stroke-width:1px;}#mermaidChart3 .cluster text{fill:#333;}#mermaidChart3 .cluster span{color:#333;}#mermaidChart3 div.mermaidTooltip{position:absolute;text-align:center;max-width:200px;padding:2px;font-family:sans-serif;font-size:12px;background:hsl(80, 100%, 96.2745098039%);border:1px solid #aaaa33;border-radius:2px;pointer-events:none;z-index:100;}#mermaidChart3 .flowchartTitleText{text-anchor:middle;font-size:18px;fill:#333;}#mermaidChart3 rect.text{fill:none;stroke-width:0;}#mermaidChart3 .icon-shape,#mermaidChart3 .image-shape{background-color:rgba(232,232,232, 0.8);text-align:center;}#mermaidChart3 .icon-shape p,#mermaidChart3 .image-shape p{background-color:rgba(232,232,232, 0.8);padding:2px;}#mermaidChart3 .icon-shape rect,#mermaidChart3 .image-shape rect{opacity:0.5;background-color:rgba(232,232,232, 0.8);fill:rgba(232,232,232, 0.8);}#mermaidChart3 .label-icon{display:inline-block;height:1em;overflow:visible;vertical-align:-0.125em;}#mermaidChart3 .node .label-icon path{fill:currentColor;stroke:revert;stroke-width:revert;}#mermaidChart3 :root{--mermaid-alt-font-family:sans-serif;}是否否是否是是否PINPOUT监控线程周期扫描 PSU 功率类 sensor识别当前是 PSU PIN 还是 PSU POUT根据 sensor 类型确定目标 PSU 和待读取的功率寄存器是否处于 PSU 升级 / Flash 模式?该 sensor 状态置为 BLOCK / 不可用检查目标 PSU 是否在位PSU 是否在位?该 sensor 状态置为 NOT_PRESENT通过 PMBus 读取原始功率值读取是否成功?该 sensor 状态置为 READING_ERR将 PMBus 原始值转换为实际功率值对读数做缩放 / 单位换算 / 抗抖处理写入该 sensor 的缓存值和状态是否为 PSU PIN?同步更新每个 PSU 的输入功率表，供总功耗统计复用仅保留该 PSU 的输出功率读数后处理阶段将缓存值拷入共享内存IPMI / SEL / 上层接口从共享内存读取该 sensor最终对外显示显示 PSU 输入功率显示 PSU 输出功率 | 数据/业务        |

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