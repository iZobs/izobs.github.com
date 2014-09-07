---
layout: post
title: "编译andriod4.4系统记录 "
date: 2014-8-20 14:25
category: "openwrt"
tags : "android"
---

###Android的源码的获取
Android 官方有有一个用git获取代码的教程，但是由于在天朝，google的服务器真心不给力，所以根本无法使用这个方法来下载代码。如果想试试，就到这里看一下，不过需要翻墙的。
[android development guide](http://source.android.com/source/index.html)

没办法，只能借助国内光纤大户下载好的src来用，悄悄从他的网盘fork到自己的网盘：
[百度网盘](http://pan.baidu.com/s/1eQy4tyy)

这个src是android 4.4 kitkat的src，下载好后需要把几个小块合成一个tar.gz，然后解压:

<pre class="prettyprint" id="bash">
<code>$ cat droidSplita* > mydroid.tar.gz </code>
<code>$ tar zxvf ./mydroid.tar,gz </code>
</pre>


###搭建android编译环境
首先你需要一个ubuntu 12.04以上的64bit的系统，在虚拟机安装可不是个好主意，除非你想等待漫长的编译时间。android官方给的搭建教程是ubuntu 12.04-64bit,所以不想遇到无法解决的erro就用这个发行版吧。

我在PC安装的是最新的ubuntu 14.04,经过验证，这个版本可以编译kitkat。下面就开始搭建吧。

####1.安装工具和依赖库

<pre class="prettyprint" id="bash">
sudo apt-get install git gnupg flex bison gperf build-essential \  
  zip curl libc6-dev libncurses5-dev:i386 x11proto-core-dev \  
    libx11-dev:i386 libreadline6-dev:i386 libgl1-mesa-glx:i386 \  
	  libgl1-mesa-dev g++-multilib mingw32 tofrodos \  
	    python-markdown libxml2-utils xsltproc zlib1g-dev:i386 
</pre>

<pre class="prettyprint" id="bash">
sudo ln -s /usr/lib/i386-linux-gnu/mesa/libGL.so.1 /usr/lib/i386-linux-gnu/libGL.so
</pre>


####2.安装sun jdk6  
对于android 编译的jdk的版本依赖，官方是这么说的;

```
The master branch of Android in the Android Open Source Project (AOSP) requires Java 7. On Ubuntu, use OpenJDK.

Java 7: For the latest version of Android

$ sudo apt-get update
$ sudo apt-get install openjdk-7-jdk

Optionally, update the default Java version by running:

$ sudo update-alternatives --config java
$ sudo update-alternatives --config javac

If you encounter version errors for Java, set its path as described in the Wrong Java Version section.

To develop older versions of Android, download and install the corresponding version of the Java JDK:
__Java 6: for Gingerbread through KitKat__
Java 5: for Cupcake through Froyo

```
我们编译的是kitkat,所以需要用JDK6,但是我们还不能用openJDK6,必须用sun的JDK：   

`Note: The lunch command in the build step will ensure that the Sun JDK is used instead of any previously installed JDK.`

获取`jdk-6u45-linux-x64.bin`;            

<pre class="prettyprint" id="bash">
 $ cd ~
 $ wget http://ghaffarian.net/downloads/Java/JDK/jdk-6u45-linux-x64.bin
</pre>

安装:         

<pre class="prettyprint" id="bash">
$ sudo chmod +x jdk-6u45-linux-x64.bin
$ sudo ./jdk-6u45-linux-x64.bin
$ sudo rm -rf jdk-6u45-linux-x64.bin
</pre>

安装到opt目录:
<pre class="prettyprint" id="bash">
$ sudo mv jdk1.6.0_45 /opt  
$ sudo mv jdk1.6.0_45 /opt  
</pre>

<pre class="prettyprint" id="bash">
$ sudo update-alternatives --install "/usr/bin/java" "java" "/opt/jdk1.6.0_45/bin/java" 1 
$ sudo update-alternatives --install "/usr/bin/javac" "javac" "/opt/jdk1.6.0_45/bin/javac" 1 
$sudo update-alternatives --install "/usr/bin/javaws" "javaws" "/opt/jdk1.6.0_45/bin/javaws" 1 
$sudo update-alternatives --install /usr/bin/javap javap /opt/jdk1.6.0_45/bin/javap 300
$ sudo update-alternatives --install "/usr/lib/mozilla/plugins/libjavaplugin.so" "mozilla-javaplugin.so" "/opt/jdk1.6.0_45/jre/lib/
</pre>

在/usr/bin创建jar和javadoc的链接:
<pre class="prettyprint" id="bash">
$ cd /usr/bin
$ sudo ln -s -f /opt/jdk1.6.0_45/bin/jar  
$ sudo ln -s -f /opt/jdk1.6.0_45/bin/javadoc  
</pre>


配置Oracle为系统默认JDK/JRE,假如你只安装了一个jdk则不需要这一步      
<pre class="prettyprint" id="bash">
$ sudo update-alternatives --set java /usr/local/java/jdk1.6.0_45/bin/java
$ sudo update-alternatives --set javac /usr/local/java/jdk1.6.0_45/bin/javac
$ sudo update-alternatives --set javaws /usr/local/java/jdk1.6.0_45/bin/javaws
</pre>

添加环境变量，在/etc/profile后面追加:            

```

JAVA_HOME=/opt/jdk1.6.0_45
PATH=$PATH:$HOME/bin:$JAVA_HOME/bin
export JAVA_HOME
export PATH   

```

到此，sun-jdk6安装完成!

###漫长的编译

首先执行环境配置:
<pre class="prettyprint" id="bash">
$ source build/envsetup.sh
</pre>

然后lunch:
<pre class="prettyprint" id="bash">
$ lunch
</pre>

![lunch-menu](/picture/lunch.png)

选择一个你要的版本，我选的是`asop_arm-eng` :

```
PLATFORM_VERSION_CODENAME=REL
PLATFORM_VERSION=4.4
TARGET_PRODUCT=aosp_arm
TARGET_BUILD_VARIANT=eng
TARGET_BUILD_TYPE=release
TARGET_BUILD_APPS=
TARGET_ARCH=arm
TARGET_ARCH_VARIANT=armv7-a
TARGET_CPU_VARIANT=generic
HOST_ARCH=x86
HOST_OS=linux
HOST_OS_EXTRA=Linux-3.13.0-32-generic-x86_64-with-Ubuntu-14.04-trusty
HOST_BUILD_TYPE=release
BUILD_ID=KRT16S
OUT_DIR=out

```
关于lunch的使用，这个文章还不错:
[lunch的分析](http://blog.csdn.net/gchww/article/details/7838947)

添加编译缓存区,CCACHE_DIR可以自己改:
<pre class="prettyprint" id="bash">
$ export USE_CCACHE=1
$ export CCACHE_DIR=/home/izobs/android/android-cache/.ccache
$ prebuilts/misc/linux-x86/ccache/ccache -M 50G
</pre>
监控缓存区的变化:
<pre class="prettyprint" id="bash">
$ watch -n1 -d prebuilts/misc/linux-x86/ccache/ccache -s
</pre>

![ccahe](/picture/ccahe.png)

可以编译了：

<pre class="prettyprint" id="bash">
$ make -j4
</pre>

大概5小时，编译完成.


###运行anroid SDK
添加环境编译:
<pre class="prettyprint" id="bash">
$ export PATH=$PATH:/home/izobs/android/kitkat/droid/mydroid/out/host/linux-x86/bin
$ export ANDROID_PRODUCT_OUT=/home/izobs/android/kitkat/droid/mydroid/out/target/product/generic
</pre>

运行:

<pre class="prettyprint" id="bash">
$ emulator
</pre>

执行emulator，如果出现如下错误：
```
emulator: ERROR: You did not specify a virtual device name, and the system

directory could not be found.

If you are an Android SDK user, please use '@<name>' or '-avd <name>'

to start a given virtual device (see -help-avd for details).

Otherwise, follow the instructions in -help-disk-images to start the emulator

```
则需要这么做:

<pre class="prettyprint" id="bash">
$ source build/envsetup.sh 
$ lunch sdk-eng
</pre>

启动要等一下，虚拟机果然有点卡
![ccahe](/picture/android.png)



__什么,为什么是MAC OX的窗口，呵呵！就是想装B而已。__
