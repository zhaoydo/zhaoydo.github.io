---
title: java反射
categories:
- java
date: 2016-07-23 16:34:38
tags:
- java
---
# 几个java反射小例子（以String为例）  
## 获取类的三种方法
```
//反射机制获取类的三种方式
//通过类名获取
Class<?> class1 = Class.forName("java.lang.String");
//通过类的class属性获取
Class<?> class2 = String.class;
//通过类对象获取
Class<?> class3 = new String().getClass();
```
<!--more-->
## 实例化类对象，有参和无参构造函数  
```
//无参构造函数
Class<?> class4 = Class.forName("java.lang.String");
String instance1 = (String)class4.newInstance();
//有参构造函数(传入一个字符串)
Constructor con = class4.getConstructor(String.class);
String instance2 = (String)con.newInstance("lalala!");
```
## 对类属性操作  
```
//获取类属性
Class<?> class5 = Class.forName("java.lang.String");
//获取所有属性
Field[] fs = class5.getFields();
//根据属性名获取指定的属性（可能会抛出java.lang.NoSuchFieldException）
Field field = class5.getDeclaredField("CASE_INSENSITIVE_ORDER");
//获取实例
String instance3 = (String)class5.newInstance();
//反射机制打破封装
field.setAccessible(true);
//赋值
field.set(instance3, null);
```  
