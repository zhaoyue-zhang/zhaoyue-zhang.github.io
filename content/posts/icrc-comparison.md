---
title: "ICRC 计算对比"
date: 2026-08-04
draft: false
tags: ["Ethernet"]
description: "四种 Ethernet 类协议（传统 802.3 FCS、UEC UET CRC、SUE R-CRC、字节跳动 ETHLINK ICRC）的 CRC 起点/终点、覆盖范围、可变字段处理策略及工程权衡；含一个决策树辅助选型。"
---

本文对比四种 Ethernet 类协议的 CRC 计算方案：

| 协议 | CRC 方案 | 端到端? |
| --- | --- | --- |
| 传统以太网 (IEEE 802.3) | FCS | ❌ 每跳重算 |
| UEC (UE-Spec 1.0.3) | UET CRC | ✅ |
| SUE (OCP Scale-Up Ethernet v1.0.2) | R-CRC | ✅（SUE Lite 无 R-CRC） |
| ETHLINK（字节跳动 OEFH + Net + Txn） | ICRC | ✅ |

下面从「字段级覆盖」「校验范围」「可变字段处理策略」三个维度展开。

## 表 1（扩展版）：四种类 Ethernet 协议 L2 头 → Payload 逐字段对比

> 新增 SUE 列（OCP Scale-Up Ethernet v1.0.2 + SUE Lite）。
> SUE 完整版用 AFH Gen 1 带 Shim Header + RH 头部；SUE Lite 用 AFH Gen 2 6B 不带 RH。

