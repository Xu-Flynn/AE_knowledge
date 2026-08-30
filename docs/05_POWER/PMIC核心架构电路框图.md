# PMIC 全产品线核心架构电路框图

> **面向角色**: AE (应用工程师)
> **文档版本**: v1.0
> **更新日期**: 2026-06

> 芯迈半导体（Silicon Magic）· 9 大产品线 · 电路框图 + 工作原理详解 · AE 工程师技术参考

---

## 一、电池管理（Battery Management）

### 1.1 开关充电器 Buck / Buck-Boost

**Buck 充电器工作原理**

- HS-FET 导通 (ton)：VBUS → L → VBAT，电感储能，电流线性上升
- HS-FET 关断 (toff)：电感续流 → CS-FET 导通，释放能量给电池
- 输出电压：$V_{BAT} = D \times V_{BUS}$（D < 1 → 仅降压）

**Buck-Boost 充电器工作原理**

- Buck 模式 (VBUS > VBAT)：HS1/LS1 交替开关，HS2 常通，LS2 常断
- Boost 模式 (VBUS < VBAT)：LS2/HS2 交替开关，HS1 常通，LS1 常断
- Buck-Boost 模式 (VBUS ≈ VBAT)：四管均参与开关，平滑过渡
- OTG 模式：电池反向 Boost 输出 5V/9V/12V（PD 协议）

> **电源路径管理 (Power Path)**：系统优先从 VBUS 取电，不足时由电池补充。即使电池深度放电，插上适配器后系统也能立即启动。

| 型号 | 拓扑 | 输入电压 | 最大充电电流 | 电源路径 | 串联电芯 | OTG | 封装 |
|------|------|----------|--------------|----------|----------|-----|------|
| SM5803A | Buck-Boost | 4.0~21.5V | 6A | ✓ | 2/3/4S | ✓ | 75 WLCSP |
| SM5424 | Buck | 4.2~13.5V | 3.5A | ✓ | 1S | ✗ | 42 WLCSP |
| SM5414 | Buck | 4.0~6.2V | 2.5A | ✓ | 1S | ✗ | 25 WLCSP |

### 1.2 电容分压器（电荷泵）Switched Capacitor

**电荷泵工作原理**

- Phase 1（充电）：S1/S3 导通，Cfly 接在 VBUS-GND 间，充至 VBUS/2
- Phase 2（转移）：S2/S4 导通，Cfly 从 GND 断开改接 VBAT，电荷转移给电池
- 2:1 转换比：输出电压 = 输入/2，天然适配 9V→4.5V 单电芯快充
- 理论效率 $\eta = 2 \times V_{out}/V_{in} \times 100\%$（当 Vout=Vin/2 时 η→100%）

> **电荷泵 vs 电感式**：电荷泵用 Cfly 替代 L → 体积更小（手机空间受限）、EMI 更低（无磁场辐射），但只能做整数比分压（2:1/3:1），不适合宽范围调压。

**工作模式**

- Forward Bypass：VBUS 直通 VBAT（输入电压匹配时）
- Forward Boost：2:1 电荷泵降压充电
- Reverse Bypass：VBAT 直通 VBUS（OTG 5V 输出）
- Reverse Boost：VBAT 升压到 VBUS（OTG 9V/12V 输出）

| 型号 | 输入电压 | 最大充电电流 | 输出功率 | 工作模式 | 电芯 | 封装 |
|------|----------|--------------|----------|----------|------|------|
| SM5440 | 6~10.5V | 9A | 60W | Charge/Rev-Bypass/Rev-Boost | 1S | 89 WLCSP |
| SM5470 | 3.3~10.5V | 5A | 25W | Fwd-Bypass/Fwd-Boost/Rev 全模式 | 1S | 36 WLCSP |
| SM5451 | 3.3~10.5V | 6A | 30W | Fwd-Bypass/Fwd-Boost/Rev 全模式 | 1S | 42 WLCSP |
| SM5441 | 6~10.5V | 7A | 60W | Direct/Rev-Bypass/Cell Balance | 2S | 72 WLCSP |

### 1.3 电量计（Fuel Gauge）Pack / System Side

**电量计工作原理**

- 库仑计数法：Rsense 采样电流 → ADC → 对时间积分 $\Delta Q = \int I \cdot dt$
- SOC 计算：$SOC = Q_{remaining} / Q_{full} \times 100\%$，需 OCV 校准消漂移
- Pack Side (SM5603)：内置 MCU，独立运行算法，适合可拆卸电池包
- System Side (SM5602)：仅 ADC+通信，算法在主 SoC 运行，成本更低

