# 基本环境搭建

# 代码编写
```c
/* Minimal STM32F103C8T6 blink example using STM32Cube HAL

- Board: genericSTM32F103C8

- Framework: stm32cube

- LED: PC13 (typical Blue Pill)

Toggle PC13 every 500 ms.

*/

  

#include "stm32f1xx_hal.h"

  

void SystemClock_Config(void);

static void MX_GPIO_Init(void);

void Error_Handler(void);

  

int main(void)

{

HAL_Init();

SystemClock_Config();

MX_GPIO_Init();

  

while (1)

{

HAL_GPIO_TogglePin(GPIOC, GPIO_PIN_13);

HAL_Delay(500);

}

}

  

void SystemClock_Config(void)

{

RCC_OscInitTypeDef RCC_OscInitStruct = {0};

RCC_ClkInitTypeDef RCC_ClkInitStruct = {0};

  

/* Initializes the CPU, AHB and APB busses clocks using HSI */

RCC_OscInitStruct.OscillatorType = RCC_OSCILLATORTYPE_HSI;

RCC_OscInitStruct.HSIState = RCC_HSI_ON;

RCC_OscInitStruct.HSICalibrationValue = RCC_HSICALIBRATION_DEFAULT;

RCC_OscInitStruct.PLL.PLLState = RCC_PLL_NONE;

if (HAL_RCC_OscConfig(&RCC_OscInitStruct) != HAL_OK)

{

Error_Handler();

}

  

RCC_ClkInitStruct.ClockType = RCC_CLOCKTYPE_HCLK|RCC_CLOCKTYPE_SYSCLK

|RCC_CLOCKTYPE_PCLK1|RCC_CLOCKTYPE_PCLK2;

RCC_ClkInitStruct.SYSCLKSource = RCC_SYSCLKSOURCE_HSI;

RCC_ClkInitStruct.AHBCLKDivider = RCC_SYSCLK_DIV1;

RCC_ClkInitStruct.APB1CLKDivider = RCC_HCLK_DIV2;

RCC_ClkInitStruct.APB2CLKDivider = RCC_HCLK_DIV1;

  

if (HAL_RCC_ClockConfig(&RCC_ClkInitStruct, FLASH_LATENCY_0) != HAL_OK)

{

Error_Handler();

}

}

  

static void MX_GPIO_Init(void)

{

/* Enable GPIOC, GPIOA and GPIOB clocks */

__HAL_RCC_GPIOC_CLK_ENABLE();

__HAL_RCC_GPIOA_CLK_ENABLE();

__HAL_RCC_GPIOB_CLK_ENABLE();

  

GPIO_InitTypeDef GPIO_InitStruct = {0};

  

/* Configure PC13 as push-pull output */

HAL_GPIO_WritePin(GPIOC, GPIO_PIN_13, GPIO_PIN_SET);

GPIO_InitStruct.Pin = GPIO_PIN_13;

GPIO_InitStruct.Mode = GPIO_MODE_OUTPUT_PP;

GPIO_InitStruct.Pull = GPIO_NOPULL;

GPIO_InitStruct.Speed = GPIO_SPEED_FREQ_LOW;

HAL_GPIO_Init(GPIOC, &GPIO_InitStruct);

  

/* Configure PA5 as push-pull output and set it HIGH */

GPIO_InitTypeDef GPIOA_Init = {0};

GPIOA_Init.Pin = GPIO_PIN_5;

GPIOA_Init.Mode = GPIO_MODE_OUTPUT_PP;

GPIOA_Init.Pull = GPIO_NOPULL;

GPIOA_Init.Speed = GPIO_SPEED_FREQ_LOW;

HAL_GPIO_Init(GPIOA, &GPIOA_Init);

HAL_GPIO_WritePin(GPIOA, GPIO_PIN_5, GPIO_PIN_SET);

  

/* Configure all PB pins as push-pull outputs and drive them LOW */

GPIO_InitTypeDef GPIOB_Init = {0};

GPIOB_Init.Pin = GPIO_PIN_All;

GPIOB_Init.Mode = GPIO_MODE_OUTPUT_PP;

GPIOB_Init.Pull = GPIO_NOPULL;

GPIOB_Init.Speed = GPIO_SPEED_FREQ_LOW;

HAL_GPIO_Init(GPIOB, &GPIOB_Init);

HAL_GPIO_WritePin(GPIOB, GPIO_PIN_All, GPIO_PIN_RESET);

}

  

void Error_Handler(void)

{

__disable_irq();

while (1)

{

}

}
```
代码中将PB口全部输出低电平。
# 代码下载
## 工程配置
```bash
; PlatformIO Project Configuration File

;

; Build options: build flags, source filter

; Upload options: custom upload port, speed and extra flags

; Library options: dependencies, extra library storages

; Advanced options: extra scripting

;

; Please visit documentation for the other options and examples

; https://docs.platformio.org/page/projectconf.html

  

[env:genericSTM32F103C8]

platform = ststm32

board = genericSTM32F103C8

framework = stm32cube

  

upload_protocol = serial

upload_port = /dev/tty.usbserial-144120

upload_flags = -b 115200 -m 8n1
```

