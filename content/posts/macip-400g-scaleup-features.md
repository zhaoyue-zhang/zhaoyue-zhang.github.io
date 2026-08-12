---
title: "MACIP 400G Scale-Up 三大特性详解"
date: 2026-08-12
draft: false
tags: ["Scale-Up", "CBFC", "LLR", "Multicast", "MAC"]
description: "从协议视角系统梳理面向 Scale-Up 超节点互联的 MAC IP 新增三大核心特性：CBFC 基于信用的流控、LLR 链路级重传、Multicast Retry 多播重传，以及它们如何协同实现无损低延迟的超节点网络。"
---

> 本文从协议视角出发，系统梳理一种面向 Scale-Up 超节点互联的 MAC IP 新增的三大核心特性：CBFC（基于信用的流控）、LLR（链路级重传）、Multicast Retry（多播重传），以及它们如何协同实现无损、低延迟的超节点网络。

## 1. CBFC — Credit Based Flow Control（基于信用的流控）

### 1.1 概述

CBFC 是 IEEE 802.1Q-2022 标准 PFC（Priority-based Flow Control）的替代方案，旨在消除因接收端缓冲区拥塞导致的帧丢失。

CBFC 采用信用（Credit）机制：发送端以 credit 为单位跟踪接收端的可用缓冲区空间，只有当接收端有足够 credit 时才允许从无损 VC 队列调度数据包发送。接收端根据端口缓冲区可用性生成 credit，并通过 CBFC 消息将 credit 返回给发送端。

### 1.2 关键特性

- **信用计数器**：支持 CC（Credit Consumption）、CF（Credit Free）、CU（Credit Used）三种信用计数器，分别用于消耗统计、释放统计和已用信用跟踪
- **VC 通道管理**：支持最多 32 个虚拟通道（VC0~VC31）独立流控，每个 VC 有独立的 Credit Limit 和状态寄存器
- **CBFC 消息类型**：CC_Update（发送端→接收端，报告信用消耗）；CF_Update（接收端→发送端，返回已释放信用）
- **溢出检测**：当 S_VC_CU 计数器达到 Credit Limit 时，溢出信号置位，支持 32bit 位掩码标识溢出 VC
- **链路故障联动**：CBFC 支持 RX/TX Link Fault 使能，链路故障时可自动丢弃 credit 错误消息
- **定时器机制**：CC_TIMER（CC 消息发送间隔）、CF_TIMER（CF 消息发送间隔）、MINI_TIMER（最小间隔）、OVERFLOW_TIMEOUT（溢出超时检测）

### 1.3 硬件接口信号

| 信号 | 说明 |
|------|------|
| TX_FIFO_READ_EN_I[31:0] | TX FIFO 读使能，每 bit 对应一个 VC，用于 CC 计数器计算 |
| RX_FIFO_WRITE_EN_I[31:0] | RX FIFO 写使能，每 bit 对应一个 VC，用于 CC 计数器计算 |
| RX_FIFO_READ_EN_I[31:0] | RX FIFO 读使能，每 bit 对应一个 VC，用于 CF 计数器计算 |
| CBFC_OVERFLOW_O[31:0] | VC 信用溢出指示，每 bit 对应一个 VC |

### 1.4 CBFC 寄存器配置要点

| 寄存器 | 地址 | 作用 |
|--------|------|------|
| CBFC_CC_TIMER | 0x200 | CBFC 使能、RX/TX LF 使能、CC 消息定时器、错误消息 credit 丢弃 |
| CBFC_VCn_CL | 0x203~0x222 | 32 个 VC 的 Sender Credit Limit 配置，每个 20bit |
| CBFC_TX/RX_PKTOVHD_VC_Nn_Nm | — | TX/RX 方向各 VC 通道的包开销配置 |
| CBFC_S_VCn_CC/CU/CF_STATUS & CREDIT_REMAIN | — | 各 VC 的 CC/CU/CF 状态及剩余信用（只读） |

## 2. LLR — Link Level Retry（链路级重传）

### 2.1 概述

