---

layout: post
title: "linux 的总线，设备，驱动模型"
date: 2014-9-7 9:34
category: "学习"
comments: false
tags : "driver"

---

##Docunment

linux有一种驱动模型由bus,device,driver三者构成。device是挂载在bus上的，而不同的device能在挂载的时候找到属于自己的driver.这三者的定义是怎么样的，而他们又是如何进行绑定的？下面从内核的Doucment/driver-model目录寻找答案。
###bus.txt

```

Bus Types 

Definition
~~~~~~~~~~
See the kerneldoc for the struct bus_type.(= = ,居然要自己找！！)

struct bus_type {
 const char  * name;//设备名称

 struct subsystem subsys;//代表自身
 struct kset  drivers;   //当前总线的设备驱动集合
 struct kset  devices; //所有设备集合
 struct klist  klist_devices;
 struct klist  klist_drivers;

 struct bus_attribute * bus_attrs;//总线属性
 struct device_attribute * dev_attrs;//设备属性
 struct driver_attribute * drv_attrs;

 int  (*match)(struct device * dev, struct device_driver * drv);//设备驱动匹配函数
 int  (*uevent)(struct device *dev, char **envp,  
      int num_envp, char *buffer, int buffer_size);//热拔插事件
 int  (*probe)(struct device * dev);
 int  (*remove)(struct device * dev);
 void  (*shutdown)(struct device * dev);
 int  (*suspend)(struct device * dev, pm_message_t state);
 int  (*resume)(struct device * dev);
};

int bus_register(struct bus_type * bus);


Declaration
~~~~~~~~~~~

Each bus type in the kernel (PCI, USB, etc) should declare one static
object of this type. They must initialize the name field, and may
optionally initialize the match callback.

每个bus类型在内核如(PCI,USB等)都应该只有一个对象声明，他们必须初始化name和自定义回调函数.match。
struct bus_type pci_bus_type = {
       .name	= "pci",
       .match	= pci_bus_match,
};

The structure should be exported to drivers in a header file:
这个结构体应该在drivers的头文件用extern声明。

extern struct bus_type pci_bus_type;


Registration
~~~~~~~~~~~~

When a bus driver is initialized, it calls bus_register. This
initializes the rest of the fields in the bus object and inserts it
into a global list of bus types. Once the bus object is registered, 
the fields in it are usable by the bus driver. 


Callbacks
~~~~~~~~~

match(): Attaching Drivers to Devices
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

The format of device ID structures and the semantics for comparing
them are inherently bus-specific. Drivers typically declare an array
of device IDs of devices they support that reside in a bus-specific
driver structure. 

通过device的ID和总线匹配，而drivers则定义了属于某个总线的device 的device IDS字符串。

The purpose of the match callback is provide the bus an opportunity to
determine if a particular driver supports a particular device by
comparing the device IDs the driver supports with the device ID of a
particular device, without sacrificing bus-specific functionality or
type-safety. 

match回调的目的是提供bus一个可以确定是否有专门的driver可以支持专门的device,具体是通过比较他们的device IDS。

When a driver is registered with the bus, the bus's list of devices is
iterated over, and the match callback is called for each device that
does not have a driver associated with it. 

当一个driver被注册到bus，bus的会扫描他的设备列表，然后match回调函数会被调用，去给那些还没有driver的device配置driver。


Device and Driver Lists
~~~~~~~~~~~~~~~~~~~~~~~

The lists of devices and drivers are intended to replace the local
lists that many buses keep. They are lists of struct devices and
struct device_drivers, respectively. Bus drivers are free to use the
lists as they please, but conversion to the bus-specific type may be
necessary. 

The LDM core provides helper functions for iterating over each list.

可以用下面的函数进行device和driver的遍历

int bus_for_each_dev(struct bus_type * bus, struct device * start, void * data,
		     int (*fn)(struct device *, void *));

int bus_for_each_drv(struct bus_type * bus, struct device_driver * start, 
		     void * data, int (*fn)(struct device_driver *, void *));

These helpers iterate over the respective list, and call the callback
for each device or driver in the list. All list accesses are
synchronized by taking the bus's lock (read currently). The reference
count on each object in the list is incremented before the callback is
called; it is decremented after the next object has been obtained. The
lock is not held when calling the callback. 


sysfs
~~~~~~~~
There is a top-level directory named 'bus'.

Each bus gets a directory in the bus directory, along with two default
directories:
bus驱动在sysfs文件系统的显示如下:

	/sys/bus/pci/
	|-- devices
	`-- drivers

