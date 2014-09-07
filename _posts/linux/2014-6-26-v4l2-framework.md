---
layout: post
title: "v4l2框架简介-V4L2-framework.txt"
date: 2014-6-26 10:50
category: "linux"
tags : "driver"
---


![V4L2.png](/picture/V4L2.png) 

##V4L2驱动框架概述

###介绍
![v4l2-intro.png](/picture/v4l2-intro.png) 

由于硬件的复杂性v412驱动往往是非常复杂的： 大多数设备有多个IC，在/dev目录下有多个设备节点, 并也创建non-V4L2的设备，如DVB，ALSA，FB，I2C和input（IR）设备。特别是v412驱动设置配套的IC做音频/视频多路复用/编码/解码，使得它更比大多数复杂的事实。通常这些芯片连接到主桥驱动器通过一个或多个I2C总线，但也可以使用其他总线。这种设备被称为“子设备”。很长一段时间有限制的video_device结构框架创建v4l设备节点和video_buf的视频缓冲区处理（请注意，本文档不讨论video_buf框架）。这意味着，所有驱动程序必须做的设备实例的设置和连接子设备本身。这部分是相当复杂，做应该做的事，很多驱动程序不这样做是正确的。 由于缺乏一个框架也有很多共同的代码不可重构。因此，这个框架的基本构建块，所有的驱动程序需要与此相同的框架应更容易进入所有驱动程序共享的实用功能重构的通用代码。

###驱动程序的结构

- 每个设备包含设备状态的实例结构
- 子设备的初始化和命令方式(如果有子设备的话)
- 创建v4l2的设备节点(/dev/videox,/dev/vbiX and /dev/radioX)
- 文件文件句柄的结构，包含每个文件句柄数据
- 视频缓冲处理

这是一个粗略的示意图，这一切是如何涉及的：

    device instances（设备实例）
      |
      +-sub-device instances（子设备实例）
      |
      \-V4L2 device nodes（V4L2的设备节点）
            |
            \-filehandle instances（文件句柄实例）

每个设备实例结构体v4l2_device（V4L2-device.h中）的代表。只是很简单的.设备可以分配这个结构，但大多数的时候你会把这个结构体嵌入到一个更大的结构体中。                              
你必须注册设备的实例：                  

	v4l2_device_register(struct device *dev, struct v4l2_device *v4l2_dev);

注册将初始化 v4l2_device 结构体. 如果 dev->driver_data字段是空, 它将连接到 v4l2_dev.
要与媒体设备框架集成的驱动程序需要设置DEV-> driver_data手动指向驱动程序特定的设备结构
嵌入结构体v4l2_device实例。这是通过一个dev_set_drvdata（）调用之前注册V4L2的设备实例。
他们必须还设置结构体v4l2_device MDEV领域指向一个正确初始化和注册media_device实例。
如果v4l2_dev->name是空，那么这将被设置为从dev取得一个值（驱动程序名称后的
 bus_id要准确）。 如果你在调用v4l2_device_register之前设置它，那么它没有改变。
如果dev是NULL，那么你‘必须’设置v4l2_dev>name在调用v4l2_device_register前。

你可以使用v4l2_device_set_name()设置名称根据驱动程序的名字和
driver-global atomic_t实例。这将产生的名字一样，ivtv0，ivtv1，等。
如果名字最后一个数字，然后将它插入一个破折号：cx18-0，cx18-1，等
这个函数返回的实例数量。

第一个参数‘dev’通常是一个pci_dev的struct device的指针，
但它是ISA设备或一个设备创建多个PCI设备时这是罕见的DEV为NULL，
因此makingit不可能联想到一个特定的父母v4l2_dev。
您也可以提供一个notify（）回调子设备，可以通过调用通知你的事件。
取决于你是否需要设置子设备。一个子设备支持的任何通知必须在头文件中定义                                       

include/media/<subdevice>.h.

注销：

	v4l2_device_unregister(struct v4l2_device *v4l2_dev);

如果dev-> driver_data字段指向v4l2_dev，它将被重置为NULL。
注销也将自动注销从设备所有子设备。

如果你有一个可热插拔设备（如USB设备），然后发生断开连接的时候父设备将变为无效。由于v4l2_device有指向父设备的指针，它被清除，以及标记父设备消失。要做到这一点调用：

	v4l2_device_disconnect(struct v4l2_device *v4l2_dev);

这并*不*注销subdevs，所以你仍然需要调用该的v4l2_device_unregister（）函数。如果你的驱动是不能热插拔，则有无需调用v4l2_device_disconnect（）。

有时你需要遍历一个特定的驱动程序注册的所有设备。
这是通常情况下如果多个设备驱动程序使用相同的硬件。
例如ivtvfb驱动程序是一个使用IVTV硬件framebuffer驱动。
同样是真实的，例如ALSA驱动程序。您可以遍历所有注册的设备如下:
{% highlight c %}

static int callback(struct device *dev, void *p)
{
    struct v4l2_device *v4l2_dev = dev_get_drvdata(dev);

    /* test if this device was inited */
    if (v4l2_dev == NULL)
        return 0;
    ...
    return 0;
}

int iterate(void *p)
{
    struct device_driver *drv;
    int err;

    /* Find driver 'ivtv' on the PCI bus.
       pci_bus_type is a global. For USB busses use usb_bus_type. */
    drv = driver_find("ivtv", &pci_bus_type);
    /* iterate over all ivtv device instances */
    err = driver_for_each_device(drv, NULL, p, callback);
    put_driver(drv);
    return err;
}

{% endhighlight %}
有时你需要保持一个设备实例运行计数器。这是常用的映射设备实例模块选项数组索引。
建议的方法如下：
{% highlight c %}

static atomic_t drv_instance = ATOMIC_INIT(0);

static int __devinit drv_probe(struct pci_dev *pdev,
                const struct pci_device_id *pci_id)
{
    ...
    state->instance = atomic_inc_return(&drv_instance) - 1;
}

{% endhighlight %}
如果你有多个设备节点然后它可以是很难知道它是安全注销v4l2_device时候。v4l2_device引用计数的支持就是对于这样做的目的。当video_register_device时被称之为增加引用计数，有设备节点被释放时减少引用计数。当引用计数达到零，则v4l2_device回调release()。你可以做最后的清理。如果其他设备节点（例如ALSA）的创建，然后你可以增加和减少以及手动调用引用计数：
       void v4l2_device_get(struct v4l2_device *v4l2_dev);
or:           
       int v4l2_device_put(struct v4l2_device *v4l2_dev);

###struct v4l2_subdev
