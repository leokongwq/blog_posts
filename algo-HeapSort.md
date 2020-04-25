---
layout: post
comments: true
title: 堆排序
date: 2017-01-19 17:24:52
tags:
categories:
- 算法
---

### 算法简介

简介来自:[堆排序](https://zh.wikipedia.org/wiki/%E5%A0%86%E6%8E%92%E5%BA%8F)

堆排序（Heapsort）是指利用堆这种数据结构所设计的一种排序算法。堆是一个近似完全二叉树的结构，并同时满足堆积的性质：即子结点的键值或索引总是小于（或者大于）它的父节点。

<!-- more -->

#### 堆节点的访问

通常堆是通过一维数组来实现的。在数组起始位置为0的情形中：

- 父节点i的左子节点在位置(2*i+1);
- 父节点i的右子节点在位置(2*i+2);
- 子节点i的父节点在位置floor((i-1)/2);

#### 堆的操作

在堆的数据结构中，堆中的最大值总是位于根节点(在优先队列中使用堆的话堆中的最小值位于根节点)。堆中定义以下几种操作：

- 最大堆调整（Max_Heapify）：将堆的末端子节点作调整，使得子节点永远小于父节点
- 创建最大堆（Build_Max_Heap）：将堆所有数据重新排序
- 堆排序（HeapSort）：移除位在第一个数据的根节点，并做最大堆调整的递归运算

### 一个例子

```java
/**
 * 堆排序
 */
public class SortAlgoHeapSort {

    /**
     * 移动节点i
     * @param arr
     * @param i
     */
    private static void shiftUp(int[] arr, int i){
        int temp = arr[i];
        int j = (i - 1) / 2; //父节点
        while (j >= 0 && i != 0){
            if (arr[j] <= temp){ //父节点小于等于子节点
                break;
            }
            arr[i] = arr[j]; // 值较大的下移
            i = j;
            j = (i - 1) / 2;
        }
        arr[i] = temp;
    }

    /**
     * 添加都是添加到最后
     * @param arr 堆数组
     * @param n 添加元素的下标
     * @param num 待添加的元素
     */
    private static void heapInsert(int[] arr, int n, int num){
        arr[n] = num;
        shiftUp(arr, n);
    }

    /**
     *
     * @param arr 待调整的堆化数组
     * @param i 待调整的节点下标
     * @param n 堆化数组大小
     */
    private static void shiftDown(int[] arr, int i, int n){
        int temp = arr[i];
        int j = 2 * i + 1;
        while (j < n) {
            //在左右孩子中找最小的
            if (j + 1 < n && arr[j + 1] < arr[j]) {
                j++;
            }
            if (arr[j] >= temp){
                break;
            }
            arr[i] = arr[j];     //把较小的子结点往上移动,替换它的父结点
            i = j;
            j = 2 * i + 1;
        }
        arr[i] = temp;
    }

    /**
     * 删除都是从顶部开始删
     * @param arr
     */
    private static void heapDelete(int[] arr, int n){
        //删除第一个元素
        int temp = arr[n - 1];
        arr[n - 1] = arr[0];
        arr[0] = temp;
        //向下调整
        shiftDown(arr, 0, n - 1);
    }

    /**
     * 建立堆
     * @param arr
     * @param n
     */
    private static void createHeap(int[] arr, int n) {
        for (int i = n / 2 - 1; i >= 0; i--){
            shiftDown(arr, i, n);
        }
    }

    public static int[] heapSort(int[] arr, int n){
        for (int i = n - 1; i >= 1; i--) {
            int temp = arr[0];
            arr[0] = arr[n - 1];
            arr[n - 1] = temp;
            shiftDown(arr, 0, i);
        }
        return arr;
    }

    public static void main(String[] args) {
        int[] arr = new int[]{3,1,41,62,73,22};
        int n = arr.length;
        createHeap(arr, n);
        heapSort(arr, n);
        System.out.println(arr);
    }
}
```



