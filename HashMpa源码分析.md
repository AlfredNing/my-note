# HashMpa源码分析

类图

![image-20220109182431871](C:\Users\nq\AppData\Roaming\Typora\typora-user-images\image-20220109182431871.png)

### 属性

```java
transient Node<K,V>[] table; // 底层存储的数组
transient Set<Map.Entry<K,V>> entrySet; // #entrySet() 方法之后的缓存
transient int size; // key-value 键值对数量
transient int modCount; //hashMap的修改次数
int threshold; // 阈值，超过此值进行扩容
final float loadFactor; // 扩容因子
```

**Node.java**

```java
 static class Node<K,V> implements Map.Entry<K,V> {
     	// 哈希值
        final int hash;
     	// key 键
        final K key;
     	// value 值
        V value;
     	// 下一个节点
        Node<K,V> next;

        Node(int hash, K key, V value, Node<K,V> next) {
            this.hash = hash;
            this.key = key;
            this.value = value;
            this.next = next;
        }
     ...
 }
实现 Map.Entry<K,V>接口
```

- TreeNode<K,V> 红黑树节点

### 构造方法

#### 无参构造

```java
static final int DEFAULT_INITIAL_CAPACITY = 1 << 4
static final float DEFAULT_LOAD_FACTOR = 0.75f;
public HashMap() {
    this.loadFactor = DEFAULT_LOAD_FACTOR; // all other fields defaulted
}
```

- 无参构造中，没有对底层数组初始化，延迟初始化，在元素put之后 **#resize()**方法初始化

#### HashMap(int initialCapacity) 构造

```java
public HashMap(int initialCapacity) {
    // 调用两个参数构造器
    this(initialCapacity, DEFAULT_LOAD_FACTOR);
}
```

#### HashMap(int initialCapacity, float loadFactor) 构造

```java
 static final int MAXIMUM_CAPACITY = 1 << 30;

public HashMap(int initialCapacity, float loadFactor) {
    // 检验初始容量大小
    if (initialCapacity < 0)
        throw new IllegalArgumentException("Illegal initial capacity: " +
                                           initialCapacity);
    // 不允许超过MAXIMUM_CAPACITY
    if (initialCapacity > MAXIMUM_CAPACITY)
        initialCapacity = MAXIMUM_CAPACITY;
    // 检验loadFactor是否合法
    if (loadFactor <= 0 || Float.isNaN(loadFactor))
        throw new IllegalArgumentException("Illegal load factor: " +
                                           loadFactor);
    // 设置loadFactor属性
    this.loadFactor = loadFactor;
    // 计算阈值
    this.threshold = tableSizeFor(initialCapacity);
}
```

```java

static final int tableSizeFor(int cap) {
    // cap 从最高位第一个为1开始的位开始，全部设置为1
    int n = cap - 1;
    n |= n >>> 1;
    n |= n >>> 2;
    n |= n >>> 4;
    n |= n >>> 8;
    n |= n >>> 16;
    // n 为 0 ，1,返回1，
    return (n < 0) ? 1 : (n >= MAXIMUM_CAPACITY) ? MAXIMUM_CAPACITY : n + 1;
}
```

#### HashMap(Map<? extends K, ? extends V> m) 构造

```java
public HashMap(Map<? extends K, ? extends V> m) {
    // 设置加载因子
    this.loadFactor = DEFAULT_LOAD_FACTOR;
    // 添加键值对
    putMapEntries(m, false);
}

 final void putMapEntries(Map<? extends K, ? extends V> m, boolean evict) {
        int s = m.size();
        if (s > 0) {
            // table 为Null，未进行初始化
            if (table == null) { // pre-size
                // 根据s的大小，计算最小的tables的大小
                float ft = ((float)s / loadFactor) + 1.0F;
                // 下面int取整，避免不够，提前+1.0F
                int t = ((ft < (float)MAXIMUM_CAPACITY) ?
                         (int)ft : MAXIMUM_CAPACITY);
                // 如果计算的值大于阈值，计算新的阈值
                if (t > threshold)
                    threshold = tableSizeFor(t);
            }
            // table非空，已经进行初始化
            else if (s > threshold)
                // 扩容
                resize();
            for (Map.Entry<? extends K, ? extends V> e : m.entrySet()) {
                // 遍历添加
                K key = e.getKey();
                V value = e.getValue();
                putVal(hash(key), key, value, false, evict);
            }
        }
    }
```

### 哈希函数

1. 性能足够高，哈希函数使用率广

2. 对于计算出来的哈希值足够离散，保证哈希冲突的概率更小

   ```java
   static final int hash(Object key) {
       int h;
       // ^ (h >>> 16) 高 16 位与自身进行异或计算，保证计算出来的 hash 更加离散
       return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
   }
   // 保证高效性与离散型
   ```

