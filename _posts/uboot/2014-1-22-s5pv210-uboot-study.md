---

layout: post
title: "s5pv210-uboot启动流程"
date: 2014-1-22 15:34
category: "学习"
comments: false
tags : "uboot"

---


# 一、启动流程  #

### 1.s5pv210的启动过程 ###

根据三星公司的《S5PV210_UM_REV1.1》手册可知,S5PV210 启动过程主要可
分为 3 个阶段

-  S5PV210 上电复位后将从 IROM 处执行已固化的启动代码 -------BL0
-  在 BL0 里初始化过程中对启动设备进行判断,并从启动设备拷贝 BL1(最大16KB ) 到 IRAM 处 , 即 刚 才 所 说 的 0xD0020000 开 始 的 地 址 , 其 中0xD0020000~0xD0020010 的 16 字节为 BL1 的校验信息和 BL1 尺寸,并对 BL1进行校验,校验通过转入 BL1 进行执行,BL1 继续初始化,并拷贝 BL2(最大80KB)到 IRAM 中并对其校验,通过后转入 BL2。
-  BL2 完成一些比较复杂的初始化,包括 DRAM 的初始化,完成后将 OS 代码拷贝到 DRAM 中,并跳到 OS 中执行并完成启动引导。

![s5pv210](/picture/s5pv210.png "s5pv210")



BL0 固化代码主要完成以下初始化:

- 关闭看门狗;
- 初始化 icache;
- 初始化栈;
- 初始化堆;
- 初始化块设备拷贝功能;
- 设置系统时钟;
- 拷贝 BL1 到 iRAM;
- 检查 BL1 的校验和,如果失败则第二启动模式(安全启动模式),校验成功则跳到 0xD0020000(IRAM)处执行。

___其中 0xD0020000 ~ 0xD0020010 里的 16 字节头部信息是什么呢?这 16 字节信息用户是不能随便设置的.在《S5PV210_iROM_ApplicationNote_Preliminary_20091126》文档中规定___



### 2.uboot中s5pv210的启动 ###

__2.1 uboot启动流程简介__
BL0是出厂的时候就固化在IROM里的，所以我们的uboot就要实现BL1和BL2，BL1在uboot里叫做u-boot-spl.bin，BL2就是我们很熟悉的u-boot.bin.上面的三个阶段对应于uboot启动，他们的功能如下：

-  BL0：出厂的时候就固化在irom中一段代码，主要负责拷贝8kb的bl1到s5pv210的一个96kb大小内部sram(Internal SRAM)中运行。值得注意的是s5pv210的Internal SRAM支持的bl1的大小可以达到16kb，容量的扩增是为了适应bootloder变得越来复杂而做的。虽然如此，但目前我们制作出来的bl1的大小仍然可以保持在8kb以内，同样能满足需求。
-  BL1：u-boot的前8kb代码(s5pv210也支持16kb大小，原因上一点提过了)，除了初始化系统时钟和一些基本的硬件外，主要负责完成代码的搬运工作(我设计成搬运bl1+bl2，而不仅仅是bl2)，也就是将完整的u-boot代码(bl1+bl2)从nand flash或者mmcSD等的存储器中读取到内存中，然后跳转到内存中运行u-boot
-  BL2：完成全面的硬件初始化和加载OS到内存中，接着运行OS。上述几个阶段的流程描述在s5pv210_irom_application手册中有详细描述.

在实际的编译中会生成两个uboot.bin。拿webee210的uboot来说，最后在根目录下生成了uboot.bin和webee210-uboot.bin连个`.bin`文件。其中`webee210-uboot.bin`是通过`/board/samsung/webee210/tools`目录下的`mkwebee210spl.bin`对`uboot.bin`处理
生成的。主要是为其添加16k的头文件信息。

![s5pv210_irom](/picture/irom.png "s5pv210_irom")


__首先把启动部分的代码分为3部分，以start.S为主，另外还有lowlevel_init.S，mem_setup.S，ctr0.S。__

- lowlevel_init.S:主要是一部分硬件的初始化，尤其是系统时钟和DRAM的初始化。如果u-boot一旦被搬运到内存中运行，那么是必须要跳过时钟和DRAM的初始化的，因为这在搬运之前已经做过了。并且如果代码在内存中运行的时侯你却去初始化DRAM，那必然导致崩溃！
- mem_setup.S：DRAM初始化代码和MMU相关代码放在这个文件中。
- ctr0.S：u-boot自带的代码文件，存放汇编函数main。

__2.2 启动原理__

 必须要明白的一点是，当代码从存储介质(nand flash，SD，norflash，onenand等)中搬运到了DRAM中后随即会跳转到内存中运行u-boot，接着会有一个重定位(relocate_code)的过程，relocate_code子函数在start.S中，而给relocate_code子函数传参数的是crt0.S中的main子函数。当判断到当前u-boot在内存的低地址处，那么relocate_code就会工作，把u-boot代码从低地址处再搬运到内存地址的顶端，然后跳转到新的位置去继续运行u-boot。而搬运的目标地址是在board_init_f()函数(此函数在/arch/arm/lib/board.c中)中计算出来的

 ![s5pv210_uboot](/picture/uboot.png "s5pv210_uboot")

__2.2.1 start.S__

<pre class="prettyprint" id="c">

#include <asm-offsets.h>
#include <config.h>
#include <version.h>
#include <common.h>
#include <configs/smart210.h>
#include <s5pc110.h>
.globl _start
_start: b	reset
	ldr	pc, _undefined_instruction  /*未定义指令异常,0x04*/  
	ldr	pc, _software_interrupt		/*软中断异常,0x08*/ 
	ldr	pc, _prefetch_abort         /*内存操作异常,0x0c*/ 
	ldr	pc, _data_abort				/*数据异常,0x10*/ 
	ldr	pc, _not_used				 /*未适用,0x14*/ 
	ldr	pc, _irq					/*慢速中断异常,0x1c*/
	ldr	pc, _fiq					/*快速中断异常,0x1c*/