LLR 是基于帧的链路级重传机制，用于实现无损传输。当对端 MAC 也支持 LLR 时，两端协同工作。TX 为每个 LLR-eligible 帧分配 20bit 序列号并存入 replay buffer；链路对端成功接收后回复 LLR_ACK；若检测到丢失帧则回复 LLR_NACK。超时机制保证在 ACK/NACK 丢失或重传损坏时仍能触发重传。

### 2.2 LLR Preamble 格式

LLR 帧与标准以太网帧的区别在于前导码（Preamble）：

- **Byte 0**：固定为 0xFB
- **LLR-ineligible 帧**：Byte 1~6 为标准 0x55，Byte 7 固定为 0xD5
- **LLR-eligible 帧**：Byte 1 低 4 位为 0x7，同时 Byte 1~3 携带 20bit 帧序列号（frame_seq[19:0]），Byte 4~6 为 0x55，Byte 7 固定为 0xD5

20bit 序列号支持 800 Gbit/s 下最小 64 字节帧的超 500μs 往返时间。

> **Preamble 对比**：LLR-ineligible = `0xFB + 0x5555555555 + 0xD5`；LLR-eligible = `0xFB + {0x7, seq[19:0]} + 0x555555 + 0xD5`

### 2.3 Control Ordered Sets (CtlOS)

LLR 使用特殊的 Control Ordered Sets 发送控制消息。PCS 以抢占方式将 CtlOS 周期性插入数据流，保证控制消息的最大延迟和带宽。共 4 种 CtlOS：

| CtlOS 类型 | 作用 |
|-----------|------|
| LLR_ACK | LLR 帧确认，用于释放 replay buffer 空间 |
| LLR_NACK | LLR 帧否定确认，表示检测到缺失的 LLR-eligible 帧 |
| LLR_INIT | 初始化对端的 next_rx_seq 状态 |
| LLR_INIT_ECHO | 表示已收到并处理 LLR_INIT，本端就绪可接收 LLR 帧 |

### 2.4 状态机行为

| 状态 | 说明 |
|------|------|
| INIT | 由 llr_init_behavior 控制 — BLOCK(0) / DISCARD(1) / BEST_EFFORT(2) |
| ADVANCE / REPLAY | 正常传输和重传状态，LLR_status = TRUE |
| FLUSH | 由 llr_flush_behavior 控制 — BLOCK(0) / DISCARD(1) / BEST_EFFORT(2) |

- **re_init_on_discard**：重传失败策略 — TRUE 自动重新初始化，FALSE 等待管理介入

### 2.5 关键配置寄存器

| 寄存器 | 位域 | 说明 |
|--------|------|------|
| LLR_EN / LLR_MODE_LOCAL / LLR_MODE_REMOTE | 0x100[2:0] | LLR 总使能及 TX/RX 独立使能 |
| ctlos_target_spacing | 0x100[30:16] | 连续 LLR_ACK/NACK CtlOS 之间的目标字节数（400~16384，建议 2048） |
| outstanding_seq_max | 0x101[19:0] | 最大未确认序列号数（0~511），默认 0x200 |
| outstanding_data_max | 0x102[19:0] | 最大未确认数据量，应设置为链路带宽延迟积 |
| replay_ct_max | 0x103[23:16] | 最大重传次数（0~255），全 1 表示无上限 |
| replay_timer_max | 0x103[15:0] | 重传定时器超时值（0~65535 ns） |

### 2.6 LLR 统计计数器（只读寄存器 0x10c~0x122）

**TX 侧**：TX_INIT_CTL_OS、TX_INIT_ECHO_CTL_OS、TX_ACK_CTL_OS、TX_NACK_CTL_OS、TX_DISCARD、TX_OK、TX_POISONED、TX_REPLAY

**RX 侧**：RX_INIT_CTL_OS、RX_INIT_ECHO_CTL_OS、RX_ACK_CTL_OS、RX_NACK_CTL_OS、RX_ACK_NACK_SEQ_ERR、RX_OK、RX_POISONED、RX_BAD、RX_EXPECTED_SEQ_GOOD/POISONED/BAD、RX_MISSING_SEQ、RX_DUPLICATE_SEQ、RX_REPLAY

