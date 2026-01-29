修改交换文件：
```
sudo swapoff /swap.img
sudo dd if=/dev/zero of=/swap.img bs=1M count=32768
sudo chmod 0600 swapfile
sudo mkswap /swap.img
sudo swapon /swap.img
```

编辑fstab文件，添加如下行:
```
/swap.img	none	swap	sw	0	0
```

安装依赖：
```bash
sudo apt install -y build-essential openjdk-8-jdk git-core gnupg flex bison gperf libxml2-utils   xz-utils zip curl zlib1g-dev g++-multilib lib32z-dev libsdl1.2-dev libncurses5-dev libssl-dev bc tofrodos python3 python3-pip python3-pexpect python3-git python3-subunit mesa-common-dev libxml2-dev   libxml2-utils bzip2 libbz2-dev squashfs-tools pngcrush lz4 liblz4-dev liblz4-1 protobuf-compiler libprotoc-dev libprotobuf-dev samba autoconf bison flex gcc g++ git libprotobuf-dev libnl-route-3-dev libtool make pkg-config protobuf-compiler 
```
**Build sandboxing disabled due to nsjail error*错误：
```
sudo vim /usr/lib/sysctl.d/10-apparmor.conf
```
将下面这行设置为0：
```
kernel.apparmor_restrict_unprivileged_userns = 0
```

找不到nsjail：
```
git clone https://gitcode.com/gh_mirrors/ns/nsjail.git
cd nsjail/
make
sudo cp nsjail /usr/local/bin
```
找不到libtinfo.so.5和libncurses.so.5的问题：
```bash
	sudo ln -s /usr/lib/x86_64-linux-gnu/libtinfo.so.6.6 /usr/lib/x86_64-linux-gnu/libtinfo.so.5
	sudo ln -s /usr/lib/x86_64-linux-gnu/libncurses.so.6.6 /usr/lib/x86_64-linux-gnu/libncurses.so.5
```

```
cd /build/rk3576_android14/
source build/envsetup.sh 
lunch
./build.sh -ACKu && echo "编译完成时间: $(date '+%Y-%m-%d %H:%M:%S')"

```