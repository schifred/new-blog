---
title: 透过散列表看HashMap
category:
  - 计算机科学
  - 算法
tags:
  - 计算机科学
  - 算法
keywords: '算法,HashMap'
abbrlink: 360c1bd
date: 2019-04-20 00:00:00
updated: 2019-04-20 00:00:00
---

散列表用于存储键值对。先举两个例子：如果使用有序数组存储键值对，那么当存在某个较大的键时，整个数组所占用的内存空间就会很大；如果使用无序数组存储键值对，那么在查找元素时就需要遍历数组项，造成了性能的低效。与这两个例子不同的是，散列表有效地平衡了时间和空间复杂度。创建散列表的流程分为：

1. 通过散列函数将键转化为散列码，以作为数组的索引。
2. 通过碰撞处理解决两个或多个散列码等值的情况。

### 散列函数

制作散列函数时需要面对的问题是：对于任意数据类型的键，都需要将其转化为可接受的数组索引；计算过程应尽可能的简便，计算结果应尽可能地均匀分布。可针对如下情形实现不同的散列策略：

1. 正整数 k：可采用除留余数法，即当数组长度为 M 时，散列码就是 k%M。当 M 是素数时，得到的散列码会均匀分布。
2. 浮点数 k：可采用 k*M 再四舍五入的方式计算码；也可采用将 k 表示为二进制数，然后再使用除留余数法。后一种方式更均匀，因为在使用前一种方式时，浮点数高位起到的作用会更大。
3. 字符串 k：可采用 horner 算法获取散列码，即

![image](hm1.svg)

其中，s.charAt(i) 将以非负 16 位整数形式获取 char 值。当 R 为较小的素数如 31 时，就可以保证结果的均匀分布。
4. 组合键 k：如果键包含多个整型变量时，可以使用如字符串的形式将其拆解。以 Date 为例，可拆解为 day, month, year 三个整型，这时就可以通过 (((day R + month) % M) R + year) % M 计算散列码。

在 Java 中，每种数据类型都有对应的散列函数。同时每种数据类型的 hashCode 方法须与 equals 方法表现一致，即当 a.equals(b) 返回 true 时，那么 a.hashCode() 结果须与 b.hashCode() 相同；反之则不然，即当 a.hashCode() 与 b.hashCode() 返回值相同时，a.equals(b) 未必返回 true。默认的散列函数会返回对象的内存地址，但只适用于极少数情况。字节型 Byte, 短整型 Short, 整型 Integer, 长整型 Long 的 hashCode 方法都以 32 位 4 字节整数作为散列码；字符型 Character 以 8 位整数为散列码；布尔型 Boolean 以 1231, 1237 作为散列码，true 时为 1231。其他类型的 hashCode 方法或可参见下方代码（特别的，对于自定义的 Java 类，可采用与 String 相类的手法计算散列码，即通过各属性的散列码计算对象的散列码）：

```java
// 单精度浮点型；双精度浮点型 Double 与此相类
public final class Float extends Number implements Comparable<Float> {
    // native 关键字的函数由操作系统实现（使用如 c 语言），java 只能调用
    // 单精度浮点型第 31 位为符号，30-23 位为指数，22-0 为有效数
    public static native int floatToRawIntBits(float value);

    public static int floatToIntBits(float value) {
        int result = floatToRawIntBits(value);
        // Check for NaN based on values of bit fields, maximum
        // exponent and nonzero significand.
        // EXP_BIT_MASK = 2139095040; SIGNIF_BIT_MASK = 8388607;
        if ( ((result & FloatConsts.EXP_BIT_MASK) ==
              FloatConsts.EXP_BIT_MASK) &&
             (result & FloatConsts.SIGNIF_BIT_MASK) != 0)
            result = 0x7fc00000;
        return result;
    }

    public static int hashCode(float value) {
        return floatToIntBits(value);
    }

    @Override
    public int hashCode() {
        return Float.hashCode(value);
    }
}

// 字符串
public final class String
    implements java.io.Serializable, Comparable<String>, CharSequence {
    // 使用 hash 作为软缓存，避免重复计算，第二次执行 hashCode 都将返回计算好的 hash
    private int hash; // Default to 0

    public int hashCode() {
        int h = hash;
        if (h == 0 && value.length > 0) {
            char val[] = value;

            for (int i = 0; i < value.length; i++) {
                h = 31 * h + val[i];
            }
            hash = h;
        }
        return h;
    }
}
```

