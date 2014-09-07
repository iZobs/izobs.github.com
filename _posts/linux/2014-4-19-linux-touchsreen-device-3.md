---

layout: post
title: "linux 电容触摸屏驱动-3"
date: 2014-4-19 19:34
category: "学习"
comments: false
tags : "driver"

---

###动手写linux i2c驱动

- 第一层：提供i2c adapter的硬件驱动，探测、初始化i2c adapter（如申请i2c的io地址和中断号），驱动soc控制的i2c adapter在硬件上产生信号（start、stop、ack）以及处理i2c中断。覆盖图中的硬件实现层

- 第二层：提供i2c adapter的algorithm，用具体适配器的xxx_xferf()函数来填充i2c_algorithm的master_xfer函数指针，并把赋值后的i2c_algorithm再赋值给i2c_adapter的algo指针。

- 第三层：实现i2c设备驱动中的i2c_driver接口，用具体的i2c device设备的attach_adapter()、detach_adapter()方法赋值给i2c_driver的成员函数指针。实现设备device与总线（或者叫adapter）的挂接。

- 第四层：实现i2c设备所对应的具体device的驱动，i2c_driver只是实现设备与总线的挂接，而挂接在总线上的设备则是千差万别的，所以要实现具体设备device的write()、read()、ioctl()等方法，赋值给file_operations，然后注册字符设备（多数是字符设备）。

一般第一、二层由i2c总线驱动完成，一般不用写在目录drivers/i2c/busses下面。而第三，四由需要根据使用的i2c设备编写完成，一般都要自己写。如我们的触摸屏驱动，在目录drivers/input/touchscreen。

###i2c总线编写
其实在linux3.8里drivers/i2c/busses目录下有i2c-s3c2410.c，他支持s3c2410,s3c6410,s5pc110等Samsung 系列的芯片多种芯片的i2c。在编译内核的时候将其打开即可。现在我们写一个精简版的总线驱动，只为容易理解。如果要写一个稳定的i2c的总线驱动还是要参考i2c-s3c2410.c来写。

/bus/i2c-webee210.c
<pre class="prettyprint" id="c">
......
struct webee210_i2c_xfer_data {
	struct i2c_msg *msgs;
	int msn_num;
	int cur_msg;
	int cur_ptr;
	int state;
	int err;
	wait_queue_head_t wait;
};

static struct webee210_i2c_xfer_data webee210_i2c_xfer_data;
static struct webee210_i2c_regs *webee210_i2c_regs;
static unsigned long *gpd1con;
static unsigned long *gpd1pud;
static unsigned long *clk_gate_ip3;

static void webee210_i2c_start(void)
{
	webee210_i2c_xfer_data.state = STATE_START;
	
	if (webee210_i2c_xfer_data.msgs->flags & I2C_M_RD) /* 读 */
	{
		webee210_i2c_regs->iicds		 = webee210_i2c_xfer_data.msgs->addr << 1;
	//	printk("slave = %x\n",webee210_i2c_xfer_data.msgs->addr);

		webee210_i2c_regs->iicstat 	 = 0xb0;	// 主机接收，启动
	}
	else /* 写 */
	{
		webee210_i2c_regs->iicds		 = webee210_i2c_xfer_data.msgs->addr << 1;
		//webee210_i2c_regs->iicds		 = webee210_i2c_xfer_data.msgs->addr;
		webee210_i2c_regs->iicstat    = 0xf0; 		// 主机发送，启动
	}
}

static void webee210_i2c_stop(int err)
{
	webee210_i2c_xfer_data.state = STATE_STOP;
	webee210_i2c_xfer_data.err   = err;

	PRINTK("STATE_STOP, err = %d\n", err);


	if (webee210_i2c_xfer_data.msgs->flags & I2C_M_RD) /* 读 */
	{
		// 下面两行恢复I2C操作，发出P信号
		webee210_i2c_regs->iicstat = 0x90;
		webee210_i2c_regs->iiccon  = 0xaf;
		ndelay(50);  // 等待一段时间以便P信号已经发出
	}
	else /* 写 */
	{
		// 下面两行用来恢复I2C操作，发出P信号
		webee210_i2c_regs->iicstat = 0xd0;
		webee210_i2c_regs->iiccon  = 0xaf;
		ndelay(50);  // 等待一段时间以便P信号已经发出
	}

	/* 唤醒 */
	wake_up(&webee210_i2c_xfer_data.wait);
	
}

