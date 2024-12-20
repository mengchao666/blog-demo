---
title: DTS设备树
tags: [kernel]
categories: [kernel]
permalink: 
poster:
  topic: null
  headline: null
  caption: null
  color: null
date: 2024-11-25 23:09:20
topic: linux_kernel
description:
cover:
banner:
references:
---
设备树(Device Tree)，将这个词分开就是“设备”和“树”，描述设备树的文件叫做 DTS(Device Tree Source)，这个 DTS 文件采用树形结构描述板级设备，也就是开发板上的设备信息，比如 CPU 数量、 内存基地址、IIC 接口上接了哪些设备、SPI 接口上接了哪些设备等等

树的主干就是系统总线，IIC 控制器、GPIO 控制器、SPI 控制器等都是接到系统主线上的分支。IIC 控制器有分为 IIC1 和 IIC2 两种，其中 IIC1 上接了 FT5206 和 AT24C02 这两个 IIC 设备，IIC2 上只接了 MPU6050 这个设备。DTS 文件的主要功能就是按照图所示的结构来描述板子上的设备信息，DTS 文件描述设备信息是有相应的语法规则要求的，稍后我们会详细的讲解 DTS 语法规则。

设备树由一系列的节点和属性组成，节点可包含子节点。在设备树中，可描述的信息包括：

* CPU数量和类型
* 内存基地址和大小
* 总线和桥
* 外设连接
* 中断控制器和中断使用情况
* GPIO控制器和GPIO使用情况
* 时钟控制器和时钟使用情况

bootload 会将这些信息传递给内核，内核开始识别这些树，并解析成 Linux 内核中 platform_device, i2c_client, spi_device等设备，而这些设备使用的内存资源，中断等信息也传递给内核。内核会将这些资源绑定给相应的设备。

### 一、设备树例子

设备树相关的包含 3 部分：DTS、DTC、DTB

DTS 是设备树源码文件， DTB 是将 DTS 编译以后得到的二进制文件。那么将 .dts 编译为 .dtbn 需要什么工具呢？需要用到 DTC 工具

dts的一个例子如下：
比如 imx6ull.dtsi 就是描述 I.MX6ULL 这颗 SOC 内部外设情况信息。

```c
#include <dt-bindings/gpio/gpio.h>
#include <dt-bindings/interrupt-controller/arm-gic.h>
#include "imx6ull-pinfunc.h"
#include "imx6ull-pinfunc-snvs.h"
#include "skeleton.dtsi"
 
/ {
	aliases {
		can0 = &flexcan1;
		...
	};
	cpus {
		#address-cells = <1>;
		#size-cells = <0>;
 
		cpu0: cpu@0 {
			compatible = "arm,cortex-a7";
			device_type = "cpu";
			reg = <0>;
			clock-latency = <61036>; /* two CLK32 periods */
			operating-points = <
				/* kHz	uV */
				996000	1275000
				792000	1225000
				528000	1175000
				396000	1025000
				198000	950000
			>;
							/* kHz	uV */
				996000	1275000
				792000	1225000
				528000	1175000
				396000	1025000
			...
	};
 
	intc: interrupt-controller@00a01000 {
		compatible = "arm,cortex-a7-gic";
		#interrupt-cells = <3>;
		interrupt-controller;
		reg = <0x00a01000 0x1000>,
		<0x00a02000 0x100>;
	};
		...
```

文件描述了 CPU arm,cortex-a7 ，支持 996MHz、 792MHz等频率， 时钟一些信息。

“/”是根节点，每个设备树文件只有一个根节点

```c
node-name@unit-address
```

其中 “node-name” 是节点名字，为 ASCII 字符串，节点名字应该能够清晰的描述出节点的功能，比如 “uart1” 就表示这个节点是 UART1 外设。“unit-address” 一般表示设备的地址或寄存器首地址，如果某个节点没有地址或者寄存器的话 “unit-address” 可以不要，比如 “cpu@0”、“interrupt-controller@00a01000”。

```c
label: node-name@unit-address
```

引入 label 的目的就是为了方便访问节点，可以直接通过 &label 来访问这个节点，比如通过&cpu0 就可以访问 “cpu@0” 这个节点。很明显通过 &intc 来访问 “interrupt-controller@00a01000” 这个节点要方便很多！

每个节点都有不同属性，不同的属性又有不同的内容，属性都是键值对，值可以为空或任意的字节流。设备树源码中常用的几种数据形式如下所示：

1、字符串

```c
compatible = "fairchild,74hc595";
```

2、32 位无符号整数

```c
reg = <0 0x123456 100>;
```

3、字符串列表

