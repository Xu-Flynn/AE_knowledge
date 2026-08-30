# MDIO 通信协议详解

> **面向角色**: AE (应用工程师)
> **文档版本**: v1.0
> **更新日期**: 2026-07-18

> Management Data Input/Output（IEEE 802.3 Clause 22 / Clause 45）— 以太网 PHY 芯片管理总线标准。覆盖 IEEE 802.3 Clause 22 / Clause 45，适用于以太网 PHY 芯片管理总线学习与调试。

---

## 一、背景与应用定位

MDIO（Management Data Input/Output，管理数据输入/输出）是一种两线串行总线协议，由 IEEE 802.3 标准定义，专门用于以太网中站管理实体（STA）对物理层芯片（PHY）的寄存器进行读写管理。它是以太网系统中 MAC 与 PHY 之间不可或缺的"管理通道"——数据通道（MII/GMII/RGMII/XGMII 等）负责以太网帧的收发，而 MDIO 负责配置、状态查询与控制。

> **一句话定位**：MDIO 是以太网 PHY 芯片的"配置与诊断总线"——没有它，MAC 就无法知道 PHY 是否链路连通、无法协商速率、无法读取错误计数，也无法完成上电初始化。

### 协议定位

| 维度 | 说明 |
|------|------|
| 标准来源 | IEEE 802.3 标准。Clause 22（802.3u-1995）定义原始协议；Clause 45（802.3ae-2002）定义增强扩展协议 |
| 总线性质 | 两线串行、同步、主从（单主多从）、半双工 |
| 物理层 | 仅 2 根信号线——MDC（时钟）+ MDIO（双向数据） |
| 别称 | MIIM（MII Management Interface）、SMI（Serial Management Interface）、Clause 22/45 接口 |

### 典型应用场景

- 以太网 PHY 管理：10M/100M/1G/2.5G/5G/10G/25G/40G/100G/400G 全速率 PHY 的寄存器配置与状态读取
- 交换芯片（Switch）：SoC/CPU 通过 MDIO 总线管理片上/外挂多端口 PHY
- 光模块管理：部分 SFP+/QSFP+ 内部 PHY 寄存器（模块 EEPROM 通常走 I2C，但 PHY 管理走 MDIO）
- 车载以太网：100BASE-T1 / 1000BASE-T1 / MultiGBASE-T1 PHY 配置
- 工业以太网：EtherCAT/IP PHY、TSN 节点 PHY 的实时状态监控
- 背板以太网：1000BASE-KX / 10GBASE-KR 等背板 PHY 管理

### 协议发展历史

| 年代 | 标准章节 | 关键变化 |
|------|----------|----------|
| 1995 | IEEE 802.3u Clause 22 | 定义原始 MDIO 协议，支持 10/100M PHY，5 位 PHY 地址 + 5 位寄存器地址（32×32） |
| 1998 | IEEE 802.3z Clause 22 | 扩展千兆以太网（1000BASE-X），新增寄存器 9/10 用于 1000BASE-T |
| 2002 | IEEE 802.3ae Clause 45 | 重大扩展：支持 10GE PHY，引入 DEVAD（设备地址）+ 16 位寄存器地址（32×32×65536） |
| 2006 | IEEE 802.3an Clause 45 | 10GBASE-T 沿用 Clause 45 |
| 2010+ | 802.3ba/bf/bs Clause 45 | 40G/100G/25G 以太网 PHY 全部使用 Clause 45 |
| 2016 | IEEE 802.3bp Clause 22/45 | 车载以太网 1000BASE-T1，支持 Clause 22/45 双模式 PHY |
| 2020+ | IEEE 802.3cy/dj Clause 45 | 多 G 车载以太网（10GBASE-T1 等），高速 PHY 强制 Clause 45 |

> **命名提示**：不同厂商对 MDIO 的叫法不同——ST 叫 SMI，Microchip/SMSC 叫 MIIM，NXP 叫 SMI，TI/ADI 通常直称 MDIO。本质上都是 IEEE 802.3 Clause 22/45 接口。