#ifdef CONFIG_SPL_BUILD
_undefined_instruction: .word _undefined_instruction
_software_interrupt:	.word _software_interrupt
_prefetch_abort:	.word _prefetch_abort
_data_abort:		.word _data_abort
_not_used:		.word _not_used
_irq:			.word _irq
_fiq:			.word _fiq
_pad:			.word 0x12345678 /* now 16*4=64 */
#else
_undefined_instruction: .word undefined_instruction
_software_interrupt:	.word software_interrupt
_prefetch_abort:	.word prefetch_abort
_data_abort:		.word data_abort
_not_used:		.word not_used
_irq:			.word irq
_fiq:			.word fiq
_pad:			.word 0x12345678 /* now 16*4=64 */
#endif	/* CONFIG_SPL_BUILD */

.global _end_vect
_end_vect:

	.balignl 16,0xdeadbeef   //所以意思就是,接下来的代码,都要16字节对齐,不知之处,用0xdeadbeef填充。

/*************************************************************************
 *
 * Startup Code (reset vector)
 *
 * do important init only if we don't start from memory!
 * setup Memory and board specific bits prior to relocation.
 * relocate armboot to ram
 * setup stack
 *
 *************************************************************************/
 /*
此地址中是一个word类型的变量,变量名
是TEXT_BASE,此值见名知意,是text的base,即代码的基地址,
在include/configs/webee210.h
#define CONFIG_SYS_TEXT_BASE            0x23E00000*/

.globl _TEXT_BASE
_TEXT_BASE:
	.word	CONFIG_SYS_TEXT_BASE

#ifdef CONFIG_TEGRA2
/*
 * Tegra2 uses 2 separate CPUs - the AVP (ARM7TDMI) and the CPU (dual A9s).
 * U-Boot runs on the AVP first, setting things up for the CPU (PLLs,
 * muxes, clocks, clamps, etc.). Then the AVP halts, and expects the CPU
 * to pick up its reset vector, which points here.
 */
.globl _armboot_start
_armboot_start:
	.word _start        //*(_armboot_start) = _start
#endif

/*
 * These are defined in the board-specific linker script.
 */
.globl _bss_start_ofs
_bss_start_ofs:
	.word __bss_start - _start

.global	_image_copy_end_ofs
_image_copy_end_ofs:
	.word 	__image_copy_end - _start

.globl _bss_end_ofs
_bss_end_ofs:
	.word __bss_end__ - _start

.globl _end_ofs
_end_ofs:
	.word _end - _start

#ifdef CONFIG_USE_IRQ
/* IRQ stack memory (calculated at run-time) */
.globl IRQ_STACK_START
IRQ_STACK_START:
	.word	0x0badc0de

/* IRQ stack memory (calculated at run-time) */
.globl FIQ_STACK_START
FIQ_STACK_START:
	.word 0x0badc0de
#endif

/* IRQ stack memory (calculated at run-time) + 8 bytes */
.globl IRQ_STACK_START_IN
IRQ_STACK_START_IN:
	.word	0x0badc0de

</pre>


reset函数里主要是设置了cpu的模式,然后就跳转到cpu_init_crit函数.

{% highlight c %}
/*
 * the actual reset code
 */

reset:
	bl	save_boot_params    //其不做任何事情，直接返回
	/*
	 * set the cpu to SVC32 mode  //设置CPU模式为SVC32
	 */
	mrs	r0, cpsr
	bic	r0, r0, #0x1f
	orr	r0, r0, #0xd3
	msr	cpsr,r0


#ifndef CONFIG_SKIP_LOWLEVEL_INIT
	bl	cpu_init_crit  //
#endif
{% endhighlight c %}


{% highlight c %}

/*************************************************************************
 *
 * CPU_init_critical registers
 *
 * setup important registers
 * setup memory timing
 *
 *************************************************************************/
cpu_init_crit:
	/*
	 * Invalidate L1 I/D
	 */
	mov	r0, #0			@ set up for MCR
	mcr	p15, 0, r0, c8, c7, 0	@ invalidate TLBs
	mcr	p15, 0, r0, c7, c5, 0	@ invalidate icache
	mcr	p15, 0, r0, c7, c5, 6	@ invalidate BP array
	mcr     p15, 0, r0, c7, c10, 4	@ DSB
	mcr     p15, 0, r0, c7, c5, 4	@ ISB

	/*
	 * disable MMU stuff and caches
	 */
	mrc	p15, 0, r0, c1, c0, 0
	bic	r0, r0, #0x00002000	@ clear bits 13 (--V-)
	bic	r0, r0, #0x00000007	@ clear bits 2:0 (-CAM)
	orr	r0, r0, #0x00000002	@ set bit 1 (--A-) Align
	orr	r0, r0, #0x00000800	@ set bit 11 (Z---) BTB
#ifdef CONFIG_SYS_ICACHE_OFF
	bic	r0, r0, #0x00001000	@ clear bit 12 (I) I-cache
#else
	orr	r0, r0, #0x00001000	@ set bit 12 (I) I-cache
#endif
	mcr	p15, 0, r0, c1, c0, 0

	/*
	 * Jump to board specific initialization...
	 * The Mask ROM will have already initialized
	 * basic memory. Go here to bump up clock rate and handle
	 * wake up conditions.
	 */
	mov	ip, lr			@ persevere link reg across call
	bl	lowlevel_init	//跳转到lowlevel_init,与开发板相关的初始化
	mov	lr, ip			@ restore link
	mov	pc, lr			@ back to my caller
#endif

{% endhighlight c %}

cpu_init_crit中主要做初始化了memory和串口，接下来就是调用C函数board_init_f了，在调用C函数之前得设置栈


lowlevel_init.s存放于`board/samsung/webee210/lowlevel_init.s`

{% highlight c %}

