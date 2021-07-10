---
title: Java之HashMap与ConcurrentHashMap
category: Java核心笔记
date: 2020-10-01 13:03:44
tags: Java核心笔记 

---

<!-- more -->

![](/Users/keithl/github/personal/xiaokunliu.github.io/websites/zimages/java/map/title.jpg)

###   JDK1.7版本的Hash

#### HashTable

##### HashTabale简述

- 实现`Map`接口,提供实现所有`Map`的接口方法,通过存储Key数据来映射对应的Value值,并且不允许存储的Key以及Value为空.
- 存储的Key数据必须要实现Object的`HashCode`以及`Equals`方法.
- `HashTable`影响性能的两个核心因素,即初始化容量(initial capacity)以及负载因子(load factor);容量capacity即为hash table对应桶的个数,初始化容量即为hash table进行初始化创建hash table需要分配的桶个数;load factor则是hash table在实现容量动态扩容前的桶个数满意度的度量器,即hash table的桶个数达到`capaticy * load factor`会实现动态扩容,即rehash.
- 默认load factor为0.75,过高则能减少分配空间的浪费但是查询的时间复杂度会增加,过低会增加分配空间的浪费,无法充分利用内存空间,相对地能提升查询的时间复杂度.
- `Map`存储的key-value数据通过创建一个迭代器并返回,迭代器具备`fail-fast`特性,即在进行迭代器遍历的过程中,如果对遍历的元素进行修改,将会抛出异常,即`ConcurrentModificationException`;而对于返回`Map`存储的key或者是value的枚举类`Enumerations`并非具备`fail-fast`特性.
- JDK源码对`Fail-Fast`的解析说明,迭代器的快速失败行为不能得到保证,因为一般来说,如果是存在非同步并发修改时不可能做出任何硬性保证.具备`Fail-Fast`的迭代器尽最大努力抛出ConcurrentModificationException.因此,如果一个程序依赖于这个异常来保证其正确性是错误的做法,因为迭代器的fail-fast行为只能用于程序bug的检测.
- 在实际使用过程中,如果当前应用场景不需要保证线程安全的并发访问,JDK推荐使用`HashMap`,而对于需要保证线程安全的并发访问,JDK推荐使用`ConcurrentHashMap`.

##### HashTable的类关系图

![](/Users/keithl/github/personal/xiaokunliu.github.io/websites/zimages/java/map/hashtabe_class.jpg)

根据上述类图,我们知道HashTable主要包含三个核心部分,一个是控制hash阀值的Holder,一个是存储key-value的Entry以及提供遍历查询的KeySet/ValueSet/迭代器Enumerator.

##### put&putAll操作

> HashTable`源码

```java
// put操作
public synchronized V put(K key, V value) {
        // Make sure the value is not null
        if (value == null) {
            throw new NullPointerException();
        }

        // Makes sure the key is not already in the hashtable.
        Entry tab[] = table;
  
  			// 对key进行hash
        int hash = hash(key);
  			
  			// 根据hash结果获取bucket的下标
  			// 查询所在下标并对重复key对应的value进行替换
        int index = (hash & 0x7FFFFFFF) % tab.length;
        for (Entry<K,V> e = tab[index] ; e != null ; e = e.next) {
            if ((e.hash == hash) && e.key.equals(key)) {
                V old = e.value;
                e.value = value;
                return old;
            }
        }

        modCount++;
        if (count >= threshold) {
            // Rehash the table if the threshold is exceeded
          	// 动态扩容
            rehash();

            tab = table;
            hash = hash(key);
            index = (hash & 0x7FFFFFFF) % tab.length;
        }

        // Creates the new entry.
        Entry<K,V> e = tab[index];
        tab[index] = new Entry<>(hash, key, value, e);
        count++;
        return null;
 }

// putAll操作
public synchronized void putAll(Map<? extends K, ? extends V> t) {
        for (Map.Entry<? extends K, ? extends V> e : t.entrySet())
            put(e.getKey(), e.getValue());
}
```

> put&putAll的核心要点

```java
// hash算法
hash(key);
// 分配计算bucket的下标
int index = (hash & 0x7FFFFFFF) % tab.length;
// 扩容
rehash();
// 创建新的entry
new Entry<>(hash, key, value, e);
```

- Hash算法

1) 对象之间的 == 与 equals区分

```java
// == 是对象存储的内存地址比较
// equals是对象属性引用的比较,可以自定义两个对象是否相同的原则
// 两者之间存在歧义的等价关系,jdk的Object定义的源码说明如下:
// The {@code equals} method for class {@code Object} implements
//     * the most discriminating possible equivalence relation on objects;
//     * that is, for any non-null reference values {@code x} and
//     * {@code y}, this method returns {@code true} if and only
//     * if {@code x} and {@code y} refer to the same object
//     * ({@code x == y} has the value {@code true}).