| 层级 | 字段 | 位宽 | 普通以太网 (IEEE 802.3) | UEC (UE-Spec 1.0.3) | SUE (OCP Scale-Up Ethernet v1.0.2) | ETHLINK (字节跳动 OEFH + Net + Txn) |
| --- | --- | --- | --- | --- | --- | --- |
| Datalink (L2) | 目的地址 | 48/16/12/6 | Dst MAC (48b) | Dst MAC (48b) | AFH Gen 1: Dst MAC (48b, 16-32b 用于 XPU-id 转发)<br>AFH Gen 2: Dst XPU-id (16b 或 32b, 嵌在 MAC 地址里) | Dst GPU-ID (16b) |
|  | 源地址 | 48/16/12/6 | Src MAC (48b) | Src MAC (48b) | AFH Gen 1: Src MAC (48b)<br>AFH Gen 2: Src XPU-id (16b 或 32b) | Src GPU-ID (16b) |
|  | 长度/类型 | 16 | Length/Type (16b) | EtherType (16b) | Ethertype (16b, 标识 AFH Gen 1/Gen 2, 可选第二 Ethertype 区分多传输头) | Ethernet Type (16b) — vlan/ethlink |
|  | 总长度 | 16 | — | — | — (AFH 内嵌在 MAC 中) | Total Length (16b) |
|  | 熵 / 哈希 | 16 | — | UET Entropy Header: entropy (16b) | AFH Gen 2 Normal: Hop Count + Entropy (在 MAC 里)<br>AFH Gen 2 Compressed: 去掉 Hop Count / Entropy | Entropy (16b) |
|  | Hop Count (SUE 特有) | var | — | — | AFH Gen 2 Normal: 嵌在 MAC 地址里 | — (用 TTL 替代) |
|  | Datalink Reserved | 16 | — | UET Entropy Header: rsvd (16b) ⚠️ tx=0, rx=ignore | Shim Header (8B): 类似 IP 头, 含 reserved (按 SUE 规则) | Reserved (16b) — 算 ICRC 时按原值 |
|  | Shim Header (SUE 特有) | 64 | — | — | AFH Gen 1 Shim (8B, 字段类似 IP 头, 见 Figure 17) | — |
|  | Datalink Reserved | 16 | — | — | — | Reserved (16b) — 算 ICRC 时按原值 |
|  | (VLAN tag) | 32 | 可选 802.1Q | — | 802.1Q 可选（无 Shim 模式下靠 VLAN 做优先级） | 在多租户场景下可选，和以太网完全兼容 |
| Network (L3) | AR 自适应路由 | 1 | — | — | — (在 Shim 头或 AFH 头中) | AR (1b) |
|  | Network Reserved | 3 | — | — | — | Reserved (3b) — tie1 |
|  | TTL | 4/8 | (IPv4) TTL (8b) | (IPv4) TTL (8b) | Shim Header 中有类似 TTL 字段 (按图 17 推测) | TTL (4b) — tie1 |
|  | DSCP | 6 | (IPv4) DSCP (6b) | (IPv4) DSCP (6b) | Shim Header 中 (DSCP → 802.1P 映射) | DSCP (6b) — tie1 |
|  | ECN | 2 | (IPv4) ECN (2b) | (IPv4) ECN (2b) | Shim Header 中 (PFC/CBFC 用) | ECN (2b) — tie1, 协议固定 b00 |
|  | Version/IHL/Flags/Frag/Proto/Cksum | 32 | IPv4 头其他字段 | IPv4/IPv6 头其他字段 | Shim 头中其他字段 | — (ETHLINK 简化头, 总长仅 2B) |
|  | Source IP | 32/128 | Src IP | Src IP | — (在 AFH/ Shim 中) | — (XPU-id 已在 L2) |
|  | Destination IP | 32/128 | Dst IP | Dst IP | — | — |
|  | UDP 头 | 64 | (上层) Src port, Dst port, Length, Checksum | (上层) Src port 用作 Entropy, Checksum 强制 0 | (上层) Src/Dst port (标准格式) | — |
| Transaction | Type | 5 | — | pds.type (5b) | RH: ver (2b) + op (2b) | pds.type (5b) — 假设沿用 PDS |
|  | Next header / Ctl type | 4 | — | pds.next_hdr (4b) | (op 已经编码 ACK/NACK/无效) | pds.next_hdr (4b) — 假设 |
|  | Reserved | 2/4 | — | (flags 内部 rsvd 4b) | RH: rsv (2b) — 算 R-CRC 时按原值 | (flags 内部 rsvd 4b) — 假设, 按原值 |
|  | XPU-id | 10 | — | (在 PDS 里是 spdcid/dpdcid) | RH: xpuid (10b) | spdcid (16b) — 假设 |
|  | Flags | 7 | — | pds.flags (7b) | — (在 op + 其他字段里) | pds.flags (7b) — 假设 |
|  | PSN | 32 | — | pds.psn (32b) | RH: npsn (16b, Next PSN) | psn (32b) — 假设 |
|  | Virtual Channel | 2 | — | (在 UDP port) | RH: vc (2b) | — |
|  | Reserved | 4 | — | — | RH: rsvd (4b) — 算 R-CRC 时按原值 | — |
|  | Partition | 10 | — | — | RH: partition (10b) | — |
|  | Src PDCID | 16 | — | spdcid (16b) | (xpuid 已在 RH 内) | spdcid (16b) — 假设 |
|  | Dst PDCID | 16 | — | dpdcid (16b) | — | dpdcid (16b) — 假设 |
|  | ACK PSN | 16 | — | ack_psn_offset / cack_psn | RH: apsn (16b, ACK/NACK PSN) | clear_psn_offset (16b) — 假设 |
|  | Clear PSN offset | 16 | — | clear_psn_offset (16b) | — | clear_psn_offset (16b) — 假设 |
|  | Total | — | — | 96 bits (12B) | 64 bits (8B) | 96 bits (12B) — 假设 |
| SES | Semantic header | var | — | ses.next_hdr + 扩展头 | — (SUE 无 SES 子层) | — |
| Payload | 业务数据 | var | 应用数据 | UET payload (含 MSG payload + 操作) | 打包 PDU (多个命令打包, 最多 4 KB; SUE Lite 1 KB) | 业务数据 |
| CRC | 完整性校验 | 32 | FCS (MAC 帧尾) — 每跳重算 | UET CRC (UET trailer) — 端到端 | R-CRC (RH+payload 之后) — 端到端; SUE Lite 无 R-CRC | ICRC (payload 之后) — 端到端 |
| Pad | 对齐填充 | var | 按需 | 按需 | 按需 (不参与 R-CRC) | 按需 (不参与 ICRC) |
| FCS | 链路层校验 | 32 | 与 ICRC 字段重叠 (即 FCS 本身) | 底层 Ethernet FCS | 底层 Ethernet FCS (LLR 也可能改 FCS, 见 spec) | 底层 Ethernet FCS |

## 表 2：四种协议 CRC/校验规则对比（核心差异）