static int webee210_i2c_xfer(struct i2c_adapter *adap,
			struct i2c_msg *msgs, int num)
{
	unsigned long timeout;
	
	/* 把num个msg的I2C数据发送出去/读进来 */
	webee210_i2c_xfer_data.msgs    = msgs;
	webee210_i2c_xfer_data.msn_num = num;
	webee210_i2c_xfer_data.cur_msg = 0;
	webee210_i2c_xfer_data.cur_ptr = 0;
	webee210_i2c_xfer_data.err     = -ENODEV;
//	printk("...............................................\n");
	PRINTK("s3c2440_i2c_xfer \n");
	
	webee210_i2c_start();

	/* 休眠 */
	timeout = wait_event_timeout(webee210_i2c_xfer_data.wait, (webee210_i2c_xfer_data.state == STATE_STOP), HZ * 5);
	if (0 == timeout)
	{
	
		printk("webee210_i2c_xfer time out\n");
		return -ETIMEDOUT;
		//return webee210_i2c_xfer_data.err;
	}
	else
	{
		return webee210_i2c_xfer_data.err;
	}
}

static u32 webee210_i2c_func(struct i2c_adapter *adap)
{
	return I2C_FUNC_I2C | I2C_FUNC_SMBUS_EMUL | I2C_FUNC_PROTOCOL_MANGLING;
}


static const struct i2c_algorithm webee210_i2c_algo = {
//	.smbus_xfer     = ,
	.master_xfer	= webee210_i2c_xfer,
	.functionality	= webee210_i2c_func,
};

/* 1. 分配/设置i2c_adapter
 */
static struct i2c_adapter webee210_i2c_adapter = {
 .name			 = "webee210_100ask",
 .algo			 = &webee210_i2c_algo,
 .owner 		 = THIS_MODULE,
};

static int isLastMsg(void)
{
	return (webee210_i2c_xfer_data.cur_msg == webee210_i2c_xfer_data.msn_num - 1);
}

static int isEndData(void)
{
	return (webee210_i2c_xfer_data.cur_ptr >= webee210_i2c_xfer_data.msgs->len);
}

static int isLastData(void)
{
	return (webee210_i2c_xfer_data.cur_ptr == webee210_i2c_xfer_data.msgs->len - 1);
}