HashMap 中的散列码通过静态方法 hash 计算，即取 key 键的散列码，并与右移 16 位的散列码进行异或。执行 (n - 1) & hash 计算，可以将散列码转化为数组索引。与此不同的是，Hashtable 的散列码通过执行 key.hashCode 直接获取，再执行 (hash & 0x7FFFFFFF) % M 计算出数组索引。散列码计算方式改易的原因不详。

```java
// HashMap 中的散列码
static final int hash(Object key) {
    int h;
    return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
}

// 常规散列码，也是 Hashtable 中的散列码
private int hash(Object key){
    return (key.hashCode() & 0x7FFFFFFF) % M;
}
```

### 碰撞冲突

当键的散列码等值时，有两种方式可用于处理这类碰撞冲突的情况：其一是拉链法，即在数组项中以链表的形式存储散列码等值的元素；其一是基于线性探测法等实现开放地址散列表，即存储的元素量不能超过数组长度，使数组足够容纳冲突的键。

#### 拉链法

拉链法既可使用原始链表实现，也可使用符号表实现。不同的是，原始链表对链表节点进行建模，符号表基于单向链表对整体进行建模。以下是基于符号表的实现：

```java
// 符号表
public class SequentialSearchST<Key, Value>{
    private Node first;
    private class Node {
        Key key;
        Value val;
        Node next;
        public Node(Key key, Value val, Node next) {
            this.key = key;
            this.val = val;
            this.next = next;
        }
    }
    public Value get(Key key) {
        for (Node x = first; x != null; x = x.next) {
            if (key.equals(x.key)) {
                return x.val;
            }
        }
        return null;
    }
    public void put(Key key, Value val) {
        for (Node x = first; x != null; x = x.next) {
            if (key.equals(x.key)) {
                x.val = val;
                return;
            }
        }
        first = new Node(key, val, first);
    }
}

// 基于符号表、拉链法的散列表
public class SeparateChainingHashST<Key, Value>{
    private static final int INIT_CAPACITY = 997;
    private int N;// 键值对总数
    private int M;// 散列表大小
    private SequentailSearchST<Key, Value>[] st;// 链表数组
    public SeparateChainingHashST() {
        this(INIT_CAPACITY);
    }
    public SeparateChainingHashST(int M) {
        this.M = M;
        // Java 不支持泛型数组，先需经过类型转换
        st = (SequentailSearchST<Key, value>[]) new SequentailSearchST[M];
        for (int i = 0; i < m; i++) {
            st[i] = new SequentailSearchST<>();
        }
    }
    private int hash(Key key) {
        return (key.hashCode() & 0x7fffffff) % M;
    }
    private Value get(Key key) {
        return (Value) st[hash(key)].get(key);
    }
    private void put(Key key, Value val) {
        st[hash(key)].put(key, val);
    }
}
```

在 java 中，Hashtable 和 HashMap 都是基于拉链法构建的，且每个数组项被称为桶。无论 Hashtable，还是 HashMap，散列表的大小都可基于插入的元素量进行动态调整，这一过程被称为再散列 rehash。

### Hashtable

Hashtable 的 rehash 过程为：当元素量超过 hashtable.threshold 阈值（散列表的长度乘以装填因子 loadFactor）时，首先创建长度翻倍的新数组，再将原有元素按新的索引值插入到数组中，最后废弃旧数组。

Hashtable 的公共方法都加上了 synchronized 关键字，因此它是线程安全的，能保证在插值过程中找到正确的索引，而不会引起当另一个线程变更数组长度时导致的索引不定问题。

