---
layout: post
title: "linux开启鼠标驱动和qt鼠标支持"
date: 2014-5-25 20:50
category: "linux"
tags : "driver"
---

###内核开启配置
在此之前，保证USB主设备驱动已经开启了

Dvice Driver-->> HID Support-->>USB HID Support

![mouse1.png](/picture/mouse1.png)
![mouse1.png](/picture/mouse1.png)

设置成功后，接入鼠标，显示如下。

	[Webee210]\# usb 1-1: USB disconnect, device number 2
	usb 1-1: new low-speed USB device number 3 using s5p-ohci
	usb 1-1: New USB device found, idVendor=093a, idProduct=2510
	usb 1-1: New USB device strings: Mfr=1, Product=2, SerialNumber=0
	usb 1-1: Product: USB Optical Mouse
	usb 1-1: Manufacturer: PixArt
	input: PixArt USB Optical Mouse as /devices/platform/s5p-ohci/usb1/1-1/1-1:1.0/input/input2
	hid-generic 0003:093A:2510.0002: input: USB HID v1.11 Mouse [PixArt USB Optical Mouse] on usb-s5p-ohci-1/input0


###qt支持鼠标和触摸屏
在`/ect/profile`中配置这句话，其中，`/dev/mouse0`为插入的鼠标设备节点。

> export QWS_MOUSE_PROTO="intelliMouse:/dev/mouse0 tslib:/dev/event0"

_注意：要使qt先支持鼠标，在开机之前，启动qt程序之前就需要把鼠标设备接入到开发板中_