---

## 二、核心工作原理

MDIO 采用两线串行、同步、主从架构。总线由一个主机（STA）和最多 32 个从机（MMD）组成。STA 通常是 MAC 控制器内部的管理实体，MMD 是 PHY 芯片内部的管理接口。所有通信由 STA 发起，MMD 被动响应。

### 核心概念

| 术语 | 全称 | 说明 |
|------|------|------|
| STA | Station Management Entity | 站管理实体，总线唯一主机，通常位于 MAC/SoC 内部，发起所有读/写操作 |
| MMD | MDIO Manageable Device | 管理数据接口设备，总线从机，位于 PHY 芯片内部，响应 STA 请求 |
| MDC | Management Data Clock | 管理数据时钟，由 STA 单向输出，所有数据在 MDC 上升沿采样。标准最大 2.5 MHz（周期 400 ns） |
| MDIO | Management Data Input/Output | 管理数据线，双向。写操作时 STA 驱动，读操作时 MMD 驱动，空闲时上拉为高 |
| PHYAD | Physical Address | PHY 物理地址，5 位（0-31），用于总线寻址。每个 MMD 有唯一 PHYAD |
| REGAD | Register Address | 寄存器地址。Clause 22 为 5 位（0-31），Clause 45 通过 DEVAD+16 位地址扩展 |
| DEVAD | Device Address | 设备地址（仅 Clause 45），5 位（0-31），区分 PHY 内部不同功能模块（PMA/PMD/WIS/PCS/...） |

### 写操作（Write）

1. **发送前导码**：STA 发送 32 bit 全 1 前导码（Preamble），唤醒总线上所有 MMD
2. **发送帧头与地址**：STA 发送 START(01) + OP(写) + PHYAD + REGAD
3. **转向位 TA = 10**：STA 驱动 MDIO 为 1，再驱动为 0，表示即将输出数据
4. **发送 16 bit 数据**：STA 在 16 个 MDC 周期内输出待写入数据（MSB 先发）
5. **释放总线**：帧结束后 STA 释放 MDIO，总线由上拉电阻拉高进入空闲态

### 读操作（Read）

1. **发送前导码**：STA 发送 32 bit 全 1 前导码
2. **发送帧头与地址**：STA 发送 START(01) + OP(读) + PHYAD + REGAD
3. **转向位 TA = Z0**：第 1 位 STA 释放 MDIO（高阻 Z），第 2 位目标 MMD 驱动 MDIO 为 0 表示响应
4. **接收 16 bit 数据**：目标 MMD（PHYAD 匹配者）在 16 个 MDC 周期内输出寄存器数据（MSB 先发）
5. **释放总线**：MMD 输出完数据后释放 MDIO，总线回到空闲态

> **关键点**：转向位（TA, Turnaround）是 MDIO 协议的精髓。它用 2 个 bit 时间完成总线控制权从 STA 到 MMD 的切换，避免了总线冲突。写操作 TA=10（STA 全程驱动），读操作 TA=Z0（第 1 位高阻切换，第 2 位 MMD 驱动 0 应答）。若读操作时 MMD 不存在或未响应，第 2 位会被上拉电阻拉为 1，STA 据此判断"无应答"。

---

## 三、核心模块深度解析 — 帧格式、寄存器与寻址

MDIO 协议的核心在于帧格式与寄存器映射。Clause 22 与 Clause 45 在帧格式上有关键差异，理解这些差异是正确使用 MDIO 的基础。

### 3.1 Clause 22 帧格式（IEEE 802.3u）

Clause 22 是最基础的 MDIO 帧格式，帧长 64 bit（含前导码），支持 32 个 PHY × 32 个寄存器。10/100/1000M PHY 普遍支持。

