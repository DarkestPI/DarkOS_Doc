



日期：2026-06-03

  

## 1. 问题现象

  

在 RK3576 MCU 工程中使用 CANFD 通信时，测试条件为：

  

- 仲裁域波特率：1 Mbps

- 数据域波特率：5 Mbps

- CANFD 标准：CANFD ISO

- CANFD 加速：开启，即 BRS 使能

- 帧格式：标准帧

  

现场出现过如下异常日志：

  

```text

[HAL INFO] CANFD0 ERROR! isr=0x40 cnt=...

[HAL INFO] CANFD0 ERROR! isr=0x1240 cnt=...

[HAL INFO] CANFD0 BUS OFF! isr=0x1240

```

  

其中：

  

- `isr=0x40` 表示 CAN 控制器检测到总线错误。

- `isr=0x1240` 包含 `ERROR_INT`、`BUS_OFF_INT`、`AUTO_RETX_FAIL_INT`，说明控制器持续发送失败并进入 Bus Off。

- `cnt` 是驱动中统计的错误中断次数，不是硬件错误码。

  

测试中还发现一个关键现象：

  

- 上位机设置 `1M/1M` 时可以接收。

- 上位机设置 `1M/5M` 时容易报错。

- 修复后 `1M/5M` 可以正常工作。

  

## 2. 排查过程

  

### 2.1 确认错误中断含义

  

在 RK3576 寄存器定义中确认：

  

```c

CAN_INT_ERROR_INT_MASK           = 0x40

CAN_INT_BUS_OFF_INT_MASK         = 0x200

CAN_INT_AUTO_RETX_FAIL_INT_MASK  = 0x1000

```

  

因此：

  

```text

0x1240 = 0x1000 + 0x0200 + 0x0040

```

  

表示自动重传失败、Bus Off、普通 CAN 错误中断同时发生。

  

### 2.2 增加 CANFD 初始化寄存器打印

  

为了确认 RK3576 实际写入的位时序，在 `periph_canfd.c` 中增加初始化日志：

  

```text

CANFD0 Enable nbps=... dbps=... nbt=0x... dbt=0x... tdc=0x...

```

  

通过该日志确认：

  

```text

dbps=4

dbt=0x980008d

tdc=0x35

```

  

表示数据域确实配置为 5 Mbps。

  

同时发现：

  

```text

nbt=0x4040d

```

  

对应仲裁域参数：

  

```text

brq   = 4

tseg1 = 13

tseg2 = 4

```

  

计算采样点：

  

```text

Sample Point = (1 + (tseg1 + 1)) / (1 + (tseg1 + 1) + (tseg2 + 1))

             = (1 + 14) / (1 + 14 + 5)

             = 75%

```

  

而上位机使用的是 `1M 80% / 5M 75%`，所以仲裁域采样点不一致。

  

## 3. 根因分析

  

本次问题主要由以下几个因素叠加导致。

  

### 3.1 CANFD 位时序配置顺序不合理

  

原 HAL 初始化流程中，RK3576 先进入工作模式，再配置 CANFD 位时序：

  

```c

SET_BIT(pReg->MODE, CAN_MODE_WORK_MODE_MASK);

HAL_CANFD_Config(pReg, initStrust->bps, initStrust->dbps);

```

  

CAN 控制器的位时序寄存器通常要求在 reset/config 模式下配置。进入工作模式后再写 `FD_NOMINAL_BITTIMING`、`FD_DATA_BITTIMING`、`FD_TDC` 等寄存器，可能导致配置不生效或部分不生效。

  

### 3.2 仲裁域 1 Mbps 默认采样点为 75%

  

RK3576 HAL 原始 1 Mbps 仲裁域配置为：

  

```c

brq = 4;

tseg1 = 13;

tseg2 = 4;

```

  

该配置对应采样点 75%。上位机配置为 1 Mbps 80%，两端不一致。

  

### 3.3 仲裁域寄存器使用 SET_BIT 写入

  

原代码中仲裁域位时序使用：

  

```c

SET_BIT(pReg->FD_NOMINAL_BITTIMING, ...);

```

  

该方式会保留旧寄存器位。若之前配置过其他波特率，可能产生旧位残留。

  

## 4. 修改内容

  

### 4.1 调整 HAL 初始化顺序

  

将 `HAL_CANFD_Config()` 放到进入工作模式之前：

  

```c

HAL_CANFD_Config(pReg, initStrust->bps, initStrust->dbps);

SET_BIT(pReg->MODE, CAN_MODE_WORK_MODE_MASK);

```

  

确保位时序配置在控制器进入工作模式前完成。

  

### 4.2 仲裁域 1 Mbps 改为 80% 采样点

  

修改 `HAL_CANFD_SetNBps()` 中 `CANFD_BPS_1MBAUD` 参数：

  

```c

case CANFD_BPS_1MBAUD:

    brq = 4;

    tseg1 = 14;

    tseg2 = 3;

    break;

```

  

计算：

  

```text

Sample Point = (1 + 15) / (1 + 15 + 4) = 80%

```

  

修复后初始化日志应显示：

  

```text

nbt=0x4030e

```

  

### 4.3 仲裁域位时序改为覆盖写

  

将：

  

```c

SET_BIT(pReg->FD_NOMINAL_BITTIMING, ...);

```

  

改为：

  

```c

WRITE_REG(pReg->FD_NOMINAL_BITTIMING, ...);

```

  

避免旧配置位残留。


  

## 5. 验证结果

  
修复后，启动日志应为：

  
```text

CANFD0 Enable nbps=8 dbps=4 nbt=0x4030e dbt=0x980008d tdc=0x35

CANFD0 app init ok, nbps=4, dbps=0, mode=0x20

```

  
含义如下：

```text

nbps=8       HAL 层 1 Mbps

dbps=4       HAL 层 5 Mbps

nbt=0x4030e  仲裁域 1 Mbps / 80%

dbt=0x980008d 数据域 5 Mbps / 75%

tdc=0x35     数据域 5 Mbps 开启 TDC

```

与上位机配置保持一致：

  
```text

CANFD ISO

CANFD 加速：是

仲裁域：1 Mbps 80%

数据域：5 Mbps 75%

```

  
  