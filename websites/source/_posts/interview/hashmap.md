---
title: hashmap与concurrenthashmap分析
category: interview
date: 2020-07-02 02:51:17
tags: interview
---


<!-- more -->

### hashmap1.7分析

#### 数据结构

- 基于数组 + 单向链表的数据结构
- 如何验证数组

```java
// 存储的数据为Entry的数组
transient Entry<K,V>[] table = (Entry<K,V>[]) EMPTY_TABLE;

static final Entry<?,?>[] EMPTY_TABLE = {};
```

- 如何验证是单向链表

```java
static class Entry<K,V> implements Map.Entry<K,V> {
        final K key;
        V value;
        Entry<K,V> next;   // Entry包含next的指针,表示相同的hash不同key对应的Entry都是通过链表进行相连
        int hash;
}
```

#### hashmap核心属性

```
/**
* The number of key-value mappings contained in this map.
*/
transient int size;// 当前map包含的key-value个数

/**
* The next size value at which to resize (capacity * load factor).
* @serial
*/
// If table == EMPTY_TABLE then this is the initial capacity at which the
// table will be created when inflated.
int threshold; // map进行扩容的阈值

// 负载因子
 final float loadFactor;
 
 // 默认配置
 // hashmap大小为16
static final int DEFAULT_INITIAL_CAPACITY = 1 << 4; // aka 16

// 最大为2^30
static final int MAXIMUM_CAPACITY = 1 << 30;

// 负载因子为0.75
static final float DEFAULT_LOAD_FACTOR = 0.75f;
 
// 空数组 
static final Entry<?,?>[] EMPTY_TABLE = {};
```

#### `put`的过程

##### put源码分析

```java
// 如果返回的数据值不为null,说明存在hash冲突,返回的是旧值
public V put(K key, V value) {
  // 如果为空数组,那么进行初始化
  if (table == EMPTY_TABLE) {
    inflateTable(threshold);
  }
  
  if (key == null)
    return putForNullKey(value);
  
  // 进行hash计算
  int hash = hash(key);
  
  // 根据hash得到table对应的index
  int i = indexFor(hash, table.length);
  
  // 如果存在hash冲突,根据key值来进行比较处理
  // 如果key相同则进行覆盖
  for (Entry<K,V> e = table[i]; e != null; e = e.next) {
    Object k;
    if (e.hash == hash && ((k = e.key) == key || key.equals(k))) {
      V oldValue = e.value;
      e.value = value;
      e.recordAccess(this);
      return oldValue;
    }
  }
 
  // 记录hashmap结构变更的次数,有新的key索引到map的数组中
  modCount++;
  addEntry(hash, key, value, i);
  return null;
}
```

##### `inflateTable`初始化数组

```
// 数组初始化/数组扩容
  private void inflateTable(int toSize) {
        // Find a power of 2 >= toSize
        // 扩容的容量保证数组大小为2^n
        int capacity = roundUpToPowerOf2(toSize);
        // 重新初始化数组
        threshold = (int) Math.min(capacity * loadFactor, MAXIMUM_CAPACITY + 1);
        table = new Entry[capacity];
        initHashSeedAsNeeded(capacity);
    }
```

##### 哈希计算

```java
final int hash(Object k) {
        int h = hashSeed;
        if (0 != h && k instanceof String) {
            return sun.misc.Hashing.stringHash32((String) k);
        }

        h ^= k.hashCode();

        // This function ensures that hashCodes that differ only by
        // constant multiples at each bit position have a bounded
        // number of collisions (approximately 8 at default load factor).
  // 将hashcode的高12位与hashcode的高20位进行异或运算
        h ^= (h >>> 20) ^ (h >>> 12);
        return h ^ (h >>> 7) ^ (h >>> 4);
}

// 使用位运算得到索引下标数据
static int indexFor(int h, int length) {
    // assert Integer.bitCount(length) == 1 : "length must be a non-zero power of 2";
  // 原始的容量为len 
  // 扩容后的容量为2len
  // 数组的数据迁移, x & (len - 1) 变更为 x & (2len - 1). 即原先的下标为0的数组进行数据迁移之后会转为0 或者 16,因为容量增加1倍
    return h & (length-1);  // hashcode % len
}
```

将节点添加到链表中

