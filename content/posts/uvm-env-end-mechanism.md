---
title: "UVM环境结束机制"
date: 2026-07-29
draft: false
tags: ["UVM Knowledge"]
description: "UVM 验证环境何时才算真正结束？从 DUT 接口收包、RM 队列状态到 objection 投票机制，梳理一套普适的环境结束判断方法。"
---

需要两个条件同时满足才能结束环境。

## 条件一：DUT 接口全部收到

- 发的包 DUT 接口全部收到，两边 cnt 相等。
- **开始发包前**：raise objection。
- **结束发包时**：drop objection。

> 实现位置：可以在 base 中实现，也可以在 seq 中实现。因为当前发包都放在 base 中，所以放在 base。

## 条件二：DUT 处理完

### 判断方式

出口所有包都发出去，出口个数 = 入口个数。

### 丢包也要考虑

丢包统计：`丢包数 + 出口数 = 入口数`。

> 例如重传模块，输出的包和入口不一致，不好统计，不能通过简单的 cnt 计数。

### 普适性结论

> 所以检测 RM 中数据结构达到预期状态，才退出。

当 RM 中所有的数据结构都为空 → drop objection。
如果数据结构非空，但 objection 都为 0 → raise 一个（表示还有数据在处理但不再有新输入）。

## Common 组件 Objection 的控制

### Monitor

- 收到 valid 握手并且包发完毕之后 → drop objection（一个包一次 raise/drop）。

### Driver & Monitor

- 是一对关系。Driver 是有包开始驱动时 raise，包驱动完毕时 drop objection。生命周期也是一个包。
- ⚠️ **不能**作为环境结束机制，因为 drop 的时候可能还有包没驱动完。

### Scoreboard (SCB)

- 收到包（不管是 exp 还是 act）→ raise。
- 如果 exp FIFO，act FIFO 不为空 → 不 drop。
- 如果 exp 和 act 对完了 → drop。

> ⚠️ SCB 也有问题：如果长时间没有发激励，或者激励因为某些原因一直没达到 SCB，SCB 就不会 raise。

### 综合结论

上述组件单独看都不够完备，需要结合起来判断。

## 为什么看 RM 队列状态就可以判断环境结束？

- Monitor 会给 RM 拿包，这个包会流转在 queue 中，最后发到 SCB 中。
- **总有队列是非空的**（只要还有数据在流转）。
- 如果队列空了 → 说明包一定发到 SCB 了，结合 SCB，就可以判断环境是否结束。

> 所以一定是 **monitor + scb + rm 状态** 三者结合，才能判断环境是否可以结束。（包已经从 DUT 出去了）

## 最终结论：两条路径

| 路径 | 说明 | 局限性 |
| --- | --- | --- |
| **1. 数 cnt** | 入包数 = 出包数 | 虽然简单，但 RTL 复杂时不行，很多环境不能用 |
| **2. 投票机制** | 所有组件结合起来，seq / driver / monitor / rm 所有 objection 都 drop 才可以 | 更普适 |

## raise / drop objection 的细节

- 可以在一个组件中写一个 `this`，就是这个组件的 objection 机制。
- base 中结束看的是 objection 的 **total cnt**。

## 预期之内的异常结束

- 检测到中断 → 直接结束。
- 根据自己需求，决定是否要异常恢复、启动业务。

## 预期之外的异常

> 待补充：需要在实际项目中根据具体场景定义处理策略。
