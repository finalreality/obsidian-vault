// Linux SPI驱动框架架构图 - Mermaid格式

```mermaid
graph TB
    subgraph "用户空间"
        APP[应用程序]
    end
    
    subgraph "内核空间"
        subgraph "SPI设备驱动层"
            DRIVER1[Flash驱动]
            DRIVER2[传感器驱动]
            DRIVER3[显示驱动]
        end
        
        subgraph "SPI核心层"
            CORE[SPI Core<br/>spi.c]
            BUS[SPI总线管理]
            DEV_MGR[设备管理]
            TRANS[传输管理]
        end
        
        subgraph "SPI控制器驱动层"
            CTRL1[SPI Master 0]
            CTRL2[SPI Master 1]
            CTRL3[SPI Master 2]
        end
        
        subgraph "硬件抽象层"
            HW1[SPI控制器硬件0]
            HW2[SPI控制器硬件1]
            HW3[SPI控制器硬件2]
        end
    end
    
    subgraph "物理层"
        DEV1[SPI Flash]
        DEV2[SPI传感器]
        DEV3[SPI显示屏]
    end
    
    APP --> DRIVER1
    APP --> DRIVER2
    APP --> DRIVER3
    
    DRIVER1 --> CORE
    DRIVER2 --> CORE
    DRIVER3 --> CORE
    
    CORE --> BUS
    CORE --> DEV_MGR
    CORE --> TRANS
    
    BUS --> CTRL1
    BUS --> CTRL2
    BUS --> CTRL3
    
    DEV_MGR --> CTRL1
    DEV_MGR --> CTRL2
    DEV_MGR --> CTRL3
    
    TRANS --> CTRL1
    TRANS --> CTRL2
    TRANS --> CTRL3
    
    CTRL1 --> HW1
    CTRL2 --> HW2
    CTRL3 --> HW3
    
    HW1 --> DEV1
    HW1 --> DEV2
    HW2 --> DEV3
    HW3 --> DEV1
```

## 架构说明

### 分层架构
1. **用户空间层** - 应用程序通过设备文件或sysfs接口访问SPI设备
2. **SPI设备驱动层** - 各种SPI外设的功能驱动
3. **SPI核心层** - 提供统一的SPI总线管理和设备管理
4. **SPI控制器驱动层** - 硬件控制器的具体实现
5. **硬件抽象层** - SPI控制器硬件寄存器操作
6. **物理层** - 实际的SPI外设设备

### 核心组件关系
- SPI Core是框架的核心，负责协调各层之间的交互
- 设备驱动通过SPI Core访问控制器驱动
- 控制器驱动负责底层的硬件操作
- 支持多控制器和多设备的并发访问

### 数据流
1. 应用程序发起I/O请求
2. SPI设备驱动封装传输请求
3. SPI Core进行设备匹配和传输调度
4. SPI控制器驱动执行硬件传输
5. 数据在物理层完成实际的信号传输