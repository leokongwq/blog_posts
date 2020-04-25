---
layout: post
comments: true
title: 二叉查找树
date: 2016-11-06 12:53:05
tags:
categories:
- 算法
---

### 定义

二叉查找树的定义非常简单，一句话就是左孩子比父节点小，右孩子比父节点大，还有一个特性就是”中序遍历“可以让结点有序
<!-- more -->
{% asset_img 2012072113392755.png %}

### 树节点

为了通用,使用了泛型.

```java
private static class TreeNode<T> {
        private T data;

        private TreeNode<T> left;

        private TreeNode<T> right;

        public TreeNode(){
        }

        public TreeNode(T data, TreeNode<T> left, TreeNode<T> right) {
            this.data = data;
            this.left = left;
            this.right = right;
        }
    }
```    

### 添加节点

```java
/**
 * 添加节点到树
 * @param data 要添加的数据
 * @return  返回最新添加的树节点
 */
public void add(T data){
    if (this.root == null){
        this.root = new TreeNode<T>(data, null, null);
        return;
    }
    addInternal(this.root, data);
}
private TreeNode<T> addInternal(TreeNode<T> node, T data){
    if (node == null){
        node = new TreeNode<T>(data, null, null);
    }
    if (node.data.compareTo(data) > 0 ){
        node.left = addInternal(node.left, data);
    }
    if (node.data.compareTo(data) < 0){
        node.right = addInternal(node.right, data);
    }
    return node;
}
```

### 遍历

```java
/**
 * 前序遍历(根-》左 -》右)
 */
public void rootLeftRightPrintTree(TreeNode treeNode){
    if (treeNode == null){
        return;
    }
    //先访问根节点
    System.out.print(treeNode.data + ",");
    //访问左子树
    rootLeftRightPrintTree(treeNode.left);
    //访问右子树
    rootLeftRightPrintTree(treeNode.right);
}
/**
 * 中序遍历(左-》根 -》右)
 */
public void leftRootRightPrintTree(TreeNode treeNode){
    if (treeNode == null){
        return;
    }
    //访问左子树
    leftRootRightPrintTree(treeNode.left);
    //先访问根节点
    System.out.print(treeNode.data + ",");
    //访问右子树
    leftRootRightPrintTree(treeNode.right);
}
/**
 * 后序遍历(左-》右 -》根)
 */
public void leftRightRootPrintTree(TreeNode treeNode){
    if (treeNode == null){
        return;
    }
    //访问左子树
    leftRightRootPrintTree(treeNode.left);
    //访问右子树
    leftRightRootPrintTree(treeNode.right);
    //先访问根节点
    System.out.print(treeNode.data + ",");
}        
```

### 范围查找

这个才是我们使用二叉查找树的最终目的，既然是范围查找，我们就知道了一个`min`和`max`，其实实现起来也很简单，

第一步：我们要在树中找到min元素，当然min元素可能不存在，但是我们可以找到min的上界，耗费时间为O(logn)。

第二步：从min开始我们中序遍历寻找max的下界。耗费时间为m。m也就是匹配到的个数。

最后时间复杂度为M+logN，要知道普通的查找需要O(N)的时间，比如在21亿的数据规模下，匹配的元素可能有30个，那么最后的结果也就是秒杀和几个小时甚至几天的巨大差异，后面我会做实验说明。

```java
/**
 * 范围查询
 * @param min
 * @param max
 * @return
 */
public List<T> rangeSearch(T min, T max){
    if (this.root == null){
        return Collections.emptyList();
    }
    List<T> result = new LinkedList<T>();
    rangeSearch(this.root, min, max, result);
    return result;
}

private void rangeSearch(TreeNode<T> tree, T min, T max, List<T> result){
    if (tree == null){
        return;
    }
    //遍历左子树查询下限
    if (min.compareTo(tree.data) < 0){
        rangeSearch(tree.left, min, max, result);
    }
    //在查找范围内
    if (tree.data.compareTo(min) >= 0 && tree.data.compareTo(max) <= 0){
        result.add(tree.data);
    }
    //遍历右子树（两种情况：1:找min的下限 2：必须在Max范围之内）
    if (min.compareTo(tree.data) > 0 || max.compareTo(tree.data) > 0){
        rangeSearch(tree.right, min, max, result);
    }
}
```

### 删除

对于树来说，删除是最复杂的，主要考虑两种情况。

**单孩子的情况**

这个比较简单，如果删除的节点有左孩子那就把左孩子顶上去，如果有右孩子就把右孩子顶上去，然后打完收工。

{% asset_img 2012072114200119.png %}

**左右都有孩子的情况**

首先可以这么想象，如果我们要删除一个数组的元素，那么我们在删除后会将其后面的一个元素顶到被删除的位置，如图

{% asset_img 2012072114272140.png %}

那么二叉树操作同样也是一样，我们根据”中序遍历“找到要删除结点的后一个结点，然后顶上去就行了，原理跟"数组”一样一样的。

{% asset_img 2012072114312025.png %}

**这个图有bug, 删除15, 被顶上去的应该是12**

```java
/**
 * 删除元素
 * @param data
 */
public void remove(T data){
    this.removeInternal(this.root, data);
}

private TreeNode<T> removeInternal(TreeNode<T> node, T data){
    if (node == null){
        return null;
    }
    //左子树
    if (data.compareTo(node.data) < 0){
        node.left = removeInternal(node.left, data);
    }
    //右子树
    if (data.compareTo(node.data) > 0){
        node.right = removeInternal(node.right, data);
    }
    //相等
    if (data.compareTo(node.data) == 0){
        if (node.left != null && node.right != null){
            //查找左子树中最大的节点
            TreeNode<T> destNode = findMax(node.left);
            //删除被移动的节点
            removeInternal(node.left, destNode.data);
            //赋值即可
            node.data = destNode.data;
        }else {
            node = node.left != null ? node.left : node.right;
        }
    }
    return node;
}

/**
 * 查找以参数root指定节点为根节点的二叉搜索树中值最大的节点
 * @param node
 * @return
 */
private TreeNode<T> findMax(TreeNode<T> node){
    if (node == null){
        return null;
    }
    while (node.right != null){
        node = node.right;
        findMax(node);
    }
    return node;
}
```

### 总结

二叉搜索树查找速度快的前提是: 该搜索树是平衡的, 如果不平衡, 则树的某个分支会退化为链表结构,这样就不会满足O(lgN)的时间复杂度,
如何保持树的平衡? 这个是需要继续学习的内容.

### 参考

[http://www.cnblogs.com/huangxincheng/archive/2012/07/21/2602375.html](http://www.cnblogs.com/huangxincheng/archive/2012/07/21/2602375.html)