> **High-side vs Low-side Rsense**：高侧采样不受地噪声干扰但需差分 ADC；低侧简单但受地回路影响。芯迈电量计支持 High & Low 两种配置。

| 型号 | 内置 MCU | 位置 | ADC 范围 | 感测电阻 | 电芯 | 封装 |
|------|----------|------|----------|----------|------|------|
| SM5603 | ✓ | Pack Side | 13A | High & Low | 1S | 12 WLCSP |
| SM5602 | ✗ | System Side | 13A | High & Low | 1S | 9 WLCSP |

---

## 二、无线供电（Wireless Power）

Qi 无线供电系统架构 TX → RX。

**无线供电工作原理**

- TX 全桥逆变器：DC → AC (100~205kHz)，驱动 TX 线圈产生交变磁场
- 谐振补偿：TX 串联 + RX 并联谐振 → 提高功率传输效率，补偿漏感
- 电磁感应：TX 线圈交变磁场 → RX 线圈感应 AC 电压（法拉第定律）
- 同步整流：MOSFET 替代二极管 → 压降从 0.7V 降至 0.3V，效率提升 5~10%
- ASK 负载调制：RX 改变自身负载 → TX 检测电流变化 → 实现 RX→TX 通信

> **FOD（异物检测）**：TX 通过监测输入功率与理论传输功率的差值，损耗过大说明线圈间有金属异物，立即停供电防止过热。WPC Qi 标准强制要求。

| 型号 | 角色 | 输入电压 | 输出电压 | 功率 | 封装 |
|------|------|----------|----------|------|------|
| SM5903 | RX | 5.15~15V | 4~12V | 20W | 52 WLCSP |
| SM5922 | RX | 5~12V | 4~6V | 5W | 40 WLCSP |
| SM5940 | TX | TBD | TBD | TBD | TBD |

---

## 三、开关稳压器（Switching Regulator）

Buck / Boost / Buck-Boost 三种拓扑 DC-DC。

**三种拓扑原理对比**

- Buck：HS 导通→L 储能，LS 续流→L 释能。$V_{out} = D \times V_{in}$，D<1 故 Vout<Vin
- Boost：LS 导通→L 从 Vin 储能，LS 断开→L 释放叠加 Vin。$V_{out} = V_{in}/(1-D)$
- Buck-Boost：四开关结构，自动切换 Buck/Boost/Buck-Boost 模式

> **高开关频率（2~2.4MHz）**：频率越高→电感值越小→体积越小→适合手机/平板。代价是开关损耗增大，需低 Qg 集成 FET。AEC-Q100 Grade-1 车规版本支持 -40°C~+150°C。

| 型号 | 拓扑 | 输入电压 | 输出电压 | 电流 | Iq | AEC-Q100 | 封装 |
|------|------|----------|----------|------|-----|----------|------|
| SM5811 | Buck-Boost | 2.5~5.5V | 1.8~5.5V | 3A | 15μA | ✗ | 15 WLCSP |
| SM6003 | Buck | 4~36V | 28V | 600mA | 75μA | Gr-1 | DFN-8 |
| SM6001 | Buck | 4~36V | 28V | 1A | 75μA | Gr-1 | DFN-8 |
| SM5820 | Buck | 2.5~5.5V | 0.35~1.39V | 3A | 40μA | ✗ | 15 WLCSP |
| SM5830 | Boost | TBD | TBD | TBD | TBD | ✗ | TBD |

---

## 四、接口 & 保护（Interface & Protection）

OVP / 负载开关 / USB Type-C / 电平转换器。

**OVP 工作原理**

- 正常：VBUS < OVP 阈值 → FET 导通 → VBUS 直通 VOUT
- 过压：VBUS 超阈值 → 比较器翻转 (30~70ns) → FET 关断 → 后端受保护

> **为什么需要 30~70ns 响应？** USB PD 中 20V 浪涌前沿达 ns 级，OVP 必须在浪涌到达后端 IC 前关断，否则 SoC/PMIC 被高压击穿。

**USB Type-C 保护要点**

- CC OVP：CC 线正常仅 5V，PD 协商时 VBUS 可达 20V → CC OVP 防误接
- VCONN：5V LDO 为 Cable 中 eMarker 供电（5A/100W Cable 必须有 eMarker）
- 2:1 Mux：USB-C 正反插需切换 D+/D- 信号到 SoC 正确引脚

