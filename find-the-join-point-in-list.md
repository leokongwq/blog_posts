---
layout: post
comments: true
title: 找出单链表有环的入口节点
date: 2016-10-12 17:04:24
tags:
    - 算法
categories:
    - 算法
    
---

#### 找出单链表有环的入口节点
 
 **题目描述**：输入一个单向链表，判断链表是否有环。如果链表存在环，如何找到环的入口点？

**解题思路**： 由上题可知，按照p2每次两步，p1每次一步的方式走，发现 p2 和 p1重合，确定了单向链表有环路了。接下来，让p2回到链表的头部，重新走，每次步长不是走2了，而是走1，那么当 p1 和 p2 再次相遇的时候，就是环路的入口了。

<!-- more -->

为什么？：假定起点到环入口点的距离为 a，p1 和 p2 的相交点M与环入口点的距离为b，环路的周长为L，当 p1 和 p2 第一次相遇的时候，假定 p1 走了 n 步。那么有：

p1走的路径： a+b ＝ n；
p2走的路径： a+b+k*L = 2*n； p2 比 p1 多走了k圈环路，总路程是p1的2倍

根据上述公式可以得到 k*L=a+b=n显然，如果从相遇点M开始，p1 再走 n 步的话，还可以再回到相遇点，同时p2从头开始走的话，经过n步，也会达到相遇点M。

显然在这个步骤当中 p1 和 p2 只有前 a 步走的路径不同，所以当 p1 和 p2 再次重合的时候，必然是在链表的环路入口点上。

**代码如下:**

{% codeblock lang:c %}
    //找到环的入口点
    Node* findLoopPort(Node *head){
        //如果head为空，或者为单结点，则不存在环
        if(head == NULL || head->next == NULL) {
            return NULL;
        }
        Node *slow,*fast;
        slow = fast = head;
    
        //先判断是否存在环
        while(fast != NULL && fast->next != NULL){
            fast = fast->next->next;
            slow = slow->next;
            if(fast == slow)
                break;
        }
    
        if(fast != slow) {
            return NULL;    //不存在环
        }
    
        fast = head;                //快指针从头开始走，步长变为1
        while(fast != slow){            //两者相遇即为入口点
            fast = fast->next;
            slow = slow->next;
        }
        return fast;
    }
{% endcodeblock %}                        
                    
                    