## 3. Multicast Retry（多播重传）

### 3.1 概述

Multicast Retry 是连接交换网络场景下的多播无损传输机制，通过双向握手协议实现。最大支持 1024 个网络节点（1023 个不同远端客户端），最多 8 个多播地址，每个多播地址最多 16 个 MAC DA。

本地 MAC 发送多播包时存入 buffer，远端 MAC 收到后回复 ACK/NACK 包，本地 MAC 收到后清除对应 buffer。

### 3.2 协议要点

- 仅当当前组所有包已确认后才可切换多播组
- 多播 buffer 满时 MAC 阻塞数据通路
- 仅当多播包的所有 ACK 都收到后才释放 buffer
- MAC 对每个收到的多播包回复 ACK，丢包时回复 NACK
- 重传采用 Go-Back-N 方式，以多播格式重传
- 收到重复多播消息时丢弃，保证接收顺序

### 3.3 包格式

每个设备需要唯一的 DEVICE_ID[9:0]（10bit，支持 1024 设备），嵌入 MAC ADDR 的 Byte0 和 Byte1[7:6] 中。

多播数据包分为标准格式和非标准格式（GEN1/GEN2/LITE），支持普通包和 VLAN 包（不支持双 VLAN）。每个多播包插入 2B 的 Multi Info 字段（含 12bit ID 序列号和 2bit SEQ 组序列号）。控制包（CTRL Packet）固定 64B，通过 ACK Info 字段（2B）传递 ACK/NACK 信息。

### 3.4 关键接口与寄存器

| 接口/寄存器 | 说明 |
|------------|------|
| MULTI_DEVICE_ID_I[159:0] | 10×16 device ID |
| MULTI_DEVICE_ID_VLD_I[15:0] | device ID 有效指示 |
| MULTI_TX/RX_FAILED_*_O | 失败设备指示 |
| MULTI_TX_IDLE_O | 所有多播包已 ACK |
| MULTI_RETRY_CFG (0xd0) | 多播检查/重传使能、CTRL 消息 VLAN 使能、刷新行为、CTRL 消息 ETH TYPE |
| MULTI_TIMEOUT (0xd1) | 多播定时器最大值、自动重试次数上限（触发中断） |

## 4. 三大特性与标准流控对比

| 特性 | 解决什么问题 | 替代/对比 |
|------|------------|----------|
| CBFC | 防拥塞丢包 | PFC（8 优先级）→ CBFC（32 VC），粒度更细 |
| LLR | 防链路误码丢包 | 链路层重传，区别于端到端的传输层重传 |
| Multicast Retry | 防多播丢包 | 专用 Go-Back-N，区别于 LLDP/IGMP Snooping |

**三层协同**：CBFC（防拥塞丢包）+ LLR（防链路误码丢包）+ Multicast Retry（防多播丢包），共同实现 Scale-Up 超节点网络的零丢包目标。

## 5. 与传输层 IP 的交互

传输层 IP 通过以下方式使用 MAC 的 Scale-Up 特性：

- **CBFC 接口信号**：RX2MAC_CBFC_CREDIT_O（credit 指示）、MAC2TX_CBFC_RESET_I、MAC2TX_CBFC_VCID_I、MAC2TX_CBFC_NUM_I、MAC2TX_CBFC_PKTOVHD_LEN_I、MAC2TX_CBFC_CREDITSIZE_I、MAC2TX_CBFC_CREDITLIMIT_I
- **CBFC 数据流**：CBFC 模式下进入 TX_PKT_GEN 的 WQE 已有链路级 credit，不会被链路阻挡。Demux 判决后 WQE 写入 retry buffer，记录选路结果用于重传
- **LLR/CBFC 恢复流程**：闪断后软件恢复 LLR/CBFC 链路状态 → 传输层确认恢复平面所有 xpu_id 对应 sub_qp 无 during_retry 标识 → 更新状态
- **报文头映射**：使能 CBFC 时 DSCP 域体现 VC 信息，pri 域同时表示 CBFC 和 PFC 优先级映射
- **配置参数**：CBFC_BUF_DEPTH（每 MAC RX CBFC 缓存深度，默认 8）、CBFC_VC_NUM（CBFC VC 数量，默认 8）、RX_CBFC_BUF（512×512×262144，1R1W RAM）

