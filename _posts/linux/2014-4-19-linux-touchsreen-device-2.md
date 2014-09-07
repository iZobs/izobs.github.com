---

layout: post
title: "linux 电容触摸屏驱动-2"
date: 2014-4-19 14:34
category: "学习"
comments: false
tags : "driver"

---

###linux i2c模型
我们的触摸屏采用的是ft5x06的i2c接口，所以我需要使用到linux的i2c设备驱动。先来看看linux的i2c模型是怎么回事.                 

![i2c](/picture/i2c.png "i2c")


Linux的I2C驱动框架属于典型的总线-驱动-设备模型。Linux的I2C驱动框架中的主要数据结构包括：i2c_driver、i2c_client、i2c_adapter和i2c_algorithm。i2c_adapter对应于物理上的一个适配器(i2c_adaptor)，这个适配器是基于不同的平台的，一个I2C适配器需要i2c_algorithm中提供的通信函数来控制适配器，因此i2c_adapter中包含其使用的i2c_algorithm的指针。i2c_algorithm中的关键函数master_xfer()以i2c_msg为单位产生I2C访问需要的信号。不同的平台所对应的master_xfer()是不同的，开发人员需要根据所用平台的硬件特性实现自己的XXX_xfer()方法以填充i2c_algorithm的master_xfer指针。i2c_ driver对应一套驱动方法，不对应于任何的物理实体。i2c_client对应于真实的物理设备，每个I2C设备都需要一个i2c_client来描述。i2c_client依附于i2c_adpater，这与I2C硬件体系中适配器和设备的关系一致。i2c_driver提供了i2c-client与i2c-adapter产生联系的函数。当attach a_dapter()函数探测物理设备时，如果确定存在一个client，则把该client使用的i2c_client数据结构的adapter指针指向对应的i2e_ adapter，driver指针指向该i2c_driver，并调用i2e_adapter的client_register()函数来注册此设备。相反的过程发生在i2c_ driver的detach_client()函数被调用的时候.由于部分英语名词翻译成中文意思会是一样了，为了不搞混了，我们下面都会用英文表示名词。                              


##i2c core

linux i2c的核心函数全在 driver/i2c/i2c-core.c下。I2C核心提供了一组不依赖于硬件平台的接口函数，I2C总线驱动和设备驱动之间依赖于I2C核心作为纽带。I2C核心提供了i2c_adapter的增加和删除函数、i2c_driver的增加和删除函数、i2c_client的依附和脱离函数以及i2c传输、发送和接收函数。i2c传输函数i2c_transfer()用于进行I2C适配器和I2C设备之间的一组消息交互i2c_master_send()函数和i2c_master_recv()函数内部会调用i2c_ transfer()函数分别完成一条写消息和一条读消息.                               

通过postcore_initcall(i2c_init)（这是内核的初始化标量，这些宏定义在文件include/linux/init.h中）i2c总线将被注册。这也是整个i2c系统的入口。我们先看i2c_init()函数，他在i2c-core.c中被定义，我们通过他来看看i2c-core.c都有什么函数:


<pre class="prettyprint" id="c">

static int __init i2c_init(void)
{
	int retval;
	
	/*注册i2c总线*/
	retval = bus_register(&i2c_bus_type);
	if (retval)
		return retval;
#ifdef CONFIG_I2C_COMPAT
	/*在/sys/class下注册i2c适配器(i2c_adapter)*/
	i2c_adapter_compat_class = class_compat_register("i2c-adapter");
	if (!i2c_adapter_compat_class) {
		retval = -ENOMEM;
		goto bus_err;
	}
#endif
	
//由于i2c总线本身不需要什么驱动,所以这里仅是一个dummy的驱动,为了方便一致吧
	retval = i2c_add_driver(&dummy_driver);
	if (retval)
		goto class_err;
	return 0;

class_err:
#ifdef CONFIG_I2C_COMPAT
	class_compat_unregister(i2c_adapter_compat_class);
bus_err:
#endif
	bus_unregister(&i2c_bus_type);
	return retval;
}

</pre>

bus_register(&i2c_bus_type) i2c总线注册过程我们先不管了，这个与plaform_bus_type总线注册的知识相关。
我接下来看看一些i2c_core.c里重要的结构和函数：

