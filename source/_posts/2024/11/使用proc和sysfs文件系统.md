---
title: 使用proc和sysfs文件系统
tags: [kernel]
categories: [kernel]
permalink: 
poster:
  topic: null
  headline: null
  caption: null
  color: null
date: 2024-11-25 23:15:30
topic: linux_kernel
description:
cover:
banner:
references:
---
proc目录详解：https://blog.csdn.net/qq_39974998/article/details/130536491

### 一、DEVICE_ATTR

在调试驱动的时候我们一般会对于驱动中某一个属性或者变量进行操作，或者是控制gpio口，这个时候我们可以在驱动中创建对应的属性，从而在应用程序或者控制台对驱动的属性进行设置,sysfs可以通过sysfs_create_files和sysfs_create_group()来创建，其中使用DEVICE_ATTR来创建属性文件

这个宏定义在kernel/include/linux/device.h中，函数定义为：

```c
#define DEVICE_ATTR(_name, _mode, _show, _store) \
	struct device_attribute dev_attr_##_name = __ATTR(_name, _mode, _show, _store)

#define __ATTR(_name, _mode, _show, _store) {				\
	.attr = {.name = __stringify(_name),				\
	.mode = VERIFY_OCTAL_PERMISSIONS(_mode) },		\
	.show	= _show,						\
	.store	= _store,						\
}
```

mode为文件的权限，定义在kernel/include/uapi/linux/stat.h

```c
#define S_IRWXU 00700 //用户可读写和执行
#define S_IRUSR 00400//用户可读
#define S_IWUSR 00200//用户可写
#define S_IXUSR 00100//用户可执行
 
#define S_IRWXG 00070//用户组可读写和执行
#define S_IRGRP 00040//用户组可读
#define S_IWGRP 00020//用户组可写
#define S_IXGRP 00010//用户组可执行
 
#define S_IRWXO 00007//其他可读写和执行
#define S_IROTH 00004//其他可读
#define S_IWOTH 00002//其他可写
#define S_IXOTH 00001//其他可执行
```

### 二、device_attribute

```c
 /* interface for exporting device attributes */
 struct device_attribute {
     struct attribute    attr;
     ssize_t (*show)(struct device *dev, struct device_attribute *attr,
             char *buf);
     ssize_t (*store)(struct device *dev, struct device_attribute *attr,
              const char *buf, size_t count);
 };
```

### 三、创建sysfs文件

#### 法1：sysfs_create_files

```c
static ssize_t led_store(struct device *dev,struct device_attribute *attr, const char *buf, size_t len)
{
    printk("led_store()\n");
    return len;//必须返回传入的长度
}

 //下面的show和store只是简单举例
static ssize_t led_show(struct device *dev, struct device_attribute*attr, char *buf)
{
         printk("led_show()\n");
         returnpr_info("store\n");
 }

//用DEVICE_ATTR宏创建属性led_sysfs文件，如果show()或是store()没有功能，就以NULL代替
static DEVICE_ATTR(led_sysfs, S_IWUSR, led_show,led_store);

//最后一项必须以NUll结尾
static const struct attribute *atk_imx6ul_led_sysfs_attrs[] = {
	&dev_attr_led_sysfs.attr,
	NULL,
};

sysfs_create_files(&led_device.device->kobj,atk_imx6ul_led_sysfs_attrs);
sysfs_remove_file(&led_device.device->kobj, atk_imx6ul_led_sysfs_attrs);//驱动退出时释放结点
```

其中kobj的定义为

```c
kobject在设备结构体中的描述：
struct platform_device
     —>struct device dev
        —>struct kobject kobj
```

#### 法2(推荐)：sysfs_create_group()

使用：将上面的attribute结构体再进行封装

```c
static ssize_t led_store(struct device *dev,struct device_attribute *attr, const char *buf, size_t len)
{
    printk("led_store()\n");
    return len;//必须返回传入的长度
}

 //下面的show和store只是简单举例
static ssize_t led_show(struct device *dev, struct device_attribute*attr, char *buf)
{
         printk("led_show()\n");
         returnpr_info("store\n");
 }

//用DEVICE_ATTR宏创建属性led_sysfs文件，如果show()或是store()没有功能，就以NULL代替
static DEVICE_ATTR(led_sysfs, S_IWUSR, led_show,led_store);

//最后一项必须以NUll结尾
static struct attribute *atk_imx6ul_led_sysfs_attrs[] = {
	&dev_attr_led_sysfs.attr,
	NULL,
};

static const struct attribute_group dev_attr_grp = {
       .attrs = atk_imx6ul_led_sysfs_attrs,
};
sysfs_create_group(&led_device.device.kobj, &dev_attr_grp) //创建接口
sysfs_remove_group(&client->dev.kobj, &dev_attr_group); //接口移除，在调用remove函数时调用
```

### 四、验证

直接在终端下通过find命令查找结点，这里只是一个示例，并不是上面举的例子

![](https://raw.githubusercontent.com/mengchao666/picture/main/blog3281136fb4b8bf1fee3e436e03651d2a.png)

实验echo命令调用store函数

![](https://raw.githubusercontent.com/mengchao666/picture/main/blog903614bed87ac3a3d4925475013fb2ea.png)

这里cat没有写出，实际上直接将上述的echo 1或者echo 0 写成cat就okay了

### 五、device_create_file

可以在创建驱动时，在创建出来的设备目录下/sys/class/xxx/xxx，新建sysfs文件

```c
/**
 \* device_create_file - create sysfs attribute file for device.
 \* @dev: device.
 \* @attr: device attribute descriptor.
 */
int device_create_file(struct device *dev, const struct device_attribute *attr)
{
 int error = 0;
 if (dev)
 error = sysfs_create_file(&dev->kobj, &attr->attr);
 return erro

}
```