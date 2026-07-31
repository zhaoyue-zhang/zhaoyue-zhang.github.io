---
title: "【SV_RANDOM】随机化与约束"
date: 2026-07-31
draft: false
tags: ["UVM Knowledge", "SystemVerilog"]
description: "SystemVerilog 随机化稳定性的判别（$random/$dist vs $urandom/randomize）、约束拆分原则、soft 约束的生效机制，以及父类 soft 与子类 dist 冲突时的实际行为。"
---

# SV 随机化与约束

## 1. 随机稳定性

SystemVerilog 提供了多套随机函数，但**不是所有都具备"随机稳定性"**。所谓稳定，是指在 RTL/约束代码改动前后，**同一个 seed 下产生的随机序列保持一致**，这样回归测试才能复现 bug。

> **不推荐使用 `$random` / `$dist`**：这两个函数不保证稳定。代码增删一行可能导致调用顺序变化，从而改变整个随机序列。

| 函数 | 是否具备随机稳定性 |
| --- | --- |
| `$random` / `$dist` | ❌ 不稳定 |
| `$urandom` / `$urandom_range` | ✅ 稳定 |
| `randomize()` | ✅ 稳定 |

工程实践：**所有受约束随机一律走 `randomize()`**；需要无约束的裸随机用 `$urandom`。

## 2. 约束设计原则

- **拆分约束**：当一个类有多个随机变量时，**按相关性拆成多个 `constraint` 块**，每块约束一组变量。这样做有两个好处——单条求解时间更短、调试时容易定位"是哪条约束卡住了"。
- **`randomize() with { ... }` 是叠加，不是覆盖**：它是**在对象原有约束之上增加**约束，**不会**删掉原约束。容易踩的坑：以为 `with` 可以临时关掉某条约束，结果旧约束和新约束一起生效。

## 3. soft 约束

`soft` 关键字声明的是"软约束"——**只要不与其它硬约束冲突，求解器就必须满足它**；冲突时会被硬约束压过。

### 3.1 容易混淆的几个点

- soft 约束**不是"优先级低"**，是"在不冲突时必须满足"——这点和"低优先级"在数学上等价，但概念不同。
- 多 soft 约束冲突时，LRM 规定**实现可以任选其一满足**，不同仿真器行为可能不同。

### 3.2 父类 soft + 子类 dist 硬约束

这是工程里**最容易踩坑**的场景。看下面的例子：

```verilog
// 父类：soft 约束默认值为 0
class parent;
    rand int var;
    constraint soft_zero { soft var == 0; }
endclass

// 子类：添加 dist 硬约束，但未覆盖父类 soft
class child extends parent;
    constraint dist_constr { var dist { 0 :/ 10, 1 :/ 10 }; }
endclass

module test;
    initial begin
        child c = new();
        repeat (10) begin
            void'(c.randomize());
            $display("var = %0d", c.var);
        end
    end
endmodule
```

**分析**：

- 父类：`soft var == 0`（希望默认值 0）
- 子类：`var dist { 0:/10, 1:/10 }`（硬约束：var 只能取 0 或 1，权重相同）
- 硬约束没有排除 0，soft 约束 `var == 0` 仍然可以满足 → **求解器同时满足硬约束和软约束**，于是 `var` 几乎始终为 0
- 想要 `var` 有机会取 1，必须在子类显式覆盖父类的 soft（例如加 `soft var == 1`，或者反着写 `soft var != 0`，或者直接重写 `constraint soft_zero`）

> **注意**：`randomize() with { ... }` 走的也是同一条 soft 处理逻辑，不会自动"覆盖"父类 soft。

### 3.3 多 soft 冲突时

```verilog
// 父类
soft var == 0;
// 子类
soft var == 1;
```

LRM 规定：**任选一个满足**。不同仿真器（VCS / Questa / Xcelium）实测行为可能不同——有的偏父、有的偏子、有的交替。如果一定要确定性，就**避免**写出多 soft 冲突。

## 4. 总结

| 关注点 | 结论 |
| --- | --- |
| 随机稳定性 | 用 `randomize()` 和 `$urandom`；不要用 `$random` / `$dist` |
| 约束拆分 | 按相关性分成多个块，单条短便于调试 |
| `with` 子句 | 叠加而非覆盖，旧的约束仍然生效 |
| soft vs 硬约束 | soft 在不冲突时必须满足；冲突时被硬约束压过 |
| 父类 soft + 子类 dist | soft 仍生效，dist 不会自动覆盖；想要真分布得显式覆盖 |
| 多 soft 冲突 | 仿真器自选，**避免**依赖这种行为 |
