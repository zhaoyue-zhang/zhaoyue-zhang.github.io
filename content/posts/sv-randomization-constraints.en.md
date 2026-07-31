---
title: "【SV_RANDOM】Randomization and Constraints"
date: 2026-07-31
draft: false
tags: ["UVM Knowledge"]
description: "How to judge randomization stability in SystemVerilog ($random/$dist vs $urandom/randomize), principles for splitting constraints, how soft constraints interact with hard constraints, and what actually happens when a parent's soft constraint meets a child's dist constraint."
---

# SV Randomization and Constraints

## 1. Randomization Stability

SystemVerilog ships several random functions, but **not all of them are "stable"**. A function is stable when, given the same seed, it produces the same sequence across RTL or constraint changes — this is what makes regression runs reproducible.

> **Avoid `$random` / `$dist`**: they are not stable. Adding or removing a line of code can shift the call order and reshuffle the whole sequence.

| Function | Stable? |
| --- | --- |
| `$random` / `$dist` | ❌ No |
| `$urandom` / `$urandom_range` | ✅ Yes |
| `randomize()` | ✅ Yes |

**Rule of thumb**: use `randomize()` for everything that has constraints; use `$urandom` for unconstrained bare random numbers.

## 2. Constraint Design Principles

- **Split constraints by variable group**: when a class has several `rand` fields, **put related fields in one `constraint` block** and unrelated fields in separate blocks. This keeps each block short, makes individual solves faster, and makes debugging far easier — you can locate "which block is failing" at a glance.
- **`randomize() with { ... }` adds, it does not replace**: the `with` clause is **layered on top of the existing constraints**; it does not remove or override them. A common trap: assuming `with` can "temporarily switch off" a constraint — in fact the old constraint is still active alongside the new one.

## 3. soft Constraints

`soft` declares a *soft* constraint: as long as the soft constraint does not conflict with any hard constraint, **the solver must satisfy it**. When it conflicts with a hard constraint, the hard one wins.

### 3.1 Common Confusions

- "soft" is **not** "low priority". The semantic is "must be satisfied when possible". Mathematically equivalent to "low priority" in most cases, but the concept is different.
- When two `soft` constraints on the same variable conflict, the LRM says the implementation **may pick either one**. Different simulators (VCS, Questa, Xcelium) behave differently in practice.

### 3.2 Parent soft + child dist hard constraint

This is the **most common pitfall** in real projects. Consider:

```verilog
// Parent: soft default of 0
class parent;
    rand int var;
    constraint soft_zero { soft var == 0; }
endclass

// Child: adds a dist hard constraint, but does not override the parent soft
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

**Analysis**:

- Parent: `soft var == 0` (prefers default of 0)
- Child: `var dist { 0:/10, 1:/10 }` (hard: var must be 0 or 1, equal weight)
- The hard constraint does not exclude 0, so the soft constraint `var == 0` is still satisfiable. The solver satisfies both, and **`var` is almost always 0**.
- To give `var` a real chance of being 1, you must explicitly override the parent soft in the child — e.g. add `soft var == 1`, write `soft var != 0`, or simply redeclare `constraint soft_zero`.

> **Note**: `randomize() with { ... }` goes through the same soft-resolution logic and does **not** automatically override the parent soft.

### 3.3 Multiple soft constraints in conflict

```verilog
// Parent
soft var == 0;
// Child
soft var == 1;
```

The LRM says: **either may be satisfied**. Different simulators pick differently — some prefer the parent, some the child, some alternate. If determinism matters, **avoid** this pattern.

## 4. Summary

| Topic | Conclusion |
| --- | --- |
| Stability | Use `randomize()` and `$urandom`; avoid `$random` / `$dist` |
| Constraint splitting | Group related variables; keep each block short for debuggability |
| `with { ... }` clause | Layers on top; does not replace existing constraints |
| soft vs hard | soft must be satisfied when non-conflicting; hard wins on conflict |
| Parent soft + child dist | soft is still active; you must explicitly override to get a real distribution |
| Conflicting multiple soft | Simulator-dependent; **avoid** relying on this |
