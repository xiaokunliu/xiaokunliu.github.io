---
title: 合并两个有序链表
category: 数据结构与算法
date: 2020-04-02 20:47:28
tags: 数据结构与算法
---

<!-- more -->
###### 1. 单向链表实现反转
> java 代码实现

```java
// SingleLinked.java
public class SingleLinked {
	
	final static class LinkedNode {
		LinkedNode next;
		int item;
	}
	
	private LinkedNode head;
	private LinkedNode tail;
	
	// 链表遍历
	public void traverse(){
		LinkedNode cur = head;
		while(cur != null){
			out.printf("%d ", cur.item);
			cur = cur.next;
		}
	}	

	// 反转链表实现
	public void reverse(){
		LinkedNode prev = null;
		LinkedNode next = null;
		LinkedNode cur = head;
		while(cur != null){
			next = cur.next;
			cur.next = prev;
			prev = cur;
			cur = next;
		}
		head = prev;
	}
	
	public void insert(LinkedNode node){
		if(head == null){
			head = node;
			head.next = null;
			return;
		}
		LinkedNode old = head;
		head = node;
		head.next = old;
	}
}
// 执行main方法测试
main(){
		SingleLinked linked = new SingleLinked();
		LinkedNode node1 = new LinkedNode(1);
		LinkedNode node4 = new LinkedNode(4);
        LinkedNode node5 = new LinkedNode(5);
        LinkedNode node6 = new LinkedNode(6);
        LinkedNode node7 = new LinkedNode(7);
        LinkedNode node8 = new LinkedNode(0);
        linked.insert(node1);
        linked.insert(node4);
        linked.insert(node5);
        linked.insert(node6);
        linked.insert(node7);
        linked.insert(node8);
        System.out.println("before reverse");
        linked.traverse();

        linked.reverse();
        System.out.println("after reverse");
        linked.traverse();
}
```

> 执行结果

```text
before reverse
1 4 5 6 7 0 
after reverse
0 7 6 5 4 1 
```

###### 2. 反转算法图解分析
> 执行反转的初始化状态代码与图解如下

```java
LinkedNode prev = null;
LinkedNode next = null;
LinkedNode cur = head;
```
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200207215728963.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dpbmRfNjAy,size_16,color_FFFFFF,t_70)

> 第一次循环执行的代码与图解

```java
next = cur.next;		// 1
cur.next = prev;		// 2
prev = cur;				// 3
cur = next;				// 4
```

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200207220216370.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dpbmRfNjAy,size_16,color_FFFFFF,t_70)

- 说明:
	- 执行1的代码之后,next的指针指向node1
	- 执行2的代码之后,head的next指针指向null
	- 执行3的代码之后,pre的指针指向head
	- 执行4的代码之后current的指针指向node1

- 于是最终结果为
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200207220439761.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dpbmRfNjAy,size_16,color_FFFFFF,t_70)

- 最后不断的循环,重复上述的动作,其过程以及最后的结果为:
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200207220704888.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dpbmRfNjAy,size_16,color_FFFFFF,t_70)
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200207220916455.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dpbmRfNjAy,size_16,color_FFFFFF,t_70)

- 小结
	- 上述的算法时间复杂度为O(n),空间复杂度为O(1)
	- 时间复杂度在于while循环,空间复杂度基本没有开辟内存空间,都是用指针引用现有的数据内存区域地址