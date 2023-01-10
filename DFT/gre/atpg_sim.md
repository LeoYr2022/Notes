# ATPG仿真时遇见的问题

## 需要事先检查的内容

### testbench上：

- 系统reset要在ijtag_reset之后；
- 由于netlist仿真可能缺失初始状态，所以最好对初始状态做规定；

### 需要force的信号：

- EMA控制信号；

### 不需要做force的信号：

- scan clock

### Generate Pattern的时候

- **对于OCC电路**，如果fast clock频率低于slow clock频率，需要将```fast_capture_mode off```
  - <font color="red">相应的SDC需不需要提高频率来约束呢?</font>
- **对于input ports**，如果其与scan不相干，可以```add_pin_constrains input_portName -CZ```

- 