| 字段 | 位宽 | Clause 22 值 | 说明 |
|------|------|--------------|------|
| Preamble | 32 | 111...1（32 个 1） | 前导码，唤醒所有 MMD。部分实现可缩短至 16 bit |
| ST (Start) | 2 | 01 | 帧起始标志，固定 01 |
| OP (Operation) | 2 | 00=写 / 01=读后递增 / 10=读 / 11=保留 | 操作码。Clause 22 中 00 是写、10 是读 |
| PHYAD | 5 | 00000~11111 (0-31) | PHY 物理地址，寻址目标 MMD |
| REGAD | 5 | 00000~11111 (0-31) | 寄存器地址，仅 32 个寄存器可寻址 |
| TA (Turnaround) | 2 | 写=10 / 读=Z0 | 转向位，切换总线控制权 |
| DATA | 16 | 任意 16 bit | 数据域，读写均为 16 bit |
| IDLE | ≥1 | 高阻（上拉为 1） | 帧间空闲，总线释放 |

### 3.2 Clause 45 帧格式（IEEE 802.3ae）

Clause 45 为支持 10GE 及以上速率 PHY 而设计，引入两段式寻址：先用 Address 帧设置 16 位寄存器地址，再用 Read/Write 帧读写数据。这使寄存器空间从 32 扩展到 65536，并通过 DEVAD 区分 PHY 内部不同功能模块。

**Clause 45 的核心改进：**

- DEVAD（设备地址）：5 位，区分 PHY 内部 PCS/PMA/PMD/WIS/AN 等不同子层，每个子层有独立寄存器空间
- 16 位寄存器地址：通过 Address 帧预先写入，支持 65536 个寄存器（Clause 22 仅 32 个）
- 新的 OP 码：00=Address / 01=Write / 10=Post-Read-Increment / 11=Read（与 Clause 22 不同！）
- 两步操作：先发 Address 帧（写 16 位地址），再发 Read/Write 帧（读写 16 位数据）
- 增量读：Post-Read-Increment 可连续读取递增地址的寄存器，高效批量读取

**Clause 45 OP 码定义（与 Clause 22 不同！）**

| OP 码 | Clause 45 含义 | Clause 22 含义 | 说明 |
|-------|----------------|----------------|------|
| 00 | Address（设置寄存器地址） | Write（写） | Clause 45 用此帧预先写入 16 位寄存器地址 |
| 01 | Write（写数据） | Read-Increment（读后递增） | Clause 45 写 16 位数据到已设置地址 |
| 10 | Post-Read-Increment（读后自增） | Read（读） | Clause 45 读数据并自动递增地址，便于连续读 |
| 11 | Read（读数据） | 保留 | Clause 45 从已设置地址读 16 位数据 |

> **易混淆点**：Clause 22 和 Clause 45 的 OP 码定义完全不同！Clause 22 中 00=写、10=读；Clause 45 中 00=地址、01=写、10=读后自增、11=读。区分两套协议的关键是起始位 ST：Clause 22 ST=01，Clause 45 ST=00。MMD 通过 ST 判断当前帧是哪种协议。

**Clause 45 DEVAD 设备类型定义**

| DEVAD | 功能模块 | 典型寄存器 |
|-------|----------|-----------|
| 0 | Reserved | 保留 |
| 1 | PMA (Physical Medium Attachment) | 链路状态、误码计数、环回 |
| 2 | PMD (Physical Medium Dependent) | 发射功率、接收灵敏度、光模块参数 |
| 3 | WIS (WAN Interface Sublayer) | 10GE WAN 接口子层 |
| 4 | PCS (Physical Coding Sublayer) | 编解码、块同步、FEC |
| 5 | PHY XS (PHY Cross-Strap) | PHY XS 接口 |
| 6 | DTE XS (DTE Cross-Strap) | DTE XS 接口 |
| 7 | Auto-Negotiation (AN) | 自协商能力、状态 |
| 8-31 | Vendor specific / Reserved | 厂商自定义 |

### 3.3 Clause 22 标准寄存器映射

