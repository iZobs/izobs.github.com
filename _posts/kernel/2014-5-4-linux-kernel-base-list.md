---
layout: post
title: "Linux Kernel数据结构——链表"
date: 2014-5-4 9:34
category: "学习"
tags: ["Kernel"]

---

###链表

链表是一种常用的组织有序数据的数据结构，它通过指针将一系列数据节点连接成一条数据链，是线性表的一种重要实现方式。相对于数组，链表具有更好的动态性，建立链表时无需预先知道数据总量，可以随机分配空间，可以高效地在链表中的任意位置实时插入或删除数据。链表的开销主要是访问的顺序性和组织链的空间损失。通常链表数据结构至少应包含两个域：数据域和指针域，数据域用于存储数据，指针域用于建立与下一个节点的联系。

####数据链表list_head

list_head链表数据结构的定义很简单：
<pre class="prettyprint" id="c">

struct list_head {
	struct list_head *next, *prev;
};

</pre>
这里的list_head没有数据域。在Linux内核链表中，不是在链表结构中包含数据，而是在要使用的数据结构中包含链表节点。它的使用如下：
<pre class="prettyprint" id="c">
struct student
{
    int value;
    struct list_head list;
};
</pre>
####数据链表list_head的操作

__1.链表的声明和初始化__

我们可以这样初始化我们的链表头：
      struct student student_head;
      INIT_LIST_HEAD(&student_head.list)
也可以直接通过
      LIST_HEAD(student_head.list)
来定义初始化一个头节点。

上面的几个宏定义如下：
<pre class="prettyprint" id="c">

#define INIT_LIST_HEAD(ptr) do { \
	(ptr)->next = (ptr); (ptr)->prev = (ptr); \
} while (0)

#define LIST_HEAD_INIT(name) { &(name), &(name) }
#define LIST_HEAD(name) struct list_head name = LIST_HEAD_INIT(name)

</pre>

__2.插入/删除/合并__

__插入操作__

    static inline void list_add(struct list_head *new, struct list_head *head)
    /*在尾部插入*/
    static inline void list_add_tail(struct list_head *new, struct list_head *head)
<pre class="prettyprint" id="c">
/**
 * list_add - add a new entry
 * @new: new entry to be added
 * @head: list head to add it after
 *
 * Insert a new entry after the specified head.
 * This is good for implementing stacks.
 */
static inline void list_add(struct list_head *new, struct list_head *head)
{
	__list_add(new, head, head->next);
}

/**
 * list_add_tail - add a new entry
 * @new: new entry to be added
 * @head: list head to add it before
 *
 * Insert a new entry before the specified head.
 * This is useful for implementing queues.
 */
static inline void list_add_tail(struct list_head *new, struct list_head *head)
{
	__list_add(new, head->prev, head);
}

static inline void __list_add(struct list_head *new,
			      struct list_head *prev,
			      struct list_head *next)
{
	next->prev = new;
	new->next = next;
	new->prev = prev;
	prev->next = new;
}
</pre>

__删除操作__

      static inline void list_del(struct list_head *entry)

<pre class="prettyprint" id="c">
static inline void list_del(struct list_head *entry)
{
	__list_del(entry->prev, entry->next);
	entry->next = LIST_POISON1;
	entry->prev = LIST_POISON2;
}

/*
 * Delete a list entry by making the prev/next entries
 * point to each other.
 *
 * This is only for internal list manipulation where we know
 * the prev/next entries already!
 */
static inline void __list_del(struct list_head * prev, struct list_head * next)
{
	next->prev = prev;
	prev->next = next;
}
</pre>

__合并__                    

除了针对节点的插入、删除操作，Linux链表还提供了整个链表的插入功能:
    //将list和head链表合并
    static inline void list_splice(struct list_head *list, struct list_head *head);
    //该函数在将list合并到head链表的基础上，调用INIT_LIST_HEAD(list)将list设置为空链。
    static inline void list_splice_init(struct list_head *list,struct list_head *head);

![list.png](/picture/list.png "list.png")

<pre class="prettyprint" id="c">
static inline void list_splice(const struct list_head *list,
				struct list_head *head)
{
	if (!list_empty(list))
		__list_splice(list, head, head->next);
}

/**
 * list_splice_tail - join two lists, each list being a queue
 * @list: the new list to add.
 * @head: the place to add it in the first list.
 */
static inline void list_splice_tail(struct list_head *list,
				struct list_head *head)
{
	if (!list_empty(list))
		__list_splice(list, head->prev, head);
}

