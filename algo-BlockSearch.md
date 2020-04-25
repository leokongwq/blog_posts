---
layout: post
comments: true
title: 分块查找
date: 2017-01-20 12:42:35
tags:
categories:
- 算法
---

### 算法思想

分块查找是折半查找和顺序查找的一种改进方法，折半查找虽然具有很好的性能，但其前提条件时线性表顺序存储而且按照关键码排序，这一前提条件在结点树很大且表元素动态变化时是难以满足的。而顺序查找可以解决表元素动态变化的要求，但查找效率很低。如果既要保持对线性表的查找具有较快的速度，又要能够满足表元素动态变化的要求，则可采用分块查找的方法。
分块查找的速度虽然不如折半查找算法，但比顺序查找算法快得多，同时又不需要对全部节点进行排序。当节点很多且块数很大时，对索引表可以采用折半查找，这样能够进一步提高查找的速度。[1] 
分块查找由于只要求索引表是有序的，对块内节点没有排序要求，因此特别适合于节点动态变化的情况。当增加或减少节以及节点的关键码改变时，只需将该节点调整到所在的块即可。在空间复杂性上，分块查找的主要代价是增加了一个辅助数组。
需要注意的是，当节点变化很频繁时，可能会导致块与块之间的节点数相差很大，没写快具有很多节点，而另一些块则可能只有很少节点，这将会导致查找效率的下降。

<!-- more -->

### 方法描述

分块查找要求把一个大的线性表分解成若干块，每块中的节点可以任意存放，但块与块之间必须排序。假设是按关键码值非递减的，那么这种块与块之间必须满足已排序要求，实际上就是对于任意的i，第i块中的所有节点的关键码值都必须小于第i+1块中的所有节点的关键码值。此外，还要建立一个索引表，把每块中的最大关键码值作为索引表的关键码值，按块的顺序存放到一个辅助数组中，显然这个辅助数组是按关键码值费递减排序的。查找时，首先在索引表中进行查找，确定要找的节点所在的块。由于索引表是排序的，因此，对索引表的查找可以采用顺序查找或折半查找；然后，在相应的块中采用顺序查找，即可找到对应的节点。
分块查找在现实生活中也很常用。例如，一个学校有很多个班级，每个班级有几十个学生。给定一个学生的学号，要求查找这个学生的相关资料。显然，每个班级的学生档案是分开存放的，没有任何两个班级的学生的学号是交叉重叠的，那么最好的查找方法实现确定这个学生所在的班级，然后再在这个学生所在班级的学生档案中查找这个学生的资料。上述查找学生资料的过程，实际上就是一个典型的分块查找。

### 一个例子

```java
package com.meiliinc.mls.algorithm.search;

import java.util.ArrayList;
import java.util.Collections;
import java.util.List;
import java.util.stream.IntStream;

/**
 * Created with IntelliJ IDEA.
 */
public class BlockSearch {

    public static void main(String[] args) {
        //arr 本身是由序的,但是不影响代码逻辑,
        int[] arr = IntStream.range(0, 10000).toArray();
        int idx = blockSearch(arr, 12);
        System.out.println(idx);
    }

    /**
     * 分块查找
     *
     * @param arr    包含 0 - 9999 的整形数组
     * @param target
     * @return
     */
    private static int blockSearch(int[] arr, int target) {
        if (arr == null || arr.length == 0) {
            return -1;
        }
        //将数组 arr分块 [0 - 99], [100 - 199] ... [9900 - 9999]
        int bucketNum = 100;
        List<List<Integer>> bucket = new ArrayList<>(bucketNum);
        for (int i = 0; i < 100; i++){
            bucket.add(new ArrayList<Integer>());
        }
        for (int num : arr) {
            int idx = num / bucketNum;
            List<Integer> subList = bucket.get(idx);
            subList.add(num);
        }
        //对每个块进行排序; 并将每个块的最大值放入 blockMaxNums
        List<Integer> blockMaxNums = new ArrayList<>();
        for (List<Integer> subList : bucket) {
            Collections.sort(subList);
            blockMaxNums.add(subList.get(subList.size() - 1));
        }
        //查找元素所在的区间
        int blockIdx = -1;
        for (int i = 0; i < blockMaxNums.size(); i++){
            if (target <= blockMaxNums.get(i)){
                blockIdx = i;
                break;
            }
        }
        if (blockIdx != -1){
            List<Integer> blockList = bucket.get(blockIdx);
            return blockList.indexOf(target);
        }
        return -1;
    }
}
```

### 思考

分块排序的难点是将数据集拆分为：**块有序的** 块序列。拆分完成后逻辑就比较简单了，只需要对每个块进行排序，然后将每个块中的最大值组织起来作为查找序列。 对该序列进行查找可以顺序查找，也可以采用二分查找。查找到目标区间后，再在目标区间进行二分查找或顺序查找就可以了。






