---
layout: post
comments: true
title: 单链表的逆序打印
date: 2016-10-12 16:41:25
tags: 
    - 算法
categories:
    - 算法

---

#### 单链表的逆序打印

**题目描述**：单链表的逆序打印
**分析**：单向链表的逆序打印就是从尾节点开始打印节点。方法1：遍历链表将每个节点放入栈中，然后每个元素出栈打印。这用方法需要分配额外的空间。方法2：利用递归的方式，当递归调用返回时我们就指定当前节点的值是最后一个节点，进行输出，层层返回就可以逐个打印每个元素。

<!-- more -->

代码如下：

{% codeblock lang:java %}
    //单链表的逆序打印
    void ReversePrintList_1(Node *head){
        if (head == NULL) {
            return;
        }
        ReversePrintList_1(head->next);
        printf("%d", head.val);
    }
{% endcodeblock %}    
    
    

                        
                    
                    

