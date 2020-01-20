---
title: 常见集合类初始化赋值方式
date: 2018-08-02 23:09:47
tags:
---

**1、匿名内部类方式**  
第一个花括号指匿名内部类  
第二个花括号指构造代码块
<!--more-->
```java
//匿名内部类方式
Map<String, Object> map = new HashMap(){
    {
        put("a", 1);
        put("b", 2);
        put("c", 3);
    }
};
List<Integer> list = new ArrayList(){
    {
        add(1);
        add(2);
    }
};
Set<Integer> set = new HashSet(){
    {
        add(1);
        add(2);
    }
};
```
**2、构造函数方式**  
对于list集合，可以用Arrays.asList借助构造函数初始化赋值
```java
//构造函数方式
List<Integer> list1 = new ArrayList<>(Arrays.asList(1,2,3));
```
