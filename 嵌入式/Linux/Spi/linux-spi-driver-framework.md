# Linux SPI驱动框架深度分析

## 概述

SPI(Serial Peripheral Interface)是一种同步串行通信接口，广泛应用于嵌入式系统中。Linux内核提供了完整的SPI驱动框架，支持主机控制器驱动和外设驱动的开发。

## 1. SPI框架核心架构

### 1.1 主要组件

Linux SPI框架主要包含以下核心组件：

1. **SPI核心层(spi.c)** - 提供SPI总线注册、设备管理、传输接口
2. **SPI主机控制器驱动** - 实现具体的SPI控制器硬件操作
3. **SPI设备驱动** - 实现具体SPI外设的功能驱动
4. **SPI设备** - 连接到SPI总线上的外设设备

### 1.2 架构层次

```
用户空间
    ↓
SPI设备驱动 (spi_device_driver)
    ↓
SPI核心层 (SPI Core)
    ↓
SPI主机控制器驱动 (spi_master)
    ↓
硬件层 (SPI控制器)
```

## 2. 核心数据结构

### 2.1 spi_master

表示SPI主机控制器，包含控制器的所有信息和操作函数：

```c
struct spi_master {
    struct device dev;
    struct list_head list;
    s16 bus_num;
    u16 num_chipselect;
    u16 mode_bits;
    u32 min_speed_hz;
    u32 max_speed_hz;
    u16 flags;
    int (*setup)(struct spi_device *spi);
    int (*transfer)(struct spi_device *spi, struct spi_message *mesg);
    void (*cleanup)(struct spi_device *spi);
    // ... 其他成员
};
```

### 2.2 spi_device

表示SPI设备，包含设备的所有配置信息：

```c
struct spi_device {
    struct device dev;
    struct spi_master *master;
    u32 max_speed_hz;
    u8 chip_select;
    u8 mode;
    u8 bits_per_word;
    int irq;
    void *controller_state;
    void *controller_data;
    char modalias[SPI_NAME_SIZE];
    // ... 其他成员
};
```

### 2.3 spi_driver

表示SPI设备驱动：

```c
struct spi_driver {
    const struct spi_device_id *id_table;
    int (*probe)(struct spi_device *spi);
    int (*remove)(struct spi_device *spi);
    void (*shutdown)(struct spi_device *spi);
    int (*suspend)(struct spi_device *spi, pm_message_t mesg);
    int (*resume)(struct spi_device *spi);
    struct device_driver driver;
};
```

### 2.4 spi_transfer

表示一次SPI传输：

```c
struct spi_transfer {
    const void *tx_buf;
    void *rx_buf;
    unsigned len;
    dma_addr_t tx_dma;
    dma_addr_t rx_dma;
    unsigned cs_change:1;
    u8 bits_per_word;
    u16 delay_usecs;
    u32 speed_hz;
    // ... 其他成员
};
```

### 2.5 spi_message

表示一个完整的SPI消息，包含多个传输：

```c
struct spi_message {
    struct list_head transfers;
    struct spi_device *spi;
    unsigned is_dma_mapped:1;
    void (*complete)(void *context);
    void *context;
    unsigned actual_length;
    int status;
    struct list_head queue;
    void *state;
    // ... 其他成员
};
```

## 3. SPI框架初始化流程

### 3.1 系统启动流程

1. **SPI总线初始化** - 在spi_init()中注册SPI总线类型
2. **SPI控制器注册** - 平台驱动注册SPI主机控制器
3. **SPI设备注册** - 通过设备树或ACPI描述SPI设备
4. **SPI驱动匹配** - SPI核心层匹配设备和驱动

### 3.2 SPI控制器注册流程

```c
spi_alloc_master()    // 分配spi_master结构
spi_register_master() // 注册SPI主机控制器
    ├── spi_master_initialize_queue() // 初始化传输队列
    ├── of_register_spi_devices()     // 注册设备树中的SPI设备
    ├── spi_register_board_info()     // 注册板级SPI设备信息
    └── device_add()                  // 添加控制器设备
```

## 4. SPI数据传输流程

