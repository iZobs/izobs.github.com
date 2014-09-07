---
layout: post
title: "openWRT-RT3070驱动的移植"
date: 2014-8-26 15:25
category: "openwrt"
tags : "openwrt"
---

###RT3070驱动的获取
RT3070的驱动源码有sta和softap两种，而RT3070的驱动源码在Ralink官网不好弄，在CSDN花了3个大洋下载了RT3070的驱动源码.这边把SotfAp修改后的源码推送到github，跟大家分享:

```
$ git clone https://github.com/iZobs/RT3070_SoftAP.git

```
因为我是想把这个驱动移植到我的webee210-WRT上，所以我把代码已经修改好了。下面说说都修改了什么地方。

###修改编译配置
__1.修改Makefile__

`注意：我们需要修改的MODULE,NETIF,UTIL,目录下的Makfile,修改的内容是一样的`

```
#PLATFORM: Target platform
#PLATFORM = PC
#PLATFORM = 5VT
#PLATFORM = IKANOS_V160
#PLATFORM = IKANOS_V180
#PLATFORM = SIGMA
#PLATFORM = SIGMA_8622
#PLATFORM = INIC
#PLATFORM = STAR
PLATFORM = IXP
#PLATFORM = INF_TWINPASS
#PLATFORM = INF_DANUBE
#PLATFORM = INF_AR9
#PLATFORM = BRCM_6358
#PLATFORM = INF_AMAZON_SE
#PLATFORM = CAVM_OCTEON
#PLATFORM = CMPC
#PLATFORM = RALINK_2880
#PLATFORM = RALINK_3052
#PLATFORM = SMDK
#PLATFORM = KODAK_DC
#PLATFORM = DM6446
#PLATFORM = FREESCALE8377

.....

ifeq ($(PLATFORM),IXP)
	LINUX_SRC = /home/webee-wrt/openWRT/webee210-WRT/Webee210-WRT/backfire_10.03.1/build_dir/linux-s5pc1xx_webee210/linux-2.6.35.7
	CROSS_COMPILE = /home/webee-wrt/OpenWrt-Toolchain-s5pc1xx-for-arm_v7-a-gcc-4.3.3+cs_eglibc-2.8_eabi/toolchain-arm_v7-a_gcc-4.3.3+cs_eglibc-2.8_eabi/usr/bin/arm-openwrt-linux-gnueabi-
	endif
	 
```

`LINUX_SRC`为你的编译好的内核的路径
`CROSS_COMPILE`为你的交叉编译器的路径

__2.修改config.mk__
如果你用我的src，这个文件则不用修改了，因为我已经改好了，这个文件注意是修改编译选项的BIG_ENDIAN选项,因为arm是小尾系统，所以我们需要这个地方。
`注意：我们需要修改的MODULE,NETIF,UTIL,目录下的os/linux下的config.mk`

```
feq ($(PLATFORM),RMI)
	WFLAGS += -DRT_BIG_ENDIAN
	endif

	ifeq ($(PLATFORM),RMI_64)
	WFLAGS += -DRT_BIG_ENDIAN
	endif
	ifeq ($(PLATFORM),IXP)                                                     
#WFLAGS += -DRT_BIG_ENDIAN            //注释掉这里
	endif

....

//把这段代码修改如下，去掉-mbig-endian选项

ifeq ($(PLATFORM),IXP)                                                     
	    CFLAGS := -v -D__KERNEL__ -DMODULE -I$(LINUX_SRC)/include -I$(RT28xx_DIR)/include -Wall -Wstrict-prototypes -Wno-trigraphs -O2 -fno-strict-aliasing -fno-common -Uarm -fno-common -pipe -mapcs-32 -D__LINUX_ARM_ARCH__=5 -mcpu=xscale -mtune=xscale -malignment-traps -msoft-float $(WFLAGS)
	        EXTRA_CFLAGS := -v $(WFLAGS) -I$(RT28xx_DIR)/include
			    export CFLAGS
				endif

				ifeq ($(PLATFORM),SMDK)
	        EXTRA_CFLAGS := $(WFLAGS) -I$(RT28xx_DIR)/include
			endif
```


__3.修改usb_main_dev.c__

在开头加上: MODULE_LICENSE("GPL");

__4.修改rt_usb_util.c__

找出AP驱动中的rt_usb_util.c文件，将`usb_buffer_alloc`和`usb_buffer_free`分别改为`usb_alloc_coherent`和 `usb_free_coherent`。

###编译安装
__1.编译__

```
$ sudo make ARCH=arm KBUILD_NOPEDANTIC=1

```

__2.安装__
将下面的文件从找MODULE,NETIF,UTIL,目录下的os/linux找出来，放在一个目录叫Wireless,然后把拷贝到你的开发板。                     

```
└── Wireless                   
    ├── RT2870AP.dat                   
    ├── rt3070ap.ko                    
    ├── rtnet3070ap.ko                  
    └── rtutil3070ap.ko                      
```
mkdir /etc/Wireless/RT2870AP/,将RT2870AP.dat存放于此                 

下面加载驱动，一定要按照顺序来哟,于是我用了一个简单脚步                  
`/etc/init.d/setup_ra0`

```
insmod /lib/RT3070/Wireless/rtutil3070ap.ko
insmod /lib/RT3070/Wireless/rt3070ap.ko
insmod /lib/RT3070/Wireless/rtnet3070ap.ko
echo "insmod rt3070 done"
ifconfig ra0 up

```
成功挂载后，如下:

```
[   40.275000] rt3070ap: module license 'RALINK' taints kernel.
[   40.275000] Disabling lock debugging due to kernel taint
[   40.345000] rtusb init --->
[   40.345000] 
[   40.345000] 
[   40.345000] === pAd = a0926000, size = 417992 ===
[   40.345000] 
[   40.345000] <-- RTMPAllocAdapterBlock, Status=0
[   40.350000] usbcore: registered new interface driver rt2870
insmod rt3070 done
[   40.615000] <-- RTMPAllocTxRxRingMemory, Status=0
[   40.615000] -->RTUSBVenderReset
[   40.615000] <--RTUSBVenderReset
[   40.900000] Key1Str is Invalid key length(0) or Type(0)
[   40.900000] Key2Str is Invalid key length(0) or Type(0)
[   40.900000] Key3Str is Invalid key length(0) or Type(0)
[   40.900000] Key4Str is Invalid key length(0) or Type(0)
[   40.905000] 1. Phy Mode = 9
[   40.905000] 2. Phy Mode = 9
[   40.905000] NVM is Efuse and its size =2d[2d0-2fc] 
[   40.990000] 3. Phy Mode = 9
[   41.045000] MCS Set = ff 00 00 00 01
[   41.075000] SYNC - BBP R4 to 20MHz.l
[   41.385000] SYNC - BBP R4 to 20MHz.l
[   41.695000] SYNC - BBP R4 to 20MHz.l
[   42.005000] SYNC - BBP R4 to 20MHz.l
[   42.315000] SYNC - BBP R4 to 20MHz.l
[   42.630000] SYNC - BBP R4 to 20MHz.l
[   42.940000] SYNC - BBP R4 to 20MHz.l
[   43.250000] SYNC - BBP R4 to 20MHz.l
[   43.775000] Main bssid = 98:3f:9f:23:c4:28
[   43.775000] <==== rt28xx_init, Status=0
[   43.780000] 0x1300 = 00064320

```

于是我们的PC就可以搜索到一个默认叫RT2860AP的wifi热点:
![wifi](/picture/wifi.png)
