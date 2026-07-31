---
title: "【UVM_FACTORY】Determining the Override Timing"
date: 2026-07-31
draft: false
tags: ["UVM Knowledge"]
description: "Three core principles of the UVM Factory override: factory-based create, when to override, type vs instance overrides, and how the position of config_db wildcards affects performance."
---

Apart from TLM- and coverage-related objects, all other classes are recommended to be created through the factory pattern `create`, and the variable name should ideally match the `name` argument of the `create` function. Use `uvm_config_db::set` for configuration; if wildcards are needed, place them toward the end of `inst_name` for better performance. As for the timing of factory overrides, every type or instance override must happen before that instance is created: in `build_phase`, if an object is a member of a component (and is created inside `super.build_phase`), the override must be set before calling the parent's `build_phase`; if the object is created inside a sequence's `body`, the override does not have to be set before `build_phase`.

## SystemVerilog Examples

The following six minimal code snippets correspond to the three principles above, demonstrating factory `create`, override timing, type/instance override, sequence timing, and the position of the `config_db` wildcard.

### 1. Factory create() vs new()

```verilog
// Recommended: created via the Factory; can be overridden
req = bus_transaction::type_id::create("req");
```

```verilog
// Not recommended: bypasses the Factory; override is ineffective
req = new("req");
```

### 2. Component build_phase: wrong order

```verilog
// Override too late: the driver has already been created as bus_driver
// inside super.build_phase
function void bus_test::build_phase(uvm_phase phase);
  super.build_phase(phase);
  bus_driver::type_id::set_type_override(error_driver::get_type());
endfunction
```

### 3. Component build_phase: correct order

```verilog
// Override comes first: it is in effect when super.build_phase creates the driver
function void bus_test::build_phase(uvm_phase phase);
  bus_driver::type_id::set_type_override(error_driver::get_type());
  super.build_phase(phase);
endfunction
```

### 4. Type override vs instance override

```verilog
// Type override: every bus_driver is replaced
bus_driver::type_id::set_type_override(error_driver::get_type());

// Instance override: only env.agent1.driver is replaced;
// all others keep their original type
bus_driver::type_id::set_inst_override(
  error_driver::get_type(),
  "env.agent1.driver"
);
```

### 5. Objects created inside sequence body()

```verilog
// Setting the override before body() is enough; it does not have to
// come before build_phase
task main_phase(uvm_phase phase);
  phase.raise_objection(this);
  bus_transaction::type_id::set_type_override(
    error_transaction::get_type()
  );
  seq.start(env.agent.sequencer);
  phase.drop_objection(this);
endtask
```

### 6. Position of uvm_config_db wildcards

```verilog
// Recommended: wildcard at the end, narrower match
uvm_config_db#(int)::set(this, "env.agent*", "timeout", 100);

// Not recommended: wildcard at the front, wider match,
// easy to hit unrelated agents by mistake
uvm_config_db#(int)::set(this, "*agent", "timeout", 100);
```

> The only criterion: a Factory override must occur before the target object's `type_id::create()`.

## Summary

- For objects created inside a parent's `build_phase`: write the override before `super.build_phase()`.
- For objects created inside a sequence's `body()`: write the override before `seq.start()`.
- For `uvm_config_db` wildcards: place them at the end of `inst_name`; the more specific, the better.