Drivers registered with the bus get a directory in the bus's drivers
directory:
drivers会有自己的目录

	/sys/bus/pci/
	|-- devices
	`-- drivers
	    |-- Intel ICH
	    |-- Intel ICH Joystick
	    |-- agpgart
	    `-- e100

Each device that is discovered on a bus of that type gets a symlink in
the bus's devices directory to the device's directory in the physical
hierarchy:

每个device被bus发现的device都有一个链接符号，这link指出了device的物理层次结构。

	/sys/bus/pci/
	|-- devices
	|   |-- 00:00.0 -> ../../../root/pci0/00:00.0
	|   |-- 00:01.0 -> ../../../root/pci0/00:01.0
	|   `-- 00:02.0 -> ../../../root/pci0/00:02.0
	`-- drivers


Exporting Attributes
~~~~~~~~~~~~~~~~~~~~
struct bus_attribute {
	struct attribute	attr;
	ssize_t (*show)(struct bus_type *, char * buf);
	ssize_t (*store)(struct bus_type *, const char * buf, size_t count);
};

Bus drivers can export attributes using the BUS_ATTR macro that works
similarly to the DEVICE_ATTR macro for devices. For example, a definition 
like this:

static BUS_ATTR(debug,0644,show_debug,store_debug);

is equivalent to declaring:

static bus_attribute bus_attr_debug;

This can then be used to add and remove the attribute from the bus's
sysfs directory using:

int bus_create_file(struct bus_type *, struct bus_attribute *);
void bus_remove_file(struct bus_type *, struct bus_attribute *);

```

###device.txt

```

The Basic Device Structure
~~~~~~~~~~~~~~~~~~~~~~~~~~

See the kerneldoc for the struct device.

Programming Interface
~~~~~~~~~~~~~~~~~~~~~
The bus driver that discovers the device uses this to register the
device with the core:

int device_register(struct device * dev);

The bus should initialize the following fields:

    - parent
    - name
    - bus_id
    - bus

A device is removed from the core when its reference count goes to
0. The reference count can be adjusted using:

struct device * get_device(struct device * dev);
void put_device(struct device * dev);

get_device() will return a pointer to the struct device passed to it
if the reference is not already 0 (if it's in the process of being
removed already).

A driver can access the lock in the device structure using: 

void lock_device(struct device * dev);
void unlock_device(struct device * dev);


Attributes
~~~~~~~~~~
struct device_attribute {
	struct attribute	attr;
	ssize_t (*show)(struct device *dev, struct device_attribute *attr,
			char *buf);
	ssize_t (*store)(struct device *dev, struct device_attribute *attr,
			 const char *buf, size_t count);
};

Attributes of devices can be exported by a device driver through sysfs.

Please see Documentation/filesystems/sysfs.txt for more information
on how sysfs works.

As explained in Documentation/kobject.txt, device attributes must be be
created before the KOBJ_ADD uevent is generated. The only way to realize
that is by defining an attribute group.

Attributes are declared using a macro called DEVICE_ATTR:

#define DEVICE_ATTR(name,mode,show,store)

Example:

static DEVICE_ATTR(type, 0444, show_type, NULL);
static DEVICE_ATTR(power, 0644, show_power, store_power);

This declares two structures of type struct device_attribute with respective
names 'dev_attr_type' and 'dev_attr_power'. These two attributes can be
organized as follows into a group:

static struct attribute *dev_attrs[] = {
	&dev_attr_type.attr,
	&dev_attr_power.attr,
	NULL,
};

static struct attribute_group dev_attr_group = {
	.attrs = dev_attrs,
};

static const struct attribute_group *dev_attr_groups[] = {
	&dev_attr_group,
	NULL,
};

This array of groups can then be associated with a device by setting the
group pointer in struct device before device_register() is invoked:

      dev->groups = dev_attr_groups;
      device_register(dev);

The device_register() function will use the 'groups' pointer to create the
device attributes and the device_unregister() function will use this pointer
to remove the device attributes.

Word of warning:  While the kernel allows device_create_file() and
device_remove_file() to be called on a device at any time, userspace has
strict expectations on when attributes get created.  When a new device is
registered in the kernel, a uevent is generated to notify userspace (like
udev) that a new device is available.  If attributes are added after the
device is registered, then userspace won't get notified and userspace will
not know about the new attributes.

This is important for device driver that need to publish additional
attributes for a device at driver probe time.  If the device driver simply
calls device_create_file() on the device structure passed to it, then
userspace will never be notified of the new attributes.