```java
// 添加节点entry
void addEntry(int hash, K key, V value, int bucketIndex) {
  // 触发扩容的条件,当前的大小达到阈值并且存在hash冲突
   if ((size >= threshold) && (null != table[bucketIndex])) {
     // 扩容为原先容量的2倍
		 resize(2 * table.length);
     hash = (null != key) ? hash(key) : 0;
     bucketIndex = indexFor(hash, table.length);
   }

   createEntry(hash, key, value, bucketIndex);
 }

// 创建节点entry并将当前的下标的entry引用指向e
void createEntry(int hash, K key, V value, int bucketIndex) {
        Entry<K,V> e = table[bucketIndex];
        table[bucketIndex] = new Entry<>(hash, key, value, e);
        size++;
    }
```

`resize`数组扩容

```java
void resize(int newCapacity) {
        Entry[] oldTable = table;
        int oldCapacity = oldTable.length;
        if (oldCapacity == MAXIMUM_CAPACITY) {
            threshold = Integer.MAX_VALUE;
            return;
        }

        Entry[] newTable = new Entry[newCapacity];
        transfer(newTable, initHashSeedAsNeeded(newCapacity));
        table = newTable;
        threshold = (int)Math.min(newCapacity * loadFactor, MAXIMUM_CAPACITY + 1);
}

    /**
     * Transfers all entries from current table to newTable.
     */
// 数据迁移
    void transfer(Entry[] newTable, boolean rehash) {
        int newCapacity = newTable.length;
        for (Entry<K,V> e : table) {
            while(null != e) {
                Entry<K,V> next = e.next;
                if (rehash) {
                    e.hash = null == e.key ? 0 : hash(e.key);
                }
                int i = indexFor(e.hash, newCapacity);
                e.next = newTable[i];
                newTable[i] = e;
                e = next;
            }
        }
    }
```

#### `get`的过程

```java
public V get(Object key) {
     if (key == null)
         return getForNullKey();
     Entry<K,V> entry = getEntry(key);

     return null == entry ? null : entry.getValue();
}

// 进行hash定位到table的位置,如果存储的数据的链表头不为null,那么就需要将其进行链表的遍历
final Entry<K,V> getEntry(Object key) {
        if (size == 0) {
            return null;
        }

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

### `ConcurrentHashMap`1.7分析

#### 数据结构

- 基于数组 + 链表

- 如何验证

```java
// 存储为Node的数组
final Segment<K,V>[] segments;

// segments本身是一把可重入的锁
static final class Segment<K,V> extends ReentrantLock {
  
  // 每一个片段中都存储一份table的数据,进行扩容的时候是对Segment的table进行扩容
  transient volatile HashEntry<K,V>[] table;
}

// 每一个Node节点都具备一个next,同时对应的value以及next具备可见性
static final class HashEntry<K,V> {
        final int hash;
        final K key;
        volatile V value;
        volatile HashEntry<K,V> next;
}

// 相当于ConcurrentHashMap是划分为一个个小片段的hashmap
```

####  `ConcurrentHashMap`核心属性

```java
// 每个片段的table大小
static final int DEFAULT_INITIAL_CAPACITY = 16;
// 负载因子
static final float DEFAULT_LOAD_FACTOR = 0.75f;
// 并发级别,表示划分的Segment的片段
 static final int DEFAULT_CONCURRENCY_LEVEL = 16;
```

#### `put`的过程

```java
public V put(K key, V value) {
    Segment<K,V> s;
  	if (value == null)
       throw new NullPointerException();
     
     // 计算hash
     int hash = hash(key);
 
  //  this.segmentShift = 32 - sshift;  // 32 - 4
  // this.segmentMask = ssize - 1;   		//默认为 15
  // 对应concurrentHashMap的容量大小,容量大小为2^n,保证定位到的j落在数组下标上
     int j = (hash >>> segmentShift) & segmentMask;
     if ((s = (Segment<K,V>)UNSAFE.getObject          // nonvolatile; recheck
        (segments, (j << SSHIFT) + SBASE)) == null) //  in ensureSegment
      s = ensureSegment(j);
     return s.put(key, hash, value, false);
}

// 初始化片段
ensureSegment

// 将数据存储到Segment对应的table中
```

- 初始化`ensureSegment`

```java

```



