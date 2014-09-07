---
layout: post
title: "在ubuntu安装zsh"
date: 2014-3-30 20:34
category: "工具"
tags : "zsh" 
---

###zsh简介
Linux/Unix提供了很多种Shell.常用的Shell有这么几种，sh、bash、csh等，想知道你的系统有几种shell，可以通过以下命令查看:

<pre class="prettyprint" id="bash">
$ cat /etc/shells
</pre>
显示如下：
<pre class="prettyprint" id="bash">
/bin/sh
/bin/bash
/sbin/nologin
/usr/bin/sh
/usr/bin/bash
/usr/sbin/nologin
/bin/zsh
/usr/bin/zsh
</pre>

`zsh`是一款强大的虚拟终端，既是一个系统的虚拟终端，也可以作为一个脚本语言的交互解析器。它在兼容Bash的同时 (默认不兼容，除非设置成 "emulate sh")，有如下优点:                        

- 强大的自动补全功能
- 丰富的插件，如git等 
- 漂亮的主题支持(_oh-my-zsh_)


###安装zsh
在ubuntu下：
1.安装`zsh`：
<pre class="prettyprint" id="bash">
$ sudo apt-get install zsh
$ sudo apt-get install git
</pre>
2.安装`oh-my-zsh`
<pre class="prettyprint" id="bash">
$ wget https://github.com/robbyrussell/oh-my-zsh/raw/master/tools/install.sh -O - | sh
</pre>

3.配置系统shell为zsh

<pre class="prettyprint" id="bash">
$ chsh -s `which zsh`
$ sudo shutdown -r 0
</pre>

4.美化
为了能够显示诸如分支（branch）、闪电（这个符号应该指拿到root权限）、错误(红色叉叉)、后台（一个齿轮）的各种符号，必须使用一个patch过的字体，在ubuntu下默认是Ubuntu Mono，OS X下坐着配的是Menlo，很多常见的等宽字体都打好了patch，当然也可以自己手动打patch。
如果没有~/.fonts首先要创建
<pre class="prettyprint" id="bash">
$ cd ~/.fonts
$ cd ~/.fonts/ && git clone https://github.com/scotu/ubuntu-mono-powerline.git && cd ~
</pre>

修改主题，可以在[zsh-thmem-preview](https://github.com/robbyrussell/oh-my-zsh/wiki/themes)预览，选择自己喜欢的theme。打开.zshrc,修改即可。
<pre class="prettyprint" id="vi">
ZSH_THEME="amuse"
</pre>
5.添加环境变量到.zshrc
<pre class="prettyprint" id="bash">
$ echo "export PATH=$PATH" >> ~/.zshrc
$ source ~/.zshrc
</pre>

假如在配置完zsh后再添加环境变量，如添加arm-linux交叉编译器环境变量时，你需要在`~/.zshrc`文件下添加.

<pre class="prettyprint" id="vi">
export PATH="/usr/local/bin:/usr/bin:/bin:/usr/local/sbin:/usr/sbin:/home/izobs/.local/bin:/home/izobs/bin:/home/izobs/workspace/opt/usr/local/arm/4.3.2/bin" 
 </pre>
