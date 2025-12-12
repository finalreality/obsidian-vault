## 
```
格式： [标签]:<名称>[@<设备地址>]
```
其中[标签]和[@<设备地址>]是可选项，<名称>是必选项。这里的<设备地址>没有实际意义，只是让节点更方便阅读和理解。如：
```bash
uart: serial@02288000
```

## reg属性
用来描述地址信息，如寄存器地址等
```bash
格式： reg = <address1 length1 address2 length2 address3 length3 ... ...>
```

如：
```bash
reg = <0x02200000 0x4000>
或
reg = <0x02200000 0x4000
		0x02205000 0x4000
		>;
```

##   \#address-cell和#size-cells属性
用来描述*子节点*中reg信息中的地址和长度信息
如：
```bash
node {
	#address-cells = <1>; //只有一个地址字段
	#size-cells = <0>; //没有长度字段
	
	node-child {
		reg = <0>; //寄存器地址为0
	};
};

node1 {
	#address-cells = <1>; //一个地址字段
	#size-cells = <1>; //一个长度字段
	node1-child {
		reg = <0x02200000 0x4000>; //寄存器地址0x02200000 寄存器长度0x4000
	};
};

node2 {
	#address-cells = <2>; //2个地址字段
	#size-cells = <0>; //没有长度字段
	
	node2-child {
		reg = <0x00 0x01>; //两个都是地址信息
	};
};
```

## model属性
用来描述信息，通常为一个字符串，如设备名称，名字等
如：
```
model = "vm8960-audio"；
或
model = "This is linux board"；
```

## status属性
status属性与设备状态有关，是一个字符串值，有如下几个状态：

|属性值 | 描述|
|---|----|
| okay | 设备为可用状态 |
| disabled | 设备为不可用状态 |
| fail | 设备为不可用状态,且设备检测到错误 |
| fail-msg | 设备为不可用状态,且设备检测到错误，msg为错误内容 |

```
/dts-v1/;
/{
	model = "This is my device";
	
	led: gpio@0x02201010 {
		status = "okay";
	};
};
```

## compatible属性
用来和驱动匹配的字段，非常重要，匹配成功后，会调用驱动的probe函数。匹配过程中使用第一个先匹配，没有匹配成功，则使用第二个。

```
compatible = "xunwei", "xunwei-board";
```

## 特殊节点aliases
用来定义别名，以方面引用节点，也可以对节点添加特殊标签来命名别名（例如之前的**led:**）。如：
```bash
aliases {
	mmc0 = &sdmmc0;
	mmc1 = &sdmmc1;
	mmc2 = &sdhci;
	
	serial0 = "/sample@fe000000/serial@11c500"
};
```
## 特殊节点chosen
用来从uboot传递参数给内核，重点是bootagrs参数，必须为根节点的子节点
```bash
chosen {
	bootargs = "root=/dev/nfs rw nfsroot=192.168.1.1 console=ttyUSB0,115200";
};
```
## device_type属性
字符串值，只用来描述cpu节点和memory节点。
```
memory@30000000 {
	device_type = "memory";
	reg = <0x30000000 0x4000000>;
};

cpu0: cpu@0 {
	device_type = "cpu";
	compatible = "arm,cortex-a35", "arm,armv8";
	reg = <0x0 0x1>;
};
```
## 自定义属性
设备树预定义属性不能满足要求时候，可以自定义属性。
```
pinnum = <0 1 2 3 4 5>;
```
## 总结
完整例子：
```bash
/dts-v1/;
/{
	model = "This is my device";
	
	chosen {
		bootargs = "root=/dev/nfs rw nfsroot=192.168.1.1 console=ttyUSB0,115200";
	};

	//别名
	aliases {
		led0 = &led; //反编译后，会被替换为：led0 = "gpio@0x02201010"
		
		nd1 = "/node1";
		nd11 = "/node1/node1"; //根节点下的node1下的node1节点
		nd2 = "/node2";
	};
	
	//下面两个属性影响子节点led的reg属性
	#address-cells = <1>;
	#size-cells = <1>;
	
	node1 {
		//下面两个属性影响子节点node1的reg属性
		#address-cells = <1>;
		#size-cells = <0>;
			
		node1 {
			reg = <0x22006000>;
		};
	};
	
	node2 {
		node-child {
			pinnum = <0 1 2 3 4 5>;
		};
	};
	
	led: gpio@0x02201010 {
		compatible = "led-myboard";
		reg = <0x22000000 0x40>;
		status = "okay";
	};
	
	
	memory@30000000 {
		device_type = "memory";
		reg = <0x30000000 0x4000000>;
	};
	
	cpu0: cpu@0 {
		device_type = "cpu";
		compatible = "arm,cortex-a35", "arm,armv8";
		reg = <0x0 0x1>;
	};
};
```

## 参考资料
DeviceTree-Specification-v0.4.pdf