```
###driver.txt

这个我们已经很熟悉了，就不看了。

###三者是如何绑定，匹配的？

```

Driver Binding

Driver binding is the process of associating a device with a device
driver that can control it. Bus drivers have typically handled this
because there have been bus-specific structures to represent the
devices and the drivers. With generic device and device driver
structures, most of the binding can take place using common code.


Bus
~~~

The bus type structure contains a list of all devices that are on that bus
type in the system. When device_register is called for a device, it is
inserted into the end of this list. The bus object also contains a
list of all drivers of that bus type. When driver_register is called
for a driver, it is inserted at the end of this list. These are the
two events which trigger driver binding.

bus 结构体包含了一个所以device的列表。当device_register被device注册调用时，这个设备将会被插入到这个列表的尾部。bus也同时包含了一个drivers的列表，当driver_register被调用时，这个driver会被插入到列表的尾部。这就是实现绑定的tigger.


device_register
~~~~~~~~~~~~~~~

When a new device is added, the bus's list of drivers is iterated over
to find one that supports it. In order to determine that, the device
ID of the device must match one of the device IDs that the driver
supports. The format and semantics for comparing IDs is bus-specific. 
Instead of trying to derive a complex state machine and matching
algorithm, it is up to the bus driver to provide a callback to compare
a device against the IDs of a driver. The bus returns 1 if a match was
found; 0 otherwise.

当一个新的device被添加时，bus将会遍历drivers列表去寻找支持这个device的driver。为了能够确定他，device的device ID必须和driver的支持的一个device IDs匹配。匹配IDs的格式和语义就是bus-specific。当bus的驱动程序通过match匹配到时，则会返回1.

int match(struct device * dev, struct device_driver * drv);

If a match is found, the device's driver field is set to the driver
and the driver's probe callback is called. This gives the driver a
chance to verify that it really does support the hardware, and that
it's in a working state. 

如果一个match被找到，那么deivce的driver被锁定，并且driver的probe回调函数将会被调用。这给了driver一个可以检查它是否支持这个硬件设备的机会。

Device Class
~~~~~~~~~~~~

Upon the successful completion of probe, the device is registered with
the class to which it belongs. Device drivers belong to one and only one
class, and that is set in the driver's devclass field. 
devclass_add_device is called to enumerate the device within the class
and actually register it with the class, which happens with the
class's register_dev callback.


Driver
~~~~~~

When a driver is attached to a device, the device is inserted into the
driver's list of devices. 


sysfs
~~~~~

A symlink is created in the bus's 'devices' directory that points to
the device's directory in the physical hierarchy.

A symlink is created in the driver's 'devices' directory that points
to the device's directory in the physical hierarchy.

A directory for the device is created in the class's directory. A
symlink is created in that directory that points to the device's
physical location in the sysfs tree. 