| 维度 | 普通以太网 (IEEE 802.3) FCS | UEC (UE-Spec 1.0.3) UET CRC | SUE (OCP Scale-Up Ethernet v1.0.2) R-CRC | ETHLINK ICRC |
| --- | --- | --- | --- | --- |
| CRC 计算起点 | L2 头 DA 首字节 | L3 头源 IP 首字节（跳过 L2） | RH 首字节（跳过 L2 AFH 头） | L2 头 Dst GPU-ID 首字节（OEFH 起点） |
| CRC 计算终点 | payload 末字节（不含 FCS） | UET payload 末字节（不含 UET CRC） | 打包 PDU 末字节（不含 R-CRC） | payload 末字节（不含 ICRC） |
| 是否覆盖 L2 头 | ✅ 是 | ❌ 否 | ❌ 否 | ✅ 是 |
| 是否覆盖 network header | — | ✅ 是（IP 头非可变字段） | ✅ Shim Header（如有） | ✅ 是（含 reserved/TTL/DSCP/ECN） |
| 是否覆盖 transaction header | — | ✅ 是（PDS 头） | ✅ 是（RH 头） | ✅ 是（含 reserved 按原值） |
| 算法 | CRC-32 (IEEE 802.3) | CRC-32C (Castagnoli) | (SUE spec 未明说) | CRC-32 (IEEE) |
| 多项式 | 0x04C11DB7 | 0x1EDC6F41 | —— | 0x04C11DB7 |
| 初始值 | 0xFFFFFFFF | 0xFFFFFFFF | —— | 0xFFFFFFFF |
| 结果处理 | XOROUT 0xFFFFFFFF (取反) | XOROUT 0xFFFFFFFF (取反) | —— | 按位取反 |
| "123456789" 参考值 | 0xCBF43926 | 0xE3069283 | —— | 0xCBF43926 |
| Datalink 头 reserved 字段 | 无 | UET Entropy rsvd: tx=0, rx=ignore | Shim 头中按 spec | 按原值参与 ICRC |
| Network 头 reserved/TTL/DSCP/ECN | — | UET CRC 不动 (按原值) | Shim 头按 spec (未明 tie1 规则) | tie1 处理 |
| Transaction 头 reserved 字段 | — | pds.flags 内部 rsvd: 按 spec 置 0, ignore | RH: rsv (2b) + rsvd (4b) — 按原值算 R-CRC | 按原值参与 ICRC |
| 报文聚合后 | (无概念) | 聚合后只算一次 UET CRC | 打包后只算一次 R-CRC | 聚合后只算一次 ICRC |
| Lite / 简化版本 | — | — | SUE Lite: 移除 R-CRC 进一步降低开销 | (无) |
| 加密互斥 | — | 与 TSS ICV 互斥 | (无) | (无) |
| 重算/端到端 | 每跳重算 | 端到端 | 端到端 | 端到端 |
| 必选/可选 | 必选 | 可选 (UET_Data_Protect) | 必选 (SUE 完整版), Lite 无 | (按你描述是必选) |

## 表 3：报文整体结构对比（侧栏视角）

> 原文是 Notion 中的交互式 HTML 对比图（被 Notion 用临时签名 URL 嵌入）。这里以 iframe 永久引用本地拷贝：

<iframe class="icrc-frame-embed" src="/icrc-embed/frame_structure_comparison.html" loading="lazy"></iframe>

## 关于可变字段的处理

路径上的交换机会修改某些字段（典型如 DSCP/ECN/TTL 重标记）。端到端 CRC 必须对"路径会改的字段"做特殊处理，否则接收端校验必失败。下面看四种策略的差异。

### 表 1：四协议可变字段处理策略对比

| 协议 | 校验类型 | 端到端? | 可变字段集合 | 处理策略 | 屏蔽值 | 实现机制 | spec 引用 |
| --- | --- | --- | --- | --- | --- | --- | --- |
| 传统以太网 (IEEE 802.3) FCS | CRC-32<br>0x04C11DB7 | ❌ 每跳重算 | • 802.1Q VLAN tag（Trunk 加 / Access 删）<br>• PCP / CFI-DEI（QoS 重标记）<br>• Preamble / SFD（物理层，不算进 FCS） | 每跳重算（不是 tie，是完全重算） | N/A | 透明转发 DA/SA/Type/Payload，每跳重新算整帧 FCS | "添加后，FCS被重新计算" |
| UEC (UE-Spec 1.0.3) UET CRC | CRC-32C<br>0x1EDC6F41 | ✅ 端到端 | IPv4/IPv6 mutable 字段（不覆盖）：<br>• IPv4: DSCP, ECN, TTL, Header Checksum, Total Length, Identification, Flags+Frag Offset<br>• IPv6: Traffic Class, Flow Label, Hop Limit, Payload Length<br>UDP header: checksum 强制清 0 | 白名单（明确只覆盖 non-mutable） | N/A | Roff 偏移量 + Figure 3-101 图形标出覆盖/不覆盖范围 | "The CRC is designed to include non-mutable portions of the IP headers" / "checksum is set to zero" |
| SUE (OCP Scale-Up Ethernet v1.0.2) R-CRC | CRC（算法未明） | ✅ 端到端 | • RH 头字段（ver/op/xpuid/npsn/vc/partition/apsn）—— 会话级，路径不变<br>• Shim header（如果存在）—— 推测有 TTL/DSCP，spec 未明说 | 未明 | 未明 | 未明 | "R-CRC 在 RH 和打包操作上添加" —— 仅此而已 |
| ETHLINK ICRC | CRC-32<br>0x04C11DB7 | ✅ 端到端 | Network header mutable 字段（tie1）：<br>• Reserved (3b) → tie1<br>• TTL (4b) → tie1<br>• DSCP (6b) → tie1<br>• ECN (2b) → tie1<br>OEFH reserved (16b × 2) → 按原值<br>Transaction reserved → 按原值 | 黑名单 tie1 | 1 | 计算前对 4 个字段做 bit-mask（OR 0xFFF...F） | 用户规则（ICRC 描述） |

