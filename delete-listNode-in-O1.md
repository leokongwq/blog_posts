---
layout: post
comments: true
title: O(1)时间内删除链表节点
date: 2016-10-12 17:03:12
tags:
    - 算法
categories:
    - 算法
---

### O(1)时间内删除链表节点 

**题目描述**：给定链表的头指针和一个节点指针，在O(1)时间删除该节点。[Google面试题]

**分析**：在单向链表中删除指定的节点，通常的做法都是从头结点开始遍历链表，逐个比较每个节点的下一个节点是否是要删除的节点，如果是，则通过`curNode->next = curNode->next->next; delete target;` 在O(1)的时间复杂度内删除指定节点似乎是不可行的，但其实可以换一种思路：**将下一个节点的值赋值给需要删除的节点，然后删除下一个节点。如果需要删除的节点是最后一个节点（target->next == null），则需要从头遍历列表找到target节点的上一个节点，然后删除。这样也是满足O(1)的时间复杂度的：假设链表总共有n个结点，我们的算法在n-1种情况下时间复杂度是O(1)，只有当给定的结点处于链表末尾的时候，时间复杂度为O(n)。那么平均时间复杂度[(n-1)*O(1)+O(n)]/n，仍然为O(1)。**

<!-- more -->

### 代码实现

{% codeblock lang:c%}
typedef struct LNode{  
   int data;  
   struct LNode *next;  
}LNode, *List; 
// LNode是定义的链表节点数据结构，List表示一个节点的指针
void InsertList(List &l, int data)//头插入节点  
{  
   List head;  
   LNode *p=new LNode;  
   p->data=data;  
   if(NULL==l)  
       p->next=NULL;  
   else  
       p->next=l;  
   l=p;  
}  
  
void PrintList(List l)//打印链表  
{  
   LNode *p=l;  
   while(p)  
   {  
       cout<<p->data<<" ";  
       p=p->next;  
   }  
   cout<<endl;  
}  

void DeleteNode(List &l, LNode *toBeDeleted)//删除链表l中的toBeDeleted节点  
{  
   LNode *p;  
   if(!l || !toBeDeleted)  
       return;  
 
   if(l==toBeDeleted)//若删除的节点时表头  
   {  
       p=l->next;  
       delete toBeDeleted ;  
       l=p;          
   }  
   else  
   {  
       if(toBeDeleted->next==NULL)//若删除的节点时最后一个节点  
       {  
           p=l;  
           while(p->next!=toBeDeleted)  
               p=p->next;  
           p->next=NULL;  
           delete toBeDeleted;  
       }  
       else//删除节点时中间节点  
       {  
           p=toBeDeleted->next;  
           toBeDeleted->data=p->data;  
           toBeDeleted->next=p->next;  
           delete p;  
 
       }  
 
   }  
  
}  
  
void main()  
{  
   List l=NULL;  
   LNode *p;  
   int n;  
   InsertList(l, 3);  
   InsertList(l, 7);  
   InsertList(l, 12);  
   InsertList(l, 56);  
   InsertList(l, 33);  
   InsertList(l, 78);  
   InsertList(l, 20);  
   InsertList(l, 89);  
 
   PrintList(l);  
   cout<<"请输入要删除的节点：";  
   cin>>n;  
 
   p=l;  
   while(p->data!=n && p)  
       p=p->next;  
 
   if(!p)  
   {  
       cout<<"不存在这样的节点！"<<endl;  
       return;  
   }  
   else  
       DeleteNode(l, p);  
 
   cout<<"删除后的链表：";  
   PrintList(l);  
{% endcodeblock %}
                                
                    
                    

