---
layout: post
title: "笔试知识-c指针"
date: 2014-5-26 20:10
category: "study"
comments: false
tagline: "Supporting tagline"
tags: [笔试,c]
---

基础知识篇
==========

## 定义问题 ##

__1.用变量a给下面定义__

- 一个整型数
- 一个指向整型数的指针( A pointer to an integer) 
- 一个指向指针的的指针，它指向的指针是指向一个整型数( A pointer to a pointer to an intege)r 
- 一个有10个整型数的数组( An array of 10 integers) 
- 一个有10个指针的数组，该指针是指向一个整型数的。(An array of 10 pointers to integers) 
- 一个指向有10个整型数数组的指针( A pointer to an array of 10 integers)
- 一个指向函数的指针，该函数有一个整型参数并返回一个整型数(A pointer to a function that takes an integer as an argument and returns an integer) 
- 一个有10个指针的数组，该指针指向一个函数，该函数有一个整型参数并返回一个整型数( An array of ten pointers to functions that take an integer argument and return an integer )

__答案__     

- a) int a; // 一个整型数 An integer 
- b) int *a; // 一个指向整型数的指针 A pointer to an integer 
- c) int **a; // 一个指向指针的的指针 A pointer to a pointer to an integer 
- d) int a[10]; // 一个有10个整型数的数组 An array of 10 integers 
- e) int *a[10]; // 一个有10个指针的数组 An array of 10 pointers to integers 
- f) int (*a)[10]; // 一个指向有10个整型数数组的指针 A pointer to an array of 10 integers 
- g) int (*a)(int); // 一个指向函数的指针 A pointer to a function a that takes an integer argument and returns an integer 
- h) int (*a[10])(int); // 一个有10个指针的数组，指向一个整形函数并有一个整形参数 An array of 10 pointers to functions that take an integer argument and return an integer


__2.加const的定义__

- a) const int a; // a是一个常整型数(不可修改值的整型数)
- b) int const a; // a是一个常整型数(不可修改值的整型数)
- c) const int *a; //a是一个指向常整型数的指针(整型数不可修改，指针可以修改)
- d) int *const a; //a是一个指向整型数的常指针(指针不可以修改)
- e) int const *a const; //指针不可变，指向的数也不可变

`Tips:const 表示不能修改,从右往左读`

__3.static 问题__

假如下面的定义发生在函数内部，则:
> char c[] = "hello world";       //这是一个局部数组
> char *c = "helllo world";       //这是一个全局数组

`static` 的常见使用方法:               
###1.static 全局变量 
   我们知道，一个进程在内存中的布局其中.text段保存进程所执行的程序二进制文件，.data段保存进程所有的已初始化的全局变量，.bss段保存进程未初始化的全局变量。在进程的整个生命周期中，.data段和.bss段内的数据时跟整个进程同生共死的，也就是在进程结束之后这些数据才会寿终就寝。         
   当一个进程的全局变量被声明为static之后，它的中文名叫静态全局变量。静态全局变量和其他的全局变量的存储地点并没有区别，都是在.data段（已初始化）或者.bss段（未初始化）内，但是它只在定义它的源文件内有效，其他源文件无法访问.               

###2.static 局部量 
普通的局部变量在栈空间上分配，这个局部变量所在的函数被多次调用时，每次调用这个局部变量在栈上的位置都不一定相同。局部变量也可以在堆上动态分配，但是记得使用完这个堆空间后要释放之。             
static局部变量中文名叫静态局部变量。它与普通的局部变量比起来有如下几个区别：              

- 1）位置：静态局部变量被编译器放在全局存储区.data（注意：不在.bss段内，原因见3）），所以它虽然是局部的，但是在程序的整个生命周期中存在。
- 2）访问权限：静态局部变量只能被其作用域内的变量或函数访问。也就是说虽然它会在程序的整个生命周期中存在，由于它是static的，它不能被其他的函数和源文件访问。
- 3）值：静态局部变量如果没有被用户初始化，则会被编译器自动赋值为0，以后每次调用静态局部变量的时候都用上次调用后的值。这个比较好理解，每次函数调用静态局部变量的时候都修改它然后离开，下次读的时候从全局存储区读出的静态局部变量就是上次修改后的值。

