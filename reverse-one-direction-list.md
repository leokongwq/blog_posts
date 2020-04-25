---
layout: post
comments: true
title: 单链表逆转 
date: 2016-10-12 17:03:27
tags:
    - 算法
categories:
    - 算法

---

### 单链表逆转 

**题目描述**：输入一个单向链表，输出逆序反转后的链表

**分析**：单向链表的转置是一个很常见、很基础的数据结构题。但就是在面试中没有做出了，**后悔**在学校没有加强数据结构的学习和编程。

<!-- more -->

### 总结
单向链表的转置总共有下面的几种方式：

 1. 最简单的方式是采用数组，遍历链表将每个节点的指针放入数组，然后反向遍历数组，逐个翻转指针指向。这个算法需要额外使用数组，所以不推荐使用。
 2. 采用p , q 两个指针, 从第二个节点开始,逐个节点进行反转; 该算法只需要额外的两个指针，可以采用。
 3. 递归的方式翻转，其实也是找到了最后的一个节点，从最后一个节点开始逐个节点进行翻转。
 
下面2，3方法的代码实现：

{% codeblock lang:c %}
    //单链表的转置,循环方法
    Node* reverseByLoop(Node *head)
    {
        if(head == NULL || head->next == NULL){
            return head;
        }        
        Node *p = head;
        Node *q = head->next;
        head->next = NULL;
        while(q != NULL){
            Node *tmp = q.next; //tmp指向剩余节点的第一个节点
            q.next = p; //翻转
            p = q; // p指向翻转后的链表的第一个节点
            q = tmp;
        }
        head = p; // head节点指向翻转后节点的第一个节点
        return head;
    }

    //单链表的转置,递归方法
    Node* reverseByRecursion(Node *head)
    {
        //第一个条件是判断异常，第二个条件是递归结束条件
        if(head == NULL || head->next == NULL) {
            return head;
        }
        //递归逆向返回时: newHead是老链表的最后一个节点; 
        // head 参数是倒数第二个节点
        Node *newHead = reverseByRecursion(head->next);
    
        head->next->next = head; 最后2个节点成环形,
        head->next = NULL; // 打断原有的正向连接, 只保留逆向的关系
        return newHead;    //返回新链表的头指针
    }
{% endcodeblock %}
                    