---
layout: post
comments: true
title: 打印NxN矩阵的对角线
date: 2016-10-12 16:41:04
tags: 
    - 算法
categories:
    - 算法

---

### 打印NxN矩阵的对角线

**题目描述**：打印NxN矩阵的对角线
**分析**： 这个问题可以分2步来解决，首先打印左上部分，然后再打印右下部分。如下的3*3矩阵

<!-- more -->

    1   2   3
    4   5   6
    7   8   9

可以先打印： [1], [2, 4], [3, 5, 7]
然后打印：[6, 8], [9]

代码如下：
{% codeblock lang:java %}
        /**
         * 打印 N* N 矩阵
         *     0  1  2
         * * * *  *  *
         * 0 * 1  2  3
         * 1 * 4  5  6
         * 2 * 7  8  9
         * @param arr
         */
        public static void printNxNMatrix(int[][] arr){
            int n = arr.length;
            //打印左上部分
            for (int i = 0; i < n; i++){
                int row = 0;
                int col = i;
                while (row <= i && col >= 0){
                    System.out.print(arr[row][col] + ",");
                    row++;
                    col--;
                }
                System.out.println();
            }
            //打印右下
            for (int j = 1; j < n; j++){
                int row  = j;
                int col = n - 1;
                while (row < n && col > 0){
                    System.out.print(arr[row][col] + ",");
                    row++;
                    col--;
                }
                System.out.println();
            }
        }
{% endcodeblock %}

### 扩展

如果从右上角开始打印？

#### 思路：
> 
将整个输出以最长的斜对角线分为两部分：右上部分和左下部分。
右上部分：对角线的起点在第一行，列数递减，对角线上相邻元素之间横坐标和纵坐标均相差1；
左下部分：对角线的起点在第一列上，行数递减，对角线上相邻元素之间横坐标和纵坐标均相差1；
复杂度：O(n^2)

#### 代码如下：
{% codeblock lang:java %}
    /**
     * 以对角线的方式打印n*n矩阵
     * @param data  矩阵数组
     * @param n 矩阵的维度
     */
    public void print(int[][] data, int n) {
        // 打印右上部分
        for (int i = n - 1; i >= 0; i--) {
            int row = 0;
            int col = i;
            while ((row >= 0 && row < n) && (col >= 0 && col < n)) {
                System.out.println(data[row][col]);
                row++;
                col++;
            }
        }
    
        // 打印左下部分
        for (int i = 1; i < n; i++) {
            int row = i;
            int col = 0;
            while ((row >= 0 && row < n) && (col >= 0 && col < n)) {
                System.out.println(data[row][col]);
                row++;
                col++;
            }
        }
    }
{% endcodeblock %}
                    

