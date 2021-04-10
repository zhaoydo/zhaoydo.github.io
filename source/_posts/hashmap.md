---
title: Java7/8 中HashMap和ConcurrentHashMap源码
date: 2021-01-19 20:00:36
updated: 2021-03-30 00:00:00
tags:
---
读Java7/8 中HashMap和ConcurrentHashMap源码，理解其数据结构与并发处理。  
<!--more-->
文章参考了[https://javadoop.com/post/hashmap#toc_16](https://javadoop.com/post/hashmap#toc_16 "Java7/8 中的 HashMap 和 ConcurrentHashMap 全解析"),图片也是直接引用这位博主的  
## Java7 HashMap
### 总体结构
![hashmap](https://www.javadoop.com/blogimages/map/1.png)
java7中，HashMap是一个数组，数组每个节点是一个单向链表，链表的每个节点是HashMap的一个元素。  
**几个重要的变量**  
**loadFactor** 负载因子，默认0.75  
**capacity** 当前数组容量，2^n,每次扩容\*2   
**size** HashMap已存在元素数量  
**threshold** 扩容阈值，=loadFactor*capacity  
### 初始化
构造函数初始化很简单，主要是新建数组（new Entry[capacity]）  
```java
/**
 * 初始化
 * @param initialCapacity 数组初始容量 默认16
 * @param loadFactor 负载因子 默认0.75f
 */
public HashMap(int initialCapacity, float loadFactor) {
    //参数校验
    if (initialCapacity < 0)
        throw new IllegalArgumentException("Illegal initial capacity: " +
                                           initialCapacity);
    if (initialCapacity > MAXIMUM_CAPACITY)
        initialCapacity = MAXIMUM_CAPACITY;
    if (loadFactor <= 0 || Float.isNaN(loadFactor))
        throw new IllegalArgumentException("Illegal load factor: " +
                                           loadFactor);

    // 找到>=initialCapacity的第一个2^n
    int capacity = 1;
    while (capacity < initialCapacity)
        capacity <<= 1;

    this.loadFactor = loadFactor;
    //计算扩容阈值
    threshold = (int)Math.min(capacity * loadFactor, MAXIMUM_CAPACITY + 1);
    //新建数组
    table = new Entry[capacity];
    useAltHashing = sun.misc.VM.isBooted() &&
            (capacity >= Holder.ALTERNATIVE_HASHING_THRESHOLD);
    //空方法
    init();
}
```
#### hash算法
```java
/**
 * 检查对象hashCode并对结果散列，用于防御低质量的散列函数
 * 这一点很关键，因为HashMap使用2次幂长度的hash表，在低位hashCode没有差异的时候会出现冲突
 * 注意key=null时 hashCoded固定0
 * @param k
 * @return
 */
final int hash(Object k) {
    int h = 0;
    //如果true 则对字符串执行可选hash，以减少弱hash导致的冲突概率
    if (useAltHashing) {
        if (k instanceof String) {
            return sun.misc.Hashing.stringHash32((String) k);
        }
        //与实例关联的随机值，为了减少hash冲突
        h = hashSeed;
    }
    //h（o or hashSeed）和key.hashCode()异或运算
    h ^= k.hashCode();
    //避免碰撞，进行四次扰动（核心是增强hash中每个位置数字的相关性，减少碰撞）
    // This function ensures that hashCodes that differ only by
    // constant multiples at each bit position have a bounded
    // number of collisions (approximately 8 at default load factor).
    h ^= (h >>> 20) ^ (h >>> 12);
    return h ^ (h >>> 7) ^ (h >>> 4);
}
```
> useAltHashing参数由配置项-Djdk.map.althashing.threshold决定是否开启。capacity>=threshold时开启。开启后会减少碰撞率  
> 1. 对字符串使用stringHash32单独计算hashCode  
> 2. hash时增加hashSeed参数  
### put
流程分三步：  
1. key=null, 直接去table[0]处理
2. 计算key的hash、在数组中的index。遍历对应数组位置链表中元素，equals则替换旧值
3. 不存在旧值则addEntry  

```java
public V put(K key, V value) {
    if (key == null)
        //key为null处理(最终entry存到table[0]中)
        return putForNullKey(value);
    //计算key的hash值
    int hash = hash(key);
    //由hash和数组长度计算对应数下标（key的hash值与数组长度取模）
    int i = indexFor(hash, table.length);
    //遍历对应数组下标上的链表
    for (Entry<K,V> e = table[i]; e != null; e = e.next) {
        Object k;
        if (e.hash == hash && ((k = e.key) == key || key.equals(k))) {
            V oldValue = e.value;
            //put赋值
            e.value = value;
            //LinkedHashMap,使用此方法将频繁访问的key移动到链表前面， 其他map的实现不做任何操作
            e.recordAccess(this);
            //返回旧值
            return oldValue;
        }
    }
    //hashMap被修改的次数（迭代器中有个expectedModCount。迭代时判断两者不相等则抛异常。保证迭代时不允许修改）
    modCount++;
    //没有遍历到key， 新增一个Entry
    addEntry(hash, key, value, i);
    return null;
}
```

put时，如果key不存在，则新增一个entry。  
1. 判断是否需要扩容。扩容时capacity*2，并重新计算key在数组中的index
2. 将数组添加到数组对应index处链表的表头  

```java
/**
 * 添加一个entry
 * @param hash hash值
 * @param key
 * @param value
 * @param bucketIndex 数组下标
 */
void addEntry(int hash, K key, V value, int bucketIndex) {
    //如果size>=threshold（扩容阈值（capacity * loadFactor）） table扩容2倍
    if ((size >= threshold) && (null != table[bucketIndex])) {
        //执行扩容
        resize(2 * table.length);
        hash = (null != key) ? hash(key) : 0;
        //重新计算下标
        bucketIndex = indexFor(hash, table.length);
    }
    //放到数组对应位置链表的表头
    createEntry(hash, key, value, bucketIndex);
}
```
```java
//将新Entry放到链表表头
void createEntry(int hash, K key, V value, int bucketIndex) {
	//拿到对应位置的链表
    Entry<K,V> e = table[bucketIndex];
    table[bucketIndex] = new Entry<>(hash, key, value, e);
    size++;
}
```
#### 数组扩容
前面新增Entry的代码中，如果size>threshold（扩容阈值（capacity * loadFactor）） table扩容2倍  
```java
//扩容
void resize(int newCapacity) {
    Entry[] oldTable = table;
    int oldCapacity = oldTable.length;
    //最大容量2^30
    if (oldCapacity == MAXIMUM_CAPACITY) {
        threshold = Integer.MAX_VALUE;
        return;
    }
    //new 一个新数组
    Entry[] newTable = new Entry[newCapacity];
    boolean oldAltHashing = useAltHashing;
    useAltHashing |= sun.misc.VM.isBooted() &&
            (newCapacity >= Holder.ALTERNATIVE_HASHING_THRESHOLD);
    //如果useAltHashing新旧不一样的话，重新计算hash
    boolean rehash = oldAltHashing ^ useAltHashing;
    //将当前数组复制到新数组中(遍历原数组、数组上每个链表元素)
    transfer(newTable, rehash);
    table = newTable;
    //重新计算扩容阈值threshold
    threshold = (int)Math.min(newCapacity * loadFactor, MAXIMUM_CAPACITY + 1);
}
```
经过扩容之后，可能原来某个节点中的元素会被分到新数组的两个节点。例如原来数组长度16，table[0]位置的链表元素，现扩容后，会分配到新数组的newTable[0]和newTable[16]中。

**扩容后是否重新计算hash值**  
看以下代码useAltHashing变了，hash值肯定也变了。这时候就重新计算每个元素的hash值  
```java
boolean oldAltHashing = useAltHashing;
useAltHashing |= sun.misc.VM.isBooted() &&
        (newCapacity >= Holder.ALTERNATIVE_HASHING_THRESHOLD);
//如果useAltHashing新旧不一样的话，重新计算hash
boolean rehash = oldAltHashing ^ useAltHashing;
```

### get
get操作很简单  
1. key=null 固定取table[0]
2. 对key进行hash，计算出数组index
3. 遍历table[index]中链表，取equals(key)的元素  

```java
public V get(Object key) {
    if (key == null)
        //获取key=null时的值，直接取table[0]中对应key（跟put时putForNullKey(value)对应）
        return getForNullKey();
    Entry<K,V> entry = getEntry(key);

    return null == entry ? null : entry.getValue();
}
```
```java
/**
 * 1、计算index
 * 2、遍历链表获取元素
 */
final Entry<K,V> getEntry(Object key) {
    int hash = (key == null) ? 0 : hash(key);
    for (Entry<K,V> e = table[indexFor(hash, table.length)];
         e != null;
         e = e.next) {
        Object k;
        if (e.hash == hash &&
            ((k = e.key) == key || (key != null && key.equals(k))))
            return e;
    }
    return null;
}
```
## Java7 ConcurrentHashMap
### 总体结构
![concurrentHashMap](https://www.javadoop.com/blogimages/map/3.png)  
ConcurrentHashMap由一个个Segment槽组成 ，每个槽中存放HashEntry数组，数组上节点存放链表。  
Segment数组不可扩容，每个槽内的HashEntry数组可以扩容。  
**几个重要的变量**  
**concurrencyLevel** 并发级别/并发Segment数量  
**initialCapacity** 指整个map数组容量，均分给Segment  
**loadFactor**  负载因子,实际中给Segment内部使用  
### 初始化
初始化segments数组，并初始化第一个元素  
```java
/**
 * @param initialCapacity 默认容量 默认16（总容量）
 * @param loadFactor 负载因子 默认0.75（给每个Segment内部使用）
 * @param concurrencyLevel 并发级别（代表分段锁数量） 默认16
 */
public ConcurrentHashMap(int initialCapacity,
                         float loadFactor, int concurrencyLevel) {
    //校验参数
    if (!(loadFactor > 0) || initialCapacity < 0 || concurrencyLevel <= 0)
        throw new IllegalArgumentException();
    if (concurrencyLevel > MAX_SEGMENTS)
        concurrencyLevel = MAX_SEGMENTS;
    //计算数组长度cap，锁数量ssize  保证是>= initialCapacity,concurrencyLevel的2^n
    int sshift = 0;
    int ssize = 1;
    //分段锁数量 2^n 默认情况下sshift=4 ssize=16
    while (ssize < concurrencyLevel) {
        ++sshift;
        ssize <<= 1;
    }
    //segmentShift segmentMask后面计算槽的数组下标用
    //默认情况下segmentShift=28 segmentMask=15
    this.segmentShift = 32 - sshift;
    this.segmentMask = ssize - 1;
    if (initialCapacity > MAXIMUM_CAPACITY)
        initialCapacity = MAXIMUM_CAPACITY;
    //容量/锁数量  计算每个分段锁中存放的数组大小cap，同样是2^n
    //默认情况下c=1， cap为2（MIN_SEGMENT_TABLE_CAPACITY）。也就是每个槽内的数组最多为2
    int c = initialCapacity / ssize;
    if (c * ssize < initialCapacity)
        ++c;
    int cap = MIN_SEGMENT_TABLE_CAPACITY;
    while (cap < c)
        cap <<= 1;
    // create segments and segments[0]
    //创建Segment数组SS，并创建第一个数组元素S0
    Segment<K,V> s0 =
        new Segment<K,V>(loadFactor, (int)(cap * loadFactor),
                         (HashEntry<K,V>[])new HashEntry[cap]);
    Segment<K,V>[] ss = (Segment<K,V>[])new Segment[ssize];
    //向ss写入s0
    UNSAFE.putOrderedObject(ss, SBASE, s0); // ordered write of segments[0]
    this.segments = ss;
}
```
### put

大致流程：  

1. 根据key hashCode定位到Segment，如果空则初始化Segment
2. 获取目标Segment的ReentrantLock锁
3. put操作，遍历链表，这里与HashMap类似。可能会触发扩容
4. 释放锁

根据key的hashCode定位到Segment后，在槽内进行put操作
```java
/**
 * put操作
 * 1、计算hash值
 * 2、根据hash值找到对应的Segment
 * 3、针对Segment操作
 */
public V put(K key, V value) {
    Segment<K,V> s;
    //并发环境下，key value均不能为空
    if (value == null)
        throw new NullPointerException();
    //计算hash
    int hash = hash(key);
    // 根据hash值计算出key在Segment数组中的位置 j
    /**
     * TODO 这里没懂  反正就是根据hash计算出key对应Segment数组下标
     * segmentShift=32-n（Segment数量=2^n）
     * segmentMask = Segment数量-1
     */
    int j = (hash >>> segmentShift) & segmentMask;
    //s = segment[j]为空时，初始化Segment[j]
    if ((s = (Segment<K,V>)UNSAFE.getObject          // nonvolatile; recheck
         (segments, (j << SSHIFT) + SBASE)) == null) //  in ensureSegment
        s = ensureSegment(j);
    //插入值到S中
    return s.put(key, hash, value, false);
}
```
#### 初始化槽
put时，如果根据key hashCode定位到的Segment为null时，需要先初始化一下  
```java
/**
 * 新建一个Segment
 * @param k
 * @return
 */
private Segment<K,V> ensureSegment(int k) {
    final Segment<K,V>[] ss = this.segments;
    //TODO 没懂
    long u = (k << SSHIFT) + SBASE; // raw offset
    Segment<K,V> seg;
    if ((seg = (Segment<K,V>)UNSAFE.getObjectVolatile(ss, u)) == null) {
        //以当前第一个Segment槽的数组长度和负载因子来初始化
        Segment<K,V> proto = ss[0]; // use segment 0 as prototype
        int cap = proto.table.length;
        float lf = proto.loadFactor;
        int threshold = (int)(cap * lf);
        HashEntry<K,V>[] tab = (HashEntry<K,V>[])new HashEntry[cap];
        //再次检查Segment是否被其他线程初始化了
        if ((seg = (Segment<K,V>)UNSAFE.getObjectVolatile(ss, u))
            == null) { // recheck
            Segment<K,V> s = new Segment<K,V>(lf, threshold, tab);
            //CAS循环，该槽被其他线程设置成功 or 当前线程设置成功后，跳出循环
            while ((seg = (Segment<K,V>)UNSAFE.getObjectVolatile(ss, u))
                   == null) {
                if (UNSAFE.compareAndSwapObject(ss, u, null, seg = s))
                    break;
            }
        }
    }
    return seg;
}
```
注意最后的cas循环。 处理了多个线程同时初始化槽的情况  
#### Segment内部put操作 
Segment内部put用ReentrantLock来保证线程安全，锁住当前Segment   
1. 获取锁
2. 遍历链表元素，进行put操作（这一步与HashMap相似）
3. 释放锁  

```java
/**
 * Segment内部put操作
 * @param key
 * @param hash
 * @param value
 * @param onlyIfAbsent
 * @return
 */
final V put(K key, int hash, V value, boolean onlyIfAbsent) {
    //尝试获取Segment独占锁
    // 1、tryLock()获取锁
    // 2、tryLock失败使用scanAndLockForPut获取锁（顺带初始化了node）
    HashEntry<K,V> node = tryLock() ? null :
        scanAndLockForPut(key, hash, value);
    V oldValue;
    try {

        //
        HashEntry<K,V>[] tab = table;
        int index = (tab.length - 1) & hash;
        HashEntry<K,V> first = entryAt(tab, index);
        //遍历链表元素，与hashMap类似，更新旧值or新建一个HashEntry元素
        for (HashEntry<K,V> e = first;;) {
            if (e != null) {
                K k;
                if ((k = e.key) == key ||
                    (e.hash == hash && key.equals(k))) {
                    oldValue = e.value;
                    if (!onlyIfAbsent) {
                        e.value = value;
                        ++modCount;
                    }
                    break;
                }
                e = e.next;
            }
            else {
                if (node != null)
                    node.setNext(first);
                else
                    node = new HashEntry<K,V>(hash, key, value, first);
                int c = count + 1;
                if (c > threshold && tab.length < MAXIMUM_CAPACITY)
                    //达到扩容阈值后扩容
                    rehash(node);
                else
                    //没有达到扩容阈值，将节点设置链表的表头
                    setEntryAt(tab, index, node);
                ++modCount;
                count = c;
                oldValue = null;
                break;
            }
        }
    } finally {
        //解锁
        unlock();
    }
    return oldValue;
}
```

#### scanAndLockForPut
注意上面put时代码  
```java
//尝试获取Segment独占锁
// 1、tryLock()获取锁
// 2、tryLock失败使用scanAndLockForPut获取锁（顺带初始化了node(key不存在)）
HashEntry<K,V> node = tryLock() ? null :
    scanAndLockForPut(key, hash, value);
```
如果第一次tryLock失败，进入scanAndLockForPut继续获取  
put获取循环获取当前Segment独占锁，并实例化node（如果key不存在）  
1. 循环获取锁
2. 循环获取锁时顺便遍历当前链表的每个节点，看key是否存在，不存在初始化node
3. 直到获取到锁  

```java
/**
 * put获取循环获取当前Segment独占锁，并实例化node（如果key不存在）
 * 1、循环获取锁
 * 2、循环获取锁时顺便遍历当前链表的每个节点，看key是否存在，不存在初始化node
 * 3、直到获取到锁
 * @param key
 * @param hash
 * @param value
 * @return
 */
private HashEntry<K,V> scanAndLockForPut(K key, int hash, V value) {
    //获取当前hash位链表第一个hashEntry
    HashEntry<K,V> first = entryForHash(this, hash);
    HashEntry<K,V> e = first;
    HashEntry<K,V> node = null;
    int retries = -1; // negative while locating node
    //循环获取lock
    while (!tryLock()) {
        //f 后面判断链表头元素是否变化用
        HashEntry<K,V> f; // to recheck first below
        //retries<0 时处理。 直到遍历到key或者遍历链表后发现无key则跳出此if语句
        if (retries < 0) {
            if (e == null) {
                //获取锁失败+遍历链表所有结点都没有找到key。 初始化node
                if (node == null) // speculatively create node
                    node = new HashEntry<K,V>(hash, key, value, null);
                retries = 0;
            }
            //遍历到key，
            else if (key.equals(e.key))
                retries = 0;
            else
                e = e.next;
        }
        //循环tryLock超出重试次数，直接阻塞获取锁（lock()）
        else if (++retries > MAX_SCAN_RETRIES) {
            lock();
            break;
        }
        //retries & 1 TODO 没理解
        //找到Doug Lea的解释http://altair.cs.oswego.edu/pipermail/concurrency-interest/2014-August/012881.html
        //单核必检查，多核循环2次检查1次，但是最终能检测到
        //链表头节点变化时，重新初始化各元素，相当于重新循环
        else if ((retries & 1) == 0 &&
                 (f = entryForHash(this, hash)) != first) {
            e = first = f; // re-traverse if entry changed
            retries = -1;
        }
    }
    return node;
}
```
### 扩容
segment数组不能扩容，扩容只对某个segment槽下的HashEntry数组扩容，每次扩容容量*2（这一点与HashMap相似）  
```java
/**
 * 槽内扩容
 * @param node 要put的node
 */
private void rehash(HashEntry<K,V> node) {
    HashEntry<K,V>[] oldTable = table;
    int oldCapacity = oldTable.length;
    int newCapacity = oldCapacity << 1;//扩容*2
    threshold = (int)(newCapacity * loadFactor);//新阈值
    //初始化新数组
    HashEntry<K,V>[] newTable =
        (HashEntry<K,V>[]) new HashEntry[newCapacity];
    //计算新掩码（与HashMap中的indexFor函数逻辑一样）
    int sizeMask = newCapacity - 1;
    //遍历oldTable、链表。放入newTable中
    /**
     * 这里做了优化，对于oldTable某个位置的链表
     * 1、从链表尾部取在newTable中位置相同的节点，直接转移到newTable
     * 2、剩下的一个个clone
     * 如果链表6个元素，后5个的新位置都一样，可以直接把这5个放入newTable新位置，剩下1个克隆到新位置
     * 据Doug Lea说，根据统计，如果使用默认的阈值，大约只有 1/6 的节点需要克隆。
     */
    for (int i = 0; i < oldCapacity ; i++) {
        HashEntry<K,V> e = oldTable[i];
        if (e != null) {
            HashEntry<K,V> next = e.next;
            //计算元素在新数组位置
            int idx = e.hash & sizeMask;
            if (next == null)   //  Single node on list
                //旧数组节点链表只有1个元素的情况
                newTable[idx] = e;
            else { // Reuse consecutive sequence at same slot
                HashEntry<K,V> lastRun = e;
                int lastIdx = idx;
                //遍历链表，lastRun及之后的节点，在newTable中的位置是一样的，所以可以直接取出来
                for (HashEntry<K,V> last = next;
                     last != null;
                     last = last.next) {
                    //计算新位置
                    int k = last.hash & sizeMask;
                    if (k != lastIdx) {
                        lastIdx = k;
                        lastRun = last;
                    }
                }
                //直接将lastRun及之后节点放到newTable中
                newTable[lastIdx] = lastRun;
                // Clone remaining nodes
                //再次从头遍历链表，遍历到lastRun（把剩下的链表节点一个个clone到新位置）
                for (HashEntry<K,V> p = e; p != lastRun; p = p.next) {
                    V v = p.value;
                    int h = p.hash;
                    int k = h & sizeMask;
                    HashEntry<K,V> n = newTable[k];
                    newTable[k] = new HashEntry<K,V>(h, p.key, v, n);
                }
            }
        }
    }
    //扩容后put新元素
    int nodeIndex = node.hash & sizeMask; // add the new node
    node.setNext(newTable[nodeIndex]);
    newTable[nodeIndex] = node;
    table = newTable;
}
```
扩容比HashMap要复杂一些。 作了一处优化  
1. 将oldTable某个节点的链表，最后一段链表（保证在newTable位置相同），直接把引用指向newTable[index]  
2. 剩下的链表节点一个个赋值  
如果链表最后一段节点index一致的很多，这样可以提高效率  

### get
代码很简单,根据key定位Segment、槽内数组链表，遍历链表得到目标数据  
```java
public V get(Object key) {
    Segment<K,V> s; // manually integrate access methods to reduce overhead
    HashEntry<K,V>[] tab;
    int h = hash(key);
    //根据hash 计算Segment数组位置，拿到目标Segment
    long u = (((h >>> segmentShift) & segmentMask) << SSHIFT) + SBASE;
    if ((s = (Segment<K,V>)UNSAFE.getObjectVolatile(segments, u)) != null &&
        (tab = s.table) != null) {
        //根据hash获得目标Segment内数组对应节点的链表，遍历链表拿value
        for (HashEntry<K,V> e = (HashEntry<K,V>) UNSAFE.getObjectVolatile
                 (tab, ((long)(((tab.length - 1) & h)) << TSHIFT) + TBASE);
             e != null; e = e.next) {
            K k;
            if ((k = e.key) == key || (e.hash == h && key.equals(k)))
                return e.value;
        }
    }
    return null;
}
```

## Java8 HashMap
Java8的HashMap数据结构上是数组+链表+红黑树组成。  
如果数组某个位置上的链表长度>=8,会将链表转换为红黑树，在这个位置上查找元素的效率为O(logN)。  
![hashMap](https://www.javadoop.com/blogimages/map/2.png)
### 初始化
```java
public HashMap() {
    this.loadFactor = DEFAULT_LOAD_FACTOR; // all other fields defaulted
}
public HashMap() {
    this.loadFactor = DEFAULT_LOAD_FACTOR; // all other fields defaulted
}
public HashMap(int initialCapacity, float loadFactor) {
    if (initialCapacity < 0)
        throw new IllegalArgumentException("Illegal initial capacity: " +
                                           initialCapacity);
    if (initialCapacity > MAXIMUM_CAPACITY)
        initialCapacity = MAXIMUM_CAPACITY;
    if (loadFactor <= 0 || Float.isNaN(loadFactor))
        throw new IllegalArgumentException("Illegal load factor: " +
                                           loadFactor);
    this.loadFactor = loadFactor;
    //计算出>initialCapacity最小的2^n 作为扩容阈值
    this.threshold = tableSizeFor(initialCapacity);
}
```
Java8初始化HashMap有三个构造函数，与Java7相比只涉及参数。不实际创建数组。  
注意this.threshold = tableSizeFor(initialCapacity); 初始化后，在第一次put的时候会触发一次扩容，根据扩容阈值threshold新建数组，新数组newCap=threshold（这点与java7不同）
#### hash算法
```java
/**
 * hash算法
 * 1、使用key自身的hash函数
 * 2、h >>> 16 取高位16位
 * 3、h ^ (h >>> 16) 得到低位16位和高位16位异或的结果
 * @param key
 * @return
 */
static final int hash(Object key) {
    int h;
    return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
}
```
**为什么要有扰动函数处理？**  
注意：元素在数组位置index= (table.length - 1) & hash  
如果直接取key的hash值的话，即时hashCode()返回的hash值很松散。如果hash低位重复多，高位不重复，得到的index也会有很多重复的。为了减少hash碰撞，同时考虑高位与低位数据。  
### put
```java
public V put(K key, V value) {
    return putVal(hash(key), key, value, false, true);
}
```
封装了一层，主要看putVal方法  
```java
/**
 * @param hash hash for key
 * @param key key
 * @param value value
 * @param onlyIfAbsent 是否只更新不存在的key
 * @param evict false 表示创建模式 HashMap实现中此参数无用
 * @return
 */
final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
               boolean evict) {
    Node<K,V>[] tab; Node<K,V> p; int n, i;
    //第一次put时，table为null，触发一次扩容
    if ((tab = table) == null || (n = tab.length) == 0)
        n = (tab = resize()).length;
    //找到数组下标，如果为空，初始化node
    if ((p = tab[i = (n - 1) & hash]) == null)
        tab[i] = newNode(hash, key, value, null);
    else {
        /**
         * 节点已存在数据处理
         * 先判断第一个是否为目标key，再分别根据红黑树、数据遍历
         */
        Node<K,V> e; K k;
        if (p.hash == hash &&
            ((k = p.key) == key || (key != null && key.equals(k))))
            // 节点第一个数据
            e = p;
        else if (p instanceof TreeNode)
            //节点为红黑树时，调用红黑树方法
            e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
        else {
            //节点为链表时，遍历next
            for (int binCount = 0; ; ++binCount) {
                if ((e = p.next) == null) {
                    //未遍历到节点，初始化node
                    p.next = newNode(hash, key, value, null);
                    //TREEIFY_THRESHOLD=8 ，put当前节点后>=8,则转换为红黑树
                    if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st
                        treeifyBin(tab, hash);
                    break;
                }
                //遍历到node，赋值给e
                if (e.hash == hash &&
                    ((k = e.key) == key || (key != null && key.equals(k))))
                    break;
                p = e;
            }
        }
        //遍历到节点e，更新value
        if (e != null) { // existing mapping for key
            V oldValue = e.value;
            if (!onlyIfAbsent || oldValue == null)
                e.value = value;
            //回调函数，空方法
            afterNodeAccess(e);
            return oldValue;
        }
    }
    ++modCount;
    //判断是否扩容
    if (++size > threshold)
        resize();
    //节点插入后回调函数,空方法
    afterNodeInsertion(evict);
    return null;
}
```
put很简单，看一遍代码就ok了。还是遍历数组对应位置节点的元素，链表和红黑树两种遍历方式。 put元素后检查一下链表长度>=8转换为红黑树。  
#### 扩容
```java
/**
 * 扩容
 * @return
 */
final Node<K,V>[] resize() {
    Node<K,V>[] oldTab = table;
    int oldCap = (oldTab == null) ? 0 : oldTab.length;
    int oldThr = threshold;
    //定义新容量、新扩容阈值
    int newCap, newThr = 0;
    //HashMap 数组存在时扩容
    if (oldCap > 0) {
        if (oldCap >= MAXIMUM_CAPACITY) {
            threshold = Integer.MAX_VALUE;
            return oldTab;
        }
        else if ((newCap = oldCap << 1) < MAXIMUM_CAPACITY &&
                 oldCap >= DEFAULT_INITIAL_CAPACITY)
            newThr = oldThr << 1; // double threshold
    }
    //使用new HashMap(int initialCapacity)初始化数组后
    else if (oldThr > 0) // initial capacity was placed in threshold
        newCap = oldThr;
    //使用new HashMap()初始化数组后
    else {               // zero initial threshold signifies using defaults
        newCap = DEFAULT_INITIAL_CAPACITY;
        newThr = (int)(DEFAULT_LOAD_FACTOR * DEFAULT_INITIAL_CAPACITY);
    }
    if (newThr == 0) {
        float ft = (float)newCap * loadFactor;
        newThr = (newCap < MAXIMUM_CAPACITY && ft < (float)MAXIMUM_CAPACITY ?
                  (int)ft : Integer.MAX_VALUE);
    }
    threshold = newThr;
    @SuppressWarnings({"rawtypes","unchecked"})
        Node<K,V>[] newTab = (Node<K,V>[])new Node[newCap];
    table = newTab;
    //旧数组不为空时，复制数组元素（这里和put有点像，链表1个元素、红黑树、链表三种情况处理）
    if (oldTab != null) {
        for (int j = 0; j < oldCap; ++j) {
            Node<K,V> e;
            if ((e = oldTab[j]) != null) {
                oldTab[j] = null;
                if (e.next == null)
                    newTab[e.hash & (newCap - 1)] = e;
                else if (e instanceof TreeNode)
                    ((TreeNode<K,V>)e).split(this, newTab, j, oldCap);
                else { // preserve order
                    //扩容后留在原位置的节点，loHead头
                    Node<K,V> loHead = null, loTail = null;
                    //扩容后留移到新位置的节点，hiHead头
                    Node<K,V> hiHead = null, hiTail = null;
                    Node<K,V> next;
                    do {
                        next = e.next;
                        /**
                         * (e.hash & oldCap) == 0 不是很好理解
                         * 参考：https://blog.csdn.net/bnmb888/article/details/77164485
                         * 比如oldCap=8,hash是3，11，19，27时，(e.hash & oldCap)的结果是0，8，0，8，
                         * 这样3，19组成新的链表，index为3；而11，27组成新的链表，新分配的index为3+8；
                         * 所以旧table相同位置元素有两种情况：1、原位置（j） 2、原位置+oldCap(j + oldCap)
                         */
                        if ((e.hash & oldCap) == 0) {
                            if (loTail == null)
                                loHead = e;
                            else
                                loTail.next = e;
                            loTail = e;
                        }
                        else {
                            if (hiTail == null)
                                hiHead = e;
                            else
                                hiTail.next = e;
                            hiTail = e;
                        }
                    } while ((e = next) != null);
                    //原位置
                    if (loTail != null) {
                        loTail.next = null;
                        newTab[j] = loHead;
                    }
                    //新位置
                    if (hiTail != null) {
                        hiTail.next = null;
                        newTab[j + oldCap] = hiHead;
                    }
                }
            }
        }
    }
    return newTab;
}
```
扩容操作，与java7有些不同，table为空时只要调节几个参数就可以了，不为空时走具体扩容逻辑并处理数组各节点数据。  
### get
```java
public V get(Object key) {
    Node<K,V> e;
    return (e = getNode(hash(key), key)) == null ? null : e.value;
}
```
封装了一层，看下面  
```java
/**
 * 获取key对应的节点
 * @param hash
 * @param key
 * @return
 */
final Node<K,V> getNode(int hash, Object key) {
    Node<K,V>[] tab; Node<K,V> first, e; int n; K k;
    if ((tab = table) != null && (n = tab.length) > 0 &&
        (first = tab[(n - 1) & hash]) != null) {
        //获取key对应节点第一个元素first，然后遍历
        //1、key为第一个元素
        if (first.hash == hash && // always check first node
            ((k = first.key) == key || (key != null && key.equals(k))))
            return first;
        if ((e = first.next) != null) {
            //2、红黑树遍历处理
            if (first instanceof TreeNode)
                return ((TreeNode<K,V>)first).getTreeNode(hash, key);
            //2、链表遍历处理
            do {
                if (e.hash == hash &&
                    ((k = e.key) == key || (key != null && key.equals(k))))
                    return e;
            } while ((e = e.next) != null);
        }
    }
    return null;
}
```
看代码，很简单。  
## Java8 ConcurrentHashMap
### 总体结构
![concurrentHashMap](https://www.javadoop.com/blogimages/map/4.png)
Java8 ConcurrentHashMap数据结构与HashMap一致，数组+链表/红黑树。 在操作上为了保证线程安全性，要复杂很多（主要是迁移代码复杂）

### 初始化
看代码   
```java
/**
 * Creates a new, empty map with the default initial table size (16).
 * 无参初始化方法
 */
public ConcurrentHashMap() {
}
```
```java
/**
 * @param initialCapacity 初始化容量
 */
public ConcurrentHashMap(int initialCapacity) {
    //参数校验
    if (initialCapacity < 0)
        throw new IllegalArgumentException();
    //sizeCtl 取initialCapacity*1.5 向上取2^n
    int cap = ((initialCapacity >= (MAXIMUM_CAPACITY >>> 1)) ?
               MAXIMUM_CAPACITY :
               tableSizeFor(initialCapacity + (initialCapacity >>> 1) + 1));
    this.sizeCtl = cap;
}
```
### put
看代码。 计算hash找到数组对应位置头节点f，给f加监视器锁来保证线程安全。  
```java
public V put(K key, V value) {
    return putVal(key, value, false);
}
```
```java
/**
 * put
 * @param key key
 * @param value value
 * @param onlyIfAbsent 是否仅更新不存在的key
 * @return
 */
final V putVal(K key, V value, boolean onlyIfAbsent) {
    if (key == null || value == null) throw new NullPointerException();
    //计算hash值，key的hash+扰动函数
    //与hashmap计算hash基本一样，但多了一步& HASH_BITS，HASH_BITS是0x7fffffff（int最大值），
    // 该步是为了消除hash最高位上的负符号 hash的负在ConcurrentHashMap中有特殊意义表示在扩容或者是树节点
    int hash = spread(key.hashCode());
    //用于记录对应链表的长度（看后面理解）
    int binCount = 0;
    for (Node<K,V>[] tab = table;;) {
        Node<K,V> f; int n, i, fh;
        if (tab == null || (n = tab.length) == 0)
            //table为空时，初始化table
            tab = initTable();
        //找到hash对应位置上数组元素f
        else if ((f = tabAt(tab, i = (n - 1) & hash)) == null) {
            //cas赋值，如果失败，继续for循环（进入下一个for循环继续处理）。 cas成功break
            if (casTabAt(tab, i, null,
                         new Node<K,V>(hash, key, value, null)))
                break;                   // no lock when adding to empty bin
        }
        // 说明链表正在扩容
        else if ((fh = f.hash) == MOVED)
            tab = helpTransfer(tab, f);
        else {
            V oldVal = null;
            //锁当前数组头节点
            synchronized (f) {
                //再次检查数组位置是否当前头节点f
                if (tabAt(tab, i) == f) {
                    //头节点hash fh>=0说明是链表
                    if (fh >= 0) {
                        binCount = 1;//记录链表长度，后面根据bitCount判断是否转换为红黑树
                        //遍历链表，替换值
                        for (Node<K,V> e = f;; ++binCount) {
                            K ek;
                            if (e.hash == hash &&
                                ((ek = e.key) == key ||
                                 (ek != null && key.equals(ek)))) {
                                oldVal = e.val;
                                if (!onlyIfAbsent)
                                    e.val = value;
                                break;
                            }
                            Node<K,V> pred = e;
                            //遍历链表全部节点，无key，链表尾部插入新节点
                            if ((e = e.next) == null) {
                                pred.next = new Node<K,V>(hash, key,
                                                          value, null);
                                break;
                            }
                        }
                    }
                    //红黑树处理逻辑
                    else if (f instanceof TreeBin) {
                        Node<K,V> p;
                        binCount = 2;
                        if ((p = ((TreeBin<K,V>)f).putTreeVal(hash, key,
                                                       value)) != null) {
                            oldVal = p.val;
                            if (!onlyIfAbsent)
                                p.val = value;
                        }
                    }
                }
            }
            if (binCount != 0) {
                //链表长度>8转红黑树（和HashMap不一样，如果数组长度不超出64只扩容数组）
                if (binCount >= TREEIFY_THRESHOLD)
                    treeifyBin(tab, i);
                if (oldVal != null)
                    return oldVal;
                break;
            }
        }
    }
    //链表长度计数器+1，判断是否扩容
    addCount(1L, binCount);
    return null;
}
```
#### 扩容
扩容的几个时机  
1. put时，元素当前位置链表长度>=8且table长度<64。 只执行扩容不转换红黑树
2. 新增节点后，调用addCount记录元素个数，并检查是否扩容，当数组元素达到阈值后，扩容
```java
/**
 * 数组扩容
 * @param size 要扩容到的size（已经是size*2了）
 */
private final void tryPresize(int size) {
    //size*1.5+1 向上取2^n
    int c = (size >= (MAXIMUM_CAPACITY >>> 1)) ? MAXIMUM_CAPACITY :
        tableSizeFor(size + (size >>> 1) + 1);
    int sc;
    while ((sc = sizeCtl) >= 0) {
        Node<K,V>[] tab = table; int n;
        //数组为空时，进行初始化数组操作，与initTable()基本一样
        if (tab == null || (n = tab.length) == 0) {
            n = (sc > c) ? sc : c;
            if (U.compareAndSwapInt(this, SIZECTL, sc, -1)) {
                try {
                    if (table == tab) {
                        @SuppressWarnings("unchecked")
                        Node<K,V>[] nt = (Node<K,V>[])new Node<?,?>[n];
                        table = nt;
                        sc = n - (n >>> 2);
                    }
                } finally {
                    sizeCtl = sc;
                }
            }
        }
        //c<=sc(其他线程扩容过了 or 正在扩容) or n 超过数组限制最大值，直接退出
        else if (c <= sc || n >= MAXIMUM_CAPACITY)
            break;
        //迁移table，两次transfer
        else if (tab == table) {
            int rs = resizeStamp(n);
            //第二次
            if (sc < 0) {
                Node<K,V>[] nt;
                if ((sc >>> RESIZE_STAMP_SHIFT) != rs || sc == rs + 1 ||
                    sc == rs + MAX_RESIZERS || (nt = nextTable) == null ||
                    transferIndex <= 0)
                    break;
                //CAS将SIZECTL+1，执行transfer
                if (U.compareAndSwapInt(this, SIZECTL, sc, sc + 1))
                    transfer(tab, nt);
            }
            // 第一次 SIZECTL置为负数
            else if (U.compareAndSwapInt(this, SIZECTL, sc,
                                         (rs << RESIZE_STAMP_SHIFT) + 2))
                transfer(tab, null);
        }
    }
}
```
#### 数据迁移
这部分代码比较复杂  
数据迁移的两个触发时机：  
1. 其他线程操作时，检测到头节点f.hash==MOVED,说明正在数据迁移，CAS获取SIZECTL锁后执行迁移
2. 扩容链表后，迁移数据  
迁移数组的方法很复杂，看代码，每次只扩容其中一小段  
```java
/**
 * 迁移数组
 * @param tab
 * @param nextTab
 */
private final void transfer(Node<K,V>[] tab, Node<K,V>[] nextTab) {
    //有n个位置需要迁移、将n分为多个任务包，每个任务包有stride个任务
    int n = tab.length, stride;
    //stride 单核下为n，多核下为n/8,最小为16  每次迁移任务包的任务个数
    if ((stride = (NCPU > 1) ? (n >>> 3) / NCPU : n) < MIN_TRANSFER_STRIDE)
        stride = MIN_TRANSFER_STRIDE; // subdivide range
    //nextTab为null，先进行一次初始化（第一个发起迁移的线程调用此方法时，nextTab为null）
    if (nextTab == null) {            // initiating
        try {
            @SuppressWarnings("unchecked")
            //数组容量翻倍
            Node<K,V>[] nt = (Node<K,V>[])new Node<?,?>[n << 1];
            nextTab = nt;
        } catch (Throwable ex) {      // try to cope with OOME
            sizeCtl = Integer.MAX_VALUE;
            return;
        }
        //赋值给全局变量
        nextTable = nextTab;
        //全局变量，控制迁移的位置
        transferIndex = n;
    }
    int nextn = nextTab.length;
    // 正在被迁移的Node，hash为MOVED， 标识用
    // 原数组某个节点处设置为这个ForwardingNode，表示此处已被其他线程吹过了
    ForwardingNode<K,V> fwd = new ForwardingNode<K,V>(nextTab);
    //表示做完一个位置的迁移工作，可以准备做下一个位置的了
    boolean advance = true;
    // 表示所有迁移操作已完成 
    boolean finishing = false; // to ensure sweep before committing nextTab
    //i为位置索引，bound =为边界  迁移bound-i之间数据
    for (int i = 0, bound = 0;;) {
        Node<K,V> f; int fh;
        //while循环计算需要迁移的左右边界
        //advance为true表示可以进行下个位置迁移
        while (advance) {
            int nextIndex, nextBound;
            if (--i >= bound || finishing)
                advance = false;
            //nextIndex指向迁移位置，，如果<=0说明其他线程处理迁移了，把这个位置往前移到0
            else if ((nextIndex = transferIndex) <= 0) {
                i = -1;
                advance = false;
            }
            //获得锁
            else if (U.compareAndSwapInt
                     (this, TRANSFERINDEX, nextIndex,
                      nextBound = (nextIndex > stride ?
                                   nextIndex - stride : 0))) {
                //将迁移边界位置nextBound（左边界），也就是上次迁移位置往前移动stride个
                bound = nextBound;
                i = nextIndex - 1;
                advance = false;
            }
        }

        if (i < 0 || i >= n || i + n >= nextn) {
            int sc;
            if (finishing) {
                //最后一次迁移的线程，nextTable赋值给原来的table，退出
                nextTable = null;
                table = nextTab;
                //sizeCtl 修改新数组长度的0.75倍
                sizeCtl = (n << 1) - (n >>> 1);
                return;
            }
            /**
             * 判断迁移是否结束
             * 1、前面每次迁移会将sizeCtl 加 1，这里每次-1代表做完了
             * 2、sizeCtl迁移前会设置为 (rs << RESIZE_STAMP_SHIFT) + 2，如果相等说明迁移结束了
             */
            if (U.compareAndSwapInt(this, SIZECTL, sc = sizeCtl, sc - 1)) {
                //迁移结束，直接退出
                if ((sc - 2) != resizeStamp(n) << RESIZE_STAMP_SHIFT)
                    return;
                //到这里迁移任务都做完了，下次循环进入if (finishing)分支
                finishing = advance = true;
                i = n; // recheck before commit
            }
        }
        //如果i处位置是空的，把ForwardingNode放入节点
        else if ((f = tabAt(tab, i)) == null)
            advance = casTabAt(tab, i, null, fwd);
        //此处已经是ForwardingNode了，
        else if ((fh = f.hash) == MOVED)
            advance = true; // already processed
        else {
            //对数组该处头节点加锁，开始迁移工作
            synchronized (f) {
                if (tabAt(tab, i) == f) {
                    Node<K,V> ln, hn;
                    //头节点hash>0,说明是链表的node节点
                    if (fh >= 0) {
                        //核心思想是扩容时将1条链表分成2部分，lastRun（不含）之后的节点与头节点f在新数组中不在一个hash位置上
                        //找到lastRun及之后的及节点一起迁移；lastRun之前的节点需要进行克隆，判断hash分到两个链表中。
                        // 这样在lastRun节点很靠前时可以显著提升效率
                        int runBit = fh & n;
                        Node<K,V> lastRun = f;
                        //循环找到lastRun（lastRun是最后1个与头节点hash不同的元素）
                        for (Node<K,V> p = f.next; p != null; p = p.next) {
                            int b = p.hash & n;
                            if (b != runBit) {
                                runBit = b;
                                lastRun = p;
                            }
                        }
                        //这里分为ln和hn代表低位hash和高位hash，数组扩容2倍后，分别放到i和i+n处
                        if (runBit == 0) {
                            //头节点hash==0，说明头节点在新数组的i+n上，lastRun后面都在i上
                            ln = lastRun;
                            hn = null;
                        }
                        else {
                            //头节点hash>0，说明头节点在新数组的i上，lastRun后面都在i+n上
                            hn = lastRun;
                            ln = null;
                        }
                        //链表分为两部分，lastRun前存i，lastRun及之后元素存i+n位置
                        for (Node<K,V> p = f; p != lastRun; p = p.next) {
                            int ph = p.hash; K pk = p.key; V pv = p.val;
                            if ((ph & n) == 0)
                                ln = new Node<K,V>(ph, pk, pv, ln);
                            else
                                hn = new Node<K,V>(ph, pk, pv, hn);
                        }
                        setTabAt(nextTab, i, ln);
                        setTabAt(nextTab, i + n, hn);
                        setTabAt(tab, i, fwd);
                        //迁移完毕
                        advance = true;
                    }
                    //红黑树处理
                    else if (f instanceof TreeBin) {
                        TreeBin<K,V> t = (TreeBin<K,V>)f;
                        TreeNode<K,V> lo = null, loTail = null;
                        TreeNode<K,V> hi = null, hiTail = null;
                        int lc = 0, hc = 0;
                        for (Node<K,V> e = t.first; e != null; e = e.next) {
                            int h = e.hash;
                            TreeNode<K,V> p = new TreeNode<K,V>
                                (h, e.key, e.val, null, null);
                            if ((h & n) == 0) {
                                if ((p.prev = loTail) == null)
                                    lo = p;
                                else
                                    loTail.next = p;
                                loTail = p;
                                ++lc;
                            }
                            else {
                                if ((p.prev = hiTail) == null)
                                    hi = p;
                                else
                                    hiTail.next = p;
                                hiTail = p;
                                ++hc;
                            }
                        }
                        ln = (lc <= UNTREEIFY_THRESHOLD) ? untreeify(lo) :
                            (hc != 0) ? new TreeBin<K,V>(lo) : t;
                        hn = (hc <= UNTREEIFY_THRESHOLD) ? untreeify(hi) :
                            (lc != 0) ? new TreeBin<K,V>(hi) : t;
                        setTabAt(nextTab, i, ln);
                        setTabAt(nextTab, i + n, hn);
                        setTabAt(tab, i, fwd);
                        //迁移完毕
                        advance = true;
                    }
                }
            }
        }
    }
}
```
### get
简单，直接代码。 通过hash确定位置，然后遍历链表or红黑树  
```java
public V get(Object key) {
    Node<K,V>[] tab; Node<K,V> e, p; int n, eh; K ek;
    int h = spread(key.hashCode());
    if ((tab = table) != null && (n = tab.length) > 0 &&
        (e = tabAt(tab, (n - 1) & h)) != null) {
        //判断头节点是否是get的元素
        if ((eh = e.hash) == h) {
            if ((ek = e.key) == key || (ek != null && key.equals(ek)))
                return e.val;
        }
        //说明正在扩容或者为红黑树
        else if (eh < 0)
            //调用红黑树的find查找
            return (p = e.find(h, key)) != null ? p.val : null;
        //遍历链表
        while ((e = e.next) != null) {
            if (e.hash == h &&
                ((ek = e.key) == key || (ek != null && key.equals(ek))))
                return e.val;
        }
    }
    return null;
}
```
## 总结
**数据结构**  
Java7  
HashMap 数组+链表  
ConcurrentHashMap Segment数组+节点数组+链表  
Java8中  
HashMap 数组+链表/红黑树  
ConcurrentHashMap 数组+链表/红黑树  
**并发操作**
Java7、8 HashMap线程不安全，不支持并发  
Java7 ConcurrentHashMap，通过ReentrantLock获得Segment上的独占锁来对节点内数组操作。CAS初始化Segment。扩容时用volatile保持内存可见性  
Java8 ConcurrentHashMap 通过给数组上头节点加监视器锁的方式来保证线程安全。扩容时CAS操作SIZECTL  
看完一遍后，ConcurrentHashMap中的扩容和数据迁移代码比较难懂，其他地方都可以理解，不少地方看到后面就理解了。