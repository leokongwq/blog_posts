---
layout: post
comments: true
title: 快速排序
date: 2017-01-19 16:40:23
tags:
categories:
- 算法
---

### 算计简介

介绍来自[快速排序](https://zh.wikipedia.org/wiki/%E5%BF%AB%E9%80%9F%E6%8E%92%E5%BA%8F)

快速排序（英语：Quicksort），又称划分交换排序（partition-exchange sort），一种排序算法，最早由东尼·霍尔提出。在平均状况下，排序n个项目要Ο(n log n)次比较。在最坏状况下则需要Ο(n2)次比较，但这种状况并不常见。事实上，快速排序通常明显比其他Ο(n log n)算法更快，因为它的内部循环（inner loop）可以在大部分的架构上很有效率地被实现出来。

<!-- more -->

### 算法原理

快速排序使用分治法（Divide and conquer）策略来把一个序列（list）分为两个子序列（sub-lists）。

步骤为：

1. 从数列中挑出一个元素，称为"基准"（pivot），
2. 重新排序数列，所有元素比基准值小的摆放在基准前面，所有元素比基准值大的摆在基准的后面（相同的数可以到任一边）。在这个分区结束之后，该基准就处于数列的中间位置。这个称为分区（partition）操作。
3. 递归地（recursive）把小于基准值元素的子数列和大于基准值元素的子数列排序。

递归的最底部情形，是数列的大小是零或一，也就是永远都已经被排序好了。虽然一直递归下去，但是这个算法总会结束，因为在每次的迭代（iteration）中，它至少会把一个元素摆到它最后的位置去。

在简单的伪代码中，此算法可以被表示为：

```
function quicksort(q)
     var list less, pivotList, greater
     if length(q) ≤ 1 {
         return q
     } else {
         select a pivot value pivot from q
         for each x in q except the pivot element
             if x < pivot then add x to less
             if x ≥ pivot then add x to greater
         add pivot to pivotList
         return concatenate(quicksort(less), pivotList, quicksort(greater))
     }
```

**原地（in-place）分区的版本**

上面简单版本的缺点是，它需要Ω(n)的额外存储空间，也就跟归并排序一样不好。额外需要的存储器空间配置，在实际上的实现，也会极度影响速度和缓存的性能。有一个比较复杂使用原地（in-place）分区算法的版本，且在好的基准选择上，平均可以达到O(log n)空间的使用复杂度。

```
function partition(a, left, right, pivotIndex)
     pivotValue := a[pivotIndex]
     swap(a[pivotIndex], a[right]) // 把pivot移到結尾
     storeIndex := left
     for i from left to right-1
         if a[i] <＝ pivotValue
             swap(a[storeIndex], a[i])
             storeIndex := storeIndex + 1
     swap(a[right], a[storeIndex]) // 把pivot移到它最後的地方
     return storeIndex
```

这是原地分区算法，它分区了标示为"左边（left）"和"右边（right）"的序列部分，借由移动小于a[pivotIndex]的所有元素到子序列的开头，留下所有大于或等于的元素接在他们后面。在这个过程它也为基准元素找寻最后摆放的位置，也就是它回传的值。它暂时地把基准元素移到子序列的结尾，而不会被前述方式影响到。由于算法只使用交换，因此最后的数列与原先的数列拥有一样的元素。要注意的是，一个元素在到达它的最后位置前，可能会被交换很多次。
一旦我们有了这个分区算法，要写快速排列本身就很容易：

```
procedure quicksort(a, left, right)
     if right > left
         select a pivot value a[pivotIndex]
         pivotNewIndex := partition(a, left, right, pivotIndex)
         quicksort(a, left, pivotNewIndex-1)
         quicksort(a, pivotNewIndex+1, right)
```

### 一个例子

```java
/**
 * 快速排序
 * 最差时间复杂度为: O(n2)
 * 平均时间复杂度为: O(n*lgN)
 * 空间复杂度: O(lgN)~O(n)
 * 稳定性: 不稳定
 */
public class SortAlgoQuickSort {
    /**
     *
     * @param arr
     * @param left
     * @param right
     * @return
     */
    public static void quickSort(int[] arr, int left, int right){
        if (left > right){
            return;
        }
        int i = left;
        int j = right;
        //基数
        int tmp = arr[left];
        //2个哨兵没有相遇
        while (i != j){
            //从右边的哨兵开始
            while (arr[j] >= tmp && i < j){
                j--;
            }
            //左边开始探测
            while (arr[i] <= tmp && i < j){
                i++;
            }
            //没有相遇, 交换
            if (i < j){
                swap(arr, i, j);
            }
        }
        //最终将基准数归位
        arr[left] = arr[i];
        arr[i] = tmp;
        //处理2个子序列
        quickSort(arr, left, i - 1);
        quickSort(arr, i + 1, right);
    }

    public static void swap(int[] arr, int i, int j){
        int tmp = arr[i];
        arr[i] = arr[j];
        arr[j] = tmp;
    }

    public static void main(String[] args) {
        int[] arr = new int[]{6,1,2,7,9,3,4,5,10,8};
        quickSort(arr, 0, arr.length - 1);
        for (int i : arr){
            System.out.printf("%d,", i);
        }
        System.out.println();
    }
}
```              
     
### 更多参考

[快速排序](https://zh.wikipedia.org/wiki/%E5%BF%AB%E9%80%9F%E6%8E%92%E5%BA%8F)

[白话经典算法系列之六 快速排序 快速搞定](http://blog.csdn.net/morewindows/article/details/6684558)