| 类别 | 型号 | 关键参数 | 封装 |
|------|------|----------|------|
| OVP | SM5334 | 8mΩ/9A, OVP 4~23V, 70ns | 20 WLCSP |
| OVP | SM5335/5362 | 9mΩ/4.5A, OVP 4~23V, 70ns | 9 WLCSP |
| 负载开关 | SM5360A | 22mΩ/5A, OVP 6~23V, 30ns | 42 WLCSP |
| USB-C | SM5517 | CC OVP + Type-C 控制器 + USB OVP | 36 WLCSP |
| eMarker | SM5516 | VCONN 2.7~5.5V | 6 WLCSP |
| 电平转换 | SM5520 | 2-bit 双向, 1.2~5.0V, 3.4Mbps | 8 DFN |

---

## 五、LED 显示背光（LED Backlight）

LED 背光驱动架构 Boost / Buck。

**LED 背光驱动工作原理**

- Boost 升压：Vin (2.7~24V) 升压到 VLED (35~55V)，驱动多颗串联 LED
- 恒流控制：每通道内置电流镜，精确设定 LED 串电流 (20~40mA)，亮度一致
- PWM 调光：200Hz~20kHz 开关 LED 电流，占空比决定亮度 → 无色偏
- 模拟调光：连续改变 LED 电流大小 → 无频闪，但低电流色温偏移
- 内置/外置 FET：手机/笔记本用内置 (小功率)，电视用外置 (大功率)

> **多通道电流匹配**：6 通道间电流匹配精度通常 <1%，直接影响背光均匀度。电流镜架构确保各通道独立设定电流且不受其他通道影响。

| 型号 | 拓扑 | 输入 | 通道 | 每串电流 | 内置 FET | 调光 | 应用 |
|------|------|------|------|----------|----------|------|------|
| SM5350 | Boost | 2.7~5.5V | 3 | 20mA | ✓ | PWM/模拟 | 手机 |
| SM4500 | Boost | 5~27V | 6 | 30mA | ✓ | PWM | 笔记本 |
| SM2006 | Boost | 2.7~24V | 6 | 40mA | ✓ | PWM | 笔记本 |
| SM2101 | Buck | 4.5~26V | 2 | 500mA | ✓ | PWM | 笔记本 |
| SM1251 | Buck | Ext. | 1 | Ext. | ✗ | PWM/模拟 | 电视 |
| SM1202 | Boost | 11~27V | 2 | Ext. | ✗ | PWM | 电视 |

---

## 六、LCD 电源管理

LCD 面板多路电源架构 AVDD/VGH/VGL/DVDD/VCOM。

**LCD 电源架构工作原理**

- AVDD：TFT 模拟电源 (10~16V)，给像素电容充电 → 纹波必须 <20mV 否则画面闪烁
- VGH：栅极开启电压 (15~30V) → 使 TFT 导通，像素写入数据电压
- VGL：栅极关断电压 (-5~-10V) → 使 TFT 截止，保持像素电压
- DVDD：数字逻辑电源 (1.8~3.3V) → TCON 和数字电路供电
- VCOM：公共电极电压 → 精密 DAC 调节 → 直接影响 Flicker
- Gamma Buffer：灰度电压缓冲 → 10bit+ 精度 → 影响色彩准确度
- Level-shifter：将 TCON 低电压信号转换为栅极驱动高电压信号
- GPM：Gate Pulse Modulation → 低功耗模式，降低栅极驱动功耗

> **VCOM 为何关键？** VCOM 偏移导致像素电压不对称 → 直流分量 → Flicker（闪烁）。必须通过 10bit+ DAC 精密调节，且随温度/老化需实时校准。Gamma Buffer 精度直接影响色深表现。

| 应用 | 代表型号 | AVDD | VGH | VGL | VCOM | Gamma | GPM | Level-shift |
|------|----------|------|-----|-----|------|-------|-----|-------------|
| 电视 | SM4805 | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | - |
| 笔记本 | SM4405 | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | - |
| 手机 | SM5119CF | ✓ | - | ✓ | - | - | - | - |
| 汽车 | SM6751Q | ✓ | ✓ | - | ✓ | - | ✓ | - |

---

## 七、SoC 电源管理

SoC PMIC 多路电源架构 多 Buck+LDO 集成。

**SoC PMIC 工作原理**