IEEE 802.3 规定了 Clause 22 下寄存器 0-15 的标准定义，所有 PHY 都应支持。寄存器 16-31 为厂商自定义。

| 寄存器 | 名称 | 功能 |
|--------|------|------|
| 0 | Control | 控制：复位/环回/速率/自协商/掉电/隔离/双工 |
| 1 | Status | 状态：能力/链路/自协商完成/Jabber |
| 2 | PHY Identifier 1 | OUI 厂商标识符高位（bit 3:18 of OUI） |
| 3 | PHY Identifier 2 | OUI 低位 + 型号 + 修订号 |
| 4 | Auto-Neg Advertisement | 自协商广播：本端能力宣告（速率/双工/流控） |
| 5 | Link Partner Ability | 对端能力：自协商后对端宣告的能力 |
| 6 | AN Expansion | 自协商扩展状态 |
| 7 | AN Next Page TX | 自协商下一页发送 |
| 8 | AN Link Partner Next Page | 对端下一页接收 |
| 9 | 1000BASE-T Control | 千兆控制：1000M 全/半双工能力宣告 |
| 10 | 1000BASE-T Status | 千兆状态：1000M 协商结果 |
| 11-13 | Reserved | 保留 |
| 14-15 | Reserved | 保留 |
| 16-31 | Vendor Specific | 厂商自定义：通常含扩展状态、错误计数、温度、眼图等 |

> **实战技巧**：读取寄存器 2 和 3 可获取 PHY 的厂商标识（OUI）和型号，这是识别未知 PHY 型号的标准方法。例如 OUI 0x08083D 是 Marvell，0x0007F0 是 TI，0x001050 是 Realtek。Linux 内核 phy_device 驱动正是通过此机制匹配驱动。

### 3.4 MDIO 时序详解

| 时序参数 | 符号 | 最小值 | 说明 |
|----------|------|--------|------|
| MDC 周期 | T_MDC | 400 ns | MDC 时钟周期，对应最大 2.5 MHz |
| MDC 高电平时间 | T_HIGH | 160 ns | MDC 高电平最小脉宽 |
| MDC 低电平时间 | T_LOW | 160 ns | MDC 低电平最小脉宽 |
| MDIO 建立时间 | T_SU | 10 ns | MDIO 数据在 MDC 上升沿前需稳定 ≥10 ns |
| MDIO 保持时间 | T_HD | 10 ns | MDIO 数据在 MDC 上升沿后需保持 ≥10 ns |
| 前导码长度 | — | 32 bit | 标准 32 bit 全 1，部分 PHY 可配置为 16 bit |
| 帧间空闲 | T_IDLE | ≥1 bit | 帧结束后至少 1 个 bit 时间的高阻 |

---

## 四、电路框图与物理连接

MDIO 的物理实现非常简洁——仅 2 根线，但电气细节决定了总线可靠性。

| 参数 | MDC | MDIO | 说明 |
|------|-----|------|------|
| 驱动方式 | 推挽（Push-Pull） | 推挽/开漏混合 | MDC 仅 STA 驱动，单向；MDIO 双向 |
| 上拉电阻 | 不需要 | 必须（1.5kΩ 推荐） | MDIO 空闲及转向位需上拉为高 |
| 电压电平 | 3.3V 或 2.5V LVCMOS | 同左 | 与 PHY 供电一致 |

> **MDIO 硬件连接要点**：MDC 由 STA 推挽驱动，无需上拉；MDIO 双向，必须接 1.5kΩ 上拉电阻到 VDDIO。总线走线应等长、<30cm，并加 ESD 保护。多 PHY 时所有 MDIO 引脚并联到同一总线上。

---

## 五、核心对比

### 5.1 Clause 22 vs Clause 45

