---
layout: post
title: "openWRT telnet登陆 "
date: 2014-8-17 15:25
category: "openwrt"
tags : "openwrt"
---

###openWRT的telnet登陆
移植的openWRT在登陆的时候出现无法登陆的问题:

```
[root@localhost dnw]# telnet 192.168.1.1
Trying 192.168.1.1...
telnet: connect to address 192.168.1.1: Connection refused

```

对openWRT和telnet完全不懂，只能去raspberry pi的openWRT镜像寻找灵感。最后发现是自己的telnet服务没开启导致的 = =。openWRT官方怎么不默认开启呢？

根据raspberry pi在/etc/init.d/telnet上做修改,
`raspberry pi`

<pre class="prettyprint" id="c++">

start() {
	    if ( ! has_ssh_pubkey && \
				         ! has_root_pwd /etc/passwd && ! has_root_pwd /etc/shadow ) || \
			       ( ! /etc/init.d/dropbear enabled 2> /dev/null && ! /etc/init.d/sshd enabled 2> /dev/null );
		    then
				        service_start /usr/sbin/telnetd -l /bin/login.sh
						    fi
}

</pre>
`webee210-WRT`
<pre class="prettyprint" id="bash">

start() {                                       
	        if ( ! has_root_pwd /etc/passwd && ! has_root_pwd /etc/shadow ) || \
	           ( [ ! -x /usr/sbin/dropbear ] && [ ! -x /usr/sbin/sshd ] );      
			        then																			 
						/usr/sbin/telnetd -l /bin/login.sh                     
					fi           
}   

</pre>
没有反应，只能在/ect/init.de/rcs中添加:

        /usr/sbin/telnetd -l /bin/login.sh 

结果成功登陆


