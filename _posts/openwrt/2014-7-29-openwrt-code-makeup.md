---
layout: post
title: "openWRT的源码结构 "
date: 2014-7-29 15:25
category: "openwrt"
tags : "openwrt"
---

###openWRT的源码的获取
下载的openWRT版本是Dreambox的openosom,因为我打算将openWRT移植到webee210(s5pv210)，而openosom是针对嵌入式有做过优化，而且支持s5pv210.openosom是从openWRT官方的backfire分支演变来的，由Dreambox团队维护。详情看这里:![openosm-blog](http://www.openosom.org/).
获取代码:

	$ git clone git://github.com/openosom/backfire_10.03.1 openosom

or

	$ svn co svn://svn.openwrt.org.cn/dreambox/branches/openosom openosom

###openWRT的目录框架

代码上来看有几个重要目录package, target, build_root, bin, dl....

---build_dir/host目录是建立工具链时的临时目录

---build_dir/toolchain-<arch>*是对应硬件的工具链的目录

---staging_dir/toolchain-<arch>* 则是工具链的安装位置

---target/linux/<platform>目录里面是各个平台(arch)的相关代码

---target/linux/<platform>/config-3.10文件就是配置文件了

---dl目录是'download'的缩写, 在编译前期，需要从网络下载的数据包都会放在这个目录下，这些软件包的一个特点就是，会自动安装在所编译的固件中，也就是我们make menuconfig的时候，为固件配置的一些软件包。如果我们需要更改这些源码包，只需要将更改好的源码包打包成相同的名字放在这个目录下，然后开始编译即可。编译时，会将软件包解压到build_dir目录下。

---而在build_dir/目录下进行解压，编译和打补丁等。

---package目录里面包含了我们在配置文件里设定的所有编译好的软件包。默认情况下，会有默认选择的软件包。在openwrt中ipk就是一切, 我们可以使用

###编译openWRT


编译内核:

`make -C /home/webee-wrt/openWRT/backfire_10.03.1/build_dir/linux-s5pc1xx_webee210/linux-2.6.35.7 CROSS_COMPILE="arm-openwrt-linux-gnueabi-" ARCH="arm" KBUILD_HAVE_NLS=no CONFIG_SHELL="/bin/bash" CC="arm-openwrt-linux-gnueabi-gcc" oldconfig prepare scripts`


修改uimage的入口地址:

`/home/izobs/virtualbox-share/openWRT/backfire_10.03.1/target/linux/s5pc1xx/image/Makefile`

查看uimage信息:

	mkimage -l  xxx
