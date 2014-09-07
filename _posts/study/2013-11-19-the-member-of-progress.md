---
layout: post
title: "linux c-堆与栈"
date: 2013-11-20 21:10
category: "linux"
comments: false
tagline: "Supporting tagline"
tags: [linux,c]
---

#程序内存区域的使用

##一.静态内存与动态内存
``1.c语言程序中的静态数据储存区分为三类：``

- 只读数据区（RO Data）
- 已初始化的读写数据区(RW Data)
- 未初始化的读写数据区(BSS)

``2.c语言程序中的动态数据储存区主要有两类：``

- 栈(stack):连续,使用线性储存方式,先入后出
- 堆(heap):从低地址向高地址生长，非连续,使用链表来实现,

##二.栈与堆的使用
###&栈
``(1).栈在空栈和满栈的不同处理``

按照满栈处理的情况：在入栈时候，入栈的内容将放入栈指针移动后的内存地址中，在出栈的时候，将弹出
当前栈指针处的内容。
按照空栈处理的情况：在入栈时候，入栈的内容将放入当前栈指针的内存地址中，在出栈的时候，将弹出
栈指针处移动后地址处的内容。

>  注意:满栈处理和空栈处理是由处理器的硬件决定的，与程序的编写无关

###代码
{% highlight c++ %}
#include <stdlib.h>
#include <stdio.h>

void fun_stack1(int p1,int p2,int p3)
{
	printf("fun_stack1:p1 = %p &p1 = %p \n",p1,(unsigned int)&p1);
	printf("fun_stack1:p2 = %x &p2 = %x \n",p2,(unsigned int)&p2);
	printf("fun_stack1:p3 = %x &p3 = %x \n",p3,(unsigned int)&p3);
	 return;
}

void fun_stack2(int p1,int p2,int p3)
{
	 fun_stack1(p1,p2,p3);
	 return;
}

int main(void)
{
	 int a,b,c;
	 a = 0x11;
	 b = 0x22;
	 c = 0x33;
	 printf("<< call fun_stack1 >>\n");
	 fun_stack1(a,b,c);
	 printf("<< call fun_stack2 >>\n");
	 fun_stack2(a,b,c);
	 return 0;
}

{% endhighlight c++ %}

`输出结果`:
> << call fun_stack1 >>                   
> fun_stack1:p1 = 11 &p1 = 59be657c 
> fun_stack1:p2 = 22 &p2 = 59be6578 
> fun_stack1:p3 = 33 &p3 = 59be6574        
> << call fun_stack2 >>                    
> fun_stack1:p1 = 11 &p1 = 59be655c 
> fun_stack1:p2 = 22 &p2 = 59be6558 
> fun_stack1:p3 = 33 &p3 = 59be6554 

**如果函数的嵌套越多，则占用的栈也将越多**                   
**内层函数可以使用外层函数的栈内存，但是外层函数不能使用内层函数的栈内存** 

###&堆

**1.与栈的不同点**

> 在一般的系统中，堆的分配方向与栈的方向的相反的.

+ 堆的分配和释放都是通过C语言的函数库stdlib.h实现的，而栈则是通过处理器的硬件机制
+ 堆内存有多个入口点，每个分配区可被单独释放。而栈内存只有一个入口点.            

**2.堆的使用**    
              
   
- 用malloc()函数返回的指针值是存放在栈上的，而该指针指向的值则存放在堆空间中
- calloc()和malloc()很类似，calloc()可以将分配好的内存初始值设置为0
- realloc()函数的调用要做3事：         
 1.按照新内存的大小重新分配一块内存         
 2.将原有的内容复制到新的内存中           
 3.释放原有的内存空间            

                    

**3.堆内存的特性**

> 1.开辟的内存没有释放，造成内存泄漏        
> 2.野指针被使用或释放            
> 3.非法释放指针                 

**野指针的避免**

{% highlight c++ %}
void fun_heap12(void)
{
    char *pa;
    pa = (char *)malloc(sizeof(char)*20);
    if(pa != NULL)
    {
        strcpy(pa,"memory leak\n");
        printf("pa = %x \n",(unsigned int)pa);
        printf("pa = %s \n",pa);
    }
    free(pa);
    pa = NULL;
    if(NULL != pa)
    {
        printf("pa = %s",(unsigned int)pa);
    }
    return;
}
{% endhighlight c++ %}

> 注意：内存被释放后，指针应该被设置为NULL，同时在使用指针的程序中
判断指针是否为NULL

**非法释放指针**      
下面是非法释放的例子:             
{% highlight c++ %}
const char readOnlyData[]="readOnlyData memory area";
char readWriteData[] = "rw data memory area";
char bssData[100];
void fun_stack3()
{
    free(readOnlyData); //错误释放只读指针
    free(readWriteData); //错误释放已经初始化的数据区指针
    free(bssData);       //错误释放未初始化的数据区指针
}
{% endhighlight c++ %}

如果释放静态存储区或代码区，同样会引发上面的错误

{% highlight c++ %}
void fun_stack4(void)
{
    char a[20];
    int b;

    free(a);
    free(&b);
    return;
}
{% endhighlight c++ %}

__a[20],b是栈上的变量,用free来释放他们显然错误__


{% highlight c++ %}
char *pa;
char *pb;

pa = (char *)malloc(sizeof(char*)*20);
pb = pa++;
free(pb);
{% endhighlight c++ %}

pb是pa的地址+1,也是一个堆内存，而且这个指针指向的地址已经被分配了内存。然而，释放内存pb依然是
个错误。这是由于这个指针不是从malloc分配出来的，而是一个中间指针值。

##堆内存和栈内存使用的比较##

###1.利用返回值传递信息

{% highlight c++ %}
int add(int a,int b)
{
    return(a+b);
}
{% endhighlight c++ %}

函数返回值是存放在C语言的储存区的栈上的。函数运行完成后，函数的调用着可以通过返回值得到这个栈上的内存.        
函数的返回值也可是指针，这个指针可以指向各种类型的内存。他可以指向静态储存区的地址，可以指向堆内存的地址，
也可以指向一个函数调用着的栈空间，但是它不应该是一个函数内部栈内存区域的地址.          
字符处理函数返回的都是函数的第一个指针形式参数dest.            
**这里传入的dest是一个需要处理的地址，所以不能传入只读储存区的地址**
{% highlight c++ %}
char *strcpy(char *dest,const char *src);
char *strcat(char *dest,const char *src);
{% endhighlight c++ %}