### 表 2：四策略工程维度对比

| 维度 | 每跳重算 (传统以太网) | 白名单 (UEC) | 黑名单 tie1 (ETHLINK) | 未明 (SUE) |
| --- | --- | --- | --- | --- |
| 校验范围 | 整 MAC 帧 | Roff 偏移量 → 终点 | OEFH → payload | RH → 打包 PDU 末 |
| 是否覆盖 L2 头 | ✅ 是 | ❌ 否（跳过） | ✅ 是 | ❌ 否（跳过 AFH） |
| 覆盖网络头 mutable 字段 | N/A（L2 没有） | ❌ 不覆盖 | ✅ 覆盖但 tie1 屏蔽 | 未明 |
| 覆盖网络头 fixed 字段 | N/A | ✅ 覆盖 | ✅ 覆盖（按原值） | 未明 |
| 处理"路径改字段"的能力 | 透明：路径改了 → 重算 FCS | 透明：mutable 不在范围 → 不受影响 | 显式：tie1 把可变值变成常数 → 不影响 | 取决于实现 |
| 硬件实现复杂度 | 简单（每跳整帧重算） | 简单（一个 Roff 配置） | 中等（计算前 +1 个 bit-mask 周期） | 未知 |
| 协议升级时 | 任何新字段都重算 | 新增 mutable 字段：自动不进 CRC（零成本）<br>新增 fixed 字段：需更新 Roff | 新增 mutable 字段：需更新 tie 列表<br>新增 fixed 字段：自动进 CRC（零成本） | 未知 |
| "报 CRC 错"时定位能力 | N/A | 排除 Roff 范围外的字段，可缩小怀疑 | 排除 tie1 字段，可缩小怀疑 | 未知 |
| "路径不能改的字段"是否进校验 | 全部进（每跳重算） | 全部进（白名单） | 部分进（fixed 进，reserved 进） | 未知 |
| 路径上"无意改某字段"的容错 | ✅ 自动（重算） | ✅ 自动（不在范围） | ✅ 自动（tie 屏蔽） | 取决于实现 |
| 路径上"无意改某字段"的检测 | ❌ 测不出（重算就过了） | ❌ 测不出（不在范围） | ❌ 测不出（tie 屏蔽） | 未知 |
| 是否支持端到端完整性证明 | ❌（每跳重算） | ✅ | ✅ | ✅ |

### 表 3：决策树

{{< mermaid >}}
flowchart TD
    start["是否需要端到端校验?"]

    start -->|"否<br/>（例如 L2 链路层）"| no_e2e["选 每跳重算<br/>（传统以太网 FCS）"]

    start -->|"是"| mutable{"路径会修改<br/>mutable 字段吗?"}

    mutable -->|"否<br/>（例如专用 fabric,<br/>端到端透明）"| blacklist_tie1["选 黑名单 tie1<br/>（ETHLINK 思路）<br/>━━━━━━━━━━━━━━━<br/>优点: 覆盖范围最广<br/>缺点: 需要 bit-mask 屏蔽"]

    mutable -->|"是<br/>（例如 AI 集群,<br/>路径有 ECN/trimming）"| field_fixed{"字段集合<br/>明确固定?"}

    field_fixed -->|"是"| whitelist["选 白名单<br/>（UEC 思路）<br/>━━━━━━━━━━━━━━━<br/>优点: 硬件简单, 数学干净<br/>缺点: 协议升级可能要改 Roff"]

    field_fixed -->|"否"| blacklist_tie01["选 黑名单 tie0/tie1<br/>（IPSec AH / ETHLINK 思路）<br/>━━━━━━━━━━━━━━━<br/>优点: 灵活, 升级时只改 tie 列表<br/>缺点: 需要维护 tie 列表"]

    mutable -->|"干脆不要端到端 CRC<br/>（短距/低延迟场景）"| sue_lite["SUE Lite 模式"]
{{< /mermaid >}}

## 小结

- **传统以太网 FCS**：每跳重算，**不是端到端**；在需要端到端完整性证明的场景不能直接用。
- **UEC UET CRC**：白名单策略（明确只覆盖 non-mutable），硬件简单但需要 spec 明确标注每个字段是否 mutable。
- **SUE R-CRC**：路径策略明确（端到端 + 不覆盖 AFH 头），但 spec 对可变字段的处理细节未公开。
- **ETHLINK ICRC**：黑名单 + tie1（计算前 bit-mask 屏蔽 mutable 字段），覆盖范围最广，协议升级时只需更新 tie 列表。