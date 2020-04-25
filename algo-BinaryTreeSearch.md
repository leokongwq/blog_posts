---
layout: post
comments: true
title: 二叉排序树查找
date: 2017-01-20 12:19:19
tags:
categories:
- 算法
---

本文转载自：[http://www.cnblogs.com/zhuyf87/archive/2012/11/09/2763113.html](http://www.cnblogs.com/zhuyf87/archive/2012/11/09/2763113.html)

### 二叉排序树概念

二叉排序树又称“二叉查找树”、“二叉搜索树”。

二叉排序树：或者是一棵空树，或者是具有下列性质的二叉树：

1. 若它的左子树不空，则左子树上所有结点的值均小于它的根结点的值；

2. 若它的右子树不空，则右子树上所有结点的值均大于它的根结点的值；

3. 它的左、右子树也分别为二叉排序树。

<!-- more -->

{% asset_img 2012110918051618.jpg %}

二叉排序树通常采用二叉链表作为存储结构。中序遍历二叉排序树可得到一个依据关键字的有序序列，一个无序序列可以通过构造一棵二叉排序树变成一个有序序列，构造树的过程即是对无序序列进行排序的过程。每次插入的新的结点都是二叉排序树上新的叶子结点，在进行插入操作时，不必移动其它结点，只需改动某个结点的指针，由空变为非空即可。搜索、插入、删除的时间复杂度等于树高，期望O(logn)，最坏O(n)（数列有序，树退化成线性表，如右斜树）。

```
/* 二叉树的二叉链表结点结构定义 */
typedef  struct BiTNode    /* 结点结构 */
{
    int data;    /* 结点数据 */
    struct BiTNode *lchild, *rchild; /* 左右孩子指针 */
} BiTNode, *BiTree;
```

虽然二叉排序树的最坏效率是O(n)，但它支持动态查找，且有很多改进版的二叉排序树可以使树高为O(logn)，如AVL、红黑树等。

### 二叉排序树的查找算法

在二叉排序树b中查找x的过程为：

 1. 若b是空树，则搜索失败，否则：

 2. 若x等于b的根节点的数据域之值，则查找成功；

 3. 若x小于b的根节点的数据域之值，则搜索左子树；

 4. 若x大于b的根节点的数据域之值，则搜索右子树。

算法实现：

```
/* 递归查找二叉排序树T中是否存在key, */
/* 指针f指向T的双亲，其初始调用值为NULL */
/* 若查找成功，则指针p指向该数据元素结点，并返回TRUE */
/* 否则指针p指向查找路径上访问的最后一个结点并返回FALSE */
Status SearchBST(BiTree T, int key, BiTree f, BiTree *p) 
{  
    if (!T)    /*  查找不成功 */
    { 
        *p = f;  
        return FALSE; 
    }
    else if (key == T->data) /*  查找成功 */
    { 
        *p = T;  
        return TRUE; 
    } 
    else if (key < T-> data) 
        return SearchBST(T -> lchild, key, T, p);  /*  在左子树中继续查找 */
    else  
        return SearchBST(T -> rchild, key, T, p);  /*  在右子树中继续查找 */
}
```

### 二叉排序树的插入算法

利用查找函数，将关键字放到树中的合适位置。

```
/*  当二叉排序树T中不存在关键字等于key的数据元素时， */
/*  插入key并返回TRUE，否则返回FALSE */
Status InsertBST(BiTree *T, int key) 
{  
    BiTree p,s;
    if (!SearchBST(*T, key, NULL, &p)) /* 查找不成功 */
    {
        s = (BiTree)malloc(sizeof(BiTNode));
        s->data = key;  
        s->lchild = s->rchild = NULL;  
        if (!p) 
            *T = s;            /*  插入s为新的根结点 */
        else if (key<p->data) 
            p->lchild = s;    /*  插入s为左孩子 */
        else 
            p->rchild = s;  /*  插入s为右孩子 */
        return TRUE;
    } 
    else 
        return FALSE;  /*  树中已有关键字相同的结点，不再插入 */
}
```

### 二叉排序树的删除算法

在二叉排序树中删去一个结点，分三种情况讨论

1. 若*p结点为叶子结点，即PL(左子树)和PR(右子树)均为空树。由于删去叶子结点不破坏整棵树的结构，则只需修改其双亲结点的指针即可。

2. 若*p结点只有左子树PL或右子树PR，此时只要令PL或PR直接成为其双亲结点*f的左子树（当*p是左子树）或右子树（当*p是右子树）即可，作此修改也不破坏二叉排序树的特性。

3. 若*p结点的左子树和右子树均不空。在删去*p之后，为保持其它元素之间的相对位置不变，可按中序遍历保持有序进行调整。比较好的做法是，找到*p的直接前驱（或直接后继）*s，用*s来替换结点*p，然后再删除结点*s。

{% asset_img 2012110918070093.jpg %}

```
/* 若二叉排序树T中存在关键字等于key的数据元素时，则删除该数据元素结点, */
/* 并返回TRUE；否则返回FALSE。 */
Status DeleteBST(BiTree *T,int key)
{ 
    if(!*T) /* 不存在关键字等于key的数据元素 */ 
        return FALSE;
    else
    {
        if (key==(*T)->data) /* 找到关键字等于key的数据元素 */ 
            return Delete(T);
        else if (key<(*T)->data)
            return DeleteBST(&(*T)->lchild,key);
        else
            return DeleteBST(&(*T)->rchild,key);
         
    }
}

/* 从二叉排序树中删除结点p，并重接它的左或右子树。 */
Status Delete(BiTree *p)
{
    BiTree q,s;
    if((*p)->rchild==NULL) /* 右子树空则只需重接它的左子树（待删结点是叶子也走此分支) */
    {
        q=*p; *p=(*p)->lchild; free(q);
    }
    else if((*p)->lchild==NULL) /* 只需重接它的右子树 */
    {
        q=*p; *p=(*p)->rchild; free(q);
    }
    else /* 左右子树均不空 */
    {
        q=*p; s=(*p)->lchild;
        while(s->rchild) /* 转左，然后向右到尽头（找待删结点的前驱） */
        {
            q=s;
            s=s->rchild;
        }
        (*p)->data=s->data; /*  s指向被删结点的直接前驱（将被删结点前驱的值取代被删结点的值） */
        if(q!=*p)
            q->rchild=s->lchild; /*  重接q的右子树 */ 
        else
            q->lchild=s->lchild; /*  重接q的左子树 */
        free(s);
    }
    return TRUE;
}
```

### 二叉排序树性能分析

每个结点的Ci为该结点的层次数。最好的情况是二叉排序树的形态和折半查找的判定树相同，其平均查找长度和logn成正比（O(log2(n))）。最坏情况下，当先后插入的关键字有序时，构成的二叉排序树为一棵斜树，树的深度为n，其平均查找长度为(n + 1) / 2。也就是时间复杂度为O(n)，等同于顺序查找。因此，如果希望对一个集合按二叉排序树查找，最好是把它构建成一棵平衡的二叉排序树（平衡二叉树）。


### Java 版本

```
/**
 * Created with IntelliJ IDEA.
 */
public class BinarySearchTree<T extends Comparable<? super T>> {

    /**
     * 根节点
     */
    private BinaryNode<T> root;

    public BinarySearchTree(){
    }

    /**
     * 是否是空树
     * @return
     */
    public boolean isEmpty(){
        return root == null;
    }
    /**
     * 是否包含指定的元素
     * @param data
     * @return
     */
    public boolean contains(T data){
        return containsInternal(root, data);
    }

    private boolean containsInternal(BinaryNode<T> node, T data){
        if (node == null){
            return false;
        }
        int result = node.data.compareTo(data);
        if (result < 0){
            return containsInternal(node.right, data);
        } else  if (result > 0){
            return containsInternal(node.left, data);
        }else {
            return true;
        }
    }

    /**
     * 查找最小的数据项
     * @return
     */
    public  T  findMin(){
        if (isEmpty()){
            return null;
        }
        return findMinInternal(root).data;
    }

    private BinaryNode<T> findMinInternal(BinaryNode<T> node){
        if (node == null){
            return null;
        }
        if (node.left == null){
            return node;
        }
        return findMinInternal(node.left);
    }

    /**
     * 查找最大的数据项
     * @return
     */
    public T findMax(){
        if (isEmpty()){
            return null;
        }
        return findMaxInternal(root).data;
    }

    private BinaryNode<T> findMaxInternal(BinaryNode<T> node){
        if (node == null){
            return null;
        }
//        非递归的实现方式
//        if (node != null){
//            while (node.right != null){
//                node = node.right;
//            }
//        }
//        return node;
        if (node.right == null){
            return node;
        }
        return findMaxInternal(node.right);
    }

    public boolean insert(T data){
        this.root = this.insertInternal(root, data);
        return true;
    }

    /**
     * 将元素 data 插入到树中
     * @param node
     * @param data
     * @return
     */
    private BinaryNode<T> insertInternal(BinaryNode<T> node, T data){
        if (node == null){
            //可以是根节点, 或者新插入的节点
            return new BinaryNode<T>(data, null, null);
        }
        int result = data.compareTo(node.data);
        if (result > 0) {
            node.right = insertInternal(node.right, data);
        }else if (result < 0){
            node.left = insertInternal(node.left, data);
        }else {
            //什么都不做, 返回老的节点(也可以在节点的附加域上加一)
        }
        return node;
    }

    /**
     * 删除参数data指定的数据项
     * @param data
     * @return
     */
    public boolean remove(T data){
        BinaryNode<T> node = removeInternal(root, data);
        return node != null;
    }

    private BinaryNode<T> removeInternal(BinaryNode<T> node , T data){
        if (node == null){
            return null;
        }
        int result = data.compareTo(node.data);
        if (result < 0) {
            //待删除的节点在节点node的左子树, 节点node的左孩子赋值为删除目标元素后的替代节点
            //每次的调用可以将node看做是只有2个叶子节点的树
            node.left = removeInternal(node.left, data);
        }else if (result > 0){
            node.right = removeInternal(node.right, data);
        }else {
            //有2个子节点
            if (node.left != null && node.right != null){
                //查找左子树中的最大节点
                BinaryNode<T> leftMaxNode = findMaxInternal(node.left);
                //删除该节点(左子树中的最大节点)
                removeInternal(node.left, leftMaxNode.data);
                node.data = leftMaxNode.data;
            }else {
                // 有一个子节点, 将左孩子或右孩子提升
                node = node.left != null ? node.left : node.right;
            }
        }
        return node;
    }

    private static class BinaryNode<T> {
        private T data;
        private BinaryNode<T> left;
        private BinaryNode<T> right;

        public BinaryNode(T data){
            this(data, null, null);
        }

        public BinaryNode(T data, BinaryNode<T> left, BinaryNode<T> right) {
            this.data = data;
            this.left = left;
            this.right = right;
        }
    }

    public static void main(String[] args) {
        BinarySearchTree<Integer> searchTree =new BinarySearchTree<Integer>();
        searchTree.insert(6);
        searchTree.insert(8);
        searchTree.insert(1);
        searchTree.insert(2);
        searchTree.insert(3);
        searchTree.insert(4);
        System.out.println(searchTree.contains(2));
        System.out.println(searchTree.findMin());
        System.out.println(searchTree.findMax());
        searchTree.remove(3);
        System.out.println(searchTree.contains(3));
    }
}
```


