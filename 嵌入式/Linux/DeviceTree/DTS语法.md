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
	#address-cells = <1>; //只有一个地址
	size-cells = <0>;
	
	node-child {
		reg = <0>;
	};
};

node1 {
	#address-cells = <1>;
	#size-cells = <1>;
	node1-child {
		reg = <0x02200000 0x4000>;
	};
};

node2 {
	#address-cells = <2>;
	#size-cells = <0>;
	
	node2-child {
		reg = <0x00 0x01>;
	};
};
```

