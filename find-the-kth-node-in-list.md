---
layout: post
comments: true
title: 找出单链表倒数第K个节点
date: 2016-10-12 17:03:50
tags:
    - 算法
categories:
    - 算法
    
---

#### 找出单链表倒数第K个节点

**题目描述**：输入一个单向链表，输出该链表中倒数第k个节点，链表的倒数第0个节点为链表的尾指针。
**分析**：设置两个指针 p1、p2，首先 p1 和 p2 都指向 head，然后 p2 向前走 k 步，这样 p1 和 p2 之间就间隔 k 个节点，最后 p1 和 p2 同时向前移动，直至 p2 走到链表末尾。

<!-- more -->

**代码如下：**

{% codeblock lang:c %}
//倒数第k个节点
Node* theKthNode(Node *head,int k){
   if(k < 0) {
       return NULL;    //异常判断
   }
   Node *slow,*fast;
   slow = fast = head;
   int i = k;
   for(;i>0 && fast!=NULL;i--){
       fast = fast->next;
   }
   if(i > 0){
       return NULL;    //考虑k大于链表长度的case
   }
    
   while(fast != NULL){
       slow = slow->next;
       fast = fast->next;
   }
   return slow;
}                        
{% endcodeblock %}
                    
                    
                    

