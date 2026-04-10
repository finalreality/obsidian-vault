### 文档 1：Android 显示子系统架构概览 (`Architecture_Overview.md`)

# Android 显示子系统架构分析

## 1. 核心分层架构
Android 显示系统的本质是一个**生产者-消费者（Producer-Consumer）**模型，通过 `BufferQueue` 机制实现图形数据的传
递。

### 1.1 分层结构
| 层级 (Layer)                           | 核心组件                                      | 职责描述                                            |
| :----------------------------------- | :---------------------------------------- | :---------------------------------------------- |
| **Application/Framework**            | `View`, `Surface`, `WindowManagerService` | 定义窗口、处理触摸事件、管理窗口布局。                             |
| **Native Service**                   | `SurfaceFlinger`, `BufferQueue`           | **系统的核心**。负责合成所有图层（Layer），决定最终像素如何显示。           |
| **HAL (Hardware Abstraction Layer)** | `HWC (Hardware Composer)`, `Gralloc`     | `HWC`负责硬件合成逻辑；`Gralloc` 负责内存分配（Graphic Buffer）。 |
| **Kernel/Driver** | `DRM/KMS`, `Display Controller (DC)` | 驱动层，控制显示控制器，将合成后的像素流推向屏幕。|

## 2. 核心组件详解

### 2.1 Surface & SurfaceControl
*   **Surface**: 应用程序的“画布”。
*   **SurfaceControl**: 在 Native 层用于控制图层的属性（位置、透明度、裁剪区域等）。

### 2BufferQueue (通信枢纽)
这是整个显示系统的“脊梁”。
*   **Producer (生产者)**: 如 App、MediaCodec、SurfaceControl。它们向 Queue 中 `enqueue` 缓冲区。
*   **Consumer (消费者)**: 如 `SurfaceFlinger`。它从 Queue 中 `dequeue` 缓冲区并进行合成。

### 2.3 SurfaceFlinger (合成器)
*   **Composition Strategy**: 决定哪些图层走 **Client Composition** (由 GPU/OpenGL 完成) ，哪些图层走 **Device
Composition** (由 HWC 硬件完成)。
*   **VSYNC 驱动**: 监听 VSYNC 信号，驱动每一帧的刷新。

### 2.4 HWC (Hardware Composer)
*   **Overlay**: 如果硬件支持，HWC 可以直接将多个图层（如状态栏、壁纸、应用窗口）进行硬件叠加，无需 CPU/GPU 参与
计算，极大地降低功耗。