```java
public class Hashtable<K,V> extends Dictionary<K,V>
    implements Map<K,V>, Cloneable, java.io.Serializable {
    private transient Entry<?,?>[] table;// 以链表数组形式存储元素
    private int threshold;// 阈值，超过该值将 rehash，值为当前长度 ✖ 装填因子
    private float loadFactor;// 装填因子

    // 添加元素
    public synchronized V put(K key, V value) {
        if (value == null) {
            throw new NullPointerException();
        }

        Entry<?,?> tab[] = table;
        int hash = key.hashCode();
        int index = (hash & 0x7FFFFFFF) % tab.length;
        @SuppressWarnings("unchecked")
        Entry<K,V> entry = (Entry<K,V>)tab[index];

        // 长度变更使得元素索引不定，需要遍历数组查找元素是否已在散列表中
        for(; entry != null ; entry = entry.next) {
            if ((entry.hash == hash) && entry.key.equals(key)) {
                V old = entry.value;
                entry.value = value;
                return old;
            }
        }

        addEntry(hash, key, value, index);
        return null;
    }

    // 私有方法，结合 rehash 添加新元素
    private void addEntry(int hash, K key, V value, int index) {
        modCount++;

        Entry<?,?> tab[] = table;
        if (count >= threshold) {
            rehash();

            tab = table;
            hash = key.hashCode();
            index = (hash & 0x7FFFFFFF) % tab.length;
        }

        @SuppressWarnings("unchecked")
        Entry<K,V> e = (Entry<K,V>) tab[index];
        tab[index] = new Entry<>(hash, key, value, e);
        count++;
    }

    // 再散列
    protected void rehash() {
        int oldCapacity = table.length;
        Entry<?,?>[] oldMap = table;

        int newCapacity = (oldCapacity << 1) + 1;// 长度放大两倍
        if (newCapacity - MAX_ARRAY_SIZE > 0) {
            if (oldCapacity == MAX_ARRAY_SIZE)
                return;
            newCapacity = MAX_ARRAY_SIZE;
        }
        Entry<?,?>[] newMap = new Entry<?,?>[newCapacity];

        modCount++;
        threshold = (int)Math.min(newCapacity * loadFactor, MAX_ARRAY_SIZE + 1);
        table = newMap;

        for (int i = oldCapacity ; i-- > 0 ;) {
            for (Entry<K,V> old = (Entry<K,V>)oldMap[i] ; old != null ; ) {
                Entry<K,V> e = old;
                old = old.next;

                int index = (e.hash & 0x7FFFFFFF) % newCapacity;
                e.next = (Entry<K,V>)newMap[index];
                newMap[index] = e;
            }
        }
    }

    // 链表节点模型
    private static class Entry<K,V> implements Map.Entry<K,V> {
        final int hash;
        final K key;
        V value;
        Entry<K,V> next;// 构成单向链表

        public int hashCode() {
            return hash ^ Objects.hashCode(value);
        }
    }
}
```

### WeakHashMap

一言以蔽之，WeakHashMap 是基于 Map 接口实现的线程不安全的 Hashtable。

### HashMap

因为 Hashtable 通过遍历链表的方式查找和插入元素并不高效，HashMap 在桶的容量超过指定值时，就会将桶的存储结构转化为红黑树，这样就提升了查找和插入的效率。关于红黑树的更多内容，可参见HashMap中的红黑树。

至于为什么在 Java 中，HashMap 被设计成线程不安全的？因为线程安全的实现方式会增加检查、加锁、解锁的开销。Java 另外提供了线程安全的 ConcurrentHashMap 类。