/**
 * list_splice_init - join two lists and reinitialise the emptied list.
 * @list: the new list to add.
 * @head: the place to add it in the first list.
 *
 * The list at @list is reinitialised
 */
static inline void list_splice_init(struct list_head *list,
				    struct list_head *head)
{
	if (!list_empty(list)) {
		__list_splice(list, head, head->next);
		INIT_LIST_HEAD(list);
	}
}

static inline void __list_splice(const struct list_head *list,
				 struct list_head *prev,
				 struct list_head *next)
{
	struct list_head *first = list->next;
	struct list_head *last = list->prev;

	first->prev = prev;
	prev->next = first;

	last->next = next;
	next->prev = last;
}

</pre>


__3.遍历__

我们知道，Linux链表中仅保存了数据项结构中list_head成员变量的地址，那么我们如何通过这个list_head成员访问到作为它的所有者的节点数据呢？Linux为此提供了一个list_entry(ptr,type,member)宏，其中ptr是指向该数据中list_head成员的指针，也就是存储在链表中的地址值，type是数据项的类型，member则是数据项类型定义中list_head成员的变量名，例如，我们要访问student_head链表中首个student_head变量value，则如此调用：

     p=list_entry(pos,struct student,list);
     printk("node %d's data :%d\n",i,p->num);
     
<pre class="prettyprint" id="c">     
/**
 * list_entry - get the struct for this entry
 * @ptr:	the &struct list_head pointer.
 * @type:	the type of the struct this is embedded in.
 * @member:	the name of the list_struct within the struct.
 */
#define list_entry(ptr, type, member) \
	container_of(ptr, type, member)

#define offsetof(TYPE, MEMBER) ((size_t) &((TYPE *)0)->MEMBER)

/**
 * container_of - cast a member of a structure out to the containing structure
 * @ptr:        the pointer to the member.
 * @type:       the type of the container struct this is embedded in.
 * @member:     the name of the member within the struct.
 *
 */
#define container_of(ptr, type, member) ({                      \
	const typeof( ((type *)0)->member ) *__mptr = (ptr);    \
	(type *)( (char *)__mptr - offsetof(type,member) );})


</pre>
关于container_of，其中：      

	  /*意思是声明一个与member同一个类型的指针常量 *__mptr,并初始化为ptr.*/
	  const typeof( ((type *)0->member ) *__mptr = (ptr);
	  /*意思是__mptr的地址减去member在该struct中的偏移量得到的地址, 再转换成type型指针. 该指针就是member的入口地址了.*/
	  (type *)( (char *)__mptr - offsetof(type,member) );
	
container_of()和offsetof()并不仅用于链表操作，这里最有趣的地方是((type *)0)->member，它将0地址强制"转换"为type结构的指针，再访问到type结构中的member成员。在container_of宏中，它用来给typeof()提供参数（typeof()是gcc的扩展，和sizeof()类似），以获得member成员的数据类型；在offsetof()中，这个member成员的地址实际上就是type数据结构中member成员相对于结构变量的偏移量。这里使用的是一个利用编译器技术的小技巧，即先求得结构成员在与结构中的偏移量，然后根据成员变量的地址反过来得出属主结构变量的地址。

遍历可用的方法：
      list_for_each_entry(pos, head, member)
 这里的pos是一个指向包含list_head节点对象的指针，可将它看做 是list_entry宏的返回值，head是一个指向头节点的指针，即遍历开始位置。member指的是list_head成员在数据结构中的名字。
 
<pre class="prettyprint" id="c">     
/**
 * list_for_each_entry	-	iterate over list of given type
 * @pos:	the type * to use as a loop cursor.
 * @head:	the head for your list.
 * @member:	the name of the list_struct within the struct.
 */
#define list_for_each_entry(pos, head, member)				\
	for (pos = list_entry((head)->next, typeof(*pos), member);	\
	     &pos->member != (head); 	\
	     pos = list_entry(pos->member.next, typeof(*pos), member))
	    	     
</pre>

举个例子，它实际的工作效果如下：
	  struct student
	  {
	      int value;
	      struct list_head list;
	  };
	  
	  struct numlist numhead;
	  struct list_head *pos;
	  struct numlist *p;
	  
	    list_for_each(pos,&numhead.list){
		    p=list_entry(pos,struct numlist,list);
		    printk("node %d's data :%d\n",i,p->num);
		}
	  
上面的等价于下面的：

	list_for_each_entry(p,&numhead.list,list)
