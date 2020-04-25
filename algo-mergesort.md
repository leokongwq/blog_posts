---
layout: post
comments: true
title: 归并排序
date: 2017-01-19 15:48:23
tags:
categories:
- 算法
---

### 算法简介

下面定义出自wikipedia [https://zh.wikipedia.org/wiki/%E5%BD%92%E5%B9%B6%E6%8E%92%E5%BA%8F](https://zh.wikipedia.org/wiki/%E5%BD%92%E5%B9%B6%E6%8E%92%E5%BA%8F)

归并排序（英语：Merge sort，或mergesort），是创建在归并操作上的一种有效的排序算法，效率为O(n log n)。1945年由约翰·冯·诺伊曼首次提出。该算法是采用分治法（Divide and Conquer）的一个非常典型的应用，且各层分治递归可以同时进行。

<!-- more -->

### 归并操作

归并操作（merge），也叫归并算法，指的是将两个已经排序的序列合并成一个序列的操作。归并排序算法依赖归并操作。

#### 迭代法

1. 申请空间，使其大小为两个已经排序序列之和，该空间用来存放合并后的序列
2. 设定两个指针，最初位置分别为两个已经排序序列的起始位置
3. 比较两个指针所指向的元素，选择相对小的元素放入到合并空间，并移动指针到下一位置
4. 重复步骤3直到某一指针到达序列尾

将另一序列剩下的所有元素直接复制到合并序列尾

#### 递归法

原理如下（假设序列共有n个元素）：

1. 将序列每相邻两个数字进行归并操作，形成 {\displaystyle floor(n/2)} floor(n/2)个序列，排序后每个序列包含两个元素
2. 将上述序列再次归并，形成 {\displaystyle floor(n/4)} floor(n/4)个序列，每个序列包含四个元素
3. 重复步骤2，直到所有元素排序完毕

### 一个例子

```java
/**
 *
 * 归并排序
 * 最好, 最坏, 平均时间复杂度 O(nlogn)
 * 空间复杂度 O(n)
 * 稳定排序
 */
public class SortAlgoMergeSort {

    /**
     * //将有二个有序数列a[start ... mid]和a[mid ... end]合并。
     * @param sorted 已经排序好的数组
     * @param start 该排序数组的的前半部分起始位置
     * @param mid 该排序数组的的前半部分中间位置
     * @param end 该排序数组的的前半部分中点位置
     * @param sorted 合并后的结果数组
     */
    private static void merge(int[] unsorted, int start, int mid, int end, int[] sorted){
        int i = start, j = mid + 1;
        int m = mid,   n = end;
        int k = 0;
        while (i <= m && j <= n){
            if (unsorted[i] > unsorted[j]){
                sorted[k++] = unsorted[j++];
            }else {
                sorted[k++] = unsorted[i++];
            }
        }
        while (i <= m){
            sorted[k++] = unsorted[i++];
        }
        while (j <= n){
            sorted[k++] = unsorted[j++];
        }
        //将排序后的数组值赋值给未排序的数组
        for (int v = 0; v < k; v++){
            unsorted[start + v] = sorted[v];
        }
    }

    /**
     * 归并排序
     * @param unSorted
     * @param first
     * @param last
     * @param sorted
     */
    private static void mergeSort(int[] unSorted, int first, int last, int[] sorted){
        if (first < last){
            int mid = first + (last - first) / 2;
            //前半部分进行归并排序,
            mergeSort(unSorted, first, mid, sorted);
            //后半部分进行归并排序,
            mergeSort(unSorted, mid + 1, last, sorted);
            //合并
            merge(unSorted, first, mid, last, sorted);
        }
    }

    public static void main(String[] args) {
        int[] arr = new int[]{3,2,4,6,5};
        int[] result = new int[arr.length];
        mergeSort(arr, 0, arr.length - 1, result);
        System.out.println(result);
    }
}
```

### 更多参考

[白话经典算法系列之五 归并排序的实现](http://blog.csdn.net/morewindows/article/details/6678165)


