// Linux SPI驱动框架类图 - Mermaid格式

```mermaid
classDiagram
    class spi_master {
        +struct device dev
        +struct list_head list
        +s16 bus_num
        +u16 num_chipselect
        +u16 mode_bits
        +u32 min_speed_hz
        +u32 max_speed_hz
        +int setup(spi_device *spi)
        +int transfer(spi_device *spi, spi_message *mesg)
        +void cleanup(spi_device *spi)
    }
    
    class spi_device {
        +struct device dev
        +struct spi_master *master
        +u32 max_speed_hz
        +u8 chip_select
        +u8 mode
        +u8 bits_per_word
        +int irq
        +void *controller_state
        +char modalias[32]
    }
    
    class spi_driver {
        +struct spi_device_id *id_table
        +int probe(spi_device *spi)
        +int remove(spi_device *spi)
        +void shutdown(spi_device *spi)
        +int suspend(spi_device *spi, pm_message_t mesg)
        +int resume(spi_device *spi)
        +struct device_driver driver
    }
    
    class spi_transfer {
        +const void *tx_buf
        +void *rx_buf
        +unsigned len
        +dma_addr_t tx_dma
        +dma_addr_t rx_dma
        +unsigned cs_change:1
        +u8 bits_per_word
        +u16 delay_usecs
        +u32 speed_hz
    }
    
    class spi_message {
        +struct list_head transfers
        +struct spi_device *spi
        +unsigned is_dma_mapped:1
        +void (*complete)(void *context)
        +void *context
        +unsigned actual_length
        +int status
        +struct list_head queue
        +void *state
    }
    
    class device {
        +struct device *parent
        +struct device_private *p
        +const char *init_name
        +struct device_type *type
        +struct bus_type *bus
        +struct device_driver *driver
        +void *platform_data
        +void *driver_data
    }
    
    class device_driver {
        +const char *name
        +struct bus_type *bus
        +struct module *owner
        +const char *mod_name
        +int probe(struct device *dev)
        +int remove(struct device *dev)
        +void shutdown(struct device *dev)
    }
    
    class spi_controller_driver {
        +struct spi_master *master
        +void *private_data
        +int setup_transfer(spi_device *spi, spi_transfer *t)
        +int transfer_one_message(spi_master *master, spi_message *mesg)
        +int prepare_transfer_hardware(spi_master *master)
        +int unprepare_transfer_hardware(spi_master *master)
        +int prepare_message(spi_master *master, spi_message *mesg)
        +int unprepare_message(spi_master *master, spi_message *mesg)
    }
    
    class spi_board_info {
        +char modalias[32]
        +const void *platform_data
        +void *controller_data
        +int irq
        +u32 max_speed_hz
        +u16 bus_num
        +u16 chip_select
        +u8 mode
    }
    
    class spi_statistics {
        +unsigned long spi_sync
        +unsigned long spi_async
        +unsigned long spi_sync_immediate
        +unsigned long spi_async_immediate
        +unsigned long spi_sync_delayed
        +unsigned long spi_async_delayed
        +unsigned long bytes_transferred
        +unsigned long messages_processed
        +unsigned long transfers_processed
        +unsigned long errors
        +unsigned long timedout
    }
    
    %% 核心类关系
    spi_master "1" --> "0..*" spi_device : manages
    spi_device "*" --> "1" spi_master : belongs_to
    spi_driver "1" --> "0..*" spi_device : drives
    spi_device "*" --> "1" spi_driver : driven_by
    
    %% 传输关系
    spi_message "1" --> "1" spi_device : target
    spi_message "1" --> "0..*" spi_transfer : contains
    spi_transfer "*" --> "1" spi_message : part_of
    
    %% 继承关系
    device <|-- spi_master : inherits
    device <|-- spi_device : inherits
    device_driver <|-- spi_driver : inherits
    
    %% 控制器驱动关系
    spi_master "1" --> "1" spi_controller_driver : implements
    spi_controller_driver "1" --> "1" spi_master : controls
    
    %% 板级信息关系
    spi_board_info "*" --> "1" spi_master : configures
    
    %% 统计信息
    spi_master "1" --> "1" spi_statistics : tracks
```

## 类图说明

### 核心类职责

1. **spi_master** - SPI主机控制器
   - 管理SPI总线和所有连接的设备
   - 提供传输接口和硬件抽象
   - 处理多设备并发访问

2. **spi_device** - SPI设备
   - 表示连接到SPI总线的具体设备
   - 包含设备配置参数
   - 维护设备状态信息

3. **spi_driver** - SPI设备驱动
   - 实现具体设备的功能逻辑
   - 处理设备探测和移除
   - 管理设备生命周期

4. **spi_transfer** - SPI传输单元
   - 表示单次SPI数据传输
   - 包含传输缓冲区、长度、时序参数
   - 支持DMA传输

5. **spi_message** - SPI消息
   - 包含一个或多个传输单元
   - 提供传输完成回调机制
   - 支持异步传输

### 设计模式

1. **设备-驱动模型** - Linux标准设备模型
2. **主从模式** - SPI控制器管理多个设备
3. **消息队列** - 异步传输的消息队列机制
4. **策略模式** - 不同的传输策略（同步/异步/DMA）

### 关键关系

- 一个SPI主控制器可以管理多个SPI设备
- 一个SPI设备只能属于一个主控制器
- 一个SPI消息可以包含多个传输单元
- 设备和驱动通过ID表进行匹配