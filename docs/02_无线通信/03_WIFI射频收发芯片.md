# WiFi射频收发芯片 (WiFi RF Transceiver)

---

## 一、产品概述

### 1.1 产品定位
WiFi射频收发芯片用于WiFi路由器/AP、终端设备、IoT设备中，实现WiFi信号的射频收发。

### 1.2 应用场景
- WiFi 7 (802.11be) / WiFi 6E (802.11ax) 路由器/AP
- 智能手机/平板 WiFi FEM (Front-End Module)
- IoT WiFi设备 (智能家居, 可穿戴)
- 企业级AP, Mesh WiFi系统

### 1.3 技术路线
- 频段: 2.4GHz + 5GHz + 6GHz (WiFi 6E/7 三频)
- 带宽: 20/40/80/160/320MHz (WiFi 7新增320MHz)
- 调制: 最高4096QAM (WiFi 7)
- MIMO: 2x2, 4x4, 8x8
- 工艺: RF-CMOS, RF-SOI (主流)
- 特殊: MLO (Multi-Link Operation), 多频段聚合

---

## 二、工作原理

### 2.1 WiFi vs 蜂窝通信差异

| 特性 | WiFi | 5G NR |
|------|------|-------|
| 频段 | 2.4/5/6 GHz ISM | 授权 Sub-6/mmWave |
| 带宽 | 20~320 MHz | 5~400 MHz |
| 调制 | up to 4096QAM | up to 256QAM (R17) |
| 发射功率 | <30dBm (1W) | 基站>40dBm |
| 接入 | CSMA/CA | 调度 |
| 成本 | 极低 (<$5) | 较高 |
| MIMO | 2x2~8x8 | 64T64R |

### 2.2 架构

WiFi RF收发芯片通常集成:
1. 射频前端: LNA, PA Driver, TR Switch
2. 收发器: Mixer, LPF, PGA, ADC/DAC
3. 频率合成器: 双频/三频PLL
4. 部分数字: 802.11 PHY基带(DFE, FFT, 信道估计, 均衡)

架构主流为ZIF (零中频):
```
RX: 天线 -> TR Switch -> LNA -> I/Q Mixer -> LPF -> VGA/PGA -> ADC -> PHY基带
TX: PHY基带 -> DAC -> LPF -> I/Q Mixer -> PA Driver -> PA -> TR Switch -> 天线
```

### 2.3 WiFi 7 (802.11be) 新特性

- 320MHz带宽 (6GHz频段)
- 4096QAM调制 (EVM要求极严: <-38dB)
- 16x16 MU-MIMO
- MLO (Multi-Link Operation, 多频段同时传输)
- 前导码打孔 (Preamble Puncturing)

---

## 三、芯片框图

```
+====================================================================+
|                  WiFi 7 RF收发芯片 (三频 2x2)                        |
|                                                                    |
|  2.4GHz Path:                                                       |
|  +------+  +------+  +-------+  +------+  +------+  +------+      |
|  |LNA+  |->| Mixer|->|  LPF  |->| PGA  |->| ADC  |->|      |      |
|  |Switch|  | I/Q  |  | 20/40 |  | 0-30 |  |12bit |  |      |      |
|  +------+  +--+---+  |  MHz  |  | dB   |  |160M  |  | PHY  |      |
|               |      +-------+  +------+  +------+  |      |      |
|  +------+  +--+---+  +-------+  +------+  +------+  | Base |      |
|  |PA    |<-| Mixer|<-|  LPF  |<-| DAC  |<-|      |<-| band |      |
|  |Driver|  | I/Q  |  | 20/40 |  |12bit |  |      |  |      |      |
|  +------+  +--+---+  |  MHz  |  |160M  |  |      |  |      |      |
|               |      +-------+  +------+  +------+  +------+      |
|               |                                                    |
|  5GHz Path:  (与2.4GHz相同, BW 20/40/80/160MHz)                    |
|  6GHz Path:  (与2.4GHz相同, BW 20/40/80/160/320MHz, WiFi 7新频段)  |
|                                                                    |
|                    +----------+----------+                         |
|                    |  PLL/Synthesizer x3  |                        |
|                    |  2.4G + 5G + 6G     |                        |
|                    +----------+----------+                         |
|                                                                    |
|  +-------------------------------------------------------------+  |
|  |  共存管理 (Coexistence): BT/WiFi共存, LTE/WiFi共存            |  |
|  +-------------------------------------------------------------+  |
+====================================================================+
```

---

## 四、关键指标

### 4.1 接收链路

| 指标 | WiFi 6E规格 | WiFi 7规格 | 说明 |
|------|------------|-----------|------|
| 频段 | 2.4/5/6 GHz | 2.4/5/6 GHz | 三频 |
| 带宽 | 20~160 MHz | 20~320 MHz | WiFi 7新增320M |
| NF | <3dB | <3.5dB | |
| 增益 | 0~55dB | 0~55dB | |
| IIP3 | >+5dBm | >+5dBm | |

### 4.2 发送链路

| 指标 | 规格 |
|------|------|
| 线性输出功率 | +5~+10dBm (PA Driver) |
| EVM (1024QAM) | <-35dB (WiFi 6) |
| EVM (4096QAM) | <-38dB (WiFi 7, 极其严格!) |
| ACLR | <-45dBc |
| LO Leakage | <-40dBc |

### 4.3 LO

| 指标 | 规格 |
|------|------|
| 相噪@100kHz | <-105 dBc/Hz |
| 相噪@1MHz | <-130 dBc/Hz |
| 积分抖动 | <150fs RMS |
| 锁定时间 | <20us (快速信道切换) |

### 4.4 系统

| 指标 | 规格 |
|------|------|
| MIMO | 2x2/4x4 |
| 三频同时工作 | 支持 (MLO) |
| 功耗 (2x2三频) | <3W |
| 封装 | WLCSP/QFN |

---

## 五、测试方法

### 5.1 标准WiFi测试

测试设备: WiFi综测仪 (如 Keysight UXM, LitePoint IQxel, Rohde CMW)

支持的标准测试模式:
- 发射测试: TX power, EVM, spectral mask, ACLR
- 接收测试: RX sensitivity (PER<10% at specified power level)
- MIMO测试: 多stream EVM, 信道相关性

### 5.2 EVM测试 (4096QAM关键!)

WiFi 7 4096QAM的EVM要求<-38dB是极大挑战:
- 需要极低相噪PLL
- 需要高线性度TX链
- 需要精确I/Q校准
- PA的非线性必须通过预失真补偿

### 5.3 快速信道切换测试

WiFi要求极快的频率切换:
- 测量PLL从频率A -> 频率B的锁定时间
- 要求 <20us (802.11ax/be要求)

### 5.4 共存测试

WiFi/BT共存 (2.4GHz):
- BT TX 对 WiFi RX的阻塞测试
- WiFi TX 对 BT RX的影响

---

## 六、AE知识要点

1. 4096QAM EVM <-38dB是WiFi 7最大挑战
2. 320MHz带宽需要更宽带模拟前端设计
3. MLO(多链路操作)需要三频同时工作能力
4. 低功耗是关键 (电池供电终端)
5. 快速信道切换 <20us
6. 共存管理: 2.4G WiFi与BT共享天线和频段
7. 成本极度敏感 (<$5 BOM成本)
8. 不同国家/地区的频谱法规和认证要求

---

*文档结束 - 版本 v1.0*