```java
public class HashMap<K,V> extends AbstractMap<K,V>
    implements Map<K,V>, Cloneable, Serializable {
    transient Node<K,V>[] table;// 以链表数组形式存储元素

    static final int hash(Object key) {
        int h;
        return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
    } 

    // 查找元素
    public V get(Object key) {
        Node<K,V> e;
        return (e = getNode(hash(key), key)) == null ? null : e.value;
    }

    final Node<K,V> getNode(int hash, Object key) {
        Node<K,V>[] tab; Node<K,V> first, e; int n; K k;
        if ((tab = table) != null && (n = tab.length) > 0 &&
            (first = tab[(n - 1) & hash]) != null) {
            // 桶中的首节点就是查找的元素
            if (first.hash == hash && // always check first node
                ((k = first.key) == key || (key != null && key.equals(k))))
                return first;
            if ((e = first.next) != null) {
                // 从红黑树查找节点
                if (first instanceof TreeNode)
                    return ((TreeNode<K,V>)first).getTreeNode(hash, key);
                // 从链表中查找节点
                do {
                    if (e.hash == hash &&
                        ((k = e.key) == key || (key != null && key.equals(k))))
                        return e;
                } while ((e = e.next) != null);
            }
        }
        return null;
    }
    
    // 插入元素
    public V put(K key, V value) {
        return putVal(hash(key), key, value, false, true);
    }

    final V putVal(int hash, K key, V value, boolean onlyIfAbsent, boolean evict) {
        Node<K,V>[] tab; Node<K,V> p; int n, i;
        // 保证散列表的长度不为 0
        if ((tab = table) == null || (n = tab.length) == 0)
            n = (tab = resize()).length;
        // 桶中首个节点
        if ((p = tab[i = (n - 1) & hash]) == null)
            tab[i] = newNode(hash, key, value, null);
        else {
            Node<K,V> e; K k;
            // 链表、红黑二叉树的首节点与插入元素含有相同 key 键
            if (p.hash == hash &&
                ((k = p.key) == key || (key != null && key.equals(k))))
                e = p;
            // 将元素插入红黑树中
            else if (p instanceof TreeNode)
                e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
            // 将元素插入链表中；当超过阈值时，转化为红黑树
            else {
                // 遍历链表
                for (int binCount = 0; ; ++binCount) {
                    if ((e = p.next) == null) {
                        // 将元素插入到链表的尾端；newNode 方法创建链表节点
                        p.next = newNode(hash, key, value, null);

                        // 超过指定长度 TREEIFY_THRESHOLD = 8，将链表转化成红黑树
                        if (binCount >= TREEIFY_THRESHOLD - 1)
                            treeifyBin(tab, hash);
                        break;
                    }
                    // 链表中存在相同的 key 键
                    if (e.hash == hash &&
                        ((k = e.key) == key || (key != null && key.equals(k))))
                        break;
                    p = e;
                }
            }
            // 链表或红黑二叉树中存在相同的 key 键
            if (e != null) {
                V oldValue = e.value;
                if (!onlyIfAbsent || oldValue == null)
                    e.value = value;
                afterNodeAccess(e);// 触发 LinkedHashMap 的动作
                return oldValue;
            }
        }
        ++modCount;
        if (++size > threshold)
            resize();
        afterNodeInsertion(evict);// 触发 LinkedHashMap 的动作
        return null;
    }

    // 将链表转化成红黑树
    final void treeifyBin(Node<K,V>[] tab, int hash) {
        int n, index; Node<K,V> e;
        // 散列表红黑树化前的最小长度 MIN_TREEIFY_CAPACITY = 64，未满足，则扩容
        if (tab == null || (n = tab.length) < MIN_TREEIFY_CAPACITY)
            resize();
        else if ((e = tab[index = (n - 1) & hash]) != null) {
            TreeNode<K,V> hd = null, tl = null;
            do {
                // replacementTreeNode 将链表节点转化为红黑树节点
                TreeNode<K,V> p = replacementTreeNode(e, null);
                if (tl == null)// 填充首节点
                    hd = p;
                // 修正链表节点的 prev, next 属性
                else {
                    p.prev = tl;
                    tl.next = p;
                }
                tl = p;// 记录上一个节点
            } while ((e = e.next) != null);
            if ((tab[index] = hd) != null)
                // 将链表转化成红黑树
                hd.treeify(tab);
        }
    }

    // 再散列，在保证容量和阈值不为 0 的前提下，把容量翻倍，老数据分拆注入新数组中
    final Node<K,V>[] resize() {
        Node<K,V>[] oldTab = table;
        int oldCap = (oldTab == null) ? 0 : oldTab.length;
        int oldThr = threshold;
        int newCap, newThr = 0;
        if (oldCap > 0) {
            // 散列表容器超过最大值 MAXIMUM_CAPACITY = 1 << 30，只调整阈值
            if (oldCap >= MAXIMUM_CAPACITY) {
                threshold = Integer.MAX_VALUE;
                return oldTab;
            }
            // 散列表容量翻倍
            else if ((newCap = oldCap << 1) < MAXIMUM_CAPACITY &&
                     oldCap >= DEFAULT_INITIAL_CAPACITY)
                newThr = oldThr << 1;
        } 
        else if (oldThr > 0)
            newCap = oldThr;
        // 设置初始容量 DEFAULT_INITIAL_CAPACITY = 1 << 4 和阈值
        // 装填因子 DEFAULT_LOAD_FACTOR = 0.75f
        else {
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
            Node<K,V>[] newTab = (Node<K,V>[])new Node[newCap];// 链表数组
        table = newTab;
        if (oldTab != null) {
            for (int j = 0; j < oldCap; ++j) {
                Node<K,V> e;
                if ((e = oldTab[j]) != null) {
                    oldTab[j] = null;
                    // 桶中只有节点，将节点直接添加到新的桶中
                    if (e.next == null)
                        newTab[e.hash & (newCap - 1)] = e;
                    // 桶的数据结构为红黑树，将红黑树分拆到两个桶中（j, j + oldCap），其数据结构为红黑树或链表
                    else if (e instanceof TreeNode)
                        ((TreeNode<K,V>)e).split(this, newTab, j, oldCap);
                    // 桶的数据结构为链表，将链表分拆到两个桶中（j, j + oldCap），其数据结构仍为链表
                    else { // preserve order
                        Node<K,V> loHead = null, loTail = null;
                        Node<K,V> hiHead = null, hiTail = null;
                        Node<K,V> next;
                        do {
                            next = e.next;
                            if ((e.hash & oldCap) == 0) {
                                if (loTail == null)
                                    loHead = e;
                                else
                                    loTail.next = e;// 记录上一个节点
                                loTail = e;
                            } else {
                                if (hiTail == null)
                                    hiHead = e;
                                else
                                    hiTail.next = e;
                                hiTail = e;
                            }
                        } while ((e = next) != null);
                        if (loTail != null) {
                            loTail.next = null;
                            newTab[j] = loHead;
                        }
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

    // 删除节点
    public V remove(Object key) {
        Node<K,V> e;
        return (e = removeNode(hash(key), key, null, false, true)) == null ?
            null : e.value;
    }

    final Node<K,V> removeNode(int hash, Object key, Object value,
                               boolean matchValue, boolean movable) {
        Node<K,V>[] tab; Node<K,V> p; int n, index;
        if ((tab = table) != null && (n = tab.length) > 0 &&
            (p = tab[index = (n - 1) & hash]) != null) {
            // 先找到待删除的节点
            Node<K,V> node = null, e; K k; V v;
            // 待删除为首节点
            if (p.hash == hash &&
                ((k = p.key) == key || (key != null && key.equals(k))))
                node = p;
            // 从红黑树或链表中找到待删除节点
            else if ((e = p.next) != null) {
                if (p instanceof TreeNode)
                    node = ((TreeNode<K,V>)p).getTreeNode(hash, key);
                else {
                    do {
                        if (e.hash == hash &&
                            ((k = e.key) == key ||
                             (key != null && key.equals(k)))) {
                            node = e;
                            break;
                        }
                        p = e;
                    } while ((e = e.next) != null);
                }
            }

            // 删除节点
            if (node != null && (!matchValue || (v = node.value) == value ||
                                 (value != null && value.equals(v)))) {
                if (node instanceof TreeNode)
                    ((TreeNode<K,V>)node).removeTreeNode(this, tab, movable);
                else if (node == p)
                    tab[index] = node.next;
                else
                    p.next = node.next;
                ++modCount;
                --size;
                afterNodeRemoval(node);
                return node;
            }
        }
        return null;
    }
}
```

