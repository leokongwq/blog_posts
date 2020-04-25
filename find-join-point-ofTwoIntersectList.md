---
layout: post
comments: true
title: 查找无环单链表的相交节点
date: 2016-10-12 17:04:47
tags:
    - 算法
categories:
    - 算法
    
---

### 查找两个无环单链表的相交节点 

**题目描述**：如果两个无环单链表相交，怎么求出他们相交的第一个节点呢？

**分析**：采用对齐的思想。计算两个链表的长度 L1,L2;分别用
两个指针 p1 , p2指向两个链表的头节点，然后从较长链表的头结点 p1（假设为 p1）开始向前移动L1-L2个节点，然后再同时向前移动p1 , p2，直到 p1 = p2。相遇的点就是相交的第一个节点。

<!-- more -->

代码如下：

{% codeblock lang:c %}
    //求两链表相交的第一个公共节点
    Node* findIntersectNode(Node *h1,Node *h2){
        int len1 = listLength(h1);          //求链表长度
        int len2 = listLength(h2);
        //对齐两个链表
        if(len1 > len2){
            for(int i=0;i<len1-len2;i++)
                h1=h1->next;
        } else {
            for(int i=0;i<len2-len1;i++)
                h2=h2->next;
        }
        while(h1 != NULL) {
            if(h1 == h2){
                return h1;
            }
            h1 = h1->next;
            h2 = h2->next;    
        }
        return NULL;
    }
{% endcodeblock %}         
    
    

                        
                    
                    