.globl lowlevel_init
lowlevel_init:
	push	{lr}   //把返回地址保存到栈中

	/* check reset status  */

	ldr	r0, =(ELFIN_CLOCK_POWER_BASE+RST_STAT_OFFSET)
	ldr	r1, [r0]
	bic	r1, r1, #0xfff6ffff
	cmp	r1, #0x10000
	beq	wakeup_reset_pre
	cmp	r1, #0x80000
	beq	wakeup_reset_from_didle

	/* IO Retention release */
	ldr	r0, =(ELFIN_CLOCK_POWER_BASE + OTHERS_OFFSET)
	ldr	r1, [r0]
	ldr	r2, =IO_RET_REL
	orr	r1, r1, r2
	str	r1, [r0]

	/* Disable Watchdog */
	ldr	r0, =ELFIN_WATCHDOG_BASE	/* 0xE2700000 */
	mov	r1, #0
	str	r1, [r0]

	/* SRAM(2MB) init for SMDKC110 */
	/* GPJ1 SROM_ADDR_16to21 */
	ldr	r0, =ELFIN_GPIO_BASE

	ldr	r1, [r0, #GPJ1CON_OFFSET]
	bic	r1, r1, #0xFFFFFF
	ldr	r2, =0x444444
	orr	r1, r1, r2
	str	r1, [r0, #GPJ1CON_OFFSET]

	ldr	r1, [r0, #GPJ1PUD_OFFSET]
	ldr	r2, =0x3ff
	bic	r1, r1, r2
	str	r1, [r0, #GPJ1PUD_OFFSET]

	/* GPJ4 SROM_ADDR_16to21 */
	ldr	r1, [r0, #GPJ4CON_OFFSET]
	bic	r1, r1, #(0xf<<16)
	ldr	r2, =(0x4<<16)
	orr	r1, r1, r2
	str	r1, [r0, #GPJ4CON_OFFSET]

	ldr	r1, [r0, #GPJ4PUD_OFFSET]
	ldr	r2, =(0x3<<8)
	bic	r1, r1, r2
	str	r1, [r0, #GPJ4PUD_OFFSET]


	/* CS0 - 16bit sram, enable nBE, Byte base address */
	ldr	r0, =ELFIN_SROM_BASE	/* 0xE8000000 */
	mov	r1, #0x1
	str	r1, [r0]

	/* PS_HOLD pin(GPH0_0) set to high */
	ldr	r0, =(ELFIN_CLOCK_POWER_BASE + PS_HOLD_CONTROL_OFFSET)
	ldr	r1, [r0]
	orr	r1, r1, #0x300
	orr	r1, r1, #0x1
	str	r1, [r0]

	/* when we already run in ram, we don't need to relocate U-Boot.
	 * and actually, memory controller must be configured before U-Boot
	 * is running in ram.
	 */
	ldr	r0, =0x00ffffff
	bic	r1, pc, r0		/* r0 <- current base addr of code */
	ldr	r2, _TEXT_BASE		/* r1 <- original base addr in ram */
	bic	r2, r2, r0		/* r0 <- current base addr of code */
	cmp     r1, r2                  /* compare r0, r1                  */
	beq     1f			/* r0 == r1 then skip sdram init   */

	/* init system clock */
	bl system_clock_init

	/* Memory initialize */
	bl mem_ctrl_asm_init

1:
	/* for UART */
	bl uart_asm_init

	bl tzpc_init

#if defined(CONFIG_ONENAND)
	bl onenandcon_init
#endif

#if defined(CONFIG_NAND)
	/* simple init for NAND */
	bl nand_asm_init
#endif

	/* check reset status  */

	ldr	r0, =(ELFIN_CLOCK_POWER_BASE+RST_STAT_OFFSET)
	ldr	r1, [r0]
	bic	r1, r1, #0xfffeffff
	cmp	r1, #0x10000
	beq	wakeup_reset_pre

	/* ABB disable */
	ldr	r0, =0xE010C300
	orr	r1, r1, #(0x1<<23)
	str	r1, [r0]

	/* Print 'K' */
	ldr	r0, =ELFIN_UART_CONSOLE_BASE
	ldr	r1, =0x4b4b4b4b
	str	r1, [r0, #UTXH_OFFSET]

	pop	{pc}

{% endhighlight c %}

总结一下上面的东东,看看,start.S是如何对CPU和开发板进行初始化的.首先start.S 执行reset函数,在reset函数里初始化了CPU 为SVC32后,就跳转到cpu_init_crit函数.在cpu_init_crit函数里设置了 TLBs、icache、BP array等,然后就跳转到`board/samsung/webee210/lowlevel_init.S`进行与开发板相关的初始化配置.在这里
关看门狗,初始化 SRAM,初始化 SROM,初始化电源保持控制器,初始化系统时钟,内存初始化
初始化串口,Nand Flash 早期初始化等.

接下来我们继续回到start.S里,看call_board_init_f这个函数.

{% highlight c %}
/* Set stackpointer in internal RAM to call board_init_f */
call_board_init_f:
	/* 为待会要执行的 C 函数设置栈 */
	ldr	sp, =(CONFIG_SYS_INIT_SP_ADDR)
	bic	sp, sp, #7 /* 8-byte alignment for ABI compliance */
	ldr	r0,=0x00000000
#if defined(CONFIG_TINY210) || defined(CONFIG_WEBEE210)
	adr	r4, _start
	ldr	r5,_TEXT_BASE
	/* 判断当前代码是否已在 RAM 上运行(DDR2) */
	/* 因为现在还在第一阶段,还在代码 SRAM 上运行,
	* 所以不会直接跳到 board_init_in_ram*/
	cmp     r5,r4
	beq	board_init_in_ram
	
	/* PRO_ID_BASE 是判断启动方式的寄存器 */
	ldr	r0, =PRO_ID_BASE
        ldr	r1, [r0,#OMR_OFFSET]
        bic	r2, r1, #0xffffffc1

	/* NAND BOOT ,判断是否通过 Nand Flash 方式启动 */
	cmp	r2, #0x0		@ 512B 4-cycle
	moveq	r3, #BOOT_NAND

	cmp	r2, #0x2		@ 2KB 5-cycle
	moveq	r3, #BOOT_NAND

	cmp	r2, #0x4		@ 4KB 5-cycle	8-bit ECC
	moveq	r3, #BOOT_NAND

	cmp	r2, #0x6		@ 4KB 5-cycle	16-bit ECC
	moveq	r3, #BOOT_NAND

	cmp	r2, #0x8		@ OneNAND Mux
	moveq	r3, #BOOT_ONENAND

	/* SD/MMC BOOT */
	cmp     r2, #0xc
	moveq   r3, #BOOT_MMCSD	

	/* NOR BOOT */
	cmp     r2, #0x14
	moveq   r3, #BOOT_NOR	

	/* Uart BOOTONG failed */
	cmp     r2, #(0x1<<4)
	moveq   r3, #BOOT_SEC_DEV
	
	ldr	r0, =INF_REG_BASE
	str	r3, [r0, #INF_REG3_OFFSET]

	ldr	r1, [r0, #INF_REG3_OFFSET]
	cmp	r1, #BOOT_NAND		/* 0x0 => boot device is nand */
	beq	nand_boot_210
	cmp     r1, #BOOT_MMCSD
	beq     mmcsd_boot_210
	
nand_boot_210:
	bl     board_init_f_nand

mmcsd_boot_210:
	bl     board_init_f
board_init_in_ram:
#endif
	bl	board_init_f

{% endhighlight c %}

假如是nand启动,`arch\arm\cpu\armv7\s5pc1xx/nand_cp.c`
{% highlight c %}
void board_init_f_nand(unsigned long bootflag)
{
        __attribute__((noreturn)) void (*uboot)(void);
        copy_uboot_to_ram_nand();

        /* Jump to U-Boot image */
        uboot = (void *)CONFIG_SYS_TEXT_BASE;
	(*uboot)();
        /* Never returns Here */
}
{% endhighlight c %}

假如是sd卡启动 ,`arch\arm\cpu\armv7\s5pc1xx/mmc_boot.c`
{% highlight c %}
void board_init_f(unsigned long bootflag)
{
        __attribute__((noreturn)) void (*uboot)(void);
        /*把第二阶段的 U-Boot 从 sd 拷贝到内存的函数*/
        copy_uboot_to_ram();

        /* Jump to U-Boot image */
        uboot = (void *)CONFIG_SYS_TEXT_BASE;
        /* 跳转到第二阶段所在的内存(DDR2)地址处 */
        (*uboot)();
        /* Never returns Here */
}
{% endhighlight c %}

到此为止,上面的内容为第一阶段所做的工作,即bl1,接下来是bl2,完成全面的硬件初始化和加载OS到内存中，接着运行OS
看回start.S里的call_board_init_f函数,在我们进行是在nand flash启动还是sd卡启动之前有一段代码如下,这段代码是判断当前的代码是否运行于RAM上的.假如不是则根据PRO_ID_BASE判断是哪种启动方式,然后调用board_init_f()或board_init_f_nand(),在他们里面有调用了copy_uboot_to_ram()0或copy_uboot_to_ram_nand(),所以uboot将会被copy到ram中,即代码开始从ram上运行.当代码已经在ram上运行了,则执行board_init_in_ram()函数.我看看接下来看看这个函数都做了什么.

{% highlight c %}

#if defined(CONFIG_TINY210) || defined(CONFIG_WEBEE210)
	adr	r4, _start
	ldr	r5,_TEXT_BASE
	/* 判断当前代码是否已在 RAM 上运行(DDR2) */
	/* 因为现在还在第一阶段,还在代码 SRAM 上运行,
	* 所以不会直接跳到 board_init_in_ram*/
	cmp     r5,r4
	beq	board_init_in_ram
	
	/* PRO_ID_BASE 是判断启动方式的寄存器 */
	ldr	r0, =PRO_ID_BASE
        ldr	r1, [r0,#OMR_OFFSET]
        bic	r2, r1, #0xffffffc1

{% endhighlight c %}


board_init_in_ram执行的也是board_init_f()这个函数,这个函数在前也出现过,当我们搜索board_init_f函数时,我们将发现在`arch\arm\cpu\armv7\s5pc1xx/mmc_boot.c`和`arch\arm\lib\board.c`中都有出现过.那么系统是如何判断是执行哪个board_init_f的呢.我觉得应该是与程序执行的内存地址不同有关.在没有判断进入ram之前,board_init_f在一个内存里,当进入ram之后,board_init_f则在另外的地址里.
这里使用相同的函数名,不知道通过什么连接方式实现不冲突,令人疑惑.网上搜到大多数的board_init_f函数都指的是`arch\arm\lib\board.c`里的这个board_init_f.
{% highlight c %}
board_init_in_ram:
#endif
	bl	board_init_f
{% endhighlight c %}

下面看看这个board_init_f都做了什么工作.
{% highlight c %}
void board_init_f(ulong bootflag)
{
	bd_t *bd;
	init_fnc_t **init_fnc_ptr;
	gd_t *id;
	ulong addr, addr_sp;

	/* Pointer is writable since we allocated a register for it */
	gd = (gd_t *) ((CONFIG_SYS_INIT_SP_ADDR) & ~0x07);
	/* compiler optimization barrier needed for GCC >= 3.4 */
	__asm__ __volatile__("": : :"memory");

	memset((void *)gd, 0, sizeof(gd_t));

	gd->mon_len = _bss_end_ofs;

 /* 依次调用 init_fnc_ptr 指针数组里的每个函数 */

	for (init_fnc_ptr = init_sequence; *init_fnc_ptr; ++init_fnc_ptr) {
		if ((*init_fnc_ptr)() != 0) {
			hang ();
		}
	}


{% endhighlight c %}

涉及到两个重要的数据结构：1）bd_t结构体，关于开发板信息（波特率，ip, 平台号，启动参数）。2）gd_t结构体成员主要是一些全局的系统初始化参数

  `gd=(gd_t *)(CONFIG_SYS_INIT_SP_ADDR) &~0x07)`

CONFIG_SYS_INIT_SP_ADDR的定义如下：
`#define CONFIG_SYS_INIT_SP_ADDR (0x22000)`
0x22000 & ~0x07，为了保证结果八字节对齐。
查芯片手册，可知0x0002_0000----0x0003_8000范围，为96K,是IRAM的空间。即gd指向IRAM的一个地址。

在本文中前面有声明DECLARE_GLOBAL_DATA_PTR
跟踪定义可看到下面形式：
`#define DECLARE_GLOBAL_DATA_PTR register volatile gd_t *gd asm ("r8")`这个声明告诉编译器使用寄存器r8来存储gd_t类型的指针gd，即这个定义声明了一个指针，并且指明了它的存储位置。（即gd值放在寄存器r8中)

- register表示变量放在机器的寄存器
- volatile用于指定变量的值可以由外部过程异步修改
- 并且这个指针现在被赋值为CONFIG_SYS_INIT_SP_ADDR) &~0x07， __asm__ __volatile__("": : :"memory"); memset((void *)gd, 0, sizeof(gd_t));
- memory 强制gcc编译器假设RAM所有内存单元均被汇编指令修改，这样cpu中的registers和cache中已缓存的内存单元中的数据将作废。cpu将不得不在需要的时候重新读取内存中的数据。这就阻止了cpu又将registers，cache中的数据用于去优化指令，而避免去访问内存。
- __asm__用于指示编译器在此插入汇编语句。                           
- volatile 用于告诉编译器，严禁将此处的汇编语句与其它的语句重组合优化。即：原原本本按原来的样子处理这这里的汇编。 memory强制gcc编译器假设RAM所有内存单元均被汇编指令修改，这样cpu中的registers和cache中已缓存的内存单元中的数据将作废。cpu将不得不在需要的时候重新读取内存中的数据。这就阻止了cpu又将registers，cache中的数据用于去优化指令，而避免去访问内存。

- "":::表示这是个空指令。

`gd->mon_len = _bss_end_ofs; `

_bss_end_ofs的定义在start.S中： .globl _bss_end_ofs

_bss_end_ofs:
        .word __bss_end__ - _start

.word就是在当前地址_bss_end_ofs放值 __bss-end__ - _start。看到是所以段的大小，此时，新开终端。打开u-boot.lds，找__bss_end__ Gd->mon_len存的值就是u-boot的实际大小。

init_fnc_ptr是一个函数指针,而init_sequence是一个函数队列,定义如下:
{% highlight c %}
init_fnc_t *init_sequence[] = {
#if defined(CONFIG_ARCH_CPU_INIT)
	arch_cpu_init,		/* basic arch cpu dependent setup */
#endif
#if defined(CONFIG_BOARD_EARLY_INIT_F)
	board_early_init_f,
#endif
	timer_init,		/* initialize timer */
#ifdef CONFIG_FSL_ESDHC
	get_clocks,
#endif
	env_init,		/* initialize environment */
	init_baudrate,		/* initialze baudrate settings */
	serial_init,		/* serial communications setup */
	console_init_f,		/* stage 1 init of console */
	display_banner,		/* say that we are here */
#if defined(CONFIG_DISPLAY_CPUINFO)
	print_cpuinfo,		/* display cpu info (and speed) */
#endif
#if defined(CONFIG_DISPLAY_BOARDINFO)
	checkboard,		/* display board info */
#endif
#if defined(CONFIG_HARD_I2C) || defined(CONFIG_SOFT_I2C)
	init_func_i2c,
#endif
	dram_init,		/* configure available RAM banks */
	NULL,
};

{% endhighlight c %}

所以通过循环，调用了函数指针数组中的一系列初始化函数：arch_cpu_init，timer_init，env_int，init_baudrate， serial_init，console_init_f，         display_banner， dram_init。


接下是board_init_f的后半段函数内容:
{% highlight c %}
	debug("monitor len: %08lX\n", gd->mon_len);
	/*
	 * Ram is setup, size stored in gd !!
	 */
	debug("ramsize: %08lX\n", gd->ram_size);
#if defined(CONFIG_SYS_MEM_TOP_HIDE)
	/*
	 * Subtract specified amount of memory to hide so that it won't
	 * get "touched" at all by U-Boot. By fixing up gd->ram_size
	 * the Linux kernel should now get passed the now "corrected"
	 * memory size and won't touch it either. This should work
	 * for arch/ppc and arch/powerpc. Only Linux board ports in
	 * arch/powerpc with bootwrapper support, that recalculate the
	 * memory size from the SDRAM controller setup will have to
	 * get fixed.
	 */
	gd->ram_size -= CONFIG_SYS_MEM_TOP_HIDE;
#endif

  /*这里addr是内存的高地址，0x20000000+256M
        关于gd->ram_size,在初始化函数dram_init中，已经给过值了 */
	addr = CONFIG_SYS_SDRAM_BASE + gd->ram_size;

#ifdef CONFIG_LOGBUFFER
#ifndef CONFIG_ALT_LB_ADDR
	/* reserve kernel log buffer */
	addr -= (LOGBUFF_RESERVE);
	debug("Reserving %dk for kernel logbuffer at %08lx\n", LOGBUFF_LEN,
		addr);
#endif
#endif

#ifdef CONFIG_PRAM
	/*
	 * reserve protected RAM
	 */
	i = getenv_r("pram", (char *)tmp, sizeof(tmp));
	reg = (i > 0) ? simple_strtoul((const char *)tmp, NULL, 10) :
		CONFIG_PRAM;
	addr -= (reg << 10);		/* size is in kB */
	debug("Reserving %ldk for protected RAM at %08lx\n", reg, addr);
#endif /* CONFIG_PRAM */

#if !(defined(CONFIG_SYS_ICACHE_OFF) && defined(CONFIG_SYS_DCACHE_OFF))
	/* reserve TLB table */
	addr -= (4096 * 4);

	/* round down to next 64 kB limit */
	addr &= ~(0x10000 - 1);

	gd->tlb_addr = addr;
	debug("TLB table at: %08lx\n", addr);
#endif

	/* round down to next 4 kB limit */
	addr &= ~(4096 - 1);
	debug("Top of RAM usable for U-Boot at: %08lx\n", addr);

#ifdef CONFIG_LCD
#ifdef CONFIG_FB_ADDR
	gd->fb_base = CONFIG_FB_ADDR;
#else
	/* reserve memory for LCD display (always full pages) */
	addr = lcd_setmem(addr);
	gd->fb_base = addr;
#endif /* CONFIG_FB_ADDR */
#endif /* CONFIG_LCD */

	/*
	 * reserve memory for U-Boot code, data & bss
	 * round down to next 4 kB limit
	 */
	addr -= gd->mon_len;
	addr &= ~(4096 - 1);

	debug("Reserving %ldk for U-Boot at: %08lx\n", gd->mon_len >> 10, addr);

/*宏CONFIG_SPL_BUILD,用ctags找，在spl/Makefile中能找到，如下：
        CONFIG_SPL_BUILD := y
        export CONFIG_SPL_BUILD
        但是从u-boot根目录的Makefile中，可以看到，ALL-$(CONFIG_SPL)，如下：
        ALL-$(CONFIG_SPL) += $(obj)spl/u-boot-spl.bin
        CONFIG_SPL没有定义（include/configs/fsc100.h中没这个宏），因此，相当于spl/Makefile没有执行。*/
#ifndef CONFIG_SPL_BUILD
	/*
	 * reserve memory for malloc() arena
	 */
	addr_sp = addr - TOTAL_MALLOC_LEN;
	debug("Reserving %dk for malloc() at: %08lx\n",
			TOTAL_MALLOC_LEN >> 10, addr_sp);
	/*
	 * (permanently) allocate a Board Info struct
	 * and a permanent copy of the "global" data
	 */
	addr_sp -= sizeof (bd_t);
	bd = (bd_t *) addr_sp;
	gd->bd = bd;
	debug("Reserving %zu Bytes for Board Info at: %08lx\n",
			sizeof (bd_t), addr_sp);

#ifdef CONFIG_MACH_TYPE
	gd->bd->bi_arch_number = CONFIG_MACH_TYPE; /* board id for Linux */
#endif

	addr_sp -= sizeof (gd_t);
	id = (gd_t *) addr_sp;
	debug("Reserving %zu Bytes for Global Data at: %08lx\n",
			sizeof (gd_t), addr_sp);

	/* setup stackpointer for exeptions */
	gd->irq_sp = addr_sp;
#ifdef CONFIG_USE_IRQ
	addr_sp -= (CONFIG_STACKSIZE_IRQ+CONFIG_STACKSIZE_FIQ);
	debug("Reserving %zu Bytes for IRQ stack at: %08lx\n",
		CONFIG_STACKSIZE_IRQ+CONFIG_STACKSIZE_FIQ, addr_sp);
#endif
	/* leave 3 words for abort-stack    */
	addr_sp -= 12;

	/* 8-byte alignment for ABI compliance */
	addr_sp &= ~0x07;
#else
	addr_sp += 128;	/* leave 32 words for abort-stack   */
	gd->irq_sp = addr_sp;
#endif

	debug("New Stack Pointer is: %08lx\n", addr_sp);

#ifdef CONFIG_POST
	post_bootmode_init();
	post_run(NULL, POST_ROM | post_bootmode_get(0));
#endif

	gd->bd->bi_baudrate = gd->baudrate;
	/* Ram ist board specific, so move it to board code ... */
	dram_init_banksize();
	display_dram_config();	/* and display it */

	gd->relocaddr = addr;
	gd->start_addr_sp = addr_sp;
	gd->reloc_off = addr - _TEXT_BASE;
	debug("relocation Offset is: %08lx\n", gd->reloc_off);
	memcpy(id, (void *)gd, sizeof(gd_t));

/* 跳去执行重定位代码 */
	relocate_code(addr_sp, id, addr);

	/* NOTREACHED - relocate_code() does not return */
}

{% endhighlight c %}

relocate_code后uboot的内存图:
![uboot-ram](/picture/uboot-ram.png "uboot-ram")



relocate_code(addr_sp, id, addr)；在start.S中定义，C又回到了汇编
（栈指针，全局数据的地方, 搬后起始地址）
完成了第二次搬运过程，把u-boot从27e00000搬到了addr．

{% highlight c %}
/*
* void relocate_code (addr_sp, gd, addr_moni)
*
* This "function" does not return, instead it continues in RAM
* after relocating the monitor code.
*
*/
          .globl relocate_code
relocate_code:
         mov r4, r0 /* save addr_sp */
         mov r5, r1 /* save addr of gd */
         mov r6, r2          /* save addr of destination */
……
/* Set up the stack */
stack_setup:
         mov sp, r4
adr r0, _start
//得到u-boot的链接地址

cmp r0, r6
//判断u-boot是否已经搬移到最终地址
moveq r9, #0      /* no relocation. relocation offset(r9) = 0 */
beq clear_bss               /* skip relocation */
//若不需要再次搬移，直接清bss段。
mov r1, r6 /* r1 <- scratch for copy_loop */
ldr r3, _image_copy_end_ofs
//需要搬移的u-boot的大小。
add r2, r0, r3 /* r2 <- source end address */
//r0和r2之间的内容全部搬走。

copy_loop:
              ldmia r0!, {r9-r10} /* copy from source address [r0] */
              stmia r1!, {r9-r10} /* copy to target address [r1] */
              cmp r0, r2 /* until source end address [r2] */
              blo copy_loop
//若u-boot需要自搬移，即不在最终地址运行，把u-boot复制到了SDRAM的高端地址

  .....

jump_2_ram:
/*
 * If I-cache is enabled invalidate it
 */
#ifndef CONFIG_SYS_ICACHE_OFF
	mcr	p15, 0, r0, c7, c5, 0	@ invalidate icache
	mcr     p15, 0, r0, c7, c10, 4	@ DSB
	mcr     p15, 0, r0, c7, c5, 4	@ ISB
#endif

//程序从_board_init_r_ofs开始执行,目的是为了跳到board_init_r
	ldr	r0, _board_init_r_ofs
	adr	r1, _start

	/* _board_init_r_ofs + _start = board_init_r */

	add	lr, r0, r1
	add	lr, lr, r9
	/* setup parameters for board_init_r */
	mov	r0, r5		/* gd_t */
	mov	r1, r6		/* dest_addr */
	/* jump to it ... */
	mov	pc, lr             /* 跳转到 board_init_r */

	 /* board_init_r - _start = _board_init_r_ofs */
_board_init_r_ofs:
	.word board_init_r - _start


{% endhighlight c %}






在board_init_r函数中,主要针对开发板自身的外设资源进行初始化,包括串口,网卡,sd卡等
{% highlight c %}

/*
 ************************************************************************
 *
 * This is the next part if the initialization sequence: we are now
 * running from RAM and have a "normal" C environment, i. e. global
 * data can be written, BSS has been cleared, the stack size in not
 * that critical any more, etc.
 *
 ************************************************************************
 */

void board_init_r(gd_t *id, ulong dest_addr)
{
	char *s;
	bd_t *bd;
	ulong malloc_start;
#if !defined(CONFIG_SYS_NO_FLASH)
	ulong flash_size;
#endif

	gd = id;
	bd = gd->bd;

	gd->flags |= GD_FLG_RELOC;	/* tell others: relocation done */

	monitor_flash_len = _end_ofs;

	/* Enable caches */
	enable_caches();

	debug("monitor flash len: %08lX\n", monitor_flash_len);
	board_init();	/* Setup chipselects */

#ifdef CONFIG_SERIAL_MULTI
	serial_initialize();
#endif

	debug("Now running in RAM - U-Boot at: %08lx\n", dest_addr);

#ifdef CONFIG_LOGBUFFER
	logbuff_init_ptrs();
#endif
#ifdef CONFIG_POST
	post_output_backlog();
#endif

	/* The Malloc area is immediately below the monitor copy in DRAM */
	malloc_start = dest_addr - TOTAL_MALLOC_LEN;
	mem_malloc_init (malloc_start, TOTAL_MALLOC_LEN);

#if !defined(CONFIG_SYS_NO_FLASH)
	puts("Flash: ");

	flash_size = flash_init();
	if (flash_size > 0) {
# ifdef CONFIG_SYS_FLASH_CHECKSUM
		print_size(flash_size, "");
		/*
		 * Compute and print flash CRC if flashchecksum is set to 'y'
		 *
		 * NOTE: Maybe we should add some WATCHDOG_RESET()? XXX
		 */
		s = getenv("flashchecksum");
		if (s && (*s == 'y')) {
			printf("  CRC: %08X", crc32(0,
				(const unsigned char *) CONFIG_SYS_FLASH_BASE,
				flash_size));
		}
		putc('\n');
# else	/* !CONFIG_SYS_FLASH_CHECKSUM */
		print_size(flash_size, "\n");
# endif /* CONFIG_SYS_FLASH_CHECKSUM */
	} else {
		puts(failed);
		hang();
	}
#endif

#if defined(CONFIG_CMD_NAND)
	puts("NAND:  ");
	nand_init();		/* go init the NAND */
#endif

#if defined(CONFIG_CMD_ONENAND)
	onenand_init();
#endif

#ifdef CONFIG_GENERIC_MMC
       puts("MMC:   ");
       mmc_initialize(bd);
#endif

#ifdef CONFIG_HAS_DATAFLASH
	AT91F_DataflashInit();
	dataflash_print_info();
#endif

	/* initialize environment */
	env_relocate();

#if defined(CONFIG_CMD_PCI) || defined(CONFIG_PCI)
	arm_pci_init();
#endif

	/* IP Address */
	gd->bd->bi_ip_addr = getenv_IPaddr("ipaddr");

	stdio_init();	/* get the devices list going. */

	jumptable_init();

#if defined(CONFIG_API)
	/* Initialize API */
	api_init();
#endif

	console_init_r();	/* fully init console as a device */

#if defined(CONFIG_ARCH_MISC_INIT)
	/* miscellaneous arch dependent initialisations */
	arch_misc_init();
#endif
#if defined(CONFIG_MISC_INIT_R)
	/* miscellaneous platform dependent initialisations */
	misc_init_r();
#endif

	 /* set up exceptions */
	interrupt_init();
	/* enable exceptions */
	enable_interrupts();

	/* Perform network card initialisation if necessary */
#if defined(CONFIG_DRIVER_SMC91111) || defined (CONFIG_DRIVER_LAN91C96)
	/* XXX: this needs to be moved to board init */
	if (getenv("ethaddr")) {
		uchar enetaddr[6];
		eth_getenv_enetaddr("ethaddr", enetaddr);
		smc_set_mac_addr(enetaddr);
	}
#endif /* CONFIG_DRIVER_SMC91111 || CONFIG_DRIVER_LAN91C96 */

	/* Initialize from environment */
	s = getenv("loadaddr");
	if (s != NULL)
		load_addr = simple_strtoul(s, NULL, 16);
#if defined(CONFIG_CMD_NET)
	s = getenv("bootfile");
	if (s != NULL)
		copy_filename(BootFile, s, sizeof(BootFile));
#endif

#ifdef BOARD_LATE_INIT
	board_late_init();
#endif

#ifdef CONFIG_BITBANGMII
	bb_miiphy_init();
#endif
#if defined(CONFIG_CMD_NET)
#if defined(CONFIG_NET_MULTI)
	puts("Net:   ");
#endif
	eth_initialize(gd->bd);
	puts("\n ");
	puts("\n");
#if defined(CONFIG_RESET_PHY_R)
	debug("Reset Ethernet PHY\n");
	reset_phy();
#endif
#endif

#ifdef CONFIG_POST
	post_run(NULL, POST_RAM | post_bootmode_get(0));
#endif

#if defined(CONFIG_PRAM) || defined(CONFIG_LOGBUFFER)
	/*
	 * Export available size of memory for Linux,
	 * taking into account the protected RAM at top of memory
	 */
	{
		ulong pram;
		uchar memsz[32];
#ifdef CONFIG_PRAM
		char *s;

		s = getenv("pram");
		if (s != NULL)
			pram = simple_strtoul(s, NULL, 10);
		else
			pram = CONFIG_PRAM;
#else
		pram = 0;
#endif
#ifdef CONFIG_LOGBUFFER
#ifndef CONFIG_ALT_LB_ADDR
		/* Also take the logbuffer into account (pram is in kB) */
		pram += (LOGBUFF_LEN + LOGBUFF_OVERHEAD) / 1024;
#endif
#endif
		sprintf((char *)memsz, "%ldk", (gd->ram_size / 1024) - pram);
		setenv("mem", (char *)memsz);
	}
#endif


/*
 * 进入主循环,也就说,U-Boot 的命令状态,
* 其实就是个死循环状态,它会不断地检测用户
* 是否有命令输入,然后分析输入的命令是什么,
* 最后执行输入命令对应的函数,之后又再继续地循环
*/

	/* main_loop() can return to retry autoboot, if so just run it again. */
	for (;;) {
		main_loop();
	}

	/* NOTREACHED - no way out of command loop except booting */
}

{% endhighlight c %}
这个main_loop函数定义于`common/main.c`

{% highlight c %}
/****************************************************************************/

void main_loop (void)
{
#ifndef CONFIG_SYS_HUSH_PARSER
	static char lastcommand[CONFIG_SYS_CBSIZE] = { 0, };
	int len;
	int rc = 1;
	int flag;
#endif

#if defined(CONFIG_BOOTDELAY) && (CONFIG_BOOTDELAY >= 0)
	char *s;
	int bootdelay;
#endif
#ifdef CONFIG_PREBOOT
	char *p;
#endif
#ifdef CONFIG_BOOTCOUNT_LIMIT
	unsigned long bootcount = 0;
	unsigned long bootlimit = 0;
	char *bcs;
	char bcs_set[16];
#endif /* CONFIG_BOOTCOUNT_LIMIT */

#ifdef CONFIG_BOOTCOUNT_LIMIT
	bootcount = bootcount_load();
	bootcount++;
	bootcount_store (bootcount);
	sprintf (bcs_set, "%lu", bootcount);
	setenv ("bootcount", bcs_set);
	bcs = getenv ("bootlimit");
	bootlimit = bcs ? simple_strtoul (bcs, NULL, 10) : 0;
#endif /* CONFIG_BOOTCOUNT_LIMIT */

#ifdef CONFIG_MODEM_SUPPORT
	debug ("DEBUG: main_loop:   do_mdm_init=%d\n", do_mdm_init);
	if (do_mdm_init) {
		char *str = strdup(getenv("mdm_cmd"));
		setenv ("preboot", str);  /* set or delete definition */
		if (str != NULL)
			free (str);
		mdm_init(); /* wait for modem connection */
	}
#endif  /* CONFIG_MODEM_SUPPORT */

#ifdef CONFIG_VERSION_VARIABLE
	{
		setenv ("ver", version_string);  /* set version variable */
	}
#endif /* CONFIG_VERSION_VARIABLE */

#ifdef CONFIG_SYS_HUSH_PARSER
	u_boot_hush_start ();
#endif

#if defined(CONFIG_HUSH_INIT_VAR)
	hush_init_var ();
#endif

#ifdef CONFIG_PREBOOT
	if ((p = getenv ("preboot")) != NULL) {
# ifdef CONFIG_AUTOBOOT_KEYED
		int prev = disable_ctrlc(1);	/* disable Control C checking */
# endif

# ifndef CONFIG_SYS_HUSH_PARSER
		run_command (p, 0);
# else
		parse_string_outer(p, FLAG_PARSE_SEMICOLON |
				    FLAG_EXIT_FROM_LOOP);
# endif

# ifdef CONFIG_AUTOBOOT_KEYED
		disable_ctrlc(prev);	/* restore Control C checking */
# endif
	}
#endif /* CONFIG_PREBOOT */

#if defined(CONFIG_UPDATE_TFTP)
	update_tftp (0UL);
#endif /* CONFIG_UPDATE_TFTP */

#if defined(CONFIG_BOOTDELAY) && (CONFIG_BOOTDELAY >= 0)
	s = getenv ("bootdelay");
	bootdelay = s ? (int)simple_strtol(s, NULL, 10) : CONFIG_BOOTDELAY;

	debug ("### main_loop entered: bootdelay=%d\n\n", bootdelay);

# ifdef CONFIG_BOOT_RETRY_TIME
	init_cmd_timeout ();
# endif	/* CONFIG_BOOT_RETRY_TIME */

#ifdef CONFIG_POST
	if (gd->flags & GD_FLG_POSTFAIL) {
		s = getenv("failbootcmd");
	}
	else
#endif /* CONFIG_POST */
#ifdef CONFIG_BOOTCOUNT_LIMIT
	if (bootlimit && (bootcount > bootlimit)) {
		printf ("Warning: Bootlimit (%u) exceeded. Using altbootcmd.\n",
		        (unsigned)bootlimit);
		s = getenv ("altbootcmd");
	}
	else
#endif /* CONFIG_BOOTCOUNT_LIMIT */
		s = getenv ("bootcmd");

	debug ("### main_loop: bootcmd=\"%s\"\n", s ? s : "<UNDEFINED>");

	if (bootdelay >= 0 && s && !abortboot (bootdelay)) {
# ifdef CONFIG_AUTOBOOT_KEYED
		int prev = disable_ctrlc(1);	/* disable Control C checking */
# endif

# ifndef CONFIG_SYS_HUSH_PARSER
		run_command (s, 0);
# else
		parse_string_outer(s, FLAG_PARSE_SEMICOLON |
				    FLAG_EXIT_FROM_LOOP);
# endif

# ifdef CONFIG_AUTOBOOT_KEYED
		disable_ctrlc(prev);	/* restore Control C checking */
# endif
	}

# ifdef CONFIG_MENUKEY
	if (menukey == CONFIG_MENUKEY) {
		s = getenv("menucmd");
		if (s) {
# ifndef CONFIG_SYS_HUSH_PARSER
			run_command(s, 0);
# else
			parse_string_outer(s, FLAG_PARSE_SEMICOLON |
						FLAG_EXIT_FROM_LOOP);
# endif
		}
	}
#endif /* CONFIG_MENUKEY */
#endif /* CONFIG_BOOTDELAY */

/*添加MENU菜单 make by nietao email: nietaooldman@126.com */
#ifdef CONFIG_CMD_MENU
	run_command("menu", 0);
#endif


	/*
	 * Main Loop for Monitor Command Processing
	 */
#ifdef CONFIG_SYS_HUSH_PARSER
	parse_file_outer();
	/* This point is never reached */
	for (;;);
#else
	for (;;) {
#ifdef CONFIG_BOOT_RETRY_TIME
		if (rc >= 0) {
			/* Saw enough of a valid command to
			 * restart the timeout.
			 */
			reset_cmd_timeout();
		}
#endif
		len = readline (CONFIG_SYS_PROMPT);

		flag = 0;	/* assume no special flags for now */
		if (len > 0)
			strcpy (lastcommand, console_buffer);
		else if (len == 0)
			flag |= CMD_FLAG_REPEAT;
#ifdef CONFIG_BOOT_RETRY_TIME
		else if (len == -2) {
			/* -2 means timed out, retry autoboot
			 */
			puts ("\nTimed out waiting for command\n");
# ifdef CONFIG_RESET_TO_RETRY
			/* Reinit board to run initialization code again */
			do_reset (NULL, 0, 0, NULL);
# else
			return;		/* retry autoboot */
# endif
		}
#endif

		if (len == -1)
			puts ("<INTERRUPT>\n");
		else
			rc = run_command (lastcommand, flag);

		if (rc <= 0) {
			/* invalid command or not repeatable, forget it */
			lastcommand[0] = 0;
		}
	}
#endif /*CONFIG_SYS_HUSH_PARSER*/
}

{% endhighlight c %}
