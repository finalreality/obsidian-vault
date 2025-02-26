在出厂系统中，该案件驱动基于input子系统而实现，所以在/dev/input目录下存在KEY0的设备节点，具体是哪个设备节点，可以查看/proc/bus/input/devices文件得知。

![3121c894-6042-11ed-8abf-dac502259ad0.png](https://file1.elecfans.com//web2/M00/97/BF/wKgaomTnNoCAHN1yAAHUn9YLVtM633.png)

```shell
#!/bin/sh
cmake -DCMAKE_TOOLCHAIN_FILE=/home/liwei66/Android/Sdk/ndk/26.1.10909125/build/cmake/android.toolchain.cmake \
      -DANDROID_ABI=arm64-v8a \
      -DANDROID_NDK=/home/liwei66/Android/Sdk/ndk/26.1.10909125 \
      -DCMAKE_BUILD_TYPE=Release \
      -DANDROID_NATIVE_API_LEVEL=30 \
      -B build
cmake --build build -j12

```
```cmake
cmake_minimum_required(VERSION 3.20)
project(test-input)

add_executable(test-input test-input.cpp)
add_executable(test-input2 test-input2.cpp)
add_executable(test-input3 test-input3.cpp)
add_executable(test-input4 test-input4.cpp)
```


```cpp
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <fcntl.h>
#include <linux/input.h>

int main() {
    int fd = open("/dev/input/event6", O_RDONLY | O_NONBLOCK);
    if (fd == -1) {
        perror("open");
        return 1;
    }

    printf("open success\n");
    struct input_event ev;
    while (true) {
	    if(read(fd, &ev, sizeof(ev)) > 0)
        printf("Event type: %d, code: %d, value: %d\n", ev.type, ev.code, ev.value);
        // 处理事件
    }

    close(fd);
    return 0;
}

```