| 维度 | Clause 22 | Clause 45 |
|------|-----------|-----------|
| 定义标准 | IEEE 802.3u (1995) | IEEE 802.3ae (2002) |
| ST 起始位 | 01 | 00 |
| 寄存器地址 | 5 bit（32 个寄存器） | 16 bit（65536 个寄存器） |
| 寻址方式 | 单帧（地址+数据同帧） | 两步（Address 帧 + Read/Write 帧） |
| DEVAD | 无 | 5 bit DEVAD 区分功能模块 |
| 典型速率 | 10M/100M/1000M | 2.5G/5G/10G/25G/40G/100G/400G |
| OP 码 | 00=写, 10=读 | 00=地址, 01=写, 10=读后自增, 11=读 |

### 5.2 MDIO vs I2C

| 维度 | MDIO | I2C |
|------|------|-----|
| 时钟线 | MDC（推挽） | SCL（开漏） |
| 数据线 | MDIO（开漏/推挽混合） | SDA（开漏） |
| 上拉 | 仅 MDIO 需要 | SCL 和 SDA 都需要 |
| 时钟频率 | ≤ 2.5 MHz | ≤ 400 kHz / 1 MHz |
| 设备地址 | 5 bit PHYAD（32 个） | 7/10 bit 设备地址 |
| 数据位宽 | 16 bit | 8 bit/字节 |
| 典型用途 | PHY 管理 | 通用外设、EEPROM、传感器 |

---

## 六、核心指标

### 6.1 时序参数

| 参数 | 指标 |
|------|------|
| MDC 周期 T_MDC | ≥ 400 ns |
| MDC 高电平 | ≥ 160 ns |
| MDC 低电平 | ≥ 160 ns |
| MDIO 建立时间 | ≥ 10 ns（MDC 上升沿前） |
| MDIO 保持时间 | ≥ 10 ns（MDC 上升沿后） |
| 前导码 | 32 bit（可配 16 bit） |
| 帧间空闲 | ≥ 1 bit 时间 |

### 6.2 电气与总线参数

| 参数 | 指标 |
|------|------|
| 电压电平 | 3.3V 或 2.5V LVCMOS |
| 上拉电阻 | 1.5 kΩ（推荐），范围 1.5k~10kΩ |
| 总线电容 | ≤ 400 pF |
| 最大节点 | 32 个 MMD |
| 典型走线长度 | < 30 cm（板内） |
| 单帧耗时 | Clause 22: 64 bit × 400ns = 25.6 µs @2.5MHz；Clause 45: 2 × 64 bit × 400ns = 51.2 µs @2.5MHz |

---

## 七、影响因素

MDIO 总线的可靠性受多个因素影响，主要源于其开漏/推挽混合 + 多点总线的电气特性。

| 因素 | 说明 |
|------|------|
| 上拉电阻取值 | 阻值过大（>10kΩ）会导致上升沿过缓、时序违例；过小（<1kΩ）驱动负载过大。推荐 1.5kΩ。总线节点多/走线长时适当减小 |
| 总线电容 | 标准限制 ≤400pF。包含所有 PHY 引脚电容 + PCB 走线 + ESD 二极管电容。电容过大会使 MDIO 上升时间 (RC) 超过建立时间，导致采样错误。每节点约 10-15pF，30cm 走线约 3pF/cm |
| MDC 时钟速率 | 速率越高，对建立/保持时间越敏感。标准 2.5MHz 下裕量充足；若超频到 10MHz，周期仅 100ns，时序裕量骤减 |
| 节点数量 | 节点越多，总线电容越大，上升沿越缓。32 节点满载时需严格评估电容预算 |
| 电平兼容性 | SoC 与 PHY 电平不一致（如 SoC 1.8V、PHY 3.3V）需加电平转换器（如 TXS0102） |
| PHY 地址冲突 | 两个 PHY 配置相同 PHYAD 会导致总线冲突。PHYAD 通常由 strapped 引脚或 EEPROM 配置 |
| 前导码长度 | 标准 32 bit。部分 PHY 支持 16 bit 以提速，但需 STA 与所有 MMD 都支持 |
| Clause 22/45 混用 | Clause 45 PHY 通常兼容 Clause 22，但仅能访问前 32 个寄存器。访问扩展寄存器必须用 Clause 45 |

