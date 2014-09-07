---

layout: post
title: "uboot的spl框架"
date: 2014-3-18 19:34
category: "学习"
comments: false
tags : "uboot"

---
##1.uboot SPL架构简介

在uboot-2011之后的版本中多了一个叫SPL的架构，这个架构有什么用呢？在uboot-2011的/doc/README.spl文件有简单的介绍：

    Generic SPL framework
    =====================
    
    Overview
    --------
    
    To unify all existing implementations for a secondary program loader (SPL)
    and to allow simply adding of new implementations this generic SPL framework
    has been created. With this framework almost all source files for a board
    can be reused. No code duplication or symlinking is necessary anymore.
    
    为了统一所有现有实现第二段的程序加载程序(SPL)并允许简单地添加新的实现这个，一个通用SPL框架已创建。用这个框架板的几乎所有的源文件可以重用。没有代码重复或符号链接是必要的了。

    
    How it works
    ------------
    
    There is a new directory TOPDIR/spl which contains only a Makefile.
    The object files are built separately for SPL and placed in this directory.
    The final binaries which are generated are u-boot-spl, u-boot-spl.bin and
    u-boot-spl.map.
    
    During the SPL build a variable named CONFIG_SPL_BUILD is exported
    in the make environment and also appended to CPPFLAGS with -DCONFIG_SPL_BUILD.
    Source files can therefore be compiled for SPL with different settings.
    ARM-based boards have previously used the option CONFIG_PRELOADER for it.
    
    For example:
    
    ifeq ($(CONFIG_SPL_BUILD),y)
    COBJS-y += board_spl.o
    else
    COBJS-y += board.o
    endif
    
    COBJS-$(CONFIG_SPL_BUILD) += foo.o
    
    #ifdef CONFIG_SPL_BUILD
            foo();
    #endif
    
    
    The building of SPL images can be with:
    
    #define CONFIG_SPL
    
    Because SPL images normally have a different text base, one have to be
    configured by defining CONFIG_SPL_TEXT_BASE. The linker script have to be
    defined with CONFIG_SPL_LDSCRIPT.
    
    To support generic U-Boot libraries and drivers in the SPL binary one can
    optionally define CONFIG_SPL_XXX_SUPPORT. Currently following options
    are supported:
    
    CONFIG_SPL_LIBCOMMON_SUPPORT (common/libcommon.o)
    CONFIG_SPL_LIBDISK_SUPPORT (disk/libdisk.o)
    CONFIG_SPL_I2C_SUPPORT (drivers/i2c/libi2c.o)
    CONFIG_SPL_GPIO_SUPPORT (drivers/gpio/libgpio.o)
    CONFIG_SPL_MMC_SUPPORT (drivers/mmc/libmmc.o)
    CONFIG_SPL_SERIAL_SUPPORT (drivers/serial/libserial.o)
    CONFIG_SPL_SPI_FLASH_SUPPORT (drivers/mtd/spi/libspi_flash.o)
    CONFIG_SPL_SPI_SUPPORT (drivers/spi/libspi.o)
    CONFIG_SPL_FAT_SUPPORT (fs/fat/libfat.o)
    CONFIG_SPL_LIBGENERIC_SUPPORT (lib/libgeneric.o)
    
    
移植过uboot的都知道，uboot的启动其实是分为BL0,BL1,BL2三个阶段的，即：ROM->SPL->uboot.img.而这个SPL架构将可以编译产生一个uboot-spl.bin。即BL1的代码。也就是说SPL结构其实做的工作就是uboot的BL1阶段的工作。

##2.SPL结构是如何工作的

