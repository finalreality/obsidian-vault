---
title: "Mac 合盖掉电（休眠掉电）解决方法"
source: "https://www.cnblogs.com/hoey94/p/16361916.html"
author:
  - "[[hoey94]]"
published: 2022-06-10T09:00:00.0000000&#x2B;08:00
created: 2025-11-29
description: "Mac在合盖之后还在掉电，第二天打开一看，只有40%的电量。针对合盖掉电的问题可能有以下解决方法。 1、重置Mac的SMC 根据Mac官方的提示，重置爱SMC可以解决某些电源，电池，风扇相关的问题。重置方法如下： 如何重置 Mac 的 SMC​support.apple.com/zh-tw/HT20"
tags:
  - "clippings"
---
## 1、重置Mac的SMC

根据Mac官方的提示，重置爱SMC可以解决某些电源，电池，风扇相关的问题。重置方法如下：

Mac官方针对不同型号的电脑都给出了相关的设置方法。根据教程设置后可能可以解决相关的问题。

## 2、在终端中修改电源选项

在终端中查询相关状态，不了解终端是什么东东的朋友可以翻看我之前的文章。

简洁版本：

为了节约大家时间，这里给一个简单版本。

1.合盖后关闭网络唤醒：

```text
sudo pmset -b tcpkeepalive 0
```

2.减小合盖后数据在内存中保留的时间（默认8个小时 = 28800秒）

```text
sudo pmset -b standbydelay 14400
```

3.减小装数据写入硬盘彻底断电的时间

```text
sudo pmset -b autopoweroffdelay 14400
```

4.修改休眠模式

```text
sudo pmset -b hibernatemode 25
```

（需要说明的是，修改为25之后，唤醒会稍微变慢，大约需要4~5s，介意者可修改为3）

注意以上操作都需要在敲入命令后输入自己的开机密码然后按回车。

详细版本

在终端中输入pmset -g，查看相关的电源设置(power managment settings，电源管理设置的意思)；在插电源使用时看到的是插电源状态下的设置，合盖掉电应该是电池状态下的问题，建议拔掉电源后再输入指令。

```text
System-wide power settings:
Currently in use:
 lidwake              1
 autopoweroff         1
 standbydelayhigh     86400
 autopoweroffdelay    28800
 proximitywake        0
 standby              1
 standbydelaylow      10800
 ttyskeepawake        1
 hibernatemode        25
 powernap             0
 gpuswitch            2
 hibernatefile        /var/vm/sleepimage
 highstandbythreshold 50
 displaysleep         5
 sleep                5 (sleep prevented by timed)
 acwake               0
 halfdim              1
 tcpkeepalive         0
 disksleep            10
```

输入pmset -g后会输出现在相关的状态信息，我这里把出现的各个项目按顺序简要说明一下。

- lidwake – 当笔记本打开盖子的时候唤醒机器（值为 1 或者 0）
- autopoweroff – 系统将写入休眠镜像并且进入到低电量芯片组睡眠。从这个状态唤醒所花的时间要比普通休眠唤醒的时间要长。如果有外部设备连接，系统不会自动切断电源，如果系统使用电池供电，或者系统被绑定在网络并且通过网络访问被唤醒功能开启。
- standbydelayhigh - 当电源电量够高时，合盖后内存保留的秒数
- autopoweroffdelay – 进入自动切断电源模式的延迟（值为表示分钟的整数）
- proximitywake - 同 iCloud 设备唤醒
- standby - 合盖后保留内存（值为0或者1）
- standbydelaylow - 电池电量低时，合盖后内存保留的秒数
- ttyskeepawake – 当任何 tty（如：远程登录会话） 在活动状态时，阻止系统空闲睡眠。tty 只能是非活动 当它的空闲时间超过系统睡眠计时器（值为 1 或者 0）
- hibernatemode – 改变休眠模式
- powernap - 电源小憩（0或1）
- gpuswitch - gpu支持（2为自动模式）
- hibernatefile – 改变休眠镜像文件位置。镜像应该只被定为到根卷中。请小心使用（值为路径）
- highstandbythreshold - 电池剩余电量百分比 standby模式的选择阈值，一般为50%
- displaysleep – 显示器睡眠计时器；替换 10.4 版本中的 dim 参数（值为分钟，或者设置 0 来禁用）
- sleep – 系统睡眠计时器（值为分钟，或者设置 0 来禁用）
- acwake – 当电源（AC 或者电池）改变的时候唤醒机器（值为 1 或者 0）
- halfdim – 显示器睡眠将使用在最大亮度和关闭显示器之间的中间亮度（值为 1 或者 0）
- tcpkeepalive - 合盖时是否保存网络连接
- disksleep - 硬盘休眠时间（值为分钟）在合盖之后，或者长时间不操作电脑之后，系统将进入等待休眠的模式，在sleep的时间之后，依据hibernatemode的模式不同，会进入不同的休眠模式。