- 多路 Buck：SoC 需要多个电压域（Core/GPU/DDR/IO），PMIC 集成 4 路 Buck 减少外部元件
- 9× LDO：PLL/RF/Audio 等对噪声敏感的模块需 LDO 供电（高 PSRR），LDO 从 Buck 输出级联以提高效率
- 上电时序：SoC 严格要求上电顺序（Core→GPU→DDR→IO→LDO→Reset），PMIC 硬件控制时序
- 2MHz 开关频率：减小电感体积 → 适合手机紧凑空间
- 线性充电器：集成了 USB 小电流线性充电，从 USB 取电补充系统

> **为何不全部用 Buck？** PLL/RF 等模块需要极低噪声电源（PSRR >60dB），Buck 输出有开关纹波 (~10mVpp)，LDO 级联可滤除纹波至 <1mV。LDO 从 Buck 输出取电（而非 VBAT）可提高整体效率。

| 型号 | 应用 | 输入电压 | Buck | Boost | LDO | 线性充电 | 开关频率 | 封装 |
|------|------|----------|------|-------|-----|----------|----------|------|
| SM5007 | 手机 | 4.0~6.1V | 4 | 0 | 9 | 1 | 2MHz | 51 WLCSP |
| SM6700 | 汽车 | - | - | - | - | - | - | - |

---

## 八、接口电源管理（Sub PMIC）

Sub PMIC 高度集成架构 充电器+电量计+闪光灯。

**Sub PMIC 工作原理**

- 高度集成：充电器 + 电量计 + OTG + Flash LED + RGB LED → 单芯片替代 3~4 颗分立方案
- 双输入 (SM5735)：支持 VBUS1 (有线) + VBUS2 (无线) 同时存在，自动切换优先级
- Flash LED (SM5038)：高电流脉冲驱动相机闪光灯，需精密电流控制和热保护（闪光时瞬时电流可达 1.5A+）
- 状态机协调：管理充电策略（CC/CV/涓流）、SOC 计算、LED 控制时序、温度保护

> **Sub PMIC vs 分立方案**：集成方案减少 BOM（省 2~3 颗 IC）、减少 PCB 面积（省 30%+）、简化软件驱动（单一 I²C 地址）。代价是灵活性降低，各子功能不能独立选型。

| 参数 | SM5735 | SM5038 |
|------|--------|--------|
| 充电器 | 3.5A | 3.5A |
| OTG | 1.5A | 1.5A |
| 双输入 | ✓ | ✗ |
| Flash LED | ✗ | ✓ |
| 电量计 | ✓ | ✓ |
| 封装 | 81 WLCSP | 81 WLCSP |

---

## 九、OLED 电源管理

OLED vs LCD 电源架构对比 ELVDD/ELVSS/Burn-in 补偿。

**OLED 电源架构工作原理**

- ELVDD (+)：OLED 像素阳极正高压，提供驱动电流 → Boost 升压产生
- ELVSS (-)：OLED 像素阴极负压 → 反相电荷泵或 Inverting Buck-Boost 产生
- 无背光：OLED 自发光，不需要 LED 背光驱动 → 但需要 ELVDD/ELVSS 大电流正负高压
- Burn-in 补偿：OLED 像素长期静态显示会老化 → 亮度衰减不均匀 → PMIC 需配合 TCON 做像素级亮度补偿
- Logic IC：OLED 时序控制与面板驱动之间的接口芯片，类似 LCD 的 TCON 功能
- Mura 补偿：OLED 面板制造不均匀导致亮度差异 → 出厂校准数据存储在面板 OTP 中

> **OLED vs LCD 核心差异**：LCD 用 VCOM 控制 Flicker（公共电极偏压），OLED 没有 VCOM 但有 Burn-in（像素老化）问题。OLED 不需要 LED 背光但需要 ELVDD/ELVSS 正负高压大电流，电源设计更复杂。

| 应用 | 型号 | 功率等级 | 说明 |
|------|------|----------|------|
| 手机 | SM30SA~D | 低→高 | 4 档功率覆盖不同屏幕尺寸 |
| 笔记本 | SM30NA/NC | 中/高 | 大尺寸需高功率 |
| 笔记本 | SM30NB | - | OLED 逻辑 IC |
| 平板 | SM30TA/TB | - | 电源 IC + 逻辑 IC |
| 电视 | SM48PM | - | OLED PMIC |
| 电视 | SM48LS | - | OLED 电平转换器 |
| 汽车 | SM30AA/AB | 中/高 | 车规 OLED 电源 IC |


