---
layout: post
title: "openWRT-usb wifi AP 上网"
date: 2014-8-27 15:25
category: "openwrt"
tags : "openwrt"
---

###配置dnsmasq
DNSmasq是一个小巧且方便地用于配置DNS和DHCP的工具，适用于小型网络，它提供了DNS功能和可选择的DHCP功能。它服务那些只在本地适用的域名，这些域名是不会在全球的DNS服务器中出现的。DHCP服务器和DNS服务器结合，并且允许DHCP分配的地址能在DNS中正常解析，而这些DHCP分配的地址和相关命令可以配置到每台主机中，也可以配置到一台核心设备中（比如路由器），DNSmasq支持静态和动态两种DHCP配置方式。

首先你需要在make menuconfig 上面将dnsmasq package添加,编译好后在/etc/init.d/rcS中启动dnsmasp服务           

{% highlight bash %}

/etc/init.d/dnsmasq start

{% endhighlight %} 

启动不了，提示的erro是           

{% highlight bash %}

cant't open /var/run/dnsmasq.pid

{% endhighlight %} 


所以需要touch /var/run/dnsmasq.pid 

{% highlight bash %}

mkdir -p /var/run
cd /var/run      
touch dnsmasq.pid
cd / 

{% endhighlight %} 


当你将无线网卡挂载好，接入wifi热点就会自动分配一个ip地址了。

{% highlight bash %}
ifconfig ra0 192.168.1.1 netmask 255.255.255.0 up
{% endhighlight %} 


###桥接ra0和eth0

eth0连接上级路由，可以上网。而ra0通过eth0也能上网，这样连接到ra0的设备就都可以上网了。这个功能是通过iptable的nat桥接功能实现。

iptable 和 NAT桥接功能需要在linux 内核开启相关的配置，主要是Netfilter。下面是我们的Network 模块的配置:
`Networking support -> Networking options`将IP字段的开头的都选上，还有WAN router。同个界面找到Netfilter进入,把里面的都选上。都选用[*]

在openWRT的make menuconfig需要选择iptable工具和libip4tc依赖库。   

openWRT并没有将iptable 桥接功能独立开来，而是将其整合到系统的firewall配置里面，所以我们需要找到openWRT firewall的配置原理.
[wiki](http://wiki.openwrt.org/doc/uci/firewall)

简单的说，我们这个桥接功能是属于转发的功能,我们只需如何配置使其生效即可。在/etc/config/firewall.我的配置如下:

{% highlight bash %}

//这个zone区是lan
config 'zone'                    
        option 'name' 'lan'      
        option 'network' 'lan'   
        option 'input' 'ACCEPT'
        option 'output' 'ACCEPT'
        option 'forward' 'ACCEPT'
//这个zone区是wan
                                 
config 'zone'                    
        option 'name' 'wan'      
        option 'network' 'wan'   
        option 'input' 'REJECT'
        option 'output' 'ACCEPT'
        option 'forward' 'REJECT'
        option 'masq' '1'        
        option 'mtu_fix' '1'     
   
//这个是转发，src是源头，dest是目的                              
config 'forwarding'              
        option 'src' 'lan'  
        option 'dest' 'wan'


{% endhighlight %} 
下面看我的/etc/config/network的配置是如何的:


{% highlight bash %}
config interface loopback
        option ifname   lo
        option proto    static
        option ipaddr   127.0.0.1
        option netmask  255.0.0.0
//这个是我的有线网卡，作为wan
config interface wan
        option ifname 'eth0'
        option proto 'dhcp'
        option peerdns '0'
        option dns '114.114.114.114 8.8.8.8'

//这个是我的无线wifi口,作为lan
config interface lan
        option ifname 'ra0'
        option proto 'static'
        option ipaddr   192.168.1.1
        option netmask  255.255.255.0


然后启动firewall 服务，就可以转发了。

{% endhighlight %} 

{% highlight bash %}

echo 1 >/proc/sys/net/ipv4/ip_forward
/etc/init.d/firewall restart

{% endhighlight %} 
连接wifi热点，ping 8.8.8.8试试。


	

