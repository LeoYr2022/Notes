# Scan & ATPG  Flow

# Scan 

## add_test_logic



### 自动修复不符合DRC的方法

![add_test_logic](https://img-blog.csdnimg.cn/d9c1334c03c1439fb7283ca007ae5b93.png?x-oss-process=image/watermark,type_ZHJvaWRzYW5zZmFsbGJhY2s,shadow_50,text_Q1NETiBAb2Nlbmlj,size_20,color_FFFFFF,t_70,g_se,x_16)

```tcl
set_test_logic \
     -set   on \
     -reset on \
     -clock on
```

### 定义不需要串chain的block

```tcl
add_non_scan_instance \
    -instance | -rtl_to_reg
```

## scan chain

### clock domain and clock edge

- default情况，工具把所有时钟域的单元串在一起；

- leading、trailing edge时钟可以被串在一条scan chain上，但首先把trailing edge clock单元串起来；

- 将clock domain和clock edge分开

  ```tcl
  add_scan_mode \
      -single_clock_domain_chains on \
      -signle_clock_edge_chains   on \
  ```

- 两个clock domain在一条scan chain上

  ```tcl
  add_scan_mode \
      -signle_clock_domain_chain off \
  ```

- Balance scan chains

  ```tcl
  add_scan_mode \
      -chain_length 400 \
      -chain_count   50 \
  ```

### scan chain family

- 同一个family中的单元才能串到一个chain中

  ```tcl
  create_scan_chain_family family_clk1 \
      -include_elements \
      [get_scan_element -filter "clock_domain == clk1"]
      
  create_scan_chain_family family_clk2_3 \
      -include_elements \
      [get_scan_element -filter "clock_domain == clk2 || clock_domain == clk3"]
  ```

### multi mode scan chain

## test point

> 为了帮助提高设计的**可测试性**，分成：
>
> - observe test points
> - control test points
> - both observe test points

![test_point](https://img-blog.csdnimg.cn/e919c233c66846fab5a5c9159f117687.png?x-oss-process=image/watermark,type_ZHJvaWRzYW5zZmFsbGJhY2s,shadow_50,text_Q1NETiBAb2Nlbmlj,size_20,color_FFFFFF,t_70,g_se,x_16)

如上图所示，无论是输入什么数值，其输出恒定为1。也就是说or gate output的SA是无法测试到的。

### VersaPoint test points

## DRC

### S1 Rule

### S2 Rule



# ATPG

## Overview

- ATPG Process：
  - Setup
  - DRC
  - configure ATPG
  - Generate Patterns
  - Save results

![ATPG_FLOW](https://img-blog.csdnimg.cn/e72f716f4fb24e1b9d993c6219e9ab54.png?x-oss-process=image/watermark,type_ZHJvaWRzYW5zZmFsbGJhY2s,shadow_50,text_Q1NETiBAb2Nlbmlj,size_20,color_FFFFFF,t_70,g_se,x_16)

## DFT Lib

- 一般是由vendor提供；

- 有的时候，需要自己通过libcomp来转化

- 如果某个module不需要串chain，也可以将其设置为blackbox；

  ```tcl
  add_black_box -module <xxx>
  add_black_box -instances /instA -pin Out Z
  ```

  ```tcl
  report_black_boxes -all
  ```

  ```tcl
  delete_black_boxes -all
  delete_black_boxes -module <xxx>
  ```

## Setup

> 这个步骤中有两种方式：
>
> - Legacy Flow
> - TSDB Flow

### Legacy Flow

#### Procedure file

![procedure](https://img-blog.csdnimg.cn/49e68abb50834cc8a3888b3ff0d73180.png?x-oss-process=image/watermark,type_ZHJvaWRzYW5zZmFsbGJhY2s,shadow_50,text_Q1NETiBAb2Nlbmlj,size_20,color_FFFFFF,t_70,g_se,x_16)

![在这里插入图片描述](https://img-blog.csdnimg.cn/5d61867e769f44a398ed9821764abb23.png?x-oss-process=image/watermark,type_ZHJvaWRzYW5zZmFsbGJhY2s,shadow_50,text_Q1NETiBAb2Nlbmlj,size_20,color_FFFFFF,t_70,g_se,x_16)

- timeplates

  ![timeplates](https://img-blog.csdnimg.cn/b4c11a2c4ae44bbc92148e15da016560.png?x-oss-process=image/watermark,type_ZHJvaWRzYW5zZmFsbGJhY2s,shadow_50,text_Q1NETiBAb2Nlbmlj,size_20,color_FFFFFF,t_70,g_se,x_16)

  > timeplate内的操作不可以随意设置，是和ATE强相关的。
  >
  > 基本语法：
  >
  > - pulse
  > - force

  ```tcl
  timeplate tp1 =
      force_pi 0;
      measure_po 10; #measure po 是在10ns 也就是 这是pre_clock 的measure
      pulse Clk 20 10; #20-30ns 是clock处于on状态的时间 也就是时钟的高电平的持续时间是10ns.
      period 50;#时钟周期是50ns
   end
  ```

- test_setup

  ```tcl
  #定义procedure 每个步骤都会执行 force pulse 的操作. 都会调用 template 的单周期的配置.
  procedure test_setup = 
  #可见此单个procedure中，有两个cycle。每个cycle在无其他force的情况下，应该与timeplate中的配置一致.
  timeplate tp1 ;
  cycle =
      force Clk 0 ; #force clk为0 
      force ScanEn 0;
      pulse Clk ;
  end;
  
  cycle =
      pulse Clk;
  end
  ```

- load_unload/shift

  ![在这里插入图片描述](https://img-blog.csdnimg.cn/fabaf710ac694f14a4e7cf894f45efdd.png?x-oss-process=image/watermark,type_ZHJvaWRzYW5zZmFsbGJhY2s,shadow_50,text_Q1NETiBAb2Nlbmlj,size_20,color_FFFFFF,t_70,g_se,x_16)

  ```tcl
  procedure load_unload =
      scan_group grp1 ;
      timeplate tp1 ;
      cycle = 
          force Clk 0;
          force ScanEn 1;
      end ;
      apply shift 2;#调用shift 的procedure 两次
  end ;
  ```

  ```tcl
  procedure shift =
      scan_group grp1 ;
      timeplate tp1 ;
      cycle =
          force_sci;
          measure_sco;
          pulse Clk ;
      end
  end;
  ```

- capture

  ```tcl
  procedure capture =
      scan_group grp1;
      timeplate tp1;
      cycle =
          force ScanEn 0;
          force_pi;
          measure_po;
          pulse_capture_clk;
      end;
  end;
  ```

  #### dofile

  > TCL中的proc 过程, 类似C 中的函数, 第一个中括号是参数列表，第二个中括号是具体的执行内容。
  >
  > ```c
  > proc add{x y}{
  >     expr x+y
  > } 
  > ```

  ```tcl
  #里面只要是 scan chain 的相关信息
  proc tessent_scan_common {} {
      add_clcoks 0 clk
  }
  
  proc tessent_scan_unwrapped_mode{}{
      #定义scanchain 的分组 以及其使用的tesproc文件
      add_scan_groups grp1 results/pipe_scan.testproc
      #添加设计中已经存在的scan chain到我们定义的scan group中并说明scan chain的输入输出.
      add_scan_chains chain grp1 {/ts_si[1]}{/ts_so[1]} 
      tessent_scan_common
  }
  
  proc tessent_scan_setup {{mode unwrapped}}{
      switch -exact -$mode{
          unwrapped {tessent_scan_unwrapped_mode}
          default{
              display_message -error "invalid scan mode: $mode"
          }
      }
  }
  ```

  - dofile中对clock和control信号的定义

    - 使用**analyze_control_signals**，他trace的全部是普通组合逻辑门，有可能不符合DFT要求；

    - 使用**add_clock**手动添加

      ```tcl
      ## 参数1为clock的off state，参数2是时钟引脚
      add_clock 0 CLKA
      ```

## Faults Coverage & Transcripts

#### Fault Universe

- 可以手动添加或是删除fault location

- **set_internal_fault on**增加fault location，其将标准单元拆分成gate cell

- 使用add_faults指定fault universe

  ```tcl
  add_faults -all # 自动将所有的faults 加进去
  /core/cpu/ix342/Y -pin # 以某个pin为条件 添加单个的fault site for both state1 and 0
  /core/cpu/alu32* -instance # 以实例的名字为条件 为所有与名字匹配的instance添加fault site
  /bus_ctr/tsd/*/EN -pin -Stuck 0 # 筛选符合条件的pin 添加 stuck at 0 的fault site
  -clock_domain  /clk_ctr/clock001 #为匹配的clock domain 添加fault site点;
  ```

- fault sampling

  只适用于try flow或是建立环境

  ```tcl
  ## 10%的fault产生pattern
  set_faults_sampling 10 
  add_faults -all
  ```

## Example

### setup

```tcl
set_context patterns -scan #设置Tessent shell context
read_verilog #读入 scan inserted 的网表
read_cell_library ../libs/adk.atpg #读入ATPG的库
set_current_design #指定设计的顶层
add_black_boxes -auto #根据需要添加 blackbox
analyze_control_signals -auto_fix # 定义时钟 和控制信号
add_input_constraints xxx -C0 #设置测试的输入约束
## add_input_constraints优先级低于test_procedure；
dofile atpg_setup.dofile #定义scan chains，读入test_procedure文件
```

### DRC

```tcl
check_design_rules
## 当DRC通过之后，Tessent从setup mode转到analyze mode
```

#### 常见DRC

- RAM-A rules
- Clock-C rules



### Configure ATPG

> 一般会在配置的时候指定block或者instance添加或者减少 fault的种类
> 或者设置一个指定的fault 模型;

### Generate Patterns

```tcl
create_patterns
# 这条命令会干啥?
#首先 本职工作是生成pattern
# 其次 它会帮助你 分析当前的设计 通过打开一些选项 帮助你修复一些DRC的问题
# 在没有 add_fualts 的指定下 工具默认是吧所有的fault 分析进去 生成pattern
#会将 运行时间 当前生成的pattern的数量 (没记错的话 是64个64个的Generate) 以及当前的覆盖率等信息 打印在控制台上.
```

### Save Results

```tcl
write_pattern <filename> <format_switch> -replace

## 保存flatten model
write_flat_model <filename> -replace
```

# Faults