## 注意事项
在单片机准备工作中，参考开发板厂家给的指导，接好线：
![[MacOS下VSCode+PlatformIO环境搭建 —— STM32F103C8T6-1.png]]
主要包括：
1. 底板和stm32cpu板上RTS、DTR两个连线
2. 3.3V选择跳线
3. 串口选择跳线
4. stm32cpu板上的BOOT1跳线
# 测试
在PC上装stm32flash
```bash
brew install stm32flash
```

运行测试：
```bash
/usr/local/Cellar/stm32flash/0.7/bin/stm32flash  /dev/tty.usbserial-144120
```
结果：
```bash
stm32flash 0.7

http://stm32flash.sourceforge.net/

Using Parser : Raw BINARY
Size         : 2804
Interface serial_posix: 115200 8E1
Failed to init device, timeout.
```
参考下载文档中手动方式：
先按下ISPK不松手，再按下RSTK，然后再依次松开RSTK，松开ISPK，然后再运行指令，结果：
```bash
/usr/local/Cellar/stm32flash/0.7/bin/stm32flash  /dev/tty.usbserial-144120
stm32flash 0.7

http://stm32flash.sourceforge.net/

Interface serial_posix: 57600 8E1
Version      : 0x22
Option 1     : 0x00
Option 2     : 0x00
Device ID    : 0x0410 (STM32F10xxx Medium-density)
- RAM        : Up to 20KiB  (512b reserved by bootloader)
- Flash      : Up to 128KiB (size first sector: 4x1024)
- Option RAM : 16b
- System RAM : 2KiB

```
我们可以发现能够识别芯片，我们进入到工程目录手动下载：

```bash
$ cd ~/Documents/PlatformIO/Projects/STM32F103/.pio/build/genericSTM32F103C8
$ stm32flash -b 115200 -w firmware.bin -v -g 0x0 /dev/tty.usbserial-144120

stm32flash 0.7

http://stm32flash.sourceforge.net/

Using Parser : Raw BINARY
Size         : 2804
Interface serial_posix: 115200 8E1
Version      : 0x22
Option 1     : 0x00
Option 2     : 0x00
Device ID    : 0x0410 (STM32F10xxx Medium-density)
- RAM        : Up to 20KiB  (512b reserved by bootloader)
- Flash      : Up to 128KiB (size first sector: 4x1024)
- Option RAM : 16b
- System RAM : 2KiB
Write to memory
Erasing memory
Wrote and verified address 0x08000af4 (100.00%) Done.

Starting execution at address 0x08000000... done.
```
我们看到，下载成功，回到VSCode，手动复位板子后，可以直接点击按钮下载成功：
![[MacOS下VSCode+PlatformIO环境搭建 —— STM32F103C8T6-2.png]]