1.i2c_driver 和i2c_client

<pre class="prettyprint" id="c">

struct i2c_client {
 unsigned short flags；//标志  
 unsigned short addr; //低7位为芯片地址  
 char name[I2C_NAME_SIZE];//设备名称
 struct i2c_adapter *adapter;//依附的i2c_adapter
 struct i2c_driver *driver;//依附的i2c_driver 
 struct device dev;//设备结构体  
 int irq;//设备所使用的结构体  
 struct list_head detected;//链表头
 };

</pre>

i2c_driver结构体：

<pre class="prettyprint" id="c">
/**
 * struct i2c_driver - represent an I2C device driver
 * @class: What kind of i2c device we instantiate (for detect)
 * @attach_adapter: Callback for bus addition (deprecated)
 * @detach_adapter: Callback for bus removal (deprecated)
 * @probe: Callback for device binding
 * @remove: Callback for device unbinding
 * @shutdown: Callback for device shutdown
 * @suspend: Callback for device suspend
 * @resume: Callback for device resume
 * @alert: Alert callback, for example for the SMBus alert protocol
 * @command: Callback for bus-wide signaling (optional)
 * @driver: Device driver model driver
 * @id_table: List of I2C devices supported by this driver
 * @detect: Callback for device detection
 * @address_list: The I2C addresses to probe (for detect)
 * @clients: List of detected clients we created (for i2c-core use only)
 *
 * The driver.owner field should be set to the module owner of this driver.
 * The driver.name field should be set to the name of this driver.
 *
 * For automatic device detection, both @detect and @address_list must
 * be defined. @class should also be set, otherwise only devices forced
 * with module parameters will be created. The detect function must
 * fill at least the name field of the i2c_board_info structure it is
 * handed upon successful detection, and possibly also the flags field.
 *
 * If @detect is missing, the driver will still work fine for enumerated
 * devices. Detected devices simply won't be supported. This is expected
 * for the many I2C/SMBus devices which can't be detected reliably, and
 * the ones which can always be enumerated in practice.
 *
 * The i2c_client structure which is handed to the @detect callback is
 * not a real i2c_client. It is initialized just enough so that you can
 * call i2c_smbus_read_byte_data and friends on it. Don't do anything
 * else with it. In particular, calling dev_dbg and friends on it is
 * not allowed.
 */
struct i2c_driver {
	unsigned int class;

	/* Notifies the driver that a new bus has appeared or is about to be
	 * removed. You should avoid using this, it will be removed in a
	 * near future.
	 */
	/*挂载到adapter和从adapter上卸载的函数*/
	int (*attach_adapter)(struct i2c_adapter *) __deprecated; 
	int (*detach_adapter)(struct i2c_adapter *) __deprecated;

	/* Standard driver model interfaces */
	int (*probe)(struct i2c_client *, const struct i2c_device_id *);
	int (*remove)(struct i2c_client *);

	/* driver model interfaces that don't relate to enumeration  */
	void (*shutdown)(struct i2c_client *);
	int (*suspend)(struct i2c_client *, pm_message_t mesg);
	int (*resume)(struct i2c_client *);

	/* Alert callback, for example for the SMBus alert protocol.
	 * The format and meaning of the data value depends on the protocol.
	 * For the SMBus alert protocol, there is a single bit of data passed
	 * as the alert response's low bit ("event flag").
	 */
	void (*alert)(struct i2c_client *, unsigned int data);

	/* a ioctl like command that can be used to perform specific functions
	 * with the device.
	 */
	int (*command)(struct i2c_client *client, unsigned int cmd, void *arg);

	/*设备驱动模型结构体*/
	struct device_driver driver;
	const struct i2c_device_id *id_table;

	/* Device detection callback for automatic device creation */
	/*探测函数*/
	int (*detect)(struct i2c_client *, struct i2c_board_info *);
	const unsigned short *address_list;
	struct list_head clients;
};

</pre>

对于上面这个结构体，我需要重点关注struct device_driver driver，这就是linux设备里最难理解的一种设备类型，很不幸。i2c也是这个类型的。
所以，如果你想继续往下看而不头晕，请先了解一下device_driver。其他结构体内的函数其实都是为这个模型服务的。

