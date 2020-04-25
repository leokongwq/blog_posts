---
layout: post
comments: true
title: 合并两个升序的单向链表
date: 2016-10-12 16:41:15
tags: 
    - 算法
categories:
    - 算法
---

### 合并两个升序的单向链表

**题目描述**：合并两个升序的单向链表
**分析**：合并排序的单向链表和合并排序好的数组是类似的；
1：分配一个新的节点，让newHead指针和tail指针指向新分配的节点；
2：从两个链表头部开始，逐个比较2个链表的节点值让tail指向节点的next域指向head1或head2, 
3: 移动tail指针
4：移动 head1 或 head2 到下一个节点

<!-- more -->

代码如下：
{% codeblock lang:java %}

    ListNode* MergeTwoSortedList(ListNode *head1, ListNode *head2){  
        if (head1 == NULL) {
        	return head2;  
        } 
            
        if (head2 == NULL) {
        	return head1;  
        } 
    	ListNode* p1 = head1, p2 = head2;
    	//分配一个新的节点
    	ListNode* newHead = (ListNode*)(malloc(size(ListNode)));
    	ListNode* tail = newHead;
    
    	while(p1 != NULL && p2 != NULL){
    		if (p1 -> val <= p2 -> val){
    			tail -> next = p1;
    			tail = p1;
    			p1 = p1 -> next;
    		}else {
    			tail -> next = p2;
    			tail = p2;
    			p2 = p2 -> next;	
    		}	
    	}
    	if (p1){
    	 	tail -> next = p1;
    	} 
    	if(p2){
    		tail -> next = p2;
    	}
    
    	return newHead -> next;
    }                            
{% endcodeblock %}                            
        
        
                         
                    
                    