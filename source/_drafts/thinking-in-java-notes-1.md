---
title: thinking-in-java-notes-1
tags:
---

最近在读《Thinking in Java》，为了避免忘记一些（我认为）重要的知识点，就开了这个系列。博客的结构和《Java编程思想（英文版 第4版）》（机械工业出版社）的目录结构一致，以便快速找到出处。

## Everything is an Object

### You must create all the objects

#### Special case: primitive types

*  布尔值类型`boolean`的大小是不确定的，只要能容纳两个值即可，由虚拟机决定大小。
*  `char`类型的大小和C++的一个字节不同，是两个字节的，存储的是Unicode编号0x0000到0xFFFF的值。从[这篇SO帖子](https://stackoverflow.com/questions/1366068/whats-the-complete-range-for-chinese-characters-in-unicode)可以发现，常见中文字的范围是在这个范围内的，根本不虚……

#### Arrays in Java

*  即使是基本类型数组，也会被初始化为0.


### You never need to destroy an object

#### Scoping

*  Java使用花括号指定变量作用域，但是不允许如下的定义：

   ```java
   {
     int x = 1;
     {
       int x = 2; // Illegal
     }
   }
   ```

## Operators

### Casting operators

#### Promotion

*  除`boolean`外，小于`int`的类型（`short`, `byte`, `char`）在参与运算之前会被自动升格为`int`后再运算（自然也会得到`int`类型的结果）。