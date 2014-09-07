---

layout: post
title: "linux 的sysfs系统和udev系统"
date: 2014-9-5 19:34
category: "学习"
comments: false
tags : "driver"

---

##sysfs系统
###内核文档的描述

```

Chinese translated version of Documentation/filesystems/sysfs.txt

If you have any comment or update to the content, please contact the
original document maintainer directly.  However, if you have a problem
communicating in English you can also ask the Chinese maintainer for
help.  Contact the Chinese maintainer if this translation is outdated
or if there is a problem with the translation.

Maintainer: Patrick Mochel	<mochel@osdl.org>
		Mike Murphy <mamurph@cs.clemson.edu>
Chinese maintainer: Fu Wei <tekkamanninja@gmail.com>
---------------------------------------------------------------------
Documentation/filesystems/sysfs.txt 的中文翻译

如果想评论或更新本文的内容，请直接联系原文档的维护者。如果你使用英文
交流有困难的话，也可以向中文版维护者求助。如果本翻译更新不及时或者翻
译存在问题，请联系中文版维护者。
英文版维护者： Patrick Mochel	<mochel@osdl.org>
		Mike Murphy <mamurph@cs.clemson.edu>
中文版维护者： 傅炜 Fu Wei <tekkamanninja@gmail.com>
中文版翻译者： 傅炜 Fu Wei <tekkamanninja@gmail.com>
中文版校译者： 傅炜 Fu Wei <tekkamanninja@gmail.com>


以下为正文
---------------------------------------------------------------------
sysfs - 用于导出内核对象(kobject)的文件系统

Patrick Mochel	<mochel@osdl.org>
Mike Murphy <mamurph@cs.clemson.edu>

修订:    16 August 2011
原始版本:   10 January 2003


sysfs 简介:
~~~~~~~~~~

sysfs 是一个最初基于 ramfs 且位于内存的文件系统。它提供导出内核
数据结构及其属性，以及它们之间的关联到用户空间的方法。

sysfs 始终与 kobject 的底层结构紧密相关。请阅读
Documentation/kobject.txt 文档以获得更多关于 kobject 接口的
信息。


使用 sysfs
~~~~~~~~~~~

只要内核配置中定义了 CONFIG_SYSFS ，sysfs 总是被编译进内核。你可
通过以下命令挂载它:

    mount -t sysfs sysfs /sys


创建目录
~~~~~~~~

任何 kobject 在系统中注册，就会有一个目录在 sysfs 中被创建。这个
目录是作为该 kobject 的父对象所在目录的子目录创建的，以准确地传递
内核的对象层次到用户空间。sysfs 中的顶层目录代表着内核对象层次的
共同祖先；例如：某些对象属于某个子系统。

Sysfs 在与其目录关联的 sysfs_dirent 对象中内部保存一个指向实现
目录的 kobject 的指针。以前，这个 kobject 指针被 sysfs 直接用于
kobject 文件打开和关闭的引用计数。而现在的 sysfs 实现中，kobject
引用计数只能通过 sysfs_schedule_callback() 函数直接修改。


属性
~~~~

kobject 的属性可在文件系统中以普通文件的形式导出。Sysfs 为属性定义
了面向文件 I/O 操作的方法，以提供对内核属性的读写。


属性应为 ASCII 码文本文件。以一个文件只存储一个属性值为宜。但一个
文件只包含一个属性值可能影响效率，所以一个包含相同数据类型的属性值
数组也被广泛地接受。

混合类型、表达多行数据以及一些怪异的数据格式会遭到强烈反对。这样做是
很丢脸的,而且其代码会在未通知作者的情况下被重写。


一个简单的属性结构定义如下:

struct attribute {
        char                    * name;
        struct module		*owner;
        umode_t                 mode;
};


int sysfs_create_file(struct kobject * kobj, const struct attribute * attr);
void sysfs_remove_file(struct kobject * kobj, const struct attribute * attr);


一个单独的属性结构并不包含读写其属性值的方法。子系统最好为增删特定
对象类型的属性定义自己的属性结构体和封装函数。

例如:驱动程序模型定义的 device_attribute 结构体如下:

struct device_attribute {
	struct attribute	attr;
	ssize_t (*show)(struct device *dev, struct device_attribute *attr,
			char *buf);
	ssize_t (*store)(struct device *dev, struct device_attribute *attr,
			 const char *buf, size_t count);
};

int device_create_file(struct device *, const struct device_attribute *);
void device_remove_file(struct device *, const struct device_attribute *);

为了定义设备属性，同时定义了一下辅助宏:

#define DEVICE_ATTR(_name, _mode, _show, _store) \
struct device_attribute dev_attr_##_name = __ATTR(_name, _mode, _show, _store)

例如:声明

static DEVICE_ATTR(foo, S_IWUSR | S_IRUGO, show_foo, store_foo);

等同于如下代码：

static struct device_attribute dev_attr_foo = {
       .attr	= {
		.name = "foo",
		.mode = S_IWUSR | S_IRUGO,
		.show = show_foo,
		.store = store_foo,
	},
};


子系统特有的回调函数
~~~~~~~~~~~~~~~~~~~

当一个子系统定义一个新的属性类型时，必须实现一系列的 sysfs 操作，
以帮助读写调用实现属性所有者的显示和储存方法。

struct sysfs_ops {
        ssize_t (*show)(struct kobject *, struct attribute *, char *);
        ssize_t (*store)(struct kobject *, struct attribute *, const char *, size_t);
};

[子系统应已经定义了一个 struct kobj_type 结构体作为这个类型的
描述符，并在此保存 sysfs_ops 的指针。更多的信息参见 kobject 的
文档]

sysfs 会为这个类型调用适当的方法。当一个文件被读写时，这个方法会
将一般的kobject 和 attribute 结构体指针转换为适当的指针类型后
调用相关联的函数。


示例:

#define to_dev(obj) container_of(obj, struct device, kobj)
#define to_dev_attr(_attr) container_of(_attr, struct device_attribute, attr)

static ssize_t dev_attr_show(struct kobject *kobj, struct attribute *attr,
                             char *buf)
{
        struct device_attribute *dev_attr = to_dev_attr(attr);
        struct device *dev = to_dev(kobj);
        ssize_t ret = -EIO;

        if (dev_attr->show)
                ret = dev_attr->show(dev, dev_attr, buf);
        if (ret >= (ssize_t)PAGE_SIZE) {
                print_symbol("dev_attr_show: %s returned bad count\n",
                                (unsigned long)dev_attr->show);
        }
        return ret;
}



读写属性数据
~~~~~~~~~~~~

在声明属性时，必须指定 show() 或 store() 方法，以实现属性的
读或写。这些方法的类型应该和以下的设备属性定义一样简单。

ssize_t (*show)(struct device *dev, struct device_attribute *attr, char *buf);
ssize_t (*store)(struct device *dev, struct device_attribute *attr,
                 const char *buf, size_t count);

也就是说,他们应只以一个处理对象、一个属性和一个缓冲指针作为参数。

sysfs 会分配一个大小为 (PAGE_SIZE) 的缓冲区并传递给这个方法。
Sysfs 将会为每次读写操作调用一次这个方法。这使得这些方法在执行时
会出现以下的行为:

- 在读方面（read(2)），show() 方法应该填充整个缓冲区。回想属性
  应只导出了一个属性值或是一个同类型属性值的数组，所以这个代价将
  不会不太高。

  这使得用户空间可以局部地读和任意的向前搜索整个文件。如果用户空间
  向后搜索到零或使用‘0’偏移执行一个pread(2)操作，show()方法将
  再次被调用，以重新填充缓存。

- 在写方面（write(2)），sysfs 希望在第一次写操作时得到整个缓冲区。
  之后 Sysfs 传递整个缓冲区给 store() 方法。

  当要写 sysfs 文件时，用户空间进程应首先读取整个文件，修该想要
  改变的值，然后回写整个缓冲区。

  在读写属性值时，属性方法的执行应操作相同的缓冲区。

注记:

- 写操作导致的 show() 方法重载，会忽略当前文件位置。

- 缓冲区应总是 PAGE_SIZE 大小。对于i386，这个值为4096。

- show() 方法应该返回写入缓冲区的字节数，也就是 snprintf()的
  返回值。

- show() 应始终使用 snprintf()。

- store() 应返回缓冲区的已用字节数。如果整个缓存都已填满，只需返回
  count 参数。

- show() 或 store() 可以返回错误值。当得到一个非法值，必须返回一个
  错误值。

- 一个传递给方法的对象将会通过 sysfs 调用对象内嵌的引用计数固定在
  内存中。尽管如此，对象代表的物理实体(如设备)可能已不存在。如有必要，
  应该实现一个检测机制。

一个简单的(未经实验证实的)设备属性实现如下：

static ssize_t show_name(struct device *dev, struct device_attribute *attr,
                         char *buf)
{
	return scnprintf(buf, PAGE_SIZE, "%s\n", dev->name);
}

static ssize_t store_name(struct device *dev, struct device_attribute *attr,
                          const char *buf, size_t count)
{
        snprintf(dev->name, sizeof(dev->name), "%.*s",
                 (int)min(count, sizeof(dev->name) - 1), buf);
	return count;
}

static DEVICE_ATTR(name, S_IRUGO, show_name, store_name);


（注意：真正的实现不允许用户空间设置设备名。）

顶层目录布局
~~~~~~~~~~~~

sysfs 目录的安排显示了内核数据结构之间的关系。

顶层 sysfs 目录如下:

block/
bus/
class/
dev/
devices/
firmware/
net/
fs/

devices/ 包含了一个设备树的文件系统表示。他直接映射了内部的内核
设备树，反映了设备的层次结构。

bus/ 包含了内核中各种总线类型的平面目录布局。每个总线目录包含两个
子目录:

	devices/
	drivers/

devices/ 包含了系统中出现的每个设备的符号链接，他们指向 root/ 下的
设备目录。

drivers/ 包含了每个已为特定总线上的设备而挂载的驱动程序的目录(这里
假定驱动没有跨越多个总线类型)。

fs/ 包含了一个为文件系统设立的目录。现在每个想要导出属性的文件系统必须
在 fs/ 下创建自己的层次结构(参见Documentation/filesystems/fuse.txt)。

dev/ 包含两个子目录： char/ 和 block/。在这两个子目录中，有以
<major>:<minor> 格式命名的符号链接。这些符号链接指向 sysfs 目录
中相应的设备。/sys/dev 提供一个通过一个 stat(2) 操作结果，查找
设备 sysfs 接口快捷的方法。

更多有关 driver-model 的特性信息可以在 Documentation/driver-model/
中找到。


TODO: 完成这一节。


当前接口
~~~~~~~~

以下的接口层普遍存在于当前的sysfs中:

- 设备 (include/linux/device.h)
----------------------------------
结构体:

struct device_attribute {
	struct attribute	attr;
	ssize_t (*show)(struct device *dev, struct device_attribute *attr,
			char *buf);
	ssize_t (*store)(struct device *dev, struct device_attribute *attr,
			 const char *buf, size_t count);
};

声明:

DEVICE_ATTR(_name, _mode, _show, _store);

增/删属性:

int device_create_file(struct device *dev, const struct device_attribute * attr);
void device_remove_file(struct device *dev, const struct device_attribute * attr);


- 总线驱动程序 (include/linux/device.h)
--------------------------------------
结构体:

struct bus_attribute {
        struct attribute        attr;
        ssize_t (*show)(struct bus_type *, char * buf);
        ssize_t (*store)(struct bus_type *, const char * buf, size_t count);
};

声明:

BUS_ATTR(_name, _mode, _show, _store)

增/删属性:

int bus_create_file(struct bus_type *, struct bus_attribute *);
void bus_remove_file(struct bus_type *, struct bus_attribute *);


- 设备驱动程序 (include/linux/device.h)
-----------------------------------------

结构体:

struct driver_attribute {
        struct attribute        attr;
        ssize_t (*show)(struct device_driver *, char * buf);
        ssize_t (*store)(struct device_driver *, const char * buf,
                         size_t count);
};

声明:

DRIVER_ATTR(_name, _mode, _show, _store)

增/删属性：

int driver_create_file(struct device_driver *, const struct driver_attribute *);
void driver_remove_file(struct device_driver *, const struct driver_attribute *);


文档
~~~~

sysfs 目录结构以及其中包含的属性定义了一个内核与用户空间之间的 ABI。
对于任何 ABI，其自身的稳定和适当的文档是非常重要的。所有新的 sysfs
属性必须在 Documentation/ABI 中有文档。详见 Documentation/ABI/README。

```