```c
compatible = "fsl,imx6ul-pxp-v4l2", "fsl,imx6sx-pxp-v4l2", "fsl,imx6sl-pxp-v4l2";
```

### 二、设备树详解

##### (1)、标准属性

1、compatible 属性

compatible 属性也叫做“兼容性”属性，这是非常重要的一个属性！ compatible 属性的值是一个字符串列表， compatible 属性用于将设备和驱动绑定起来。字符串列表用于选择设备所要使用的驱动程序，compatible 属性的值格式如下所示：

```c
"manufacturer,model"
```

其中 manufacturer 表示厂商，model 一般是模块对应的驱动名字。比如 imx6ull-alientek-emmc.dts 中 sound 节点是 I.MX6U-ALPHA 开发板的音频设备节点，I.MX6U-ALPHA 开发板上的音频芯片采用的欧胜(WOLFSON)出品的 WM8960，sound 节点的 compatible 属性值如下：

```c
compatible = "fsl,imx6ul-evk-wm8960","fsl,imx-audio-wm8960";
```

其中 “fsl” 表示厂商是飞思卡尔，“imx6ul-evk-wm8960” 和 “imx-audio-wm8960” 表示驱动模块名字。设备首先使用第一个兼容值在 Linux 内核里面查找，如果没有找到的话就使用第二个兼容值查。

2、 model 属性

model 属性值也是一个字符串，一般 model 属性描述设备模块信息，比如名字什么的，比如：

```c
model = "wm8960-audio";
```

3、status 属性

status 属性看名字就知道是和设备状态有关的，status 属性值也是字符串，字符串是设备的状态信息，可选的状态如下表所示：