## 6. 验证关键点

### 6.1 CBFC 验证

- Credit 计数正确性（CC/CF/CU 一致性检查）
- 32 VC 隔离与独立流控
- Overflow 检测与处理
- 链路故障（LF）下的 CBFC 行为
- 不同交换机 CBFC 适配差异验证

### 6.2 LLR 验证

- 20bit 序列号分配与回卷
- 4 种 CtlOS (ACK/NACK/INIT/ECHO) 生成与解析
- Replay Buffer 管理：满/空/回收状态
- FLUSH/INIT 行为组合测试 (BLOCK/DISCARD/BEST_EFFORT × 2)
- 超时重传 (replay_timer_max) 与最大重传次数 (replay_ct_max)
- 链路闪断恢复 + CtlOS 间距 (ctlos_target_spacing) 验证

### 6.3 Multicast Retry 验证

- 多播组切换条件验证
- 全 ACK 确认后 buffer 释放
- Go-Back-N 重传正确性
- 重复包丢弃 + DeviceID 嵌入与识别
- 最大 1024 节点压力测试

### 6.4 集成验证

- MAC + 传输层联合 CBFC/LLR 场景
- 多平面乱序传输 + 重排序
- 闪断旁路重传
- 不同协议帧头压缩兼容性

## 7. Q&A：TX_FIFO_READ_EN_I 信号深度解析

> 疑问：TX_FIFO_READ_EN_I[31:0] 为什么叫 read enable？它和 CC 计数器之间是什么对应关系？

### 数据通路回顾

TX 方向数据流：上层/传输层 → TX FIFO（MAC 内部）→ MAC TX 处理 → MII → PCS → 发送到链路。

TX_FIFO_READ_EN_I 是外部（上层/传输层）告知 MAC 某个 VC 的数据正在被 MAC 从 TX FIFO 读出并发送到链路上。

### "Read Enable"的含义

关键在于站在 MAC 的视角理解 FIFO 操作：

- **Write** = 上层把数据写入 TX FIFO（数据排队等待发送）
- **Read** = MAC 从 TX FIFO 读出数据并通过 MII 发送出去

所以 TX_FIFO_READ_EN 指的就是 MAC 正在从某个 VC 的 TX 队列中读取数据并往外发送——**发送 = 读出 FIFO = read enable = 1**。

### 与 CC 计数器的关系

TX_FIFO_READ_EN_I[x] 置位（MAC 从 VC_x 队列读出一个包准备发送）→ 该包的发送将消耗接收端的 buffer 空间 → S_VC_CC[x] 计数器递增（发送端记录：我为 VC_x 消耗了多少信用）→ 定时通过 CC_Update 消息发送给接收方。

也就是说，read enable 信号就是 CC 计数器的触发源——每读出一个包发送，就消耗一份 credit。

### 三组信号与三类计数器的完整对应

| 信号 | 方向 | 含义 | 驱动计数器 |
|------|------|------|----------|
| TX_FIFO_READ_EN_I[31:0] | TX 侧 | MAC 读出数据 → 发送 → 消耗对端 buffer | S_VC_CC（发送方记录消耗量） |
| RX_FIFO_WRITE_EN_I[31:0] | RX 侧 | 链路数据到达 → 写入 RX FIFO → 占用本端 buffer | R_VC_CC（接收方记录消耗量） |
| RX_FIFO_READ_EN_I[31:0] | RX 侧 | 上层读出 RX FIFO 数据 → 释放 buffer 空间 | R_VC_CF（接收方记录释放量 → CF_Update 返回发送方） |

### 完整闭环流程

