---
title: java集合
categories:
- java
date: 2016-07-23 16:34:38
tags:
- java
---
#  java集合
## 关系图
{% asset_img 1.png 集合关系图 image %}
## Collection接口
java集合类库姜接口（interface）与实现（implementation）分离。除Map集合（实现Map接口）外，List，Set，Queue都实现了Collection接口  
<!--more-->
### Collection主要方法  
返回值 | 方法名 | 说明
--- | ---- | --------
boolean | add(E o) | 添加指定方法
boolean | addAll(Collection<? extends E> c) | 将集合中所有元素添加到新的集合中
void | clear() | 移除所有元素
boolean | contains(Object o) | 集合是否包含特定元素
boolean | containsAll(Collection<?> c) | 集合是否包含特定集合中所有元素
boolean | isEmpty() | 集合是否为空
Iterator<E> | iterator() | 返回迭代器
boolean | remove(Object o) | 移除特定元素
Object[] | toArray() | 返回包含集合所有元素的数组（集合转数组）
<T> T[] | toArray(T[] a) | 同上（利用泛型返回指定数据类型的数组）  
### Set接口，两个实现类HashSet、TreeSet
-  HashSet：Set接口最常用的实现类，无序、元素唯一、可为null  
-  TreeSet：有序的Set集合，根据创建时提供的Comparator排序
### List接口，两个实现类ArrayList、LinkedList
- ArrayList：动态数组，有序可重复，可以用get和set方法随机访问每个元素（LinkedList只能利用迭代器）
- LinkedList：双向链接的链表（java中所有的链表都是双向链接的，即每个节点存放下一个和上一个节点的引用），每次查找元素都要遍历（LinkedList实现方法中做了优化，如果索引大于size()/2就从尾端遍历）  
```
/**
 * LinkedArray实现类获取指定索引节点的方法
 * index>(size/2)从尾端遍历
 */
Node<E> node(int index) {
    // assert isElementIndex(index);

    if (index < (size >> 1)) {
        Node<E> x = first;
        for (int i = 0; i < index; i++)
            x = x.next;
        return x;
    } else {
        Node<E> x = last;
        for (int i = size - 1; i > index; i--)
            x = x.prev;
        return x;
    }
}
```
## Map接口
 存放键值对（key/value），无序、key不能重复，key可为null，常用的有两种实现HashMap和TreeMap  
提供三个方法与Collection联系
```
Set<K> keySet();//获取key的set集合
Collection<V> values();//获取values集合
Set<Map.Entity<K,V>> entrySet();//将每一个Map键值对放到Set集合里面
```
### HashMap
最常用的Map集合
### TreeMap
根据创建时提供的Comparator排序  
HashMap和TreeMap看起来和HashSet、TreeSet相似，验证一下
```
Map<String,String> map = new HashMap<String, String>();
map.put("a", null);
map.put("b", null);
System.out.println(map.hashCode()==map.keySet().hashCode());
```
打印出：true
# Java库中的具体集合  

集合类型 | 描述 
--- | --- 
ArrayList | 一种可以动态增长和缩减的索引序列
LinkedList | 一种可以在任何位置进行高效地插入和删除操作的有序序列
ArrayDeque | 一种用循环数组实现的双端队列
HashSet | 一种没有重复元素的集合
TreeSet | 一种有序集
EnumSet | 一种包含枚举类型值的集
LinkedHashSet | 一种可以记住元素插入次序的集
PriorityQueue | 一种允许高效删除最小元素的集合
HashMap | 一种存储键/值关联的数据结构
TreeMap | 一种键值有序排列的映射表
EnumMap | 一种键值属于枚举类型的映射表
LinkedHashMap | 一种可以记住键/值添加次序的映射表
WeakHashMap | 一种其值无用武之地后可以被垃圾回收器回收的映射表
IdentityHashMap | 一种用==而不是用equals比较键值的映射表