​	

### 添加单个元素

```java
public V put(K key, V value) {
    return putVal(hash(key), key, value, false, true);
}
final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
                   boolean evict) {
    	// talbe数组
        Node<K,V>[] tab;
    	// node节点
    	Node<K,V> p; 
    	// n数组大小，i 对应的table位置
    	int n, i;
    	// talbe为空或者容量为0，进行扩容
        if ((tab = table) == null || (n = tab.length) == 0)
            n = (tab = resize()).length;
    	// 对应Node为空，创建节点
        if ((p = tab[i = (n - 1) & hash]) == null)
            tab[i] = newNode(hash, key, value, null);
        else {
            // e代表key已在map中存在的老节点
            Node<K,V> e; K k;
            // 如果p节点就是所求节点
            if (p.hash == hash &&
                ((k = p.key) == key || (key != null && key.equals(k))))
                e = p;
            // 如果p是红黑树节点，直接添加到树中
            else if (p instanceof TreeNode)
                e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
            else {
                // 遍历列表
                for (int binCount = 0; ; ++binCount) {
                    // 当前节点的下一个节点，链表结尾为空，说明key不存在
                    if ((e = p.next) == null) {
                        // 创建新节点
                        p.next = newNode(hash, key, value, null);
                        // 链表长度达到TREEIFY_THRESHOLD（8）转换成红黑树
                        if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st
                            treeifyBin(tab, hash);
                        break;
                    }
                    //  e节点就是目前要找的，直接使用
                    if (e.hash == hash &&
                        ((k = e.key) == key || (key != null && key.equals(k))))
                        break;
                    p = e;
                }
            }
            // 找到了对应的节点
            if (e != null) { // existing mapping for key
                V oldValue = e.value;
                // 修改节点value
                if (!onlyIfAbsent || oldValue == null)
                    e.value = value;
                // 节点访问回调
                afterNodeAccess(e);
                // 返回老的值
                return oldValue;
            }
        }
    	// 增加修改次数
        ++modCount;
    	// 超过阈值，进行扩容
        if (++size > threshold)
            resize();
    	// 添加节点后回调
        afterNodeInsertion(evict);
    	// 返回null
        return null;
    }
// 当key不存在时候，添加key-value键值对到其中
 public V putIfAbsent(K key, V value) {
        return putVal(hash(key), key, value, true, true);
    }
```

### 扩容

