---
layout: post
title: "openWRT 连接路由上网 "
date: 2014-8-16 15:25
category: "openwrt"
tags : "openwrt"
---

###openWRT的上网配置

将网口与路由连接，
首先修改/etc/config/network
<pre class="prettyprint" id="bash">

config interface net
        option ifname "eth0"
		option proto "dhcp"
</pre>

然后再修改/etc/init.d/rcs
<pre class="prettyprint" id="bash">

/etc/init.d/network reload

</pre>
测试：ping www.baidu.com 是无法ping通的，因为系统没配置dns服务。只能ping其ip 地址：
<pre class="prettyprint" id="bash">

root@(none):/etc/init.d# ping 202.108.22.5
			 PING 202.108.22.5 (202.108.22.5): 56 data bytes
		   64 bytes from 202.108.22.5: seq=0 ttl=51 time=41.307 ms
	       64 bytes from 202.108.22.5: seq=1 ttl=51 time=36.746 ms
		   64 bytes from 202.108.22.5: seq=2 ttl=51 time=44.290 ms
</pre>
