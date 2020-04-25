---
layout: post
comments: true
title: 找出单链表的中间节点
date: 2016-10-12 17:04:00
tags:
    - 算法
categories:
    - 算法
    
---

### 找出单链表的中间节点 

**题目描述**：求链表的中间节点，如果链表的长度为偶数，返回中间两个节点的任意一个，若为奇数，则返回中间节点。

**分析**：此题的解决思路和「求链表的倒数第 k 个节点」很相似。可以先求链表的长度，然后计算出中间节点所在链表顺序的位置。但是如果要求只能扫描一遍链表，如何解决呢？通过两个指针来完成。用两个指针从链表头节点开始，一个指针每次向后移动两步，一个每次移动一步，直到快指针移到到尾节点，那么慢指针即是所求。

<!-- more -->

**代码如下**：

{% codeblock lang:c %}
//求链表的中间节点
Node* theMiddleNode(Node *head)
{
   if(head == NULL)
       return NULL;
   Node *slow,*fast;
   slow = fast = head;
   //如果要求在链表长度为偶数的情况下，返回中间两个节点的第一个，可以用下面的循环条件
   //while(fast && fast->next != NULL && fast->next->next != NULL)  
   while(fast != NULL && fast->next != NULL)
   {
       fast = fast->next->next;
       slow = slow->next;
   }
   return slow;
}
{% endcodeblock %}                        
                    
                    
                    

