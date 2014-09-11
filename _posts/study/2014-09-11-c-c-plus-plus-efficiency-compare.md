---
layout: post
title: "c和c++的效率比较"
category: cpp
modified: 2014-09-11 11:29:44 +0800
tags: [C,C++]
image:
  feature: 
  credit: 
  creditlink: 
comments: ture
share: ture
---

> 转载自知乎网

有两个方面影响最终执行效率，第一是语言特性，第二是编译器优化效率。

从语言特性角度上来看，C++是C的超集。在(C++) - C的这部分语言特性中有很多会降低执行效率。一个例子是dynamic_cast，执行一个dynamic_cast要消耗100-300个CPU cycles，因为机器要跳到一段特别的snippet(一小段程序)去检查type inheritance，在内层循环中使用它无非是大大浪费时间。另一个例子是大家都很熟悉的vtable，这里不多说了。    

但是一些增加了的语言特性会极大地提高编译器识别并对代码进行优化的能力。最简单的就是inline关键词。在C中程序员是不能显示地告诉编译器要不要inline某个函数，C++有了这个能力，也就是说把控制权更多地交给了写代码的人（虽然最终不一定会inline）。inline和const这两个关键词使得在global constant propagation这个编译器优化过程里，一些在底层函数中，在C里不能被全局识别的常数都能被顺利地展开，这样生成的代码必然比C要快得多。（这一段需要一些编译器优化的知识才能理解）

题外说一句，编译器很重要。说一个之前优化代码的例子：在做循环内的运算优化时，如果运算数据在内存里相邻并对齐，GCC可以检测出来并生成并行代码（在X86上是Intel SSE指令）。最终我通过数据对齐让GCC生成SIMD代码将循环速度提升了20倍。这一点上C和C++收到的效果是一样的。

总而言之，你不能简单地说C和C++哪个效率更高。它们各有各的特性，如何利用它们各自的特性生成运行效率优秀的程序，是一个程序员应该思考的事情。希望我的回答能对问题有所补充。

-------------
@冯东 提出了一些问题，我也继续说说我的看法。

我不明白为什么使用Template的话CPU cache会失效增加？假如template生成两个实例，那在执行template的两个实例时两者的execution path应该是类似的，于是CPU cache失效率也应该是类似的。代码量的增加并不是Cache miss的条件，很多情况下在优化代码时我们会刻意增加代码长度，来减少cache miss，增加编译器发现并输出cache preload指令的可能。

然而实际上，template的使用会提高编译器发现优化的可能性，因为在编译时template会提供更多的信息，使得静态编译器有更大的机会优化代码，并且更有可能输出cache miss较少的binary。

至于，C++ binary interface，通常工业界的做法是做个C的API wrapper，然后内部用C++实现，这样C++代码就有了C这样干净的binary interface了。 
