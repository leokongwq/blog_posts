---
layout: post
comments: true
title: 素数相关
date: 2016-11-16 14:17:33
tags:
- algo
categories:
- 算法
---

### 素数定义

> 素数（质数）：只有1和它本身两个因数的正整数叫做素数（质数）。例如，7是一个素数因为它只有1和7两个因数. 注意：1不是素数，因为它只有一个因数.

<!-- more -->

### 题目1:判断正整数N是否为素数

#### 解法1(简单粗暴)

```java
private static boolean isPrime(int n){
    if (n <= 1){
        return false;
    }
    for (int i = 2; i < n; i++){
        if (n % i == 0){
            return false;
        }
    }
    return true;
}
```
评价: 确实简单粗暴, 拿到问题首先就能思考出的解法.


#### 解法2(一点优化)

如果N有除了1和它本身的其它因数, 则该因数的范围一定是在[2, N/2]这个区间.

```java
private static boolean isPrime(int n){
    if (n <= 1){
        return false;
    }
    for (int i = 2; i < n / 2; i++){
        if (n % i == 0){
            return false;
        }
    }
    return true;
}
```

#### 解法3

除了2以外，所有可能的质因数都是奇数(如果是偶数, 则肯定能被2整除). 先尝试 2, 再尝试2以后的所有奇数直到N/2.

```java
private static boolean isPrime(int n){
    if (n <= 1){
        return false;
    }
    if (n == 2){
        return true;
    }
    if (n % 2 == 0){
        return false;
    }
    int max = n / 2;
    int i = 3;
    while (i <= max){
        if (n % i == 0){
            return false;
        }
        i += 2;
    }
    return true;
}
```

### 解法4

一个正整数等于它的平方根的平方, 也可以表示为它的两个因数的乘积. 其中两个因数必须有一个是小于它的平方根的.

```java
private static boolean isPrime(int n){
        if (n <= 1){
            return false;
        }
        if (n == 2){
            return true;
        }
        if (n % 2 == 0){
            return false;
        }
        int max = (int) Math.sqrt((double)n);
        int i = 3;
        while (i <= max){
            if (n % i == 0){
                return false;
            }
            i += 2;
        }
        return true;
    }
```

### 求N以内的所有素数

#### 方法一

```java
public static List<Integer> findPrime(int n) {
    List<Integer> primes = new ArrayList<Integer>();
    primes.add(2);
    for(int i = 3; i <= n; i+=2) { //除了2, 所有的素数都是奇数
        for(int j = 0; j < primes.size(); j++) {
            if(i % primes.get(j) == 0) {
                break;
            }
            if(j == primes.size() - 1) { 
                primes.add(i); 
                break; 
            }
        }
    }
    return primes;
}
```
#### 方法二(筛法)
    
```java
public class Prime {
    // 返回n以内的素数列表
    public static int[] getPrimes(int n) {
        if(n < 2 || n > 1000000) {
            throw new IllegalArgumentException("输入参数n错误！");
        }
        int[] a = new int[n];
        for(int i = 2; i < n; i ++) {
            a[i] = i;
        }
        // 筛法
        for(int i = 2; i < n; i ++) {
            if (a[i] != 0) {
                for(int j = i * 2; j < n; j = j + i) {
                    a[j] = 0;
                }
            }
        }
        int count = 0;
        for(int i = 2; i < n; i++) {
            if (a[i] != 0) {
                count ++;
            }
        }
        if (count > 0) {
            int[] primes  = new int[count];
            int j = 0;
            for (int i = 2; i < n; i ++) {
                if(a[i] != 0) {
                    primes[j] = a[i];
                    j ++;
                }
            }
            return primes;
        }
        return null;
    }

    public static void main(String[] args) {
        int[] a = getPrimes(10);
        for (int i = 0; i < a.length; i ++) {
            System.out.println(a[i]);
        }
        System.out.println();
    }
}
```

#### 我的方法

```java
private static void test(int n){
        //假设所有的数都是素数
        int[] primes = new int[n + 1];
        //1不是素数
        primes[1] = 1;
        for (int i = 2; i <=n; i++){
            if (i == 2){
                continue;
            }
            //所有能被二整除的数都不是素数
            if (i % 2 == 0){
                primes[i] = 1;
            }
        }
        for (int i = 3; i <=n; i+=2){
            if (primes[i] == 0){ //是奇数下标, 把它的所有倍数下标置为1
                int k = 2;
                int m = k * i;
                while (m <= n){
                    primes[m] = 1;
                    m = i * (k++);
                }
            }
        }
        //
//        for (int i = 2; i < n; i++){
//            if (primes[i] == 0){
//                System.out.print(i + ",");
//            }
//        }
        System.out.println();
    }
```

### 参考

[https://program-think.blogspot.com/2011/12/prime-algorithm-1.html](https://program-think.blogspot.com/2011/12/prime-algorithm-1.html)
[http://blog.csdn.net/liyuming0000/article/details/49622771](http://blog.csdn.net/liyuming0000/article/details/49622771)
[http://coolshell.cn/articles/3738.html](http://coolshell.cn/articles/3738.html)
[http://dongxicheng.org/structure/prime/](http://dongxicheng.org/structure/prime/)

