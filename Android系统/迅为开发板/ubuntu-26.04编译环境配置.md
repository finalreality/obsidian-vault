```bash
sudo apt install -y build-essential openjdk-8-jdk git-core gnupg flex bison gperf libxml2-utils   xz-utils zip curl zlib1g-dev g++-multilib lib32z-dev libsdl1.2-dev libncurses5-dev libssl-dev   bc tofrodos python3 python3-pip python3-pexpect python3-git python3-subunit mesa-common-dev libxml2-dev   libxml2-utils bzip2 libbz2-dev squashfs-tools pngcrush liblz4-dev liblz4-1
```

找不到libtinfo.so.5问题
```bash
	sudo ln -s /usr/lib/x86_64-linux-gnu/libtinfo.so.6.6 /usr/lib/x86_64-linux-gnu/libtinfo.so.5
	sudo ln -s /usr/lib/x86_64-linux-gnu/libncurses.so.6.6 /usr/lib/x86_64-linux-gnu/libncurses.so.5
```