A symlink can be created (though this isn't done yet) in the device's
physical directory to either its class directory, or the class's
top-level directory. One can also be created to point to its driver's
directory also. 


driver_register
~~~~~~~~~~~~~~~

The process is almost identical for when a new driver is added. 
The bus's list of devices is iterated over to find a match. Devices
that already have a driver are skipped. All the devices are iterated
over, to bind as many devices as possible to the driver.


Removal
~~~~~~~

When a device is removed, the reference count for it will eventually
go to 0. When it does, the remove callback of the driver is called. It
is removed from the driver's list of devices and the reference count
of the driver is decremented. All symlinks between the two are removed.

When a driver is removed, the list of devices that it supports is
iterated over, and the driver's remove callback is called for each
one. The device is removed from that list and the symlinks removed. 


```

##翠花，上代码

下面来个简单的例子看看。

`mybus.c`

{% highlight c %}

#include <linux/device.h>
#include <linux/module.h>
#include <linux/kernel.h>
#include <linux/init.h>
#include <linux/string.h>

MODULE_AUTHOR("David Xie");
MODULE_LICENSE("Dual BSD/GPL");

static char *Version = "$Revision: 1.9 $";

static int my_match(struct device *dev, struct device_driver *driver)
{
 return !strncmp(dev->init_name, driver->name, strlen(driver->name));
}

static void my_bus_release(struct device *dev)
{
 printk(KERN_DEBUG "my bus release\n");
}
 
struct device my_bus = {
 .init_name = "my_bus0",
 .release  = my_bus_release,
};


struct bus_type my_bus_type = {
 .name = "my_bus",
 .match = my_match,
};

EXPORT_SYMBOL(my_bus);
EXPORT_SYMBOL(my_bus_type);


/*
 * Export a simple attribute.
 */
static ssize_t show_bus_version(struct bus_type *bus, char *buf)
{
 return snprintf(buf, PAGE_SIZE, "%s\n", Version);
}

static BUS_ATTR(version, S_IRUGO, show_bus_version, NULL);


static int __init my_bus_init(void)
{
 int ret;
       
        /*注册总线*/
 ret = bus_register(&my_bus_type);
 if (ret)
  return ret;
  
 /*创建属性文件*/ 
 if (bus_create_file(&my_bus_type, &bus_attr_version))
  printk(KERN_NOTICE "Fail to create version attribute!\n");
 
 /*注册总线设备*/
 ret = device_register(&my_bus);
 if (ret)
  printk(KERN_NOTICE "Fail to register device:my_bus!\n");
  
 return ret;
}

static void my_bus_exit(void)
{
 device_unregister(&my_bus);
 bus_unregister(&my_bus_type);
}

module_init(my_bus_init);
module_exit(my_bus_exit);


{% endhighlight %}

`mydev.c`

{% highlight c %}
#include <linux/device.h>
#include <linux/module.h>
#include <linux/kernel.h>
#include <linux/init.h>
#include <linux/string.h>

MODULE_AUTHOR("David Xie");
MODULE_LICENSE("Dual BSD/GPL");

extern struct device my_bus;
extern struct bus_type my_bus_type;

/* Why need this ?*/
static void my_dev_release(struct device *dev)
{
 
}

struct device my_dev = {
 .bus = &my_bus_type,//与总线接上关系
 .parent = &my_bus,//与总线设备接上关系
 .release = my_dev_release,
};

/*
 * Export a simple attribute.
 */
static ssize_t mydev_show(struct device *dev,struct device_attribute *attr, char *buf)
{
 return sprintf(buf, "%s\n", "This is my device!");
}

static DEVICE_ATTR(dev, S_IRUGO, mydev_show, NULL);

static int __init my_device_init(void)
{
 int ret = 0;
       
        /* 初始化设备 */
 dev_set_name(&my_dev,"my_dev"); 
        /*注册设备*/
 device_register(&my_dev);
  
 /*创建属性文件*/
 device_create_file(&my_dev, &dev_attr_dev);
 
 return ret; 

}

static void my_device_exit(void)
{
 device_unregister(&my_dev);
}

module_init(my_device_init);
module_exit(my_device_exit);
{% endhighlight %}


`mydrv.c`

{% highlight c %}
#include <linux/device.h>
#include <linux/module.h>
#include <linux/kernel.h>
#include <linux/init.h>
#include <linux/string.h>

MODULE_AUTHOR("David Xie");
MODULE_LICENSE("Dual BSD/GPL");

extern struct bus_type my_bus_type;

static int my_probe(struct device *dev)
{
    printk("Driver found device which my driver can handle!\n");
    return 0;
}

static int my_remove(struct device *dev)
{
    printk("Driver found device unpluged!\n");
    return 0;
}

struct device_driver my_driver = {
 .name = "my_dev",//对应的设备名称
 .bus = &my_bus_type,//挂载的总线
 .probe = my_probe,//这个函数就是在找到与自己对应的设备时被调用。
 .remove = my_remove,
};

/*
 * Export a simple attribute.
 */
static ssize_t mydriver_show(struct device_driver *driver, char *buf)
{
 return sprintf(buf, "%s\n", "This is my driver!");
}

static DRIVER_ATTR(drv, S_IRUGO, mydriver_show, NULL);

static int __init my_driver_init(void)
{
 int ret = 0;
       
        /*注册驱动*/
 driver_register(&my_driver);
  
 /*创建属性文件*/
 driver_create_file(&my_driver, &driver_attr_drv);
 
 return ret; 

}

static void my_driver_exit(void)
{
 driver_unregister(&my_driver);
}

module_init(my_driver_init);
module_exit(my_driver_exit);

{% endhighlight %}

结果如下:

```
devices  drivers  drivers_autoprobe  drivers_probe  uevent  version
root@vm:/sys/bus/my_bus# tree
.
├── devices
│   └── my_dev -> ../../../devices/my_bus0/my_dev
├── drivers
│   └── my_dev
├── drivers_autoprobe
├── drivers_probe
├── uevent
└── version

4 directories, 4 files
root@vm:/sys/bus/my_bus# 

```
