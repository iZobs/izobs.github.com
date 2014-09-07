---
layout: post
title: "linux rmmod bug "
date: 2014-6-25 20:25
category: "linux"
tags : "Bug"
---

	###rmmod Bug
	[Webee210]\# rmmod ./mem-funtion-test.ko 
	rmmod: chdir(3.8.0-ge8e9b24): No such file or directory
	[Webee210]\# rmmod /dev/mem-test 
	rmmod: chdir(3.8.0-ge8e9b24): No such file or directory
	[Webee210]\# rmmod *
	rmmod: chdir(3.8.0-ge8e9b24): No such file or directory
	[Webee210]\# rmmod mem-funtion-test    
	rmmod: chdir(3.8.0-ge8e9b24): No such file or directory
	[Webee210]\# rmmod mem-funtion-test.ko 
	rmmod: chdir(3.8.0-ge8e9b24): No such file or directory


##BUG 解决

你必须创建/lib/modules/"uname-r" 这样一个空目录了，否则不能卸载ko模块
但是这样到可以卸载掉模块了，不过会一直有这样一个提示：
     rmmod: module 模块名' not found.

如下：

	[Webee210]\# pwd
	/lib/modules/3.8.0-ge8e9b24
	[Webee210]\# ls
	modules.dep.bb
	[Webee210]\# 
再测试一次：

	[Webee210]\# insmod ./mem-funtion-test.ko 
	mem-test        initialized
	[Webee210]\# rmmod ./mem-funtion-test.ko 
	[Webee210]\# 
