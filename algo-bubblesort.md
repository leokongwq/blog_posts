---
layout: post
comments: true
title: 冒泡排序
date: 2017-01-19 15:32:45
tags:
categories:
- 算法
---

### 算法简介

冒泡排序（英语：Bubble Sort，台湾另外一种译名为：泡沫排序）是一种简单的排序算法。它重复地走访过要排序的数列，一次比较两个元素，如果他们的顺序错误就把他们交换过来。走访数列的工作是重复地进行直到没有再需要交换，也就是说该数列已经排序完成。这个算法的名字由来是因为越小的元素会经由交换慢慢“浮”到数列的顶端。

冒泡排序对 N 个项目需要O(n2)的比较次数，且可以原地排序。尽管这个算法是最简单了解和实现的排序算法之一，但它对于少数元素之外的数列排序是很没有效率的。

冒泡排序是与插入排序拥有相等的运行时间，但是两种算法在需要的交换次数却很大地不同。在最好的情况，冒泡排序需要O(n2)次交换，而插入排序只要最多 O(N)交换。冒泡排序的实现（类似下面）通常会对已经排序好的数列拙劣地运行O(n2)，而插入排序在这个例子只需要 O(n)个运算。因此很多现代的算法教科书避免使用冒泡排序，而用插入排序替换之。冒泡排序如果能在内部循环第一次运行时，使用一个旗标来表示有无需要交换的可能，也可以把最好的复杂度降低到 O(n)。在这个情况，已经排序好的数列就无交换的需要。若在每次走访数列时，把走访顺序反过来，也可以稍微地改进效率。有时候称为鸡尾酒排序，因为算法会从数列的一端到另一端之间穿梭往返。

冒泡排序算法的运作如下：

1. 比较相邻的元素。如果第一个比第二个大，就交换他们两个。
2. 对每一对相邻元素作同样的工作，从开始第一对到结尾的最后一对。这步做完后，最后的元素会是最大的数。
3. 针对所有的元素重复以上的步骤，除了最后一个。
4. 持续每次对越来越少的元素重复上面的步骤，直到没有任何一对数字需要比较。

由于它的简洁，冒泡排序通常被用来对于程序设计入门的学生介绍算法的概念。

### 代码实现

```java
/**
 *
 * 冒泡排序(时间换空间)
 * 最差时间复杂度为: O(n2)
 * 平均时间复杂度为: O(n2)
 * 空间复杂度: O(1)
 * 稳定性: 稳定
 */
public class SortAlgoBubbleSort {

    public static int[] bubbleSort(int[] arr){
        int n = arr.length;
        for (int i = 0; i < n - 1; i++){
            for (int j = i + 1; j < n; j++){
                if (arr[j] < arr[i]){
                    int tmp = arr[j];
                    arr[j] = arr[i];
                    arr[i] = tmp;
                }
            }
        }
        return arr;
    }

    /**
     * 冒泡排序总共需要比较n-1轮(最后一轮只有一个元素,没有比较的必要),每一轮只能将一个元素归位
     * 每一轮都需要将该轮(idx - 1)位置的元素和所有未归位的元素进行一一比较。
     * @param arr
     * @return
     */
    public static int[] bubbleSortV1(int[] arr){
        for (int i = 0; i < arr.length - 1; i++){
            for (int j = i; j < arr.length - 1; j++){
                if (arr[j] > arr[j+1]){
                    int tmp = arr[j];
                    arr[j] = arr[j+1];
                    arr[j+1] = tmp;
                }
            }
        }
        return arr;
    }

    public static void main(String[] args) {
        int[] arr = new int[]{4, 3, 2, 1};
        arr = bubbleSort(arr);
        System.out.println(arr);

        arr = bubbleSortV1(arr);
        System.out.println(arr);
    }
}
```

### 参考

[冒泡排序](https://zh.wikipedia.org/wiki/%E5%86%92%E6%B3%A1%E6%8E%92%E5%BA%8F)