```
发送方                                接收方
──────                              ──────
TX_FIFO_READ_EN → S_VC_CC++
打包 CC_Update 发送 ────────────→ 收到 CC_Update
                                   对比本地 CF，确认信用消耗

收到 CF_Update ←──────────── 打包 CF_Update 发送
S_VC_CU = CC - returned_credits    RX_FIFO_READ_EN → R_VC_CF++

S_VC_CU >= Credit_Limit ?
  是 → CBFC_OVERFLOW_O 置位
        → 暂停该 VC 发送
```

> **总结**："read enable" 就是 "MAC 正在发送这个 VC 的数据" 的信号。每发一个包就消耗对端一份 credit，所以它直接驱动 CC（Credit Consumption）计数器累加。命名是从 FIFO 操作视角来看的——发送 = 读出 FIFO = read enable = 1。

验证启示：在仿真中，可以通过监控 TX_FIFO_READ_EN_I 的位翻转来验证 CC 计数器是否按预期累加，以及对应 VC 的 credit 消耗是否正确反映在 CC_Update 消息中。同时注意区分 TX_FIFO_READ_EN_I（发一个包消耗一个 credit 单位）与 CC_Update 中上报值的量纲关系。

## 8. Q&A：CBFC 中的 LF（Link Fault）是什么意思

> 疑问：CBFC 寄存器中多次出现的 LF 是什么含义？

### 定义

LF = Link Fault（链路故障），是 IEEE 802.3 标准中定义的物理链路层状态指示信号。当底层 PCS/PMA/PMD 检测到物理链路出现问题时，会向上层 MAC 报告 Link Fault。

常见的 Link Fault 触发场景：

- 光纤断裂或铜缆断开（Loss of Signal）
- SerDes 失锁（Loss of Lock / CDR unlock）
- 信号质量不可恢复（过高 BER，Block Lock 丢失）
- 对端设备断电或复位
- PHY 内部致命错误

### 在 CBFC 中的体现

CBFC_CC_TIMER 寄存器 (0x200) 中定义了 CBFC 对 Link Fault 的响应使能：

| 位 | 名称 | 作用 |
|----|------|------|
| Bit[24] | CBFC RX LF ENABLE | RX 方向检测到 Link Fault 时的 CBFC 响应使能 |
| Bit[23] | CBFC TX LF ENABLE | TX 方向检测到 Link Fault 时的 CBFC 响应使能 |
| Bit[25] | CBFC Error message credit discard enable | FCS 错误包的 credit 是否丢弃（0=不丢弃，1=丢弃） |

### 为什么 CBFC 需要感知 Link Fault

CBFC 的信用循环依赖链路的双向通信：

- 发送方通过 CC_Update 消息向接收方报告信用消耗
- 接收方通过 CF_Update CtlOS 向发送方返还信用

一旦链路断了，这个循环就被切断：

> ⚠️ 链路故障 → 接收方收不到数据也无法返回 CF_Update → 发送方 credit 耗尽、等待释放 → 链路恢复后 credit 计数错乱或死锁

所以 RX/TX LF ENABLE 的作用就是在检测到 Link Fault 时，自动执行保护动作——例如丢弃积压的 credit 错误消息、复位 CBFC 状态机、或在链路恢复后触发 LLR/CBFC 的 re-init 流程。这保证了链路闪断后 credit 计数体系不会出现难以恢复的错乱。

### 验证要点

- LF 触发时 CBFC 状态机是否正确进入/退出安全状态
- LF 恢复后 credit 计数器是否清零/补回合理值
- RX LF ENABLE 和 TX LF ENABLE 分别独立使能的场景覆盖
- LF 与 LLR re-init 的联动

## 9. Q&A：CBFC_TX_PKTOVHD 寄存器详解

> 疑问：PKTOVHD 为什么用 2's complement？为什么范围是 -16 到 +127 而非 -128 到 +127？

### 名称拆解

- **CBFC** = Credit Based Flow Control
- **TX** = 发送方向
- **PKTOVHD** = Packet Overhead（每包开销字节数）
- **VC_N0_N1** = 本寄存器配置 Virtual Channel 0 和 Channel 1

