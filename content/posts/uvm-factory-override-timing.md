---
title: "【UVM_FACTORY】重载时机的确定"
date: 2026-07-31
draft: false
tags: ["UVM Knowledge"]
description: "梳理 UVM Factory 重载的三条核心原则：工厂 create、重载时机、类型/实例重载，以及 config_db 通配符的位置对性能的影响。"
---

除了 TLM 和覆盖率相关的对象之外，其余所有的类都建议使用工厂模式 `create` 进行例化，且变量名最好和 `create` 函数的 `name` 参数一致。配置使用 `uvm_config_db::set` 函数；如果使用通配符，建议尽量放在 `inst_name` 的后面，以提高性能。工厂重载的时机：所有的类型或实例重载必须在该实例例化之前完成——在 `build_phase` 中，如果某个对象是组件的成员（会在 `super.build_phase` 中创建），则重载必须放在调用父类 `build_phase` 之前；如果对象是在 sequence 的 `body` 中创建，则不必放在 `build_phase` 前完成重载。

## SystemVerilog 示例

下面用 6 段最小代码对应上面 3 条原则，分别展示工厂 create、重载时机、类型/实例重载、sequence 时机、config_db 通配符位置。

### 1. 工厂 create() vs new()

```verilog
// 推荐：通过 Factory 创建，可被重载
req = bus_transaction::type_id::create("req");
```

```verilog
// 不推荐：绕过 Factory，重载无效
req = new("req");
```

### 2. 组件 build_phase：错误顺序

```verilog
// 重载太晚：driver 已在 super.build_phase 中被创建为 bus_driver
function void bus_test::build_phase(uvm_phase phase);
  super.build_phase(phase);
  bus_driver::type_id::set_type_override(error_driver::get_type());
endfunction
```

### 3. 组件 build_phase：正确顺序

```verilog
// 重载在前：super.build_phase 中创建 driver 时已经生效
function void bus_test::build_phase(uvm_phase phase);
  bus_driver::type_id::set_type_override(error_driver::get_type());
  super.build_phase(phase);
endfunction
```

### 4. 类型重载 vs 实例重载

```verilog
// 类型重载：所有 bus_driver 都被替换
bus_driver::type_id::set_type_override(error_driver::get_type());

// 实例重载：只替换 env.agent1.driver，其它保持原类型
bus_driver::type_id::set_inst_override(
  error_driver::get_type(),
  "env.agent1.driver"
);
```

### 5. sequence body() 中创建对象

```verilog
// body() 之前设置即可，不必放在 build_phase 之前
task main_phase(uvm_phase phase);
  phase.raise_objection(this);
  bus_transaction::type_id::set_type_override(
    error_transaction::get_type()
  );
  seq.start(env.agent.sequencer);
  phase.drop_objection(this);
endtask
```

### 6. uvm_config_db 通配符位置

```verilog
// 推荐：通配符靠后，匹配范围小
uvm_config_db#(int)::set(this, "env.agent*", "timeout", 100);

// 不推荐：通配符靠前，匹配范围大、容易误伤其它 agent
uvm_config_db#(int)::set(this, "*agent", "timeout", 100);
```

> 唯一判断标准：Factory override 必须发生在目标对象 `type_id::create()` 之前。

## 总结

- 父类 `build_phase` 中会创建的对象：重载写在 `super.build_phase()` 之前。
- sequence `body()` 中创建的对象：重载写在 `seq.start()` 之前即可。
- `uvm_config_db` 通配符：放在 `inst_name` 后段，范围越精确越好。