### 4.1 同步传输

```c
spi_sync(struct spi_device *spi, struct spi_message *message)
```

流程：
1. 检查参数有效性
2. 获取控制器锁
3. 调用控制器具体的传输函数
4. 等待传输完成
5. 释放控制器锁

### 4.2 异步传输

```c
spi_async(struct spi_device *spi, struct spi_message *message)
```

流程：
1. 检查参数有效性
2. 将消息添加到控制器队列
3. 调度工作队列处理传输
4. 传输完成后调用完成回调函数

### 4.3 DMA传输

支持DMA的SPI控制器可以配置DMA传输：

```c
spi_message.is_dma_mapped = 1;
spi_transfer.tx_dma = dma_map_single(...);
spi_transfer.rx_dma = dma_map_single(...);
```

## 5. SPI设备驱动开发

### 5.1 基本驱动框架

```c
#include <linux/spi/spi.h>

static struct spi_driver my_spi_driver = {
    .driver = {
        .name = "my_spi_device",
        .owner = THIS_MODULE,
    },
    .probe = my_spi_probe,
    .remove = my_spi_remove,
    .id_table = my_spi_id,
};

static int __init my_spi_init(void)
{
    return spi_register_driver(&my_spi_driver);
}

static void __exit my_spi_exit(void)
{
    spi_unregister_driver(&my_spi_driver);
}

module_init(my_spi_init);
module_exit(my_spi_exit);
```

### 5.2 设备探测函数

```c
static int my_spi_probe(struct spi_device *spi)
{
    /* 配置SPI设备参数 */
    spi->mode = SPI_MODE_0;
    spi->max_speed_hz = 1000000;
    spi->bits_per_word = 8;
    
    spi_setup(spi);
    
    /* 分配私有数据结构 */
    struct my_spi_data *data = devm_kzalloc(&spi->dev, sizeof(*data), GFP_KERNEL);
    if (!data)
        return -ENOMEM;
    
    spi_set_drvdata(spi, data);
    
    /* 初始化设备 */
    // ... 设备特定的初始化代码
    
    return 0;
}
```

### 5.3 数据传输示例

```c
static int my_spi_read_reg(struct spi_device *spi, u8 reg, u8 *val)
{
    struct spi_transfer t[2];
    struct spi_message m;
    u8 tx_buf[2], rx_buf[2];
    
    spi_message_init(&m);
    memset(t, 0, sizeof(t));
    
    /* 寄存器地址传输 */
    tx_buf[0] = reg | 0x80;  /* 读操作标志 */
    t[0].tx_buf = tx_buf;
    t[0].rx_buf = rx_buf;
    t[0].len = 1;
    spi_message_add_tail(&t[0], &m);
    
    /* 数据读取 */
    t[1].rx_buf = val;
    t[1].len = 1;
    spi_message_add_tail(&t[1], &m);
    
    return spi_sync(spi, &m);
}
```

## 6. SPI主机控制器驱动开发

### 6.1 控制器驱动框架

```c
static struct spi_master *master;

static int my_spi_setup(struct spi_device *spi)
{
    /* 配置SPI控制器硬件参数 */
    // ... 硬件特定的配置
    return 0;
}

static int my_spi_transfer(struct spi_device *spi, struct spi_message *mesg)
{
    struct spi_transfer *t;
    
    list_for_each_entry(t, &mesg->transfers, transfer_list) {
        /* 执行实际的SPI传输 */
        // ... 硬件传输代码
    }
    
    mesg->status = 0;
    mesg->complete(mesg->context);
    
    return 0;
}

static int __init my_spi_controller_init(void)
{
    master = spi_alloc_master(&pdev->dev, 0);
    if (!master)
        return -ENOMEM;
    
    master->setup = my_spi_setup;
    master->transfer = my_spi_transfer;
    master->bus_num = 0;
    master->num_chipselect = 4;
    master->mode_bits = SPI_CPOL | SPI_CPHA | SPI_CS_HIGH;
    master->min_speed_hz = 100000;
    master->max_speed_hz = 50000000;
    
    return spi_register_master(master);
}
```

