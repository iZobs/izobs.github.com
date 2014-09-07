---
layout: post
title: "openWRT luci登陆 "
date: 2014-8-19 15:25
category: "openwrt"
tags : "openwrt"
---

###openWRT的luci配置

修改feed.conf
{% highlight text %}

src-svn packages svn://svn.openwrt.org.cn/dreambox/feeds/packages_10.03.2
src-svn qpe svn://svn.openwrt.org.cn/dreambox/feeds/qpe
src-svn device svn://svn.openwrt.org.cn/dreambox/feeds/device
src-svn luci http://svn.luci.subsignal.org/luci/tags/0.10.0/contrib/package
`src-svn dreambox_packages svn://svn.openwrt.org.cn/dreambox/feeds/dreambox_packages`
#src-svn xwrt http://x-wrt.googlecode.com/svn/branches/backfire_10.03/package
#src-svn phone svn://svn.openwrt.org/openwrt/feeds/phone
#src-svn efl svn://svn.openwrt.org/openwrt/feeds/efl
#src-svn desktop svn://svn.openwrt.org/openwrt/feeds/desktop
#src-svn xfce svn://svn.openwrt.org/openwrt/feeds/xfce
#src-link custom /usr/src/openwrt/custom-feed

## Android for Goldfish (Android Emulator)
#src-git android git://nbd.name/borg.git
#src-svn android16 http://openwrt-for-embedded.googlecode.com/svn/feeds/android16

{% endhighlight %} 

如果不修改，很多软件包都会无法下载，最后构造成的openwrtxxx-yaffs.img就会启动不了luci等很多问题，这个问题然后我panic了一周。

更新openwrt：
{% highlight bash %}

./scripts/feeds update -a

{% endhighlight %} 

安装所有包:
{% highlight bash %}

./scripts/feeds install -a

{% endhighlight %} 

这个时候，我觉得有必要把LUCI编译进内核，这样就方便以后我们通过web来控制openwrt，而不是通过命令行来控制了。所以加上以下命令:
{% highlight bash %}

./scripts/feeds update packages luci

./scripts/feeds install -a -p luci

{% endhighlight %} 

这样之后，在编译内核时才会出现LUci选项。

修改/etc/init.d/rCS
{% highlight bash %}

/sbin/ifconfig eth0 192.168.1.1 netmask 255.255.255.0 up

/etc/init.d/uhttpd enable
/etc/init.d/uhttpd start

{% endhighlight %} 