i2c_driver对应一套驱动方法，其主要函数是attach_adapter()和detach_client()
i2c_client对应真实的i2c物理设备device，每个i2c设备都需要一个i2c_client来描述
i2c_driver与i2c_client的关系是一对多。一个i2c_driver上可以支持多个同等类型的i2c_client.

2._i2c adapter和i2c algorithm_


<pre class="prettyprint" id="c">
/*
 * i2c_adapter is the structure used to identify a physical i2c bus along
 * with the access algorithms necessary to access it.
 */
struct i2c_adapter {
	struct module *owner;
	unsigned int class;		  /* classes to allow probing for */
	const struct i2c_algorithm *algo; /* the algorithm to access the bus */
	void *algo_data;

	/* data fields that are valid for all devices	*/
	struct rt_mutex bus_lock;

	int timeout;			/* in jiffies */
	int retries;
	struct device dev;		/* the adapter device */

	int nr;				/*该成员描述了总线号*/
	char name[48];
	struct completion dev_released;

	struct mutex userspace_clients_lock;
	struct list_head userspace_clients;   /* i2c_client结构链表，该结构包含device，driver和  adapter结构*/  
};


/*
 * The following structs are for those who like to implement new bus drivers:
 * i2c_algorithm is the interface to a class of hardware solutions which can
 * be addressed using the same bus algorithms - i.e. bit-banging or the PCF8584
 * to name two of the most common.
 */
struct i2c_algorithm {
	/* If an adapter algorithm can't do I2C-level access, set master_xfer
	   to NULL. If an adapter algorithm can do SMBus access, set
	   smbus_xfer. If set to NULL, the SMBus protocol is simulated
	   using common I2C messages */
	/* master_xfer should return the number of messages successfully
	   processed, or a negative value on error */
	int (*master_xfer)(struct i2c_adapter *adap, struct i2c_msg *msgs,
			   int num);
	int (*smbus_xfer) (struct i2c_adapter *adap, u16 addr,
			   unsigned short flags, char read_write,
			   u8 command, int size, union i2c_smbus_data *data);

	/* To determine what the adapter supports */
	u32 (*functionality) (struct i2c_adapter *);
};


</pre>

　i2c_adapter对应与物理上的一个适配器，如s5pv210有两个。而i2c_algorithm对应一套通信方法，一个i2c适配器需要i2c_algorithm中提供的（i2c_algorithm中的又是更下层与硬件相关的代码提供）通信函数来控制适配器上产生特定的访问周期。缺少i2c_algorithm的i2c_adapter什么也做不了，因此i2c_adapter中包含其使用i2c_algorithm的指针。i2c_algorithm中的关键函数master_xfer()用于产生i2c访问周期需要的start stop ack信号，以i2c_msg（即i2c消息）为单位发送和接收通信数据。　i2c_msg也非常关键，调用驱动中的发送接收函数需要填充该结构体

i2c_adapter和i2c_client的关系与i2c硬件体系中适配器和设备的关系一致，即i2c_client依附于i2c_adapter,由于一个适配器上可以连接多个i2c设备，所以i2c_adapter中包含依附于它的i2c_client的链表。

3._一些重要函数_

<pre class="prettyprint" id="c">
//增加/删除i2c_adapter
int i2c_add_adapter(struct i2c_adapter *adapter)
int i2c_del_adapter(struct i2c_adapter *adap)

//增加/删除i2c_driver
int i2c_register_driver(struct module *owner, struct i2c_driver *driver)
void i2c_del_driver(struct i2c_driver *driver)

//i2c_client依附/脱离
int i2c_attach_client(struct i2c_client *client)

//增加/删除i2c_driver
int i2c_register_driver(struct module *owner, struct i2c_driver *driver)
void i2c_del_driver(struct i2c_driver *driver)

//i2c_client依附/脱离
int i2c_attach_client(struct i2c_client *client)
int i2c_detach_client(struct i2c_client *client)

//I2C传输,发送和接收
int i2c_master_send(struct i2c_client *client,const char *buf ,int count)
int i2c_master_recv(struct i2c_client *client, char *buf ,int count)
int i2c_transfer(struct i2c_adapter *adap, struct i2c_msg *msgs, int num)

</pre>