public class Person {
  
    private String name;
    private int age;
    private String hobby;
 	
  	
  	 @Override
    public boolean equals(Object o) {
      	return false;
    }
  
  	 public static void main(String[] args) {
        Person p4 = new Person("1113", 4, "xxx3");
        Person p5 = p4;
       	
       // 很明显,下面equals是不合理的,引用同一份内存地址,对象存储的属性以及地址分配空间应当是保持一致性的
       // 于是在逻辑上,如果a1 == a2, 那么一定会有a1.equals(a2), 反之不一定成立,于是"=="是"equals"的充分条件,而"equals"是"=="的必要条件.
        System.out.println(p4 == p5);       // 地址相同,true
        System.out.println(p4.equals(p5));	// 上述自定义为false,所以永远为false
     }
} 

// 根据jdk语法含义,equals方法是值的比较,或者说是对象属性值的比较,于是重写equals方法应当如下:
 @Override
public boolean equals(Object o) {
  // 引用同一份内存地址,必须返回true
  if (this == o) return true;

  if (!(o instanceof Person)) return false;

  Person person = (Person) o;

  // 不同对象但是具备相同的属性也作为两个对象相同.
  return new EqualsBuilder()
    .append(age, person.age)
    .append(name, person.name)
    .append(hobby, person.hobby)
    .isEquals();
}
```

2) hash算法需要将对象进行`hashCode`以及`equals`方法的必要性

```java
// 基于Java语法规范
// The method equals definesanotion of object equality,which is based on value,not reference, comparison
// The method hashCode is very useful, together with the method equals, in hashtables such as java.util.HashMap
//The equals method in Enum is a final method that merely invokes super.equals on its argument and returns the result, thus performing an identity comparison

// Object定义的equals方法以及hashcode方法
public class Object {
	
  // Object的hashCode是基于底层C语言实现
  // 存在的含义: 返回一个对象的hashcode,如果对象是存储在HashMap数据结构中,将有利于hashMap进行hash运算来存储到对应HashTable的桶中
  // 1. 如果两个对象通过equals比较是相同的,那么两个对象的hashcode也必须是一致的
  // 2. 如果两个对象通过equals比较返回为false,那么对应的hashcode不一定是需要相等的,对于不相等的对象(地址空间不同)但是具备相同的hashcode数值,可以考虑重写equals的方法以确保对象保持相同的hashcode以及equals来提升hashMap的性能.
  public native int hashCode();
  
  // equals方法的含义,对于非空的x,y,z
  // 1. 具备自反性,x.equals(x) - true
  // 2. 具备对称性,x.equals(y) 与 y.equals(x) 是一致的
  // 3. 具备传递性,x.equals(y), y.equals(z) =》x.equals(z) 结果相同
  // 4. 总是返回false,对于任意非空x,y,z有 x.equals(null) = false
  // 根据上述hashCode的源码说明,当使用HashMap存储对象作为key的时候,需要将对象的equals方法以及hashCode进行覆盖重写以保持通过equals比较返回的对象为true具备相同的hascode值,以提升hashMap的性能.
  public boolean equals(Object obj) {
       return (this == obj);
  }
} 

// HashTable以及HashMap的存储源码(为什么需要重写hashcode的同时也要重写equals方法能提升map的性能)
// 在put方法中存在以下代码

int hash = hash(key);
int index = (hash & 0x7FFFFFFF) % tab.length;
for (Entry<K,V> e = tab[index] ; e != null ; e = e.next) {
  // 判断hashCode以及equals方法来确定是否属于同一个key,是的话直接覆盖
  // 存在两个不同对象但是具备相同的hash,属性值不同的情况,因此在这里比较hash以及equals来保证是相同key.
  // 不同属性值但是相同的hash会存放在链表中
  if ((e.hash == hash) && e.key.equals(key)) {
    V old = e.value;
    e.value = value;
    return old;
  }
}

