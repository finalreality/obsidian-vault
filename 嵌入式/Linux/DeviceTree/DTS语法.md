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
	
	node1 {
		node1 {
		
		};
	};
	
	node2 {
		node-child {
		
		};
	};
	
	led: gpio@0x022010100 {
	
	};
};
```

## compatible属性
用来和驱动匹配的字段，非常重要，匹配成功后，会调用驱动的probe函数。匹配过程中使用第一个先匹配，没有匹配成功，则使用第二个。

```
compatible = "xunwei", "xunwei-board";
```