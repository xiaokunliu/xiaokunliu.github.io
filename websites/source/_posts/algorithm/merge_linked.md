---
title: 单向链表实现反转算法
category: 数据结构与算法
date: 2020-04-02 20:47:28
tags: 数据结构与算法
---

<!-- more -->

###### 1. 合并两个有序链表实现
> 使用递归方式

```java
public static LinkedNode mergeWithRecursive(LinkedNode head1, LinkedNode head2){
	if(head1 == null && head2 == null){
		return null;
	}
	
	if(head1 == null){
		return head2;
	}
	
	if(head2 == null){
		return head1;
	}
	
	// head1 & head2 not null
	LinkedNode head = null;
	if(head1.item <= head2.item){
		head = head1;
		head.next = mergeWithRecursive(head1.next, head2);
	}else{
		head = head2;
		head.next = mergeWithRecursive(head1, head2.next);
	}
	return head;
}
```

> 使用非递归方式

```java
public static LinkedNode mergeSortedLinked(LinkedNode head1, LinkedNode head2){
	if (head1 == null || head2 == null){
		return head1 == null? head2:head1;
	}
	
	// 第一个元素最小值的链表
	LinkedNode min_cur = head1.item <= head2.item?head1:head2;
	// 第一元素较大的链表
	LinkedNode max_cur =  min_cur == head1?head2:head1;
	
	// 以min_cur为基准,将max_cur合并到min_cur
	LinkedNode min_pre = null;	// min_cur的上一个节点
	LinkedNode max_next = null;	// max_cur的下一个节点
	
	// 目标节点
	LinkedNode head = min_cur;
	while(min_cur != null && max_cur != null){
		if(min_cur.item <= max_cur.item){
			// min的节点小于max节点,继续循环遍历
			min_pre = min_cur;
			min_cur = min_cur.next;
		}else{
			// 因为以min为基准,因此第一次循环的时候会在上面的if获取到min_pre的值
			min_pre.next = max_cur;
			// 保存max的next节点
			max_next = max_cur.next;
			// 连接到min的节点上,断开当前的max节点,将其的next指向min当前的节点
			max_cur.next = min_cur;
			// min_cur的上一个节点移动到当前的max节点
			min_pre = max_cur;
			// 	循环遍历下一个max的节点
			max_cur = next;
		}
	}
	// 最后需要判断下节点,将末尾的所有节点连接起来
	prev.next = min_cur == null?max_cur:min_cur;
	return head;
}
```

###### 2.合并两个链表非递归图解说明
> 初始化状态

- 代码如下
```java
// 确定min和max节点
LinkedNode min_cur = head1.item <= head2.item?head1:head2;
LinkedNode max_cur =  min_cur == head1?head2:head1;
```
- 其初始化图解如下
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200208132522917.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dpbmRfNjAy,size_16,color_FFFFFF,t_70)
> 第一次循环遍历
- 此时执行的代码
```java
if(min_cur.item <= max_cur.item){
	// min的节点小于max节点,继续循环遍历
	min_pre = min_cur;
	min_cur = min_cur.next;
}else{
	// ...
}
```
- 图解如下
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200208132836507.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dpbmRfNjAy,size_16,color_FFFFFF,t_70)
- 此时min_pre指向min链表中值为1的节点
- min_cur,即当前的指针指向min链表中值为2的节点

> 循环执行到else的语句分析
- 执行代码
```java
// 因为以min为基准,因此第一次循环的时候会在上面的if获取到min_pre的值
min_pre.next = max_cur;			// 1
// 保存max的next节点
max_next = max_cur.next;		// 2
// 连接到min的节点上,断开当前的max节点,将其的next指向min当前的节点
max_cur.next = min_cur;			// 3
// min_cur的上一个节点移动到当前的min节点
min_pre = max_cur;				// 4
// 	循环遍历下一个max的节点
max_cur = next;					// 5
```
- 执行代码前的图解
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200208133035167.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dpbmRfNjAy,size_16,color_FFFFFF,t_70)
- 执行代码过程图解
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200208141819843.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dpbmRfNjAy,size_16,color_FFFFFF,t_70)
- min_pre此时的指针为**min链表中值为2的节点**
- 执行1的代码之后,min_prev的next指针指向max当前的节点,也就是**值为2的max链表的节点**
- 执行2的代码之后,max_next的指针**指向max当前节点的下一个节点,也就是值为4的节点**
- 执行3的代码之后,max_cur的next指针指向min当前的节点,也就是**min链表中值为5的节点**
- 执行4的代码之后,min_prev的指针指向当前max的节点,也就是**值为2的max链表的节点**
- 执行5的代码之后,**max_cur当前指针指向max值为4的节点**
- 执行代码后的图解
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200208142012862.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dpbmRfNjAy,size_16,color_FFFFFF,t_70)
- 根据上述代码以及图解的节点值再执行else代码
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200208142457416.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dpbmRfNjAy,size_16,color_FFFFFF,t_70)
- 此时的min_prev的指针**指向max链表值为2的节点**
- 执行1代码之后,min_prev的**next指针指向max_cur节点,也就是值为4的max节点**
- 执行2代码之后,max_next的**指针指向值为6的max节点**
- 执行3代码之后,max_cur当前的**next指针指向当前min_cur的节点,也就是值为5的min节点**
- 执行4代码之后,min_prev的**指针指向当前max的节点,也就是值为4的max节点**
- 执行5代码之后,**max指针指向max_next的节点,也就是当前值为6的max节点**
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200208142613319.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dpbmRfNjAy,size_16,color_FFFFFF,t_70)
- 此时再次循环,min_cur.item < max_cur.item,执行流程如下
```java
prev = min_cur; //1
min_cur = min_cur.next; //2
```
执行过程
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200208145218834.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dpbmRfNjAy,size_16,color_FFFFFF,t_70)
执行结果
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200208145313288.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dpbmRfNjAy,size_16,color_FFFFFF,t_70)
接下来动作重复上述流程,可以看到**prev 始终跟在min_cur之后**,在经历整个循环之后,说明至少有一个链表已经遍历完成,最后只需要将prev的next节点与剩下的节点连接即可,最后max所有节点都将有序地合并到min中


- 小结
	- 使用的时间复杂度为O(n),空间复杂度为O(1)