// hashcode相同但是equals不相同以及hashCode不相同的情况
modCount++;
if (count >= threshold) {
  // Rehash the table if the threshold is exceeded
  rehash();

  tab = table;
  hash = hash(key);
  index = (hash & 0x7FFFFFFF) % tab.length;
}

// Creates the new entry.
Entry<K,V> e = tab[index];

// 如果hashCode不同,那么直接插入hashTable对应的桶位置
// 如果hashCode相同,那么将会以链表的形式插入链表头部,并指向hashTbale的桶位置中
// 因此我们可以知道,当不重写equals方法的时候,这个时候产生的链表长度会越来越长,查询的时间复杂度为O(N),影响hashMap的查询效率
tab[index] = new Entry<>(hash, key, value, e);
```

4) hash算法实现原理

```java
/**
  * A randomizing value associated with this instance that is applied to
  * hash code of keys to make hash collisions harder to find.
  */
transient int hashSeed;  // 与此实例相关联的随机值，应用于键的哈希码，降低hash冲突

/**
  * Initialize the hashing mask value.
  */
final boolean initHashSeedAsNeeded(int capacity) {
  boolean currentAltHashing = hashSeed != 0;
  boolean useAltHashing = sun.misc.VM.isBooted() &&
    (capacity >= Holder.ALTERNATIVE_HASHING_THRESHOLD);
  boolean switching = currentAltHashing ^ useAltHashing;
  if (switching) {
    hashSeed = useAltHashing
      ? sun.misc.Hashing.randomHashSeed(this)
      : 0;
  }
  return switching;
}

// 进行异或运算
// 如果vm.isBooted 开启并且容量大于threshold会分配hashSeed
// 配置jdk.map.althashing.threshold

// -Djdk.map.althashing.threshold=-1:表示不做优化（不配置这个值作用一样）
// -Djdk.map.althashing.threshold<0:报错
// -Djdk.map.althashing.threshold=1:表示总是启用随机HashSeed
// -Djdk.map.althashing.threshold>=0:便是hashMap内部的数组长度超过该值了就使用随机HashSeed，降低碰撞

// 使用hashSeed主要是为了降低hash冲突
 private int hash(Object k) {
    // hashSeed will be zero if alternative hashing is disabled.
    // key的hashcode 与 hashSeed 进行异或
     return hashSeed ^ k.hashCode();
}
```

- 分配并计算key对应的桶的位置

```java
// hashTable的put方法
put() {
 				 // ...
        int index = (hash & 0x7FFFFFFF) % tab.length;
        for (Entry<K,V> e = tab[index] ; e != null ; e = e.next) {
            if ((e.hash == hash) && e.key.equals(key)) {
                V old = e.value;
                e.value = value;
                return old;
            }
        }

        modCount++;
        if (count >= threshold) {
            // Rehash the table if the threshold is exceeded
            rehash();

            tab = table;
            hash = hash(key);
            index = (hash & 0x7FFFFFFF) % tab.length;
        }
  // ...
}
```

通过上述的代码,有几个细节需要注意:

1) 其一,在java中最大以及最小的整数值为

```java
public final class Integer extends Number implements Comparable<Integer> {
    /**
     * A constant holding the minimum value an {@code int} can
     * have, -2<sup>31</sup>.
     */
    @Native public static final int   MIN_VALUE = 0x80000000;

    /**
     * A constant holding the maximum value an {@code int} can
     * have, 2<sup>31</sup>-1.
     */
    @Native public static final int   MAX_VALUE = 0x7fffffff
}
```

通过上述可知,hash在java中取值范围是在MIN_VALUE - MAX_VALUE之间,那么对于hash & MAX_VALUE得到最终的结果为 0 - 2147483647,也就是说不论hash是否为负数还是正数,都能够保证得到的index一定是落在大于等于0的区间,这样就不会导致分配的hashTable计算出现不合法的index,即index < 0的情况.

2) 其二是跟进hashTable的桶长度进行取模,得到最终的index下标一定是落在(0 - len-1)的下标范围内

3) 其三是不同的key但是hash相同的数据,会查找hashTable桶对应下标的位置并逐个遍历链表,通过比较hashcode以及equals方法来定位是否存在相同的key来进行覆盖更新,如下代码

```java
// 在这里我们可以思考一件事情,如果重写hashcode的同时不重写equals方法,会导致什么结果?
// 1. 不会覆盖更新,那么就是在相同的hashTable桶中的index位置插入新的entry
// 2. 这个时候产生的问题就是entry链条增加,那么查询的时间复杂度也会相应增加.
// 3. 这也是为什么在进行使用map/set存储的对象中,如果是作为key,需要重写hashcode并重写equals方法的原因,提升查询效率,减少时间复杂度.
if ((e.hash == hash) && e.key.equals(key)) {
  V old = e.value;
  e.value = value;
  return old;
}
```

4) 这里有三个变量`count`,`modCount`以及`threshold`

```java
// 记录hashTable的结构发生变化的次数,即相同位置对应的entry长度发生变化异或是hashTable因rahash内部发生变化
private transient int modCount = 0