static irqreturn_t webee210_i2c_xfer_irq(int irq, void *dev_id)
{
	unsigned int iicSt;
	iicSt  = webee210_i2c_regs->iicstat; 

	if(iicSt & 0x8){ printk("Bus arbitration failed\n\r"); }

	PRINTK("webee210_i2c_xfer_irq \n");

	switch (webee210_i2c_xfer_data.state)
	{
		case STATE_START : /* 发出S和设备地址后,产生中断 */
		{
			PRINTK("Start\n");
			/* 如果没有ACK, 返回错误 */
			if (iicSt & S3C2410_IICSTAT_LASTBIT)
			{
				webee210_i2c_stop(-ENODEV);
				break;
			}

			if (isLastMsg() && isEndData())
			{
				webee210_i2c_stop(0);
				break;
			}

			/* 进入下一个状态 */
			if (webee210_i2c_xfer_data.msgs->flags & I2C_M_RD) /* 读 */
			{
				webee210_i2c_xfer_data.state = STATE_READ;
				goto next_read;
			}
			else
			{
				webee210_i2c_xfer_data.state = STATE_WRITE;
			}	
		}

		case STATE_WRITE:
		{
			PRINTK("STATE_WRITE\n");
			/* 如果没有ACK, 返回错误 */
			if (iicSt & S3C2410_IICSTAT_LASTBIT)
			{
				webee210_i2c_stop(-ENODEV);
				break;
			}

			if (!isEndData())  /* 如果当前msg还有数据要发送 */
			{
				webee210_i2c_regs->iicds = webee210_i2c_xfer_data.msgs->buf[webee210_i2c_xfer_data.cur_ptr];
				webee210_i2c_xfer_data.cur_ptr++;
				
				// 将数据写入IICDS后，需要一段时间才能出现在SDA线上
				ndelay(50);	
				
				webee210_i2c_regs->iiccon = 0xaf;		// 恢复I2C传输
				break;				
			}
			else if (!isLastMsg())
			{
				/* 开始处理下一个消息 */
				webee210_i2c_xfer_data.msgs++;
				webee210_i2c_xfer_data.cur_msg++;
				webee210_i2c_xfer_data.cur_ptr = 0;
				webee210_i2c_xfer_data.state = STATE_START;
				/* 发出START信号和发出设备地址 */
				webee210_i2c_start();
				break;
			}
			else
			{
				/* 是最后一个消息的最后一个数据 */
				webee210_i2c_stop(0);
				break;				
			}

			break;
		}

		case STATE_READ:
		{
			PRINTK("STATE_READ\n");
			/* 读出数据 */
			webee210_i2c_xfer_data.msgs->buf[webee210_i2c_xfer_data.cur_ptr] = webee210_i2c_regs->iicds;			
			webee210_i2c_xfer_data.cur_ptr++;
next_read:
			if (!isEndData()) /* 如果数据没读写, 继续发起读操作 */
			{
				if (isLastData())  /* 如果即将读的数据是最后一个, 不发ack */
				{
					webee210_i2c_regs->iiccon = 0x2f;   // 恢复I2C传输，接收到下一数据时无ACK
				}
				else
				{
					webee210_i2c_regs->iiccon = 0xaf;   // 恢复I2C传输，接收到下一数据时发出ACK
				}				
				break;
			}
			else if (!isLastMsg())
			{
				/* 开始处理下一个消息 */
				webee210_i2c_xfer_data.msgs++;
				webee210_i2c_xfer_data.cur_msg++;
				webee210_i2c_xfer_data.cur_ptr = 0;
				webee210_i2c_xfer_data.state = STATE_START;
				/* 发出START信号和发出设备地址 */
				webee210_i2c_start();
				break;
			}
			else
			{
				/* 是最后一个消息的最后一个数据 */
				webee210_i2c_stop(0);
				break;								
			}
			break;
		}

		default: break;
	}

	/* 清中断 */
	webee210_i2c_regs->iiccon &= ~(S3C2410_IICCON_IRQPEND);

	return IRQ_HANDLED;	
}

/*
 * I2C初始化
 */
static void webee210_i2c_init(void)
{
	/*使能时钟*/
	*clk_gate_ip3 = 0xffffffff;

	// 选择引脚功能：GPE15:IICSDA, GPE14:IICSCL
	*gpd1con |= 0x22<<16;
	*gpd1pud |= 0x5<<8;
	
//	GPH1CON |= 0xf<<24;//EINT14 ENABLE


	/* bit[7] = 1, 使能ACK
	* bit[6] = 0, IICCLK = PCLK/16
	* bit[5] = 1, 使能中断
	* bit[3:0] = 0xf, Tx clock = IICCLK/16
	* PCLK = 50MHz, IICCLK = 3.125MHz, Tx Clock = 0.195MHz
	*/
	webee210_i2c_regs->iiccon = (1<<7) | (0<<6) | (1<<5) | (0xf);  // 0xaf

	webee210_i2c_regs->iicadd = 0x10;     // S3C24xx slave address = [7:1]
	webee210_i2c_regs->iicstat = 0x10;     // I2C串行输出使能(Rx/Tx)
}