> **总线电容与上升时间的关系**：$t_r \approx 2.2 \times R_{pull} \times C_{bus}$（10%~90% 上升时间）。例：Rpull=1.5kΩ，Cbus=200pF → tr ≈ 2.2 × 1.5k × 200p = 660 ns。而 MDC 半周期（2.5MHz）仅 200 ns——此时上升沿无法在一个 bit 内完成，必须减小上拉阻值或降低时钟。

> **典型故障模式**：读寄存器始终返回 0xFFFF。原因排查顺序：①MDIO 上拉电阻缺失/虚焊 → ②PHYAD 配置错误（PHY 未响应）→ ③PHY 未上电/复位未释放 → ④MDC 时钟未输出 → ⑤总线电容过大导致时序违例 → ⑥Clause 22/45 协议不匹配。

---

## 八、测试方法

### 测试一：MDIO 总线功能性测试（逻辑分析仪）

1. **连接探头**：逻辑分析仪 CH0 接 MDC，CH1 接 MDIO。同时接 SoC 的 MDIO 引脚和 PHY 侧引脚以对比信号完整性
2. **配置协议解码**：在逻辑分析仪软件中选择 MDIO 协议解码（Clause 22 / Clause 45），设置采样率 ≥20 MHz（MDC 2.5MHz 的 8 倍以上）
3. **触发读操作**：通过 SoC 驱动（如 Linux mdio-tools 或裸机寄存器操作）发起一次读操作，读取 PHY 寄存器 1（Status）
4. **验证帧格式**：检查解码结果：Preamble(32×1) → ST(01/00) → OP → PHYAD → REGAD/DEVAD → TA → DATA → IDLE。确认 TA 转向位正确（读=Z0，写=10）
5. **验证数据正确性**：对比逻辑分析仪读到的 DATA 与 PHY datasheet 中寄存器 1 的默认值。若读到 0xFFFF，说明 PHY 未响应
6. **遍历所有 PHY 地址**：对 PHYAD 0-31 逐一发起读操作（扫描总线），记录哪些地址有响应

### 测试二：信号完整性测试（示波器）

1. **测量 MDC 与 MDIO 波形**：示波器（≥100 MHz 带宽）探头接 MDC 和 MDIO，触发连续读操作，观察波形上升/下降沿、振铃、过冲
2. **测量建立/保持时间**：在 MDC 上升沿处展开波形，测量 MDIO 数据建立时间和保持时间，均应 ≥10 ns
3. **测量上升时间**：测量 MDIO 上升沿 10%~90% 时间，验证 tr = 2.2×R×C 是否在预算内
4. **检查高阻态**：读操作的 TA 第 1 位应为高阻（由上拉拉高）。若此位为低或中间电平，说明 STA 未正确释放总线或上拉失效

### 测试三：软件层验证（Linux 环境）

```bash
# 1. 使用 mdio-tools（用户态工具）扫描总线
mdio-tool list eth0                      # 列出总线上所有 PHY
mdio-tool read eth0 0x01 0x1             # 读 PHYAD=1 的寄存器 1 (Status)
mdio-tool write eth0 0x01 0x0 0x8000     # 写 PHYAD=1 寄存器 0, bit15=1 (软复位)

# 2. Linux 内核 sysfs 接口
cat /sys/bus/mdio_bus/*/phy_id           # 读取 PHY 标识 (OUI+型号)
cat /sys/class/net/eth0/phydev/link      # 读取链路状态

# 3. Clause 45 访问（需 phylib 支持，格式: phydev:mmd:register）
mdio-tool read eth0 0x01:0x1:0x0001      # Clause 45: PORTAD=1, DEVAD=1, REG=0x0001

# 4. ethtool 间接访问
ethtool -d eth0                          # dump PHY 寄存器
ethtool --phy-statistics eth0            # PHY 错误统计
```

