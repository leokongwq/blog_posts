---
layout: post
comments: true
title: 判断单链表是否有环
date: 2016-10-12 17:04:12
tags:
- 算法
categories:
- 算法
---

#### 判断单链表是否有环
 
**题目描述**：输入一个单向链表，判断链表是否有环？

**分析**：通过两个指针，分别从链表的头节点出发，一个每次向后移动一步，另一个移动两步，两个指针移动速度不一样，如果存在环，那么两个指针一定会在环里相遇。这其实就是追赶游戏，跑的快的肯定能追的上慢的。

<!-- more -->

**代码如下**：

{% codeblock lang:c %}
bool hasCircle(Node *head){
   Node *slow,*fast;
   slow = fast = head;
   while(fast != NULL && fast->next != NULL){
       fast = fast->next->next;
       slow = slow->next;
       if(fast == slow){
           circleNode = fast;
           return true;
       }
   }
   return false;
}                        
{% endcodeblock %}   

### 另一种解法

**分析:** 利用一个Map和一个指针p，从头结点出发，依次判断每个节点的指针是否存在于map中，如果存在则表示有环，且p就是环的入口点，否则循环结束后p==null，没有环；

**代码如下：**

{% codeblock lang:c %}
bool hasCircle(Node *head){
   if(head == NULL || head->next == NULL){
       return false;
   }
   Node *p = head;
   Map map = new Map();
   while(p != NULL){
       if(map.contains(p)){
           return true;
       }
       map.put(p);
       p = p->next;
   }
   return false;
}       
{% endcodeblock %}                       
                    

