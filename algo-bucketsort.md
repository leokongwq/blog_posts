---
layout: post
comments: true
title: 桶排序
date: 2017-01-19 15:37:49
tags:
categories:
- 算法
---

### 算法介绍

下面定义出自wikipedia[https://zh.wikipedia.org/wiki/%E6%A1%B6%E6%8E%92%E5%BA%8F](https://zh.wikipedia.org/wiki/%E6%A1%B6%E6%8E%92%E5%BA%8F)

桶排序（Bucket sort）或所谓的箱排序，是一个排序算法，工作的原理是将数组分到有限数量的桶里。每个桶再个别排序（有可能再使用别的排序算法或是以递归方式继续使用桶排序进行排序）。桶排序是鸽巢排序的一种归纳结果。当要被排序的数组内的数值是均匀分配的时候，桶排序使用线性时间（Θ(n)）。但桶排序并不是比较排序，他不受到O(n log n)下限的影响。

<!-- more -->

桶排序以下列程序进行：

1. 设置一个定量的数组当作空桶子。
2. 寻访序列，并且把项目一个一个放到对应的桶子去。
3. 对每个不是空的桶子进行排序。
4. 从不是空的桶子里把项目再放回原来的序列中。

### 一个例子

```java
/**
 * 桶排序
 */
public class SortAlgoBucketSort {
    /**
     * 算法基本思想: 桶排序的思想近乎彻底的分治思想。假设现在需要对一亿个数进行排序。
     * 我们可以将其等长地分到10000个虚拟的“桶”里面，这样，平均每个桶只有10000个数。
     * 如果每个桶都有序了，则只需要依次输出为有序序列即可。具体思路是这样的：
     * 1. 将待排数据按一个映射函数f(x)分为连续的若干段。理论上最佳的分段方法应该使数据平均分布；
     * 实际上，通常采用的方法都做不到这一点。显然，对于一个已知输入范围在【0，10000】的数组，
     * 最简单的分段方法莫过于x/m这种方法，例如，f(x)=x/100。
     * "连续的”这个条件非常重要，它是后面数据按顺序输出的理论保证
     * 2. 分配足够的桶，按照f(x)从数组起始处向后扫描，并把数据放到合适的桶中。对于上面的例子，
     * 如果数据有10000个，则我们需要分配101个桶（因为要考虑边界条件：f(x)=x/100会产生【0，100】共101种情况），
     * 理想情况下，每个桶有大约100个数据
     *3. 对每个桶进行内部排序，例如，使用快速排序。注意，如果数据足够大，这里可以继续递归使用桶排序，
     * 直到数据大小降到合适的范围。
     * 4. 按顺序从每个桶输出数据。例如，1号桶【112，123，145，189】，2号桶【234，235，250，250】，3号桶【361】，
     * 则输出序列为【112，123，145，189，234，235，250，250，361】。
     * 5. 排序完成
     *
     * @param arr 输入的待排序数组
     * @return 排序后的数组
     */
    public static int[] bucketSort(int[] arr){
        //10个桶, 每个桶里面10个元素
        int bucketNum = 10;
        Integer[][] buckets = new Integer[bucketNum][arr.length];
        for (int num : arr){
            int bucketIdx = num / 10;
            //查找空位,并放入指定编号的桶
            for (int j = 0; j < arr.length; j++){
                if (buckets[bucketIdx][j] == null){
                    buckets[bucketIdx][j] = num;
                    break;
                }
            }
        }

        //对每个桶进行排序
        //小桶排序
        for (int i = 0; i < buckets.length; i++){
            //insertion sort
            for (int j = 1; j < buckets[i].length; ++j){
                if(buckets[i][j] == null){
                    break;
                }
                int value = buckets[i][j];
                int position = j;
                while (position > 0 && buckets[i][position-1] > value){
                    buckets[i][position] = buckets[i][position-1];
                    position--;
                }
                buckets[i][position] = value;
            }
        }
        int k = 0;
        //输出
        for (int i = 0; i < bucketNum; i++){
            for (int j = 0; j < buckets[i].length; j++){
                if (null == buckets[i][j]){
                    continue;
                }
                arr[k++] = buckets[i][j];
            }
        }
        return arr;
    }

    private static void printArray(int[] arr){
        for (int num : arr){
            System.out.printf("%d,", num);
        }
    }
    public static void main(String[] args) {
        int[] arr = new int[]{3,1,41,62,73,22};
        arr = bucketSort(arr);
        printArray(arr);
    }
}
```

### 更多参考

[https://www.byvoid.com/blog/sort-radix](https://www.byvoid.com/blog/sort-radix)