### 位域结构（32bit）

```
Bit[31:22]  VC1 per-packet overhead  8bit 有符号 2's comp, -16~+127 bytes
Bit[21:12]  VC0 per-packet overhead  8bit 有符号 2's comp, -16~+127 bytes
Bit[11:0]   CreditSize               信用单元: 32|64|128|256|512|1024|2048 bytes
```

### 为什么需要 PKTOVHD

TX_FIFO_READ_EN 只标记了从 FIFO 读出的 payload 数据量，但一个以太网帧的实际链路占用远比 payload 大：

```
实际链路占用 = payload + Preamble(8B) + SFD(1B) + DA/SA(12B)
             + EtherType(2B) + FCS(4B) + IPG(≥12B)
             + VLAN(+4B) + 封装头...
```

如果只按 FIFO 读出的 payload 字节计 credit，就会系统性低估信用消耗，累积误差最终导致接收端 buffer 溢出。PKTOVHD 在每个包的 credit 计算中额外加上这笔开销：

```
credit_消耗 = ceil((payload_bytes + PKTOVHD) / CreditSize)

例：payload=64B, PKTOVHD=+32, CreditSize=64B
    credit = ceil(96/64) = 2 个 credit 单位
```

### 为什么用 2's Complement（二进制补码）

2's complement 是硬件表示有符号整数的标准方式。同一个寄存器域段既表示正偏移也表示负偏移，无需额外的符号位字段。硬件做加法时统一按补码处理，不区分正负，电路最简洁。

那为什么 per-packet overhead 需要负值？主要有两类场景：

- **基线校准**：系统可能在别处（如传输层的 credit 计算中）已预扣了默认开销，CBFC 若再用正偏移就会重复计数。配负的 PKTOVHD 可以抵消这部分 double count
- **压缩头协议**：AFH 压缩头（GEN1/GEN2/LITE）比标准以太网头更短，实际开销小于默认值。PKTOVHD 配负值使信用消耗更准确

### 为什么范围是 -16 到 +127（而非 -128 到 +127）

标准 8-bit 2's complement 范围是 -128 ~ +127。文档明确写 -16 ~ +127，说明负向做了限幅（clamping）。原因有三：

1. **物理约束 -16 够用**：标准以太网帧最小开销约 24B（前导码+IPG+FCS），压缩头最多减掉十几字节。配置小于 -16 在物理上不成立——你不会希望 credit 公式认为一个包比它的 payload 还小很多。限制负值范围防止了灾难性误配置
2. **正向用满 127**：各种封装组合（VLAN 堆叠、QinQ、UDP/IP encap、多平面标记等）可能累加到很大的 per-packet overhead，正向需要完整的 0~+127 范围
3. **硬件实现简化**：配置值 < -16 时硬件内部直接 clamp 到 -16，加法器/比较器无需处理极端负值的 corner case，减少组合逻辑面积和时序路径深度

> **总结**：-16 是硬件设计的 safety clamp——用最小的硬件代价消除了一个永远用不到但可能出 bug 的配置空间。

### TX 与 RX PKTOVHD 寄存器布局差异

- **TX_PKTOVHD**：每寄存器存 2 个 VC（首个寄存器另有 12bit CreditSize），共需 16 个寄存器覆盖 32 VC
- **RX_PKTOVHD**：每寄存器存 3 个 VC（不含 CreditSize，共享 TX 侧的配置），共需约 11 个寄存器覆盖 32 VC
- **差异原因**：TX 侧首个寄存器因夹了 CreditSize 字段，只能放 2 个 VC；RX 侧无此字段，可以每寄存器放 3 个 VC

### 验证要点

- PKTOVHD = 0 时 credit 消耗是否与裸 payload 一致
- 正值/负值场景下 credit 计算正确性
- PKTOVHD = -16（下限）和 +127（上限）边界测试
- 配置值 < -16 时硬件是否正确 clamp
- 不同 CreditSize (32~2048B) 与不同 PKTOVHD 的组合遍历