// hashTable中entry的总个数
private transient int count;

// 当hashTable的entry个数大小超过阀值时会进行rehash
private int threshold;
```

- 扩容机制Rehash (当hash链的entry个数大于等于(`容量大小capacity*负载因子loadFactor`)threshold将会进行rehash)

```java
protected void rehash() {
  int oldCapacity = table.length;
  Entry<K,V>[] oldMap = table;

  // overflow-conscious code
  // 新的容量 = 旧有的容量加 + 1
  int newCapacity = (oldCapacity << 1) + 1;
  if (newCapacity - MAX_ARRAY_SIZE > 0) {
    if (oldCapacity == MAX_ARRAY_SIZE)
      // Keep running with MAX_ARRAY_SIZE buckets
      return;
    newCapacity = MAX_ARRAY_SIZE;
  }
  Entry<K,V>[] newMap = new Entry[newCapacity];

  // rehash,改变hashTable结构,自增加
  modCount++;
  threshold = (int)Math.min(newCapacity * loadFactor, MAX_ARRAY_SIZE + 1);
  
  // 重新分配hash种子
  boolean rehash = initHashSeedAsNeeded(newCapacity);

  table = newMap;

  for (int i = oldCapacity ; i-- > 0 ;) {
    for (Entry<K,V> old = oldMap[i] ; old != null ; ) {
      Entry<K,V> e = old;
      old = old.next;

      // hash seed是否变更进行重新hash
      if (rehash) {
        e.hash = hash(e.key);
      }
      int index = (e.hash & 0x7FFFFFFF) % newCapacity;
      e.next = newMap[index];
      newMap[index] = e;
    }
  }
```

- 创建新的Entry并追加到hashTable中

```java
// Creates the new entry.
// entry个数增加 
Entry<K,V> e = tab[index];
tab[index] = new Entry<>(hash, key, value, e);
count++;
```

##### 删除Remove

```java
// 同步删除并返回删除对应key的value数据值
public synchronized V remove(Object key) {
       // 获取下标索引位置
        Entry tab[] = table;
        int hash = hash(key);
        int index = (hash & 0x7FFFFFFF) % tab.length;
      // 判断下标索引对应的链表信息
        for (Entry<K,V> e = tab[index], prev = null ; e != null ; prev = e, e = e.next) {
           // 根据hashcode以及equals方法判断 
            if ((e.hash == hash) && e.key.equals(key)) {
                modCount++;
                if (prev != null) {
                    prev.next = e.next;
                } else {
                    tab[index] = e.next;
                }
                count--;
                V oldValue = e.value;
                e.value = null;
                return oldValue;
            }
        }
        return null;
```

##### 遍历查询

```java
// 同步加锁遍历
public synchronized V get(Object key) {
  // 获取hash下标并遍历下标对应的链表,直到查找到相同的key为止
  Entry tab[] = table;
  int hash = hash(key);
  int index = (hash & 0x7FFFFFFF) % tab.length;
  for (Entry<K,V> e = tab[index] ; e != null ; e = e.next) {
    if ((e.hash == hash) && e.key.equals(key)) {
      return e.value;
    }
  }
  return null;
    
```

#### HashMap

##### HashMap简述

- 实现`Map`接口,提供了所有可选`Map`的可选方法操作,与`HashTable`类基本一致,但是区分在于`HashMap`不存在同步操作,即线程安全以及`HashMap`允许存储的Key和Value数据为空.
- 初始化容量

```java
// 初始化容量
static final int DEFAULT_INITIAL_CAPACITY = 1 << 4; // aka 16

// 默认负载因子
static final float DEFAULT_LOAD_FACTOR = 0.75f;

// 最大容量
static final int MAXIMUM_CAPACITY = 1 << 30;

public HashMap() {
        this(DEFAULT_INITIAL_CAPACITY, DEFAULT_LOAD_FACTOR);
 }
```

- 对于HashMap的`put`方法如下:

```java
 public V put(K key, V value) {
   // 如果hash桶为空,那么膨胀桶为2^15大小的hashMap信息
        if (table == EMPTY_TABLE) {
            inflateTable(threshold);
        }
   // key为空,也会存储在hashMap中
        if (key == null)
            return putForNullKey(value);
        int hash = hash(key);
        int i = indexFor(hash, table.length);
        for (Entry<K,V> e = table[i]; e != null; e = e.next) {
            Object k;
            if (e.hash == hash && ((k = e.key) == key || key.equals(k))) {
                V oldValue = e.value;
                e.value = value;
                // 不论value是否被覆盖都会唤醒当前方法并记录访问
                e.recordAccess(this);
                return oldValue;
            }
        }
				
   // hashTable结构发生变更,次数+1
        modCount++;
        addEntry(hash, key, value, i);
        return null;
}

// 存放key为null的数据
 private V putForNullKey(V value) {
   for (Entry<K,V> e = table[0]; e != null; e = e.next) {
     if (e.key == null) {
       V oldValue = e.value;
       e.value = value;
       e.recordAccess(this);
       return oldValue;
     }
   }
   modCount++;
   addEntry(0, null, value, 0);
   return null;
 }

// 添加链表的entry
void addEntry(int hash, K key, V value, int bucketIndex) {
  
  // 如果当前的大小大于等于最大的entry个数并且对应的桶位置非空(也就是存在hashTable桶下的链表非空)才会进行扩容
  if ((size >= threshold) && (null != table[bucketIndex])) {
    resize(2 * table.length);
    hash = (null != key) ? hash(key) : 0;
    bucketIndex = indexFor(hash, table.length);
  }

  createEntry(hash, key, value, bucketIndex);
} 

// 扩容为原来大小的2倍
void resize(int newCapacity) {
  Entry[] oldTable = table;
  int oldCapacity = oldTable.length;
  if (oldCapacity == MAXIMUM_CAPACITY) {
    threshold = Integer.MAX_VALUE;
    return;
  }

  Entry[] newTable = new Entry[newCapacity];
  
  // 重新分配hashSeed并将旧的hashTable复制到newTable中
  transfer(newTable, initHashSeedAsNeeded(newCapacity));
  table = newTable;
  threshold = (int)Math.min(newCapacity * loadFactor, MAXIMUM_CAPACITY + 1);
}

// 创建entry并追加到链表中
 void createEntry(int hash, K key, V value, int bucketIndex) {
        Entry<K,V> e = table[bucketIndex];
        table[bucketIndex] = new Entry<>(hash, key, value, e);
        size++;
 }

// 追加entry到链表中
 Entry(int h, K k, V v, Entry<K,V> n) {
   value = v;
   next = n;
   key = k;
   hash = h;
 }
```

- HashMap的`remove`方法

```java
public V remove(Object key) {
  Entry<K,V> e = removeEntryForKey(key);
  return (e == null ? null : e.value);
}

 final Entry<K,V> removeEntryForKey(Object key) {
   if (size == 0) {
     return null;
   }
   int hash = (key == null) ? 0 : hash(key);
   int i = indexFor(hash, table.length);
   Entry<K,V> prev = table[i];
   Entry<K,V> e = prev;

   // 遍历链表并根据key查询对应的entry,并从链表中删除
   while (e != null) {
     Entry<K,V> next = e.next;
     Object k;
     if (e.hash == hash &&
         ((k = e.key) == key || (key != null && key.equals(k)))) {
       modCount++;
       size--;
       if (prev == e)
         table[i] = next;
       else
         prev.next = next;
       e.recordRemoval(this);
       return e;
     }
     prev = e;
     e = next;
   }

   return e;
 }
```

- HashMap的`get`方法

```java
// 根据key获取value信息 
public V get(Object key) {
        if (key == null)
            return getForNullKey();
        Entry<K,V> entry = getEntry(key);

        return null == entry ? null : entry.getValue();
  }
```













#### ConcurrentHashMap







### JDK1.8版本的Hash

#### HashTable







#### HashMap









#### ConcurrentHashMap