**推荐设备**：

- 逻辑分析仪：Saleae Logic Pro 16 / Siglent SDLA / Keysight 1680 系列（支持 MDIO 协议解码）
- 示波器：≥4 通道、≥100 MHz 带宽（如 Siglent SDS1104X、Rigol DS1054Z）
- 软件工具：Linux mdio-tools、ethtool、phytool；裸机环境可用厂商提供的 MDIO 调试命令

---

## 九、注意事项

| 注意事项 | 说明 |
|----------|------|
| 🔌 上拉电阻不可省 | MDIO 必须有外部上拉（1.5kΩ）。漏加上拉会导致空闲态为低、转向位高阻失效、读操作返回错误数据。这是最常见的硬件设计错误 |
| 📋 PHY 地址必须唯一 | 总线上的 PHY 必须配置不同的 PHYAD（通过 strapped 引脚或 EEPROM）。地址冲突会导致读操作时两个 PHY 同时驱动 MDIO，数据损坏 |
| 🔄 Link Status 是锁存型 | 寄存器 1 的 bit2（Link Status）是 latch-low 型：一旦链路断开会锁存 0，即使重新连通也保持 0，直到被读取。读取后清锁存，需读第二次才得到实时值 |
| ⚡ 复位后需等待 | 写寄存器 0 的 bit15=1 执行软复位后，PHY 需要一定时间完成内部初始化（通常几十 µs~几 ms）。复位后立即读取可能返回错误值 |
| 📖 Clause 22/45 不可混用 | 同一总线上 Clause 22 和 Clause 45 帧通过 ST 位区分。若 SoC 控制器只支持 Clause 22，则无法访问 Clause 45 扩展寄存器，10G+ PHY 配置将受限 |
| 📐 总线长度受限 | MDIO 是板内总线，走线应 <30cm。若需跨板管理（如背板），需加 MDIO 中继/桥接芯片，或改用以太网管理帧（如 NC-SI） |
| 🔍 0xFFFF 不一定是真值 | 读操作返回 0xFFFF 有两种可能：①PHY 寄存器真值确实是 0xFFFF（罕见）；②PHY 未响应，MDIO 被上拉拉高为全 1。若所有寄存器都返回 0xFFFF，几乎可判定是通信失败 |
| 🚗 车载 PHY 特殊性 | 100BASE-T1/1000BASE-T1 等车载以太网 PHY 通常强制使用 Clause 45（因寄存器空间大），且 Master/Slave 模式需通过 MDIO 配置 |

### 常见问题速查表

| 现象 | 可能原因 | 排查方法 |
|------|----------|----------|
| 所有寄存器读 0xFFFF | MDIO 上拉缺失 / PHY 未上电 / PHYAD 错误 | 示波器查 MDIO 空闲电平；万用表查 PHY 供电；逻辑分析仪查 TA 位是否为 0 |
| 部分寄存器读值错误 | 时序违例 / 总线电容过大 | 降低 MDC 速率重试；示波器查上升时间 |
| 写操作无效 | OP 码错误 / Clause 协议不匹配 | 逻辑分析仪解码帧，确认 ST 和 OP 值 |
| 链路状态读取异常 | Link Status 锁存机制 | 连续读两次寄存器 1，取第二次 bit2 |
| 总线偶发错误 | 信号完整性差 / EMI 干扰 | 示波器查振铃过冲；缩短走线；加 ESD/去耦 |
| Clause 45 访问失败 | SoC 控制器不支持 / PHY 未启用 Clause 45 | 查 SoC 手册 MDIO 控制器章节；确认 PHY 配置 |

---

## 十、选型指南

### 场景一：按以太网速率选择协议

