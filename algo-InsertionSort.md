---
layout: post
comments: true
title: 插入排序
date: 2017-01-19 16:03:15
tags:
categories:
- 算法
---

### 算法简介

来自[插入排序](https://zh.wikipedia.org/wiki/%E6%8F%92%E5%85%A5%E6%8E%92%E5%BA%8F)

插入排序（英语：Insertion Sort）是一种简单直观的排序算法。它的工作原理是通过构建有序序列，对于未排序数据，在已排序序列中从后向前扫描，找到相应位置并插入。插入排序在实现上，通常采用in-place排序（即只需用到O(1)的额外空间的排序），因而在从后向前扫描过程中，需要反复把已排序元素逐步向后挪位，为最新元素提供插入空间。

<!-- more -->

### 算法历史

最早拥有排序概念的机器出现在1901至1904年间由Hollerith发明出使用基数排序法的分类机，此机器系统包括打孔，制表等功能，1908年分类机第一次应用于人口普查，并且在两年内完成了所有的普查数据和归档。 Hollerith在1896年创立的分类机公司的前身，为电脑制表记录公司（CTR）。他在电脑制表记录公司（CTR）曾担任顾问工程师，直到1921年退休，而电脑制表记录公司（CTR）在1924年正式改名为IBM。

### 算法描述

一般来说，插入排序都采用in-place在数组上实现。具体算法描述如下：

1. 从第一个元素开始，该元素可以认为已经被排序
2. 取出下一个元素，在已经排序的元素序列中从后向前扫描
3. 如果该元素（已排序）大于新元素，将该元素移到下一位置
4. 重复步骤3，直到找到已排序的元素小于或者等于新元素的位置
5. 将新元素插入到该位置后
6. 重复步骤2~5

如果比较操作的代价比交换操作大的话，可以采用二分查找法来减少比较操作的数目。该算法可以认为是插入排序的一个变种，称为二分查找插入排序。

### 示例

```java
/**
 * 插入排序
 * 最差时间复杂度为: O(n2)
 * 平均时间复杂度为: O(n2)
 * 空间复杂度: O(1)
 * 稳定性: 稳定
 */
public class SortAlgoInsertSort {

    /**
     * 插入排序
     * @param arr
     * @return
     */
    public static int[] insertSort(int[] arr){
        int n = arr.length;
        int j = 0;
        for (int i = 1; i < n; i++){
            int k = j;
            //查找插入位置
            while (k >= 0 && arr[k] > arr[i]){
                k--;
            }
            if (k == j){
                j++;
                continue;
            }
            int tmp = arr[i];
            for(int m = i; m > k + 1; m--){
                arr[m] = arr[m-1];
            }
            arr[k+1] = tmp;
            // 4 3 2 1
            j++;
        }
        return arr;
    }

    public static void main(String[] args) {
        //int[] arr = new int[]{3,1,5,8,2,4};
        int[] arr = new int[]{4, 3, 2, 1};
        arr = insertSort(arr);
        System.out.println(arr);
    }
}
```