```java
final Node<K,V>[] resize() {
    Node<K,V>[] oldTab = table;
    int oldCap = (oldTab == null) ? 0 : oldTab.length;
    int oldThr = threshold;
    int newCap, newThr = 0;
    // talbe非空
    if (oldCap > 0) {
        if (oldCap >= MAXIMUM_CAPACITY) {
            // 超过最大容量，则直接设置 threshold 阀值为 Integer.MAX_VALUE ，不再允许扩容
            threshold = Integer.MAX_VALUE;
            return oldTab;
        }
        // 扩容两倍 如果 oldCap >= DEFAULT_INITIAL_CAPACITY 满足，说明当前容量大于默认值（16），则 2 倍阀值
        else if ((newCap = oldCap << 1) < MAXIMUM_CAPACITY &&
                 oldCap >= DEFAULT_INITIAL_CAPACITY)
            newThr = oldThr << 1; // double threshold
    }
    // 【非默认构造方法】oldThr 大于 0 ，则使用 oldThr 作为新的容量
    else if (oldThr > 0) // initial capacity was placed in threshold
        newCap = oldThr;
    else {              
        // 【默认构造方法】oldThr 等于 0 ，则使用 DEFAULT_INITIAL_CAPACITY 作为新的容量，使用 DEFAULT_LOAD_FACTOR * DEFAULT_INITIAL_CAPACITY 作为新的容量
        newCap = DEFAULT_INITIAL_CAPACITY;
        newThr = (int)(DEFAULT_LOAD_FACTOR * DEFAULT_INITIAL_CAPACITY);
    }
    // 如果上述的逻辑，未计算新的阀值，则使用 newCap * loadFactor 作为新的阀值
    if (newThr == 0) {
        float ft = (float)newCap * loadFactor;
        newThr = (newCap < MAXIMUM_CAPACITY && ft < (float)MAXIMUM_CAPACITY ?
                  (int)ft : Integer.MAX_VALUE);
    }
    // 将 newThr 赋值给 threshold 属性
    threshold = newThr;
    @SuppressWarnings({"rawtypes","unchecked"})
    // 创建新的 Node 数组，赋值给 table 属性
    Node<K,V>[] newTab = (Node<K,V>[])new Node[newCap];
    table = newTab;
    if (oldTab != null) {
        // 老node非空，进行搬运
        for (int j = 0; j < oldCap; ++j) {
            // 获得老的 table 数组第 j 位置的 Node 节点 e
            Node<K,V> e;
            if ((e = oldTab[j]) != null) {
                // 置空老的 table 数组第 j 位置
                oldTab[j] = null;
                // 如果 e 节点只有一个元素，直接赋值给新的 table 即可
                if (e.next == null)
                    newTab[e.hash & (newCap - 1)] = e;
                else if (e instanceof TreeNode)
                    // 如果 e 节点是红黑树节点，则通过红黑树分裂处理
                    ((TreeNode<K,V>)e).split(this, newTab, j, oldCap);
                else { // preserve order
                    // 是链表 如果结果为 0 时，则放置到低位，如果结果非 1 时，则放置到高位
                    Node<K,V> loHead = null, loTail = null;
                    Node<K,V> hiHead = null, hiTail = null;
                    Node<K,V> next;
                    // 这里 do while 的原因是，e 已经非空，所以减少一次判断
                    do {
                        // 指向下一个节点
                        next = e.next;
                        // 满足地位
                        if ((e.hash & oldCap) == 0) {
                            if (loTail == null)
                                loHead = e;
                            else
                                loTail.next = e;
                            loTail = e;
                        }
                        // 满足高位
                        else {
                            if (hiTail == null)
                                hiHead = e;
                            else
                                hiTail.next = e;
                            hiTail = e;
                        }
                    } while ((e = next) != null);
                    // 设置低位到新的 newTab 的 j 位置上
                    if (loTail != null) {
                        loTail.next = null;
                        newTab[j] = loHead;
                    }
                    // 设置高位到新的 newTab 的 j + oldCap 位置上
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

### 添加多个元素

```java
public void putAll(Map<? extends K, ? extends V> m) {
    putMapEntries(m, true);
}
// 构造方法也调用该函数
final void putMapEntries(Map<? extends K, ? extends V> m, boolean evict) {
        int s = m.size();
        if (s > 0) {
            if (table == null) { // pre-size
                float ft = ((float)s / loadFactor) + 1.0F;
                int t = ((ft < (float)MAXIMUM_CAPACITY) ?
                         (int)ft : MAXIMUM_CAPACITY);
                if (t > threshold)
                    threshold = tableSizeFor(t);
            }
            else if (s > threshold)
                resize();
            for (Map.Entry<? extends K, ? extends V> e : m.entrySet()) {
                K key = e.getKey();
                V value = e.getValue();
                putVal(hash(key), key, value, false, evict);
            }
        }
    }
```

### 移出单个元素

```java
public V remove(Object key) {
    Node<K,V> e;
    return (e = removeNode(hash(key), key, null, false, true)) == null ?
        null : e.value;
}

final Node<K,V> removeNode(int hash, Object key, Object value,
                               boolean matchValue, boolean movable) {
        Node<K,V>[] tab; // talbe数组
    	Node<K,V> p;	// hash对应的table位置p节点
    	int n, index;
    		// 查找 hash 对应 table 位置的 p 节点
        if ((tab = table) != null && (n = tab.length) > 0 &&
            (p = tab[index = (n - 1) & hash]) != null) {
            // 查找 hash 对应 table 位置的 p 节点
            Node<K,V> node = null, e; K k; V v;
            if (p.hash == hash &&
                // 如果找到的 p 节点，就是要找的，则则直接使用即可
                ((k = p.key) == key || (key != null && key.equals(k))))
                node = p;
            else if ((e = p.next) != null) {
                //  如果找到的 p 节点，是红黑树 Node 节点，则直接在红黑树中查找
                if (p instanceof TreeNode)
                    node = ((TreeNode<K,V>)p).getTreeNode(hash, key);
                else {
                    do {
                        // 如果遍历的 e 节点，就是要找的，则则直接使用即可
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
            // 如果有要求匹配 value 的条件，这里会进行一次判断先移除
            if (node != null && (!matchValue || (v = node.value) == value ||
                                 (value != null && value.equals(v)))) {
                // 如果找到的 node 节点，是红黑树 Node 节点，则直接在红黑树中删除
                if (node instanceof TreeNode)
                    ((TreeNode<K,V>)node).removeTreeNode(this, tab, movable);
                else if (node == p)
                    // 如果查找到的是链表的头节点，则直接将 table 对应位置指向 node 的下一个节点，实现删除
                    tab[index] = node.next;
                else
                    // 如果查找到的是链表的中间节点，则将 p 指向 node 的下一个节点，实现删除
                    p.next = node.next;
                // 增加修改次数
                ++modCount;
                // 减少hashMap数量
                --size;
                // 移出node后的回调
                afterNodeRemoval(node);
                // 返回 node
                return node;
            }
        }
    	// 查找不到，则返回 null
        return null;
    }
```