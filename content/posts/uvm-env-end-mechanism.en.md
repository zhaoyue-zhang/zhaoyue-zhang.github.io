---
title: "UVM Environment End-of-Simulation Mechanism"
date: 2026-07-29
draft: false
tags: ["UVM Knowledge"]
description: "When is a UVM verification environment truly done? From DUT interface packet reception and RM queue status to the objection voting mechanism, a general-purpose approach to determining environment completion."
---

Two conditions must be satisfied simultaneously before the environment can end.

## Condition 1: All packets received at the DUT interface

- All sent packets are received at the DUT interface; the counts on both sides are equal.
- **Before sending starts**: raise objection.
- **When sending finishes**: drop objection.

> Where to implement: in the base sequence or in the sequence itself. Since packet sending currently lives in the base, it is placed there.

## Condition 2: The DUT has finished processing

### How to judge

All packets have been sent out of the output; output count = input count.

### Dropped packets must also be considered

Drop accounting: `dropped + output = input`.

> For example, a retransmission module's output may not match its input, making simple cnt counting unreliable.

### General conclusion

> So we only exit when the data structures inside the RM have reached their expected state.

When all data structures in the RM are empty → drop objection.
If a data structure is non-empty but all objections are already 0 → raise one (indicating data is still being processed but no new input is coming).

## Objection control in common components

### Monitor

- After a valid handshake is seen and the packet has been fully emitted → drop objection (one raise/drop per packet).

### Driver & Monitor

- They come as a pair. The Driver raises objection when it starts driving a packet and drops it when driving is done; its lifecycle is also one packet.
- ⚠️ **Cannot** serve as the environment end mechanism, because at drop time there may still be packets that haven't finished being driven.

### Scoreboard (SCB)

- On receiving a packet (either expected or actual) → raise.
- If the exp FIFO or act FIFO is non-empty → do not drop.
- If exp and act are fully matched → drop.

> ⚠️ SCB has issues too: if no stimulus is sent for a long time, or stimulus never reaches the SCB for some reason, the SCB never raises.

### Combined conclusion

None of the above components is sufficient on its own; they must be combined to make the judgment.

## Why can RM queue status tell us the environment is done?

- The Monitor hands packets to the RM; these packets flow through the queue and finally arrive at the SCB.
- **Some queue is always non-empty** (as long as data is still flowing).
- If the queues are empty → the packets must have reached the SCB; combined with the SCB, we can decide whether the environment is done.

> So it must be **monitor + scb + rm status** combined to judge whether the environment can end. (The packets have already left the DUT.)

## Final conclusion: two approaches

| Approach | Description | Limitation |
| --- | --- | --- |
| **1. Count cnt** | input count = output count | Simple, but fails with complex RTL; unusable in many environments |
| **2. Voting mechanism** | All components combined; the environment ends only when objections from seq / driver / monitor / rm have all dropped | More general |

## raise / drop objection details

- You can write a `this` inside a component, which is that component's objection mechanism.
- The base sequence ends based on the **total cnt** of objections.

## Expected abnormal termination

- On detecting an interrupt → end immediately.
- Depending on your needs, decide whether to recover from the exception and resume traffic.

## Unexpected exceptions

> TBD: handling strategies must be defined per scenario in real projects.
