```mermaid
flowchart LR
    A[dtsi、dts] --> B[dtc]
    B  --编译--> C[dtb]
```

Dts文件路径：
arch/arm[arm64]/boot/dts/xxx

最简单dts：

```c my_device_tree.dts
/dts-v1/;
/ {

};
```
编译：
```
dtc -I dts -O dtb --o 
```