依据Mac的休眠流程，我们主要需要修改的参数有以下几个： sleep，hibernatemode，提高standby的电量阈值；tcpkeepalive设置为0，合盖后中断网络链接；proximitywake设置成 0, 关闭被同一网络下的同 iCloud 设备唤醒。

这里需要对hibernatemode的不同参数说明一下。

- hibernatemode = 0

1. iMac, Mac Mini等 Mac桌面设备默认参数
2. 持续向内存供电，将数据保留在内存
3. 唤醒速度快，减少硬盘占用
4. 数据有丢失风险
5. 耗电量大
6. hibernatemode = 25
7. 将数据写入硬盘
8. 不向内存供电，将内存镜像直接写入硬盘
9. 数据不易丢失，镜像占用硬盘空间
10. 唤醒速度慢
11. 耗电量少
12. hibernatemode = 3
13. MacBook 笔记本设备默认参数
14. safe sleep, 数据既写入内存又写入硬盘
15. 持续向内存供电
16. 唤醒时，根据设备电量自动选择从 内存/硬盘 恢复

依据以上参数含义，我们将电池供电下的状态设置如下：

```text
// 5 分钟后进入休眠
sudo pmset -b sleep 5
// 向硬盘写入镜像，不向内存供电
sudo pmset -b hibernatemode 25
// 显示器休眠时间：15 分钟
sudo pmset -b displaysleep 15
// 硬盘休眠时间：30 分钟
sudo pmset -b disksleep 30
// 休眠时断网
sudo pmset -b tcpkeepalive 0
// 高电量下 standby: 4小时
sudo pmset -b standbydelayhigh 14400
// 低电量下 standby: 2小时
sudo pmset -b standbydelaylow 7200
// standby 电量阈值：75%
sudo pmset -b highstandbythreshold 75
// 开盖唤醒
sudo pmset -b lidwake 1
// 关闭被同一 iCloud 下的设备唤醒
sudo pmset -b acwake 0
```

经过以上设置之后，应该就可以减少合盖后Mac的掉电情况。实测掉电很少，一晚上大约1%吧。希望以上内容可以对大家有所帮助，谢谢大家。

参考资料：

[MacOS 关闭 tcpkeepalive 解决合盖掉电问题](https://link.zhihu.com/?target=https%3A//blog.upall.cn/1897.html)

[通过 pmset 工具管理 masOS 睡眠，让你的 Mac 睡得更好 - 少数派](https://link.zhihu.com/?target=https%3A//sspai.com/post/61379)

[How to Hibernate a Mac](https://link.zhihu.com/?target=https%3A//computers.tutsplus.com/tutorials/how-to-hibernate-a-mac--cms-23235)

[https://en.wikipedia.org/wiki/P](https://link.zhihu.com/?target=https%3A//en.wikipedia.org/wiki/Pmset)

转自：https://zhuanlan.zhihu.com/p/294859171