static int i2c_bus_webee210_init(void)
{
	/* 2. 硬件相关的设置 */
	webee210_i2c_regs = ioremap(0xE1A00000, sizeof(struct webee210_i2c_regs));
	gpd1con = ioremap(0xE02000C0,4); 
	gpd1pud = ioremap(0xE02000C8,4);
	clk_gate_ip3 = ioremap(0xE010046C,4);

	/*I2C初始化，寄存器，io口*/
	webee210_i2c_init();
	
	/*为i2c申请中断*/
	if(request_irq(IRQ_IIC2, webee210_i2c_xfer_irq, 0, "webee210-i2c", NULL))
	{
		printk("request_irq failed");
		return -EAGAIN;
	}

	init_waitqueue_head(&webee210_i2c_xfer_data.wait);
	
	/* 3. 注册i2c_adapter */
	i2c_add_adapter(&webee210_i2c_adapter);
	
	return 0;
}

static void i2c_bus_webee210_exit(void)
{
	i2c_del_adapter(&webee210_i2c_adapter);	
	free_irq(IRQ_IIC, NULL);
	iounmap(webee210_i2c_regs);
}

module_init(i2c_bus_webee210_init);
module_exit(i2c_bus_webee210_exit);
MODULE_LICENSE("GPL");

</pre>

上面关键的部分，可以用下面一个简单流程图表示。

![i2c-bus](/picture/i2c-bus.png "i2c-bus")



###i2c设备驱动编写

在我们开始编写i2c设备驱动之前,我们先来了解一些关于Linux中注册使用I2C设备的方法,这个内容可以在Documentation/i2c/instantiating-devices文件中找到。(linux 3.8)

- 通过总线号声明设备
- 立即探测设备
- 通过Probe探测相应设备
- 在用户空间立即探测

 简单来说，第一种方式一般应用在嵌入式设备中。因为对于嵌入式设备来说，外围器件基本都是固定的，只需提供有限几款器件的支持即可。使用这种方式的时候，需要在板级配置文件中定义并设置 i2c_board_info 这个结构体的内容。其中需要配置设备名称和设备地址，此外设备中断和私有数据结构也可以选择设置。然后使用 i2c_register_board_info 这个接口对设置的设备进行注册使用。需要注意的是这种方法注册的设备是在注册I2C总线驱动时进行驱动适配的。

第二种方法可以通过给定的I2C适配器以及相应的I2C板级结构体，自行通过 i2c_new_device 接口进行添加注册所需的设备。这种方法灵活性要较第一种方法大，可以很方便的在模块中使用。也可以枚举挂载i2c_client.

第三种方法是 2.6 内核之前的做法，使用 detect 方法去探测总线上的设备驱动。因为探测机制的原因，会导致一些副作用的发生，所以不建议使用，除非真的没有别的办法。

第四种方法是在Linux的控制台上，在用户空间通过sysfs，使用 /sys/bus/i2c/devices/i2c-3/new_device 节点进行设备的添加注册。 


我们将采用方式二，因为这也是现在内核所推荐和广泛使用的方式。根据Documentation/i2c/instantiating-devices的描述，我们在整理一下。

	struct i2c_client * i2c_new_device(struct i2c_adapter *adap, struct i2c_board_info const *info);

这个函数将会使用info提供的信息建立一个i2c_client并与第一个参数指向的i2c_adapter绑定。返回的参数是一个i2c_client指针。驱动中可以直接使用i2c_client指针和设备通信了。

如在arm/arch/mach-s5pv210/mach-smdk210.c中，有下面的代码:
<pre class="prettyprint" id="c">
static struct i2c_board_info smdkv210_i2c_devs0[] __initdata = {
	{ I2C_BOARD_INFO("24c08", 0x50), },     /* Samsung S524AD0XD1 */

};
/////////////////////////////////////////////
#include <linux/input/edt-ft5x06.h>

static struct i2c_board_info smdkv210_i2c_devs1[] __initdata = {
	/* To Be Updated */
         {
                I2C_BOARD_INFO("edt_ft5x06", (0x70 >> 1)),
         },
	
};

static struct i2c_board_info smdkv210_i2c_devs2[] __initdata = {
	/* To Be Updated */
};
</pre>

这个方法是一个比较简单的方法。

<pre class="prettyprint" id="c">
static struct i2c_board_info sfe4001_hwmon_info = {
    I2C_BOARD_INFO("max6647", 0x4e),
};

int sfe4001_init(struct efx_nic *efx)
{
    (...)
    efx->board_info.hwmon_client =
        i2c_new_device(&efx->i2c_adap, &sfe4001_hwmon_info);

    (...)
}                                                                                                                                      

