---
layout: post
comments: true
title: 用栈实现后缀表达式的计算
date: 2016-11-18 12:11:47
tags:
categories:
- 算法
---

> 栈非常常见并且用的场景非常多.原理和实现掌握起来很简单, 它的用法取决于问题的场景和你的想象力.
 
<!-- more --> 
 
### 用栈来实现后缀表达式(算术表达式的逆波兰表示)的计算
  
```java
/**
 * Created with IntelliJ IDEA.
 * User: jiexiu
 * Date: 16/11/18
 * Time: 上午10:52
 * Email:jiexiu@mogujie.com
 */
public class MyStack {

    public static void main(String[] args) {
        //后缀表达式
        String exp = "6523+8*+3+*";
        int result = calculatePostfixExp(exp);
        System.out.println("calculate result is : " + result);
    }

    /**
     * 用栈计算后缀表达式的值
     * @param exp
     * @return
     */
    private static int calculatePostfixExp(String exp){
        int n = exp.length();
        Stack<String> stack = new Stack<String>();
        for (int i = 0; i < n; i++){
            char c = exp.charAt(i);
            //如果是数字直接入栈
            if (Character.isDigit(c)){
                stack.push(Character.toString(c));
            }else {
                //操作符
                int c1 = Integer.parseInt(stack.pop());
                int c2 = Integer.parseInt(stack.pop());
                int opRet = oprateTwoNum(c1, c2, c);
                stack.push(String.valueOf(opRet));
            }
        }
        return Integer.parseInt(stack.pop());
    }

    private static int oprateTwoNum(int n, int m, char op){
        switch (op){
            case '+' :
                return n + m;
            case '-' :
                return n - m;
            case '*' :
                return n * m;
            case '/' :
                return n / m;
            default:
                return 0;
        }
    }
}
```  

### 备注

上面的代码比较粗, 只能计算简单的`+0*/`表达式, 如果有括号就不适用了.