### 概率问题

本节所要致力于解决的问题是，当散列表的大小为 M 时，长度为 k 的链表出现的概率是多少？为什么要解决这个问题呢？因为只有在链表长度足够短时，查找和插入节点才会显得高效。

假设散列函数能将所有的元素均匀且独立地分配到数组中，即元素放入某个链表中的概率为 1/M，没有放入该链表的概率为 1-1/M。因此由二项分布可知，该链表长度为 k 的概率为（α = N/M 为期望）：

![image](hm2.svg)

当 α 足够小时，可以转化为泊松分布的数学表达式：

![image](hm3.svg)

介于此，对于初始大小为 16 的 HashMap，链表长度等于 8 的可能性为 0.00000006，这时就需要将链表转换成红黑树。

### 线性探测法

因为开放地址散列表的长度大于待插入的元素量，当插入元素的数组索引已被占用时，就可以通过索引自增 1 的方式向下查找并插入。在这个过程中，插入元素可能已经存在在散列表中，因此就需要检测数组元素的键是否和插入元素的键相同，这一过程称为探测。需要说明的是，开放地址散列表中的空位譬如磁盘碎片，不只有一处，而是会散落多处。探测的成本在于需要遍历连续无间断的元素量（即键簇中包含的元素量）。

开放地址散列表的性能也依赖于 N/M 的比值，即数组的使用率。我们需要动态调整数组大小的方式来保证使用率在 1/8 到 1/2 之间。

