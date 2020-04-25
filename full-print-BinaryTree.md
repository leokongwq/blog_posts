---
layout: post
comments: true
title: full-print-BinaryTree
date: 2016-11-03 11:20:48
tags:
- java
categories:
- 算法
---

### 前言

> 二叉树是非常有用的数据结构.对二叉树的操作也能体现一个开发人员的基本功或者对数据结构的掌握能力.这篇文章就把二叉树的遍历方式整理一下.加强记忆.


### 遍历方式分类

1. 前序遍历-根左右
2. 中序遍历-左根右
3. 后序遍历-左右根

<!-- more -->

### 代码最靠谱

```java
import java.util.Stack;

/**
 * 二叉树的先序，中序，以及后序，递归以及非递归的实现
 */
public class FullScan {

    public static void main(String[] args) {
        Node head = createTree();
        recurseFront(head);
        recurseMid(head);
        recurseEnd(head);
        front(head);
        mid(head);
        endWith2Stack(head);
        endWithOneStack(head);
    }

    /**
     * 非递归实现的二叉树后序遍历<br>
     * 借助于一个栈进行实现
     * 
     * @param head
     */
    public static void endWithOneStack(Node head) {
        System.out.println();
        if (head == null) {
            return;
        } else {
            Stack<Node> stack = new Stack<Node>();
            stack.push(head);
            // 该节点代表已经打印过的节点，待会会及时的进行更新
            Node printedNode = null;
            while (!stack.isEmpty()) {
                // 获取 栈顶的元素的值，而不是pop掉栈顶的值
                head = stack.peek();
                // 如果当前栈顶元素的左节点不为空，左右节点均未被打印过，说明该节点是全新的，所以压入栈中
                if (head.getLeft() != null && printedNode != head.getLeft() && printedNode != head.getRight()) {
                    stack.push(head.getLeft());
                } else if (head.getRight() != null && printedNode != head.getRight()) {
                    // 第一层不满足，则说明该节点的左子树已经被打印过了。如果栈顶元素的右节点未被打印过，则将右节点压入栈中
                    stack.push(head.getRight());
                } else {
                    // 上面两种情况均不满足的时候则说明左右子树均被打印过，此时只需要弹出栈顶元素，打印该值即可
                    System.out.println("当前值为：" + stack.pop().getValue());
                    // 记得实时的更新打印过的节点的值
                    printedNode = head;
                }
            }
        }
    }

    /**
     * 非递归实现的二叉树的后序遍历<br>
     * 借助于两个栈来实现
     * 
     * @param head
     */
    public static void endWith2Stack(Node head) {
        System.out.println();
        if (head == null) {
            return;
        } else {
            Stack<Node> stack1 = new Stack<Node>();
            Stack<Node> stack2 = new Stack<Node>();

            stack1.push(head);
            // 对每一个头结点进行判断，先将头结点放入栈2中，然后依次将该节点的子元素放入栈1.
            // 顺序为left-->right。便是因为后序遍历为“左右根”
            while (!stack1.isEmpty()) {
                head = stack1.pop();
                stack2.push(head);
                if (head.getLeft() != null) {
                    stack1.push(head.getLeft());
                }

                if (head.getRight() != null) {
                    stack1.push(head.getRight());
                }
            }

            // 直接遍历输出栈2，即可实现后序遍历的节点值的输出
            while (!stack2.isEmpty()) {
                System.out.println("当前节点的值：" + stack2.pop().getValue());
            }
        }
    }

    /**
     * 非递归实现的二叉树的中序遍历
     * 
     * @param head
     */
    public static void mid(Node head) {
        System.out.println();
        if (head == null) {
            return;
        } else {
            Stack<Node> nodes = new Stack<Node>();

            // 使用或的方式是因为 第一次的时候栈中元素为空，head的非null特性可以保证程序可以执行下去
            while (!nodes.isEmpty() || head != null) {
                // 当前节点元素值不为空，则放入栈中，否则先打印出当前节点的值，然后将头结点变为当前节点的右子节点。
                if (head != null) {
                    nodes.push(head);
                    head = head.getLeft();
                } else {
                    Node temp = nodes.pop();
                    System.out.println("当前节点的值：" + temp.getValue());
                    head = temp.getRight();
                }
            }

        }
    }

    /**
     * 非递归实现的二叉树的先序遍历
     * 
     * @param head
     */
    public static void front(Node head) {
        System.out.println();

        // 如果头结点为空，则没有遍历的必要性，直接返回即可
        if (head == null) {
            return;
        } else {
            // 初始化用于存放节点顺序的栈结构
            Stack<Node> nodes = new Stack<Node>();
            // 先把head节点放入栈中，便于接下来的循环放入节点操作
            nodes.add(head);

            while (!nodes.isEmpty()) {
                // 取出栈顶元素，判断其是否有子节点
                Node temp = nodes.pop();

                System.out.println("当前节点的值：" + temp.getValue());
                // 先放入右边子节点的原因是先序遍历的话输出的时候左节点优先于右节点输出，而栈的特性决定了要先放入右边的节点
                if (temp.getRight() != null) {
                    nodes.push(temp.getRight());
                }
                if (temp.getLeft() != null) {
                    nodes.push(temp.getLeft());
                }
            }
        }
    }

    /**
     * 递归实现的先序遍历
     * 
     * @param head
     */
    public static void recurseFront(Node head) {
        System.out.println();

        if (head == null) {
            return;
        }
        System.out.println("当前节点值：" + head.getValue());
        recurseFront(head.left);
        recurseFront(head.right);
    }

    /**
     * 递归实现的中序遍历
     * 
     * @param head
     */
    public static void recurseMid(Node head) {
        System.out.println();
        if (head == null)
            return;
        recurseMid(head.getLeft());
        System.out.println("当前节点的值：" + head.getValue());
        recurseMid(head.getRight());
    }

    /**
     * 递归实现的后序遍历递归实现
     * 
     * @param head
     */
    public static void recurseEnd(Node head) {
        System.out.println();
        if (head == null)
            return;
        recurseEnd(head.getLeft());
        recurseEnd(head.getRight());
        System.out.println("当前节点的值为：" + head.getValue());
    }

    public static Node createTree() {
        // 初始化节点
        Node head = new Node(1);
        Node headLeft = new Node(2);
        Node headRight = new Node(3);
        Node headLeftLeft = new Node(4);
        Node headLeftRigth = new Node(5);
        Node headRightLeft = new Node(6);
        // 为head节点 赋予左右值
        head.setLeft(headLeft);
        head.setRight(headRight);

        headLeft.setLeft(headLeftLeft);
        headLeft.setRight(headLeftRigth);
        headRight.setLeft(headRightLeft);

        // 返回树根节点
        return head;
    }

}
```

### 总结

递归和栈是孪生兄弟,在解决问题时能用递归实现的问题一般栈也可以.