</pre>


如果连i2c设备的地址都是不固定的，甚至在不同的板子上有不同的地址，可以提供一个地址列表供系统探测。
此时应该使用的函数是i2c_new_probe_device.文档如下:
	A variant of this is when you don't know for sure if an I2C device is
	present or not (for example for an optional feature which is not present
	on cheap variants of a board but you have no way to tell them apart), or
	it may have different addresses from one board to the next (manufacturer
	changing its design without notice). In this case, you can call
	i2c_new_probed_device() instead of i2c_new_device().

获取i2c_adapter指针的函数是：
struct i2c_adapter* i2c_get_adapter(int id)；//它的参数是i2c总线编号。
使用完要释放：
void i2c_put_adapter(struct i2c_adapter *adap)；


<pre class="prettyprint" id="c">
static const unsigned short normal_i2c[] = { 0x2c, 0x2d, I2C_CLIENT_END };

static int usb_hcd_nxp_probe(struct platform_device *pdev)
{
    (...)
    struct i2c_adapter *i2c_adap;
    struct i2c_board_info i2c_info;

    (...)
    i2c_adap = i2c_get_adapter(2);
    memset(&i2c_info, 0, sizeof(struct i2c_board_info));
    strlcpy(i2c_info.type, "isp1301_nxp", I2C_NAME_SIZE);
    isp1301_i2c_client = i2c_new_probed_device(i2c_adap, &i2c_info,
                           normal_i2c, NULL);
    i2c_put_adapter(i2c_adap);
    (...)
}
 
</pre>

关于i2c总线号的获取，可以在你insmod i2c总线驱动后，进入`/sys/class/i2c-dev`下查看，如我的:

	[Webee210]\# pwd
	/sys/class/i2c-dev
	[Webee210]\# ls
	i2c-3
	[Webee210]\# 


下面，我们看我们的代码：

/dev/dev.c
<pre class="prettyprint" id="c">

#include <linux/kernel.h>
#include <linux/module.h>
#include <linux/platform_device.h>
#include <linux/i2c.h>
#include <linux/err.h>
#include <linux/slab.h>
/*
static struct i2c_board_info ft5x06_info = {	
	I2C_BOARD_INFO("at24c08", 0x1c),
};
*/
/*ft5x06 为0x38 */
static const unsigned short addr_list[] = {0x38, 0x1C, 0x70, 0xe, I2C_CLIENT_END};
static struct i2c_client *ft5x06_client;

static int ft5x06_dev_init(void)
{
	struct i2c_adapter *i2c_adap;

	struct i2c_board_info ft5x06_info;

	memset(&ft5x06_info, 0, sizeof(struct i2c_board_info));	
	/*我们自己建一个i2c_board_info，名字叫ft5x06,i2c_client 将根据这个名字寻找驱动*/
	strlcpy(ft5x06_info.type, "ft5x06", I2C_NAME_SIZE);
	/*这个根据自己系统的i2c总线的adapter号进行修改*/
	i2c_adap = i2c_get_adapter(3);
	if(!i2c_adap)
	printk("i2c_adap_fail\n");
	
	ft5x06_client = i2c_new_probed_device(i2c_adap, &ft5x06_info, addr_list, NULL);
	
	i2c_put_adapter(i2c_adap);
		
	if(!ft5x06_client)
	printk("<0> ft5x06_client_fail\n");	

	return 0;
}

static void ft5x06_dev_exit(void)
{
	i2c_unregister_device(ft5x06_client);
}

module_init(ft5x06_dev_init);
module_exit(ft5x06_dev_exit);
MODULE_LICENSE("GPL");

</pre>


这个dev.c的代码也可以完全整合到drv.c文件中，在i2c_driver中的attach_adapter调用i2c_new_device()或者i2c_new_probed_device（）。这里为了结构清晰，所以分开来写。


最后，我们来看/drv/drv.c,在这里，我们注册i2c_driver。不过因为这个内容涉及到linux的input设备类型，所以我们在下篇再文章一起看吧。