| 速率 | 推荐协议 | 说明 |
|------|----------|------|
| 10M / 100M | Clause 22 | 传统 PHY，Clause 22 足够。如 RTL8201、DP83848 |
| 1000M (1G) | Clause 22（兼容） | 多数 1G PHY 双模支持，Clause 22 可访问基本寄存器 |
| 2.5G / 5G | Clause 45（推荐） | 扩展寄存器多，建议 Clause 45。如 Marvell 88E2580 |
| 10G / 25G | Clause 45（必须） | 10G+ PHY 寄存器空间大，Clause 22 无法完整配置 |
| 40G / 100G / 400G | Clause 45（必须） | 多通道 PHY，强依赖 Clause 45 的 DEVAD 寻址 |
| 车载 100/1000BASE-T1 | Clause 45（推荐） | 车载 PHY 通常强制 Clause 45 |

### 场景二：SoC 控制器选型要点

| 选型维度 | 建议 |
|----------|------|
| Clause 45 支持 | 若涉及 10G+ PHY，SoC MDIO 控制器必须支持 Clause 45 |
| MDC 时钟可调 | 控制器应支持 MDC 分频配置（如 1/2/4/8 分频），以适配不同 PHY 的最大时钟 |
| 前导码长度可配 | 支持 32/16 bit 前导码切换，兼容不同 PHY |
| DMA / 中断 | 频繁轮询 PHY 状态时，支持中断驱动的控制器可降低 CPU 占用 |
| 多总线实例 | Switch 芯片管理多组 PHY 时，SoC 最好有多个独立 MDIO 控制器实例 |
| Linux 驱动支持 | 确认 SoC 的 MDIO 控制器有主流内核驱动（libphy / mdio_bus） |

### 场景三：PHY 选型要点

**低速 PHY（10/100/1000M）**

- 协议：Clause 22（多数也兼容 Clause 45）
- PHYAD 配置：strapped 引脚（2-3 位，地址 0-4 或 0-7）
- 典型型号：RTL8201F、DP83848I、KSZ8081、LAN8720
- 关注：寄存器 0-10 标准寄存器完整、支持中断输出
- 价格：低（<$0.5）

**高速 PHY（10G+）**

- 协议：Clause 45（必须）
- DEVAD 支持：PMA/PMD/PCS/AN 全子层寄存器可访问
- 典型型号：Marvell 88X3310、TI DP83867/DP83869、Aquantia AQR107
- 关注：LED 配置、SerDes 环回、PRBS 测试、眼图寄存器
- 价格：高（$3-$20+）

### 决策流程

1. **确定 PHY 速率等级**：10/100M → Clause 22；1G → Clause 22/45 双模；10G+ → 强制 Clause 45
2. **确认 SoC 控制器能力**：查 SoC datasheet 的 MDIO 章节，确认支持 Clause 22/45 及 MDC 时钟范围
3. **分配 PHY 地址**：列出总线上所有 PHY，分配唯一 PHYAD（0-31），在原理图标注 strapped 配置
4. **设计硬件电路**：MDIO 加 1.5kΩ 上拉；MDC/MDIO 等长并行走线；ESD 保护；评估总线电容
5. **软件驱动适配**：配置 SoC MDIO 控制器（时钟、前导码、协议模式）；移植 libphy 驱动；验证读写

> **选型总结**：对于新设计，强烈推荐选择同时支持 Clause 22 和 Clause 45 的 PHY 与 SoC 控制器。这样既能用 Clause 22 快速访问标准寄存器，又能用 Clause 45 配置扩展功能。Linux 内核 libphy 框架原生支持双协议自动协商，软件层无需额外适配。

**学习资源推荐**：

- IEEE 802.3 标准 Clause 22（第 22 章）与 Clause 45（第 45 章）——最权威的协议定义
- Linux 内核源码 drivers/net/phy/phy_device.c 与 drivers/net/mdio.c——工业级实现参考
- 各 PHY 厂商 datasheet 的 "MDIO Interface" 章节——具体寄存器定义与时序
- mdio-tools 项目（GitHub）——用户态调试工具源码