cd 进入spl目录，从编译文件看这个架构都包括了什么文件：
我们先来看该目录Makefile：
 当**CONFIG_SPL_BUILD := y** driver下设备驱动和arch/arm与cup相关的东东将被编译。
    CONFIG_SPL_BUILD := y
    export CONFIG_SPL_BUILD
    
    include $(TOPDIR)/config.mk
    
    # We want the final binaries in this directory
    obj := $(OBJTREE)/spl/
    
        HAVE_VENDOR_COMMON_LIB := $(shell [ -f $(SRCTREE)/board/$(VENDOR)/common/Makefile ] \
        			&& echo y || echo n)
    
    START := $(CPUDIR)/start.o
    
    LIBS-y += arch/$(ARCH)/lib/lib$(ARCH).o
    LIBS-y += $(CPUDIR)/lib$(CPU).o
    ifdef SOC
    LIBS-y += $(CPUDIR)/$(SOC)/lib$(SOC).o
    endif
    LIBS-y += board/$(BOARDDIR)/lib$(BOARD).o
    LIBS-$(HAVE_VENDOR_COMMON_LIB) += board/$(VENDOR)/common/lib$(VENDOR).o
    
    LIBS-$(CONFIG_SPL_LIBCOMMON_SUPPORT) += common/libcommon.o
    LIBS-$(CONFIG_SPL_LIBDISK_SUPPORT) += disk/libdisk.o
    LIBS-$(CONFIG_SPL_I2C_SUPPORT) += drivers/i2c/libi2c.o
    LIBS-$(CONFIG_SPL_GPIO_SUPPORT) += drivers/gpio/libgpio.o
    LIBS-$(CONFIG_SPL_MMC_SUPPORT) += drivers/mmc/libmmc.o
    LIBS-$(CONFIG_SPL_SERIAL_SUPPORT) += drivers/serial/libserial.o
    LIBS-$(CONFIG_SPL_SPI_FLASH_SUPPORT) += drivers/mtd/spi/libspi_flash.o
    LIBS-$(CONFIG_SPL_SPI_SUPPORT) += drivers/spi/libspi.o
    LIBS-$(CONFIG_SPL_FAT_SUPPORT) += fs/fat/libfat.o
    LIBS-$(CONFIG_SPL_LIBGENERIC_SUPPORT) += lib/libgeneric.o

接下来是连接到一个脚本，u-boot-spl.lds

        # Linker Script
        ifdef CONFIG_SPL_LDSCRIPT
        # need to strip off double quotes
        LDSCRIPT := $(addprefix $(SRCTREE)/,$(subst ",,$(CONFIG_SPL_LDSCRIPT)))
        endif
        
        ifeq ($(wildcard $(LDSCRIPT)),)
        	LDSCRIPT := $(TOPDIR)/board/$(BOARDDIR)/u-boot-spl.lds
        endif
        ifeq ($(wildcard $(LDSCRIPT)),)
        	LDSCRIPT := $(TOPDIR)/$(CPUDIR)/u-boot-spl.lds
        endif
        ifeq ($(wildcard $(LDSCRIPT)),)
        $(error could not find linker script)
        endif

在arm/cpu/armv7目录下找到`u-boot-spl.lds`这个连接脚本
    
    MEMORY { .sram : ORIGIN = CONFIG_SPL_TEXT_BASE,\
    		LENGTH = CONFIG_SPL_MAX_SIZE }
    MEMORY { .sdram : ORIGIN = CONFIG_SPL_BSS_START_ADDR, \
    		LENGTH = CONFIG_SPL_BSS_MAX_SIZE }
    
    OUTPUT_FORMAT("elf32-littlearm", "elf32-littlearm", "elf32-littlearm")
    OUTPUT_ARCH(arm)
    ENTRY(_start)
    SECTIONS
    {
    	.text      :
    	{
    	__start = .;
    	  arch/arm/cpu/armv7/start.o	(.text)
    	  *(.text*)
    	} >.sram
    
    	. = ALIGN(4);
    	.rodata : { *(SORT_BY_ALIGNMENT(.rodata*)) } >.sram
    
    	. = ALIGN(4);
    	.data : { *(SORT_BY_ALIGNMENT(.data*)) } >.sram
    	. = ALIGN(4);
    	__image_copy_end = .;
    	_end = .;
    
    	.bss :
    	{
    		. = ALIGN(4);
    		__bss_start = .;
    		*(.bss*)
    		. = ALIGN(4);
    		__bss_end__ = .;
    	} >.sdram
    }

  其实这个连接脚本最后连接到还是`arch/arm/cpu/armv7/start.S`,作为他的入口。
  
  (reset) <arch/arm/cpu/armv7/start.S> (b lowlevel_init: arch/arm/cpu/armv7/lowlevel_init.S) (b _main) --> <arch/arm/lib/crt0.S> (bl board_init_f) --> <arch/arm/lib/spl.c> (board_init_r) --> <common/spl/spl.c> (jump_to_image_no_args去启动u-boot) 到此SPL的生命周期结束。
    
    


    