```java
public class LinearProbingHashST<Key, Value>{
    private int N;
    private int M;
    private Key[] keys;
    private Value[] vals;
    public LinearProbingHashST() {
        keys = (Key[]) new Object[M];
        values = (Value[]) new Object[M];
    }
    public void put(Key key, Value val) {
        if (N >= M / 2) resize(M * 2);
        
        int i;
        // 采用余数计算索引，在保证索引正确的同时，也能跳回到 0 索引位置
        for (i = hash(key); keys[i] != null; i = (i + 1) % M) {
            if (key.equals(keys[i])) {
                vals[i] = val;
                return;
            }
        }
        keys[i] = key;
        vals[i] = val;
        N++;
    }
    public Value get(Key key) {
        for (int i = hash(key); keys[i] != null; i = (i + 1) % M) {
            if (key.equals(keys[i])) {
                return vals[i];
            }
        }
        return null;
    }
    public void delete(Key key) {
        if (!contains(key)) {
            return;
        }

        int i = hash(key);
        // 找到 key 键
        while (!key.equals(keys[i])) {
            i = (i + 1) % M;
        }

        keys[i] = null;
        values[i] = null;

        i = (i + 1) % M;
        // 遍历调整后续元素的位置
        while (keys[i] != null) {
            Key keyToRehash = keys[i];
            Value valueToRehash = values[i];
            keys[i] = null;
            values[i] = null;
            N--;
            put(keyToRehash, valueToRehash);
            i = (i + 1) % M;
        }
        N--;
        if (N > 0 && N == M / 8) {
            resize(M / 2);
        }
    }
}
```

### 参考

[Java中Native关键字的作用](https://www.cnblogs.com/KingIceMou/p/7239668.html)
[如何通俗理解泊松分布？](https://blog.csdn.net/ccnt_2012/article/details/81114920)
[HashMap桶中链表转红黑树为什么选择数字8？](https://blog.csdn.net/Mollychin/article/details/80444967)
[hashMap线程不安全的原因及表现](https://blog.csdn.net/VIP_WangSai/article/details/70182933)