sysfs与kobject的关系是非常紧密的，一般，一个kobject对象将会在sysfs系统中建立一个文件来记录kobject对象的属性。

关于这个kobject结构体，我们再看一下文档的描述：

```

Everything you never wanted to know about kobjects, ksets, and ktypes
这里有你想要的关于kobjects, ksets, and ktypes的一切

Greg Kroah-Hartman <gregkh@linuxfoundation.org>

Based on an original article by Jon Corbet for lwn.net written October 1,
2003 and located at http://lwn.net/Articles/51437/

Last updated December 19, 2007


Part of the difficulty in understanding the driver model - and the kobject
abstraction upon which it is built - is that there is no obvious starting
place. Dealing with kobjects requires understanding a few different types,
all of which make reference to each other. In an attempt to make things
easier, we'll take a multi-pass approach, starting with vague terms and
adding detail as we go. To that end, here are some quick definitions of
some terms we will be working with.

下面是kobject,ktype,kset的定义：

 - A kobject is an object of type struct kobject.  Kobjects have a name
   and a reference count.  A kobject also has a parent pointer (allowing
   objects to be arranged into hierarchies), a specific type, and,
   usually, a representation in the sysfs virtual filesystem.

   Kobjects are generally not interesting on their own; instead, they are
   usually embedded within some other structure which contains the stuff
   the code is really interested in.

   Kobjects通常是被嵌套在其它结构体中的

   No structure should EVER have more than one kobject embedded within it.
   If it does, the reference counting for the object is sure to be messed
   up and incorrect, and your code will be buggy.  So do not do this.
   一个结构体只能嵌套一个kobject结构体。

 - A ktype is the type of object that embeds a kobject.  Every structure
   that embeds a kobject needs a corresponding ktype.  The ktype controls
   what happens to the kobject when it is created and destroyed.
   ktype是嵌套在kobject结构体中的一个结构体，每个嵌套kobject结构体都需要一个正确的ktype。ktype控制着kobject注册和摧毁。

 - A kset is a group of kobjects.  These kobjects can be of the same ktype
   or belong to different ktypes.  The kset is the basic container type for
   collections of kobjects. Ksets contain their own kobjects, but you can
   safely ignore that implementation detail as the kset core code handles
   this kobject automatically.
   kset是一个kobjects的集会。但这些kobjects可以有相同或不同的ktype。

   When you see a sysfs directory full of other directories, generally each
   of those directories corresponds to a kobject in the same kset.

We'll look at how to create and manipulate all of these types. A bottom-up
approach will be taken, so we'll go back to kobjects.


Embedding kobjects

It is rare for kernel code to create a standalone kobject, with one major
exception explained below.  Instead, kobjects are used to control access to
a larger, domain-specific object.  To this end, kobjects will be found
embedded in other structures.  If you are used to thinking of things in
object-oriented terms, kobjects can be seen as a top-level, abstract class
from which other classes are derived.  A kobject implements a set of
capabilities which are not particularly useful by themselves, but which are
nice to have in other objects.  The C language does not allow for the
direct expression of inheritance, so other techniques - such as structure
embedding - must be used.

(As an aside, for those familiar with the kernel linked list implementation,
this is analogous as to how "list_head" structs are rarely useful on
their own, but are invariably found embedded in the larger objects of
interest.)

So, for example, the UIO code in drivers/uio/uio.c has a structure that
defines the memory region associated with a uio device:

    struct uio_map {
	struct kobject kobj;
	struct uio_mem *mem;
    };

If you have a struct uio_map structure, finding its embedded kobject is
just a matter of using the kobj member.  Code that works with kobjects will
often have the opposite problem, however: given a struct kobject pointer,
what is the pointer to the containing structure?  You must avoid tricks
(such as assuming that the kobject is at the beginning of the structure)
and, instead, use the container_of() macro, found in <linux/kernel.h>:

可以通过使用container_of寻找嵌入在结构体的变量所在的父结构体。
    container_of(pointer, type, member)

where:

  * "pointer" is the pointer to the embedded kobject,
  * "type" is the type of the containing structure, and
  * "member" is the name of the structure field to which "pointer" points.

The return value from container_of() is a pointer to the corresponding
container type. So, for example, a pointer "kp" to a struct kobject
embedded *within* a struct uio_map could be converted to a pointer to the
*containing* uio_map structure with:

    struct uio_map *u_map = container_of(kp, struct uio_map, kobj);

For convenience, programmers often define a simple macro for "back-casting"
kobject pointers to the containing type.  Exactly this happens in the
earlier drivers/uio/uio.c, as you can see here:

    struct uio_map {
        struct kobject kobj;
        struct uio_mem *mem;
    };

    #define to_map(map) container_of(map, struct uio_map, kobj)

where the macro argument "map" is a pointer to the struct kobject in
question.  That macro is subsequently invoked with:

    struct uio_map *map = to_map(kobj);


Initialization of kobjects

Code which creates a kobject must, of course, initialize that object. Some
of the internal fields are setup with a (mandatory) call to kobject_init():

    void kobject_init(struct kobject *kobj, struct kobj_type *ktype);

初始化kobject的同时也要初始化kobj_type。

The ktype is required for a kobject to be created properly, as every kobject
must have an associated kobj_type.  After calling kobject_init(), to
register the kobject with sysfs, the function kobject_add() must be called:

    int kobject_add(struct kobject *kobj, struct kobject *parent, const char *fmt, ...);

This sets up the parent of the kobject and the name for the kobject
properly.  If the kobject is to be associated with a specific kset,
kobj->kset must be assigned before calling kobject_add().  If a kset is
associated with a kobject, then the parent for the kobject can be set to
NULL in the call to kobject_add() and then the kobject's parent will be the
kset itself.kobject
设置kobjec父母和他的名字。kobj->kset 必须要在kobject_add()之前被注册。

As the name of the kobject is set when it is added to the kernel, the name
of the kobject should never be manipulated directly.  If you must change
the name of the kobject, call kobject_rename():

    int kobject_rename(struct kobject *kobj, const char *new_name);

kobject_rename does not perform any locking or have a solid notion of
what names are valid so the caller must provide their own sanity checking
and serialization.

There is a function called kobject_set_name() but that is legacy cruft and
is being removed.  If your code needs to call this function, it is
incorrect and needs to be fixed.

To properly access the name of the kobject, use the function
kobject_name():

    const char *kobject_name(const struct kobject * kobj);

There is a helper function to both initialize and add the kobject to the
kernel at the same time, called surprisingly enough kobject_init_and_add():

    int kobject_init_and_add(struct kobject *kobj, struct kobj_type *ktype,
                             struct kobject *parent, const char *fmt, ...);

The arguments are the same as the individual kobject_init() and
kobject_add() functions described above.


Uevents

After a kobject has been registered with the kobject core, you need to
announce to the world that it has been created.  This can be done with a
call to kobject_uevent():
kobject被注册到内核中后，我们需要通过kobject_uevent来向系统通知我们的已经被创建了。


    int kobject_uevent(struct kobject *kobj, enum kobject_action action);

Use the KOBJ_ADD action for when the kobject is first added to the kernel.
This should be done only after any attributes or children of the kobject
have been initialized properly, as userspace will instantly start to look
for them when this call happens.

When the kobject is removed from the kernel (details on how to do that is
below), the uevent for KOBJ_REMOVE will be automatically created by the
kobject core, so the caller does not have to worry about doing that by
hand.


Reference counts

One of the key functions of a kobject is to serve as a reference counter
for the object in which it is embedded. As long as references to the object
exist, the object (and the code which supports it) must continue to exist.
The low-level functions for manipulating a kobject's reference counts are:
 
“reference counter”，kobject的引用指针，当有引用时，kobject就必须保持存在。

    struct kobject *kobject_get(struct kobject *kobj);  //增加1
    void kobject_put(struct kobject *kobj);             //减少1

A successful call to kobject_get() will increment the kobject's reference
counter and return the pointer to the kobject.

When a reference is released, the call to kobject_put() will decrement the
reference count and, possibly, free the object. Note that kobject_init()
sets the reference count to one, so the code which sets up the kobject will
need to do a kobject_put() eventually to release that reference.

Because kobjects are dynamic, they must not be declared statically or on
the stack, but instead, always allocated dynamically.  Future versions of
the kernel will contain a run-time check for kobjects that are created
statically and will warn the developer of this improper usage.

If all that you want to use a kobject for is to provide a reference counter
for your structure, please use the struct kref instead; a kobject would be
overkill.  For more information on how to use struct kref, please see the
file Documentation/kref.txt in the Linux kernel source tree.


Creating "simple" kobjects

Sometimes all that a developer wants is a way to create a simple directory
in the sysfs hierarchy, and not have to mess with the whole complication of
ksets, show and store functions, and other details.  This is the one
exception where a single kobject should be created.  To create such an
entry, use the function:

    struct kobject *kobject_create_and_add(char *name, struct kobject *parent);

This function will create a kobject and place it in sysfs in the location
underneath the specified parent kobject.  To create simple attributes
associated with this kobject, use:

    int sysfs_create_file(struct kobject *kobj, struct attribute *attr);
or
    int sysfs_create_group(struct kobject *kobj, struct attribute_group *grp);

Both types of attributes used here, with a kobject that has been created
with the kobject_create_and_add(), can be of type kobj_attribute, so no
special custom attribute is needed to be created.

See the example module, samples/kobject/kobject-example.c for an
implementation of a simple kobject and attributes.



ktypes and release methods

One important thing still missing from the discussion is what happens to a
kobject when its reference count reaches zero. The code which created the
kobject generally does not know when that will happen; if it did, there
would be little point in using a kobject in the first place. Even
predictable object lifecycles become more complicated when sysfs is brought
in as other portions of the kernel can get a reference on any kobject that
is registered in the system.

The end result is that a structure protected by a kobject cannot be freed
before its reference count goes to zero. The reference count is not under
the direct control of the code which created the kobject. So that code must
be notified asynchronously whenever the last reference to one of its
kobjects goes away.

Once you registered your kobject via kobject_add(), you must never use
kfree() to free it directly. The only safe way is to use kobject_put(). It
is good practice to always use kobject_put() after kobject_init() to avoid
errors creeping in.

当kobject的“reference counter”为0时，系统将自动调用这个release函数。
This notification is done through a kobject's release() method. Usually
such a method has a form like:

    void my_object_release(struct kobject *kobj)
    {
    	    struct my_object *mine = container_of(kobj, struct my_object, kobj);

	    /* Perform any additional cleanup on this object, then... */
	    kfree(mine);
    }

One important point cannot be overstated: every kobject must have a
release() method, and the kobject must persist (in a consistent state)
until that method is called. If these constraints are not met, the code is
flawed.  Note that the kernel will warn you if you forget to provide a
release() method.  Do not try to get rid of this warning by providing an
"empty" release function; you will be mocked mercilessly by the kobject
maintainer if you attempt this.

Note, the name of the kobject is available in the release function, but it
must NOT be changed within this callback.  Otherwise there will be a memory
leak in the kobject core, which makes people unhappy.


release() method是和kobj_type关联的。
Interestingly, the release() method is not stored in the kobject itself;
instead, it is associated with the ktype. So let us introduce struct
kobj_type:

    struct kobj_type {
	    void (*release)(struct kobject *kobj);
	    const struct sysfs_ops *sysfs_ops;
	    struct attribute **default_attrs;
	    const struct kobj_ns_type_operations *(*child_ns_type)(struct kobject *kobj);
	    const void *(*namespace)(struct kobject *kobj);
    };

This structure is used to describe a particular type of kobject (or, more
correctly, of containing object). Every kobject needs to have an associated
kobj_type structure; a pointer to that structure must be specified when you
call kobject_init() or kobject_init_and_add().

The release field in struct kobj_type is, of course, a pointer to the
release() method for this type of kobject. The other two fields (sysfs_ops
and default_attrs) control how objects of this type are represented in
sysfs; they are beyond the scope of this document.

sysfs_ops and default_attrs 描述了kobject在sysfs系统的属性描述。

The default_attrs pointer is a list of default attributes that will be
automatically created for any kobject that is registered with this ktype.


ksets

A kset is merely a collection of kobjects that want to be associated with
each other.  There is no restriction that they be of the same ktype, but be
very careful if they are not.

A kset serves these functions:

 - It serves as a bag containing a group of objects. A kset can be used by
   the kernel to track "all block devices" or "all PCI device drivers."

 - A kset is also a subdirectory in sysfs, where the associated kobjects
   with the kset can show up.  Every kset contains a kobject which can be
   set up to be the parent of other kobjects; the top-level directories of
   the sysfs hierarchy are constructed in this way.

 - Ksets can support the "hotplugging" of kobjects and influence how
   uevent events are reported to user space.

In object-oriented terms, "kset" is the top-level container class; ksets
contain their own kobject, but that kobject is managed by the kset code and
should not be manipulated by any other user.

A kset keeps its children in a standard kernel linked list.  Kobjects point
back to their containing kset via their kset field. In almost all cases,
the kobjects belonging to a kset have that kset (or, strictly, its embedded
kobject) in their parent.

As a kset contains a kobject within it, it should always be dynamically
created and never declared statically or on the stack.  To create a new
kset use:
  struct kset *kset_create_and_add(const char *name,
				   struct kset_uevent_ops *u,
				   struct kobject *parent);

When you are finished with the kset, call:
  void kset_unregister(struct kset *kset);
to destroy it.

An example of using a kset can be seen in the
samples/kobject/kset-example.c file in the kernel tree.

If a kset wishes to control the uevent operations of the kobjects
associated with it, it can use the struct kset_uevent_ops to handle it:

struct kset_uevent_ops {
        int (*filter)(struct kset *kset, struct kobject *kobj);
        const char *(*name)(struct kset *kset, struct kobject *kobj);
        int (*uevent)(struct kset *kset, struct kobject *kobj,
                      struct kobj_uevent_env *env);
};


The filter function allows a kset to prevent a uevent from being emitted to
userspace for a specific kobject.  If the function returns 0, the uevent
will not be emitted.

The name function will be called to override the default name of the kset
that the uevent sends to userspace.  By default, the name will be the same
as the kset itself, but this function, if present, can override that name.

The uevent function will be called when the uevent is about to be sent to
userspace to allow more environment variables to be added to the uevent.

One might ask how, exactly, a kobject is added to a kset, given that no
functions which perform that function have been presented.  The answer is
that this task is handled by kobject_add().  When a kobject is passed to
kobject_add(), its kset member should point to the kset to which the
kobject will belong.  kobject_add() will handle the rest.

If the kobject belonging to a kset has no parent kobject set, it will be
added to the kset's directory.  Not all members of a kset do necessarily
live in the kset directory.  If an explicit parent kobject is assigned
before the kobject is added, the kobject is registered with the kset, but
added below the parent kobject.


Kobject removal

After a kobject has been registered with the kobject core successfully, it
must be cleaned up when the code is finished with it.  To do that, call
kobject_put().  By doing this, the kobject core will automatically clean up
all of the memory allocated by this kobject.  If a KOBJ_ADD uevent has been
sent for the object, a corresponding KOBJ_REMOVE uevent will be sent, and
any other sysfs housekeeping will be handled for the caller properly.

If you need to do a two-stage delete of the kobject (say you are not
allowed to sleep when you need to destroy the object), then call
kobject_del() which will unregister the kobject from sysfs.  This makes the
kobject "invisible", but it is not cleaned up, and the reference count of
the object is still the same.  At a later time call kobject_put() to finish
the cleanup of the memory associated with the kobject.

kobject_del() can be used to drop the reference to the parent object, if
circular references are constructed.  It is valid in some cases, that a
parent objects references a child.  Circular references _must_ be broken
with an explicit call to kobject_del(), so that a release functions will be
called, and the objects in the former circle release each other.


Example code to copy from
在samples/kobject/存放了一些例子。

For a more complete example of using ksets and kobjects properly, see the
example programs samples/kobject/{kobject-example.c,kset-example.c},
which will be built as loadable modules if you select CONFIG_SAMPLE_KOBJECT.


```

###基友udev系统
##devfs与udev系统的争论

devfs是在linux2.4内核引入的，其有下面的作用:

- 可以通过程序在设备初始化时在/dev目录下创建文件，卸载时把它删除
- 设备驱动程序可以指定设备名，权限，用户空间仍然可以修改
- 不需要为设备驱动分配主设备号和处理次设备号

尽管devfs很有用，但是在linux 2.6中，有人对他提出质疑，并用udev取代了他，原因如下:
- devfs做的工作可以在用户态来完成
- devfs被加入内核之时，大家希望他的质量可以改善
- devfs被发现了一些不可修复的bug
- devfs的维护者对devfs希望，并停止了维护工作

udev相比devfs，其完全在用户态工作，利用设备的hotplug event来实现工作。在热插拔时，设备的详细信息会显示在/sys的sysfs系统中。在使用udev后，在/dev就会出现真正设备的名字。            

所以udev的工作实现都是基于他的基友sysfs的系统信息输出，根据规则提取信息，然后挂载设备。没有sysfs系统就没有udev，也就没有/dev目录下动态创建的设备节点。






