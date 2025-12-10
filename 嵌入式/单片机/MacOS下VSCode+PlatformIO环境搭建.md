# 环境配置
## 下载
```
https://code.visualstudio.com/Download
```
## 添加插件
![[MacOS下VSCode+PlatformIO环境搭建-1.png]]
## 安装platformio core和sdcc
```
brew install platformio
brew install sdcc
```

重新启动vscode后：
![[MacOS下VSCode+PlatformIO环境搭建-2.png]]

# 测试工程
在PlatformIO Home界面，“New Profect”新建工程:
![[MacOS下VSCode+PlatformIO环境搭建-3.png]]
我们Board选择和自己匹配的，我们这里选择STC89C52RC,点击Finish后，会进行相关配置，配置完成后：
![[MacOS下VSCode+PlatformIO环境搭建-4.png]]
我们可以看到工作区中已经有了我们新建的工程，已经有了源文件目录，但里面没有任何文件，需要我们自己添加所需代码。
在配置工程的过程中可能会卡住，这里的解决方式是在新建的工程目录中，手动运行工程配置指令：
```
pio init --board STC89C52RC
```
然后再在vscode中重新打开工程即可。
# 编写代码
```c
//

// Created by liwei on 2025/12/9.

//

#include <8052.h>

  

#define speaker P1_5

unsigned char timer0h, timer0l, time;

//--------------------------------------

// 单片机晶振采用11.0592MHz

// 频率-半周期数据表 高八位 本软件共保存了四个八度的28个频率数据

__code unsigned char FREQH[] = {

0xF2, 0xF3, 0xF5, 0xF5, 0xF6, 0xF7, 0xF8, // 低音1234567

0xF9, 0xF9, 0xFA, 0xFA, 0xFB, 0xFB, 0xFC, 0xFC, // 1,2,3,4,5,6,7,i

0xFC, 0xFD, 0xFD, 0xFD, 0xFD, 0xFE, // 高音 234567

0xFE, 0xFE, 0xFE, 0xFE, 0xFE, 0xFE, 0xFF}; // 超高音 1234567// 频率-半周期数据表 低八位

__code unsigned char FREQL[] = {

0x42, 0xC1, 0x17, 0xB6, 0xD0, 0xD1, 0xB6, // 低音1234567

0x21, 0xE1, 0x8C, 0xD8, 0x68, 0xE9, 0x5B, 0x8F, // 1,2,3,4,5,6,7,i

0xEE, 0x44, 0x6B, 0xB4, 0xF4, 0x2D, // 高音 234567

0x47, 0x77, 0xA2, 0xB6, 0xDA, 0xFA, 0x16}; // 超高音 1234567

  

//--------------------------------------

__code unsigned char sszymmh[] = {

// 第一段：长亭外，古道边，芳草碧连天

5, 2, 2, // 长 (sol, 中音, 1拍 = 2半拍)

3, 2, 1, // 亭 (re)

5, 2, 1,

1, 3, 4, // 外 (mi, 2拍)

  

6, 2, 2, // 古

1, 3, 1, // 道

6, 2, 1, // 边 (2拍)

5, 2, 4,

  

5, 2, 2, // 芳 (mi)

1, 2, 1, // 草 (sol)

2, 2, 1,

3, 2, 2, /// 碧 (la)

  

2, 2, 1, // 连 (sol)

1, 2, 1,

2, 2, 6, // 天 (mi, 3拍 = 附点二分音符)

0, 0, 2,

  

// 第二段：晚风拂柳笛声残，夕阳山外山

5, 2, 2, // 晚 (la)

3, 2, 1, // 风 (sol)

5, 2, 1, // 拂 (mi, 2拍)

1, 3, 3,

7, 2, 1, // 柳 （xi, 半拍)

  

6, 2, 2, // 笛 (la)

1, 3, 2, // 声 (sol)

5, 2, 2, // 残 (re, 2拍)

0, 0, 2,

  

5, 2, 2, // 夕 (sol)

2, 2, 1, // 阳 (la)

3, 2, 1,

4, 2, 3, // 山 (高音 do)

7, 1, 1, // 外 (la)

  

1, 2, 6, // 山 (sol, 3拍)

0, 0, 2, // 较长休止

  

6, 2, 2, // 天

1, 3, 2, // 之

1, 3, 4, // 涯

  

7, 2, 2, // 地之角

6, 2, 1,

7, 2, 1,

1, 3, 4,

  

// 知交半零落

6, 2, 1,

7, 2, 1,

1, 3, 1,

6, 2, 1,

6, 2, 1,

5, 2, 1,

3, 2, 1,

1, 2, 1,

2, 2, 8,

  

// 一壶浊酒尽余欢

5, 2, 2,

3, 2, 1,

5, 2, 1,

1, 3, 3,

7, 2, 1,

6, 2, 2,

1, 3, 2,

5, 2, 2,

0, 0, 2,

  

// 今宵别梦寒

5, 2, 2,

2, 2, 1,

3, 2, 1,

4, 2, 3,

7, 1, 1,

1, 2, 6,

0, 0, 2,

  

0, 0, 0 // 结束标志（time=0 退出）

};

  

//--------------------------------------

void t0_int() __interrupt(1) // T0中断程序，控制发音的音调

{

TR0 = 0; // 先关闭T0

speaker = !speaker; // 输出方波, 发音

TH0 = timer0h; // 下次的中断时间, 这个时间, 控制音调高低

TL0 = timer0l;

TR0 = 1; // 启动T0

}

//-------------------------------------

#define T 3 * 8000 // 延时常数, 控制发音的节奏

void delay(unsigned char t) // 延时程序，控制发音的时间长度

{

unsigned char t1;

unsigned long t2;

for (t1 = 0; t1 < t; t1++) // 双重循环, 共延时t个半拍

{

for (t2 = 0; t2 < T; t2++)

; // 延时期间, 可进入T0中断去发音

}

TR0 = 0; // 关闭T0, 停止发音

}

  

//--------------------------------------

void song() // 演奏一个音符

{

TH0 = timer0h; // 控制音调

TL0 = timer0l;

TR0 = 1; // 启动T0, 由T0输出方波去发音

delay(time); // 控制时间长度

}

//--------------------------------------

  

void main(void)

{

unsigned char k, i;

TMOD = 1;

ET0 = 1;

EA = 1;

while (1)

{

i = 0;

time = 1;

while (time)

{

unsigned char note = sszymmh[i];

unsigned char octave = sszymmh[i + 1];

time = sszymmh[i + 2];

  

if (note == 0)

{

// 休止符：关闭蜂鸣器，延时

TR0 = 0;

speaker = 0;

delay(time);

}

else

{

k = note + 7 * octave - 1; // 正确索引：1~7 + 八度

timer0h = FREQH[k];

timer0l = FREQL[k];

song(); // 发声

}

i += 3;

}

}

}
```
# 代码下载
在使用stcgal下载hex过程中，始终会卡死在Wait cpu之前，通过手动命令来进行是可以的：
```
stcgal -P stc89 -p /dev/tty.usbserial-144120 -b 9600 -D .pio/build/STC89C52RC/firmware.hex
```
于是重新设置预定义指令来实现：
```
[env:STC89C52RC]

platform = intel_mcs51
board = STC89C52RC

upload_port = /dev/cu.usbserial-144120
upload_speed = 9600

upload_protocol = custom

# 自定义烧录命令
upload_command = /usr/local/bin/stcgal -P stc89 -p /dev/tty.usbserial-144120 -b 9600 -D .pio/build/STC89C52RC/firmware.hex
```