![](https://raw.githubusercontent.com/mengchao666/picture/main/blog803172a7f4daa540ee467df243b092a8.png)

4、★#address-cells 和#size-cells 属性

#address-cells# 属性值决定了子节点 reg 属性中地址信息所占用的字长(32 位)
#size-cells# 属性值决定了子节点 reg 属性中长度信息所占的字长(32 位)
#address-cells# 和 #size-cells# 表明了子节点应该如何编写 reg 属性值，一般 reg 属性都是和地址有关的内容，和地址相关的信息有两种：起始地址和地址长度， reg 属性的格式一为：

> reg = <address1 length1 address2 length2 address3 length3……>

每个 “address length” 组合表示一个地址范围，其中 address 是起始地址， length 是地址长度.
#address-cells# 表明 address 这个数据所占用的字长， #size-cells# 表明 length 这个数据所占用的字长

```c
aips3: aips-bus@02200000 {
	compatible = "fsl,aips-bus", "simple-bus";
	#address-cells = <1>;
	#size-cells = <1>;

	dcp: dcp@02280000 {
		compatible = "fsl,imx6sl-dcp";
		reg = <0x02280000 0x4000>;
	};
};
```

说明 aips3: aips-bus@02200000 节点起始地址长度所占用的字长为 1，地址长度所占用的字长也为 1
子节点 dcp: dcp@02280000 的 reg 属性值为<0x02280000 0x4000>相当于设置了起始地址为 0x02280000，地址长度为 0x40000，但是 dcp的地址长度(范围)并没有 0x4000 这么多

5、 ★reg 属性

reg 属性一般用于描述设备地址空间资源信息，一般都是某个外设的寄存器地址范围信息,一般是(address， length)组成，详情如上所述！

6、ranges 属性

ranges 属性值可以为空或者按照 (child-bus-address,parent-bus-address,length) 格式编写的数字矩阵， ranges 是一个地址映射/转换表， ranges 属性每个项目由子地址、父地址和地址空间长度这三部分组成：

child-bus-address：子总线地址空间的物理地址，由父节点的 #address-cells# 确定此物理地址所占用的字长。
parent-bus-address：父总线地址空间的物理地址，同样由父节点的 #address-cells# 确定此物理地址所占用的字长。
length：子地址空间的长度，由父节点的 #size-cells# 确定此地址长度所占用的字长。

如果 ranges 属性值为空值，说明子地址空间和父地址空间完全相同，不需要进行地址转换，例程如下：

```c
soc {
	#address-cells = <1>;
	#size-cells = <1>;
	compatible = "simple-bus";
	interrupt-parent = <&gpc>;
	ranges;//为空
```

ranges 属性不为空的示例代码如下所示：

```c
soc {
	compatible = "simple-bus";
	#address-cells = <1>;
	#size-cells = <1>;
	ranges = <0x0 0xe0000000 0x00100000>;

	serial {
		device_type = "serial";
		compatible = "ns16550";
		reg = <0x4600 0x100>;
		clock-frequency = <0>;
		interrupts = <0xA 0x8>;
		interrupt-parent = <&ipic>;
	};
};
```

节点 soc 定义的 ranges 属性，值为 <0x0 0xe0000000 0x00100000>，此属性值指定了一个 1024KB(0x00100000)的地址范围，子地址空间的物理起始地址为 0x0，父地址空间的物理起始地址为 0xe0000000。

serial 是串口设备节点，reg 属性定义了 serial 设备寄存器的起始地址为 0x4600，寄存器长度为 0x100。经过地址转换，serial 设备可以从 0xe0004600 开始进行读写操作，0xe0004600=0x4600+0xe0000000。

### 三、向节点追加或修改内容

imx6ull.dtsi 有以下内容，表示 I2C 节点。不同的 I2C 设备有不通的详细属性，采用追加节点方法不会对共有信息带来污染。

```c
i2c1: i2c@021a0000 {
	#address-cells = <1>;
	#size-cells = <0>;
	compatible = "fsl,imx6ul-i2c", "fsl,imx21-i2c";
	reg = <0x021a0000 0x4000>;
	interrupts = <GIC_SPI 36 IRQ_TYPE_LEVEL_HIGH>;
	clocks = <&clks IMX6UL_CLK_I2C1>;
	status = "disabled";
};
```

现在要在 i2c1 节点下创建一个子节点，这个子节点就是 fxls8471，最简单的方法就是在 i2c1 下直接添加一个名为 fxls8471 的子节点，如下所示：

```c
&i2c1 {
	clock-frequency = <100000>;
	pinctrl-names = "default";
	pinctrl-0 = <&pinctrl_i2c1>;
	status = "okay";

	fxls8471@1e {
		compatible = "fsl,fxls8471";
		reg = <0x1e>;
		position = <0>;
		interrupt-parent = <&gpio5>;
		interrupts = <0 8>;
};
```

子节点可以修改增加一些属性；
比如子节点中 clock-frequency 新增加的属性。
status 状态由disabled变成 okay

### 四、设备树在目录中的体现

运行 cd /proc/device-tree 后，ls -a 查询当前目录下的文本情况

![](https://raw.githubusercontent.com/mengchao666/picture/main/blog7fa958da33cda8112329b01e3bd409d8.png)

1、在当前目录下执行 cat model

> model 的内容是 “Freescale i.MX6 ULL 14x14 EVK Board”
> compatible 的内容为 “fsl,imx6ull-14x14-evkfsl,imx6ull”

![](https://raw.githubusercontent.com/mengchao666/picture/main/blogeb77e936e496f83c48bbafe2a30b0af0.png)

打开文件 imx6ull-alientek-emmc.dts 查看一下，这正是根节点 “/” 的 model 和 compatible 属性值

2、soc子节点

![](https://raw.githubusercontent.com/mengchao666/picture/main/blog20241125230745.png)

3、aliases 子节点

![image](assets/image-20241125230849-nfh9y2i.png)

与imx6ull.dtsi中的 aliases一致

4、chosen 子节点

chosen 并不是一个真实的设备， chosen 节点主要是为了 uboot 向 Linux 内核传递数据，重点是 bootargs 参数，一般.dts 文件中 chosen 节点通常为空或者内容很少， imx6ull-alientekemmc.dts 中 chosen 节点内容如下所示：

```c
chosen {
	stdout-path = &uart1;
}
```

chosen 节点仅仅设置了属性 “stdout-path”，表示标准输出使用 uart1。

```c
root@ATK-IMX6U:/proc/device-tree/chosen# ls
bootargs  name  stdout-path
```

我们可以发现 chosen 内存在 boot 的启动参数 bootargs！

![](https://raw.githubusercontent.com/mengchao666/picture/main/blog14c3de917c139c77d2b33b3b125e7b5e.png)

cat 查看确实是启动信息
1，我们并没有在设备树中设置 chosen 节点的 bootargs 属性，那么 bootargs这个属性是怎么产生的如何关联起来的呢？
2，为什么和 uboot 中的参数不一致？

chosen 节点的 bootargs 属性不是我们在设备树里面设置的，那么只有一种可能，那就是 uboot 自己在 chosen 节点里面添加了 bootargs 属性，并且设置 bootargs 属性的值为 bootargs环境变量的值。

uboot 源码中搜索 “chosen”，在文件 common/fdt_support.c 中

```c
int fdt_chosen(void *fdt)
{
	 //寻找chosen节点
	nodeoffset = fdt_find_or_add_subnode(fdt, 0, "chosen");

	if (nodeoffset < 0)
		return nodeoffset;
	//读取bootargs环境
	str = getenv("bootargs");
}
```

### 五、Linux 内核解析 DTB 文件

启动内核流程函数

![](https://raw.githubusercontent.com/mengchao666/picture/main/blogcf9237850e99e117ecac39bdde6c4911.png)

start_kernel 函数中最终调用了函数为 unflatten_dt_node（很多初始化操作都在start_kernel ）！

### 六、设备树节点的操作函数

Linux 驱动程序往往需要去读取到 Linux 内核中附带的 dts 文件，并操作设备树 DTS 的相关节点！接下来我们来学习一下，如何进行设备树节点操作！

#### 1、查找节点的 of 函数

Linux 内核使用 device_node 结构体来描述一个节点

1、 of_find_node_by_name 函数

```c
//通过节点名字查找指定的节点
struct device_node *of_find_node_by_name(struct device_node *from, const char *name);
//from：开始查找的节点，如果为 NULL 表示从根节点开始查找整个设备树。
//name：要查找的节点名字
```

2、of_find_node_by_type 函数

```c
//通过device_type查找指定的节点
struct device_node *of_find_node_by_type(struct device_node *from, const char *type)
```

3、 of_find_compatible_node 函数

```c
device_type 和 compatible 这两个属性查找指定的节点
struct device_node *of_find_compatible_node(struct device_node *from,
											const char *type,
											const char *compatible)
```

4、of_find_matching_node_and_match 函数

```c
//通过 of_device_id 匹配表来查找指定的节点
struct device_node *of_find_matching_node_and_match(struct device_node *from,
						const struct of_device_id *matches,
						const struct of_device_id **match)
```

5、of_find_node_by_path 函数

```c
struct device_node *of_get_parent(const struct device_node *node)
//node 要查找的父节点的节点
```

### 七、查找父/子节点的 OF 函数

1、of_get_parent 函数
用于获取指定节点的父节点

```c
struct device_node *of_get_parent(const struct device_node *node)
//node 要查找的父节点的节点
```

2、of_get_next_child 函数

```c
//用迭代的查找子节点
struct device_node *of_get_next_child(const struct device_node *node,
						              struct device_node *prev)
//node：父节点。
//prev：前一个子节点，也就是从哪一个子节点开始迭代的查找下一个子//节点。可以设置为NULL，表示从第一个子节点开始。
//返回 找到的下一个子节点。
```

### 八、提取属性值的 OF 函数

property 结构体，此结构体定义在文件 include/linux/of.h 中

```c
struct property {
	char *name; /* 属性名字 */
	int length; /* 属性长度 */
	void *value; /* 属性值 */
	struct property *next; /* 下一个属性 */
	unsigned long _flags;
	unsigned int unique_id;
	struct bin_attribute attr;
};
```

1、 of_find_property 函数

```c
property *of_find_property(const struct device_node *np,
						   const char *name,
						   int *lenp)
//np：设备节点。
//name： 属性名字。
//lenp：属性值的字节数
//返回找到的属性
```

2、 of_property_count_elems_of_size 函数
用于获取属性中元素的数量，比如 reg 属性值是一个数组，那么使用此函数可以获取到这个数组的大小

```c
int of_property_count_elems_of_size(const struct device_node *np,
							        const char *propname,
							        int elem_size)
//np：设备节点。
//proname： 需要统计元素数量的属性名字。
//elem_size：元素长度。
//返回 得到的属性元素数量
```

3、 of_property_read_u32_index 函数
从属性中获取指定标号的 u32 类型数据值(无符号 32 位)，比如某个属性有多个 u32 类型的值，那么就可以使用此函数来获取指定标号的数据值

```c
int of_property_read_u32_index(const struct device_node *np,
					           const char *propname,
					           u32 index,
					           u32 *out_value)
```

4、of_property_read_u8_array 函数
of_property_read_u16_array 函数
of_property_read_u32_array 函数
of_property_read_u64_array 函数

分别是读取属性中 u8、 u16、 u32 和 u64 类型的数组数据，比如大多数的 reg 属性都是数组数据，可以使用这 4 个函数一次读取出 reg 属性中的所有数据

5、 of_property_read_string 函数

```c
//用于读取属性中字符串值
int of_property_read_string(struct device_node *np,
                            const char *propname,
                            const char **out_string)
```

6、 of_n_addr_cells 函数
用于获取#address-cells 属性值

7、 of_n_size_cells 函数
of_size_cells 函数用于获取#size-cells 属性值

8、of_iomap 函数
采用设备树以后就可以直接通过 of_iomap 函数来获取内存地址所对应的虚拟地址，不需要使用 ioremap 函数了