### 6.2 中断处理

```c
static irqreturn_t my_spi_irq(int irq, void *dev_id)
{
    struct spi_master *master = dev_id;
    struct my_spi *spi = spi_master_get_devdata(master);
    
    /* 处理SPI中断 */
    if (spi->irq_status & SPI_IRQ_TX_COMPLETE) {
        /* 传输完成处理 */
        complete(&spi->tx_complete);
    }
    
    return IRQ_HANDLED;
}
```

## 7. 设备树配置

### 7.1 SPI控制器配置

```dts
spi0: spi@40000000 {
    compatible = "mycompany,my-spi";
    reg = <0x40000000 0x1000>;
    interrupts = <10 IRQ_TYPE_LEVEL_HIGH>;
    clocks = <&spi_clk>;
    clock-names = "spi";
    #address-cells = <1>;
    #size-cells = <0>;
    status = "okay";
};
```

### 7.2 SPI设备配置

```dts
spi_flash: flash@0 {
    compatible = "jedec,spi-nor";
    reg = <0>;  /* CS0 */
    spi-max-frequency = <50000000>;
    spi-cpol;
    spi-cpha;
};

spi_sensor: sensor@1 {
    compatible = "mycompany,my-sensor";
    reg = <1>;  /* CS1 */
    spi-max-frequency = <1000000>;
    interrupt-parent = <&gpio1>;
    interrupts = <5 IRQ_TYPE_EDGE_RISING>;
};
```

## 8. 调试和故障排除

### 8.1 调试工具

1. **spidev** - SPI用户空间接口
2. **spi-tools** - SPI调试工具集
3. **逻辑分析仪** - 硬件级信号分析

### 8.2 常见问题

1. **传输失败** - 检查时钟频率、模式设置
2. **数据错误** - 验证位顺序、字长设置
3. **CS信号问题** - 检查片选逻辑和时序

### 8.3 调试技巧

```c
/* 启用SPI调试信息 */
#define DEBUG
#include <linux/spi/spi.h>

/* 打印传输信息 */
dev_dbg(&spi->dev, "SPI transfer: speed=%d, mode=%d, bits=%d\n",
        spi->max_speed_hz, spi->mode, spi->bits_per_word);
```

## 9. 性能优化

### 9.1 DMA优化

- 使用DMA传输减少CPU占用
- 合理设置DMA缓冲区大小
- 避免频繁的DMA映射/解映射

### 9.2 中断优化

- 使用中断驱动传输
- 合理设置中断优先级
- 减少中断处理时间

### 9.3 批处理优化

- 合并多个小传输
- 使用SPI消息链
- 减少CS切换次数

## 10. 实际应用案例

### 10.1 SPI Flash驱动

SPI Flash是最常见的SPI设备之一：

```c
static const struct spi_device_id spi_flash_ids[] = {
    { "m25p80", 0 },
    { "n25q128", 0 },
    { "w25q32", 0 },
    { }
};
MODULE_DEVICE_TABLE(spi, spi_flash_ids);
```

### 10.2 SPI传感器驱动

SPI传感器的典型实现：

```c
static int sensor_read_data(struct spi_device *spi, u8 *data, int len)
{
    struct spi_transfer t = {
        .rx_buf = data,
        .len = len,
        .speed_hz = 1000000,
    };
    struct spi_message m;
    
    spi_message_init(&m);
    spi_message_add_tail(&t, &m);
    
    return spi_sync(spi, &m);
}
```

## 总结

Linux SPI驱动框架提供了完整的SPI设备支持，包括：

1. **分层架构** - 清晰的核心层、控制器驱动、设备驱动分离
2. **灵活配置** - 支持多种SPI模式和传输参数
3. **高性能** - 支持DMA、中断驱动的异步传输
4. **易于扩展** - 标准化的接口便于添加新设备支持

理解SPI框架的核心数据结构和传输流程对于开发高质量的SPI驱动至关重要。通过合理的架构设计和性能优化，可以构建出稳定高效的SPI通信系统。

---

*本文档基于Linux 5.x内核版本编写，具体实现可能因内核版本而异。*