---
layout: post
comments: true
title: 单链表-链表有环如何判断相交
date: 2016-10-12 17:04:34
tags:
    - 算法
categories:
    - 算法
    
---

#### 单链表-链表有环如何判断相交 

**题目描述**：上面的问题都是针对链表无环的，那么如果现在，链表是有环的呢?上面的方法还同样有效么?

**分析**：如果有环且两个链表相交，则两个链表都有共同一个环，即环上的任意一个节点都存在于两个链表上。因此，就可以判断一链表上俩指针相遇的那个节点，在不在另一条链表上。

<!-- more -->

代码如下：

{% codeblock lang:c %}
    //判断两个带环链表是否相交
    bool isIntersectWithLoop(Node *h1,Node *h2){
        Node *circleNode1,*circleNode2;
        if(!hasCircle(h1,circleNode1))    //判断链表带不带环，并保存环内节点
            return false;                //不带环，异常退出
        if(!hasCircle(h2,circleNode2)){
            return false;
        }
        Node *temp = circleNode2->next;
        while(temp != circleNode2){
            if(temp == circleNode1){
                return true;
            }    
            temp = temp->next;
        }
        return false;
    }
{% endcodeblock %}     

                        
                    
                    