###3.static 函数
还记得C++面向对象编程中的private函数吗，私有函数只有该类的成员变量或成员函数可以访问。在C语言中，也有“private函数”，它就是接下来要说的static函数，完成面向对象编程中private函数的功能。    
当你的程序中有很多个源文件的时候，你肯定会让某个源文件只提供一些外界需要的接口，其他的函数可能是为了实现这些接口而编写，这些其他的函数你可能并不希望被外界（非本源文件）所看到，这时候就可以用static修饰这些“其他的函数”。               
所以static函数的作用域是本源文件，把它想象为面向对象中的private函数就可以了.static函数可以很好地解决不同原文件中函数同名的问题，因为一个源文件对于其他源文件中的static函数是不可见的.          

## 指针访问问题 ##

__1.如何给一个绝对地址赋值?__             

__答案__:           
{% highlight c %}
int *ptr;
ptr = (int *)0x6819;
*ptr = 11;
{% endhighlight %}

__2.volatile有什么用__                 

volatile的意思是易变的，也就是说，在程序运行过程中，有一些变量可能会被莫名其妙的改变，而优化器为了节约时间，有时候不会重读这个变量的真实值，而是去读在寄存器的备份，这样的话，这个变量的真实值反而被优化器给“优化”掉了，用时髦的词说就是被“和谐”了。如果使用了这个修饰词，就是通知编译器别犯懒，老老实实去重新读一遍！可能我说的太“通俗”了，那么我引用一下“大师”的标准解释：volatile的本意是“易变的”。
由于访问寄存器的速度要快过RAM,所以编译器一般都会作减少存取外部RAM的优化，但有可能会读脏数据。当要求使用volatile声明的变量的值的时候，系统总是重新从它所在的内存读取数据，即使它前面的指令刚刚从该处读取过数据。而且读取的数据立刻被保存。
精确地说就是，优化器在用到这个变量时必须每次都小心地重新读取这个变量的值，而不是使用保存在寄存器里的备份。下面是volatile变量的几个例子：     

- 1).并行设备的硬件寄存器（如：状态寄存器）
- 2).一个中断服务子程序中会访问到的非自动变量(Non-automatic variables) 3).多线程应用中被几个任务共享的变量
嵌入式系统程序员经常同硬件、中断、RTOS等等打交道，所用这些都要求volatile变量。不懂得volatile内容将会带来灾难


## 指针大小问题 ##

__1.存放一个地址需要几个字节？__
__答案__:和一个 int 类型的大小相同：4字节。 

所以,若有： 
{% highlight c %}

int* pInt; 

char* pChar; 

bool* pBool; 

float* pFloat; 

double* pDouble; 

则: sizeof(pInt)、sizeof(pChar)、sizeof(pBool)、sizeof(pFloat)、sizeof(pDouble)的值全部为：4。 

{% endhighlight %}

__2.sizeof和strlen的区别__                         
__❀第一个例子__：                                 
char* ss = "0123456789";              
1、sizeof(ss)的结果是4,ss是指向字符串常量的字符指针        
2、sizeof(*ss)的结果是1，*ss是第一个字符           

__❀第二个例子__：           
char ss[] = "01233456789";       
1、sizeof(ss)结果是11，ss是数组，计算到'\0'的位置，因此是10+1       
2、sizeof(*ss)结果是1，*ss是第一个字符         

__❀第三个例子__:          
char ss[100] = "0123456789";          
1、sizeof(ss)的结果是100，ss表示在内存中预分配的大小：100*1        
2、strlen(ss)的结果是10，它的内部实现是用一个循环计算字符串的长度，直到'\0'为止。   



## 函数指针 ##
__1.函数指针变量__                             
在Ｃ语言中规定，一个函数总是占用一段连续的内存区，而函数名就是该函数所占内存区的首地址。我们可以把函数的这个首地址（或称入口地址）赋予一个指针变量，使该指针变量指向该函数。然后通过指针变量就可以找到并调用这个函数。我们把这种指向函数的指针变量称为"函数指针变量"。                          
函数指针变量定义的一般形式为：类型说明符（*指针变量名）（）；           

其中"类型说明符"表示被指函数的返回值的类型。"（*指针变量名）"表示"*"后面的变量是定义的指针变量。最后的空括号表示指针变量所指的是一个函数。例如：int（*pf）（）；表示pf是一个指向函数入口的指针变量，该函数的返回值（函数值）是整型       

看下面的例子：                        

A) char * (*fun1)(char * p1,char * p2);          

B) char * *fun2(char * p1,char * p2);            

C) char * fun3(char * p1,char * p2);                

看看上面三个表达式分别是什么意思？              

C）：这很容易，fun3 是函数名，p1，p2 是参数，其类型为char *型，函数的返回值为char *类型。                 

B)：也很简单，与C）表达式相比，唯一不同的就是函数的返回值类型为char**，是个二级指针。          

