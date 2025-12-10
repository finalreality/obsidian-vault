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