A)：fun1是函数名吗？回忆一下数组指针时的情形。数组指针(指向数组的指针)这么定义或许更清晰：`int (*)[10] p`；       
再看看A）,这里fun1 不是什么函数名，而是一个指针变量，它指向一个函数。这个函数有两个指针类型的参数，函数的返回值也是一个指针.         

{% highlight c %}
/*
 * ===============================================================
 *
 *       Filename:  func_pointer.c
 *
 *    Description:  函数指针测试程序
 *
 *        Version:  1.0
 *        Created:  2014年05月26日 21时32分27秒
 *       Revision:  none
 *       Compiler:  gcc
 *
 *         Author:  izobs (Lin), ivincentlin@gmail.com
 *   Organization:  
 *
 * ===============================================================
 */

#include <stdlib.h>
#include <stdio.h>
#include <string.h>

char *fun(char *p1,char *p2)
{
	int i = 0;
	i = strcmp(p1,p2);
	if(0 == i)
	{
		return p1;
	}
	else
	{
		return p2;
	}
}
int main()
{
	char *ss = NULL;
	char *(*pf)(char *p1,char *p2);
	pf = &fun;
	ss = (*pf) ("aa","bb");
	printf("the bigger string is %s\n",ss);
	return 0;
}

{% endhighlight %}
结果为：
> $ ./func_pointer 
> the bigger string is bb

__2、*(int*)&p ----这是什么?__
{% highlight c %}

#include <stdlib.h>
#include <stdio.h>
#include <string.h>

char fun(char *p1,char *p2)
{
	return 4;
}
int main()
{
    void (*p)();
    *(int *)&p = (int) fun;
    (*p) ();
	return 0;
}

{% endhighlight %}
`void (*p)()`;这行代码定义了一个指针变量p，p 指向一个函数，这个函数的参数和返回值都是void。
&p 是求指针变量p 本身的地址，这是一个32 位的二进制常数（32 位系统）。           
`(int*)&p` 表示将地址强制转换成指向int 类型数据的指针。                 
`(int)fun` 表示将函数的入口地址强制转换成int 类型的数据
使用函数指针的好处在于，可以将实现同一功能的多个模块统一起来标识，这样一来更容易后期的维护，系统结构更加清晰。或者归纳为：便于分层设计、利于系统抽象、降低耦合度以及使接口与实现分开      

__3.`(*(void(*) ())0)()`------这是什么？__                

这是《C Traps and Pitfalls》这本经典的书中的一个例子。：         

- 第一步：`void(*) ()`，可以明白这是一个函数指针类型。这个函数没有参数，没有返回值。

- 第二步：`(void(*) ())0`，这是将0 强制转换为函数指针类型，0 是一个地址，也就是说一个函数存在首地址为0 的一段区域内。

- 第三步：`(*(void(*) ())0)`，这是取0 地址开始的一段内存里面的内容，其内容就是保存在首地址为0 的一段区域内的函数。

- 第四步：`(*(void(*) ())0)()`，这是函数调用。

__4.函数指针数组__                  

现在我们清楚表达式“`char * (*pf)(char * p)`”定义的是一个函数指针pf。既然pf 是一个指针，那就可以储存在一个数组里。把上式修改一下：               

> char * (*pf[3])(char * p);

这是定义一个函数指针数组。它是一个数组，数组名为pf，数组内存储了3 个指向函数的指针。这些指针指向一些返回值类型为指向字符的指针、参数为一个指向字符的指针的函数。         

{% highlight c %}
#include <string.h>
#include <stdio.h>

char * fun1(char *p)
{
    printf("%s\n",p);
    return p;
}

char * fun2(char *p)
{
    printf("%s\n",p);
    return p;
}

char * fun3(char *p)
{
    printf("%s\n",p);
    return p;
}

int main()
{

    char * (*pf[3])(char *p);
    pf[0] = fun1;    //直接用函数名
    pf[1] = &fun2;    //可以用函数名加上取地址符
    pf[2] = &fun3;
    pf[0]("fun1");
    pf[1]("fun2");
    pf[2]("fun3");
    return 0;
}
{% endhighlight %}

## 内存分配问题 ##
###1.内存分配和free函数 ###

> void *calloc(unsigned num,unsigned size)
分配内存大小大小位num*size,并将内存空间初始化为`0`或`NULL`.
> void *realloc(void *ptr,size_t size)
ptr指向已有的内存空间，size用来指定重新分配之后所得的整个空间大小。

> 注意:free函数释放内存后，要用if(prt == NUll)来判断prt是否为无效，防止再次被使用。所以在使用完free后，最好令`prt = NULL`.
