---
title: Java集合之HashMap 1.8 源码解析
date: 2019-12-06 15:38:06
tags: 源码解析
categories: Java
---

> 本文源码基于HashMap 1.8,下载地址：[Java 8]()
>
> 另外本文不分析红黑树相关的源码

# 前言

在对HashMap进行源码解析前，我们很有必要搞清楚下面这几个名词，这对于下文的阅读有很大的帮助。

- 哈希表：这里指的就是HashMap
- 哈希桶：HashMap的底层数据结构，即数组
- 链表：哈希桶的下标装的就是链表
- 节点：链表上的节点就是哈希表上的元素
- 哈希表的容量：元素的总个数
- 哈希桶的容量：数组的个数，由于当发生哈希冲突时，采用链地址法解决冲突，故哈希桶的容量<=哈希表的容量

> 注意：一定要区分哈希表的容量和哈希桶的容量，一开始很容易将这两个定义搞混淆

# 一、链表节点

> HashMap.Node

```java
    static class Node<K,V> implements Map.Entry<K,V> {
        final int hash;//哈希值
        final K key;
        V value;
        Node<K,V> next;//下一个结点

        Node(int hash, K key, V value, Node<K,V> next) {
            this.hash = hash;
            this.key = key;
            this.value = value;
            this.next = next;
        }

        public final K getKey()        { return key; }
        public final V getValue()      { return value; }
        public final String toString() { return key + "=" + value; }

		//结点的hash值等于key和value哈希值的异或
        public final int hashCode() {
            return Objects.hashCode(key) ^ Objects.hashCode(value);
        }

		//设置新的value，同时返回旧的value
        public final V setValue(V newValue) {
            V oldValue = value;
            value = newValue;
            return oldValue;
        }
        
        //key和value都相等才被认为是相同的节点
        public final boolean equals(Object o) {
            if (o == this)
                return true;
            if (o instanceof Map.Entry) {
                Map.Entry<?,?> e = (Map.Entry<?,?>)o;
                if (Objects.equals(key, e.getKey()) &&
                    Objects.equals(value, e.getValue()))
                    return true;
            }
            return false;
        }
    }
```

从上面可以发现哈希桶的链表就是单链表结构，并且节点的hash值会等于key和value哈希值的异或。

# 二、成员属性

> HashMap

## 1. 常量

```java
    //哈希桶默认容量为16
    static final int DEFAULT_INITIAL_CAPACITY = 1 << 4; // aka 16

	//哈希桶最大容量2的30次方
    static final int MAXIMUM_CAPACITY = 1 << 30;

    //默认加载因子
    static final float DEFAULT_LOAD_FACTOR = 0.75f;

    //转化成红黑树的阈值，当哈希桶的链表结点数量大于等于8时，转化成红黑树
    static final int TREEIFY_THRESHOLD = 8;

    //如果当前是红黑树结构，那么当桶的链表结点数量小于6时，会转换成链表
    static final int UNTREEIFY_THRESHOLD = 6;
	
    //当哈希表的容量达到64时，也会转换为红黑树结构
    static final int MIN_TREEIFY_CAPACITY = 64;
```

## 2. 变量

```java
    //哈希桶，存放链表。transient关键字表示该属性不能被序列化
    transient Node<K,V>[] table;

    //迭代功能
    transient Set<Map.Entry<K,V>> entrySet;

    //哈希表元素数量
    transient int size;

    //统计该map修改的次数
    transient int modCount;

    //阈值，当元素数量，即哈希表的容量达到阈值时，会进行扩容
    int threshold;

    //加载因子，用于计算哈希表的阈值。threshold = 哈希桶的容量*loadFactor
    final float loadFactor;
```

# 三、构造器

## 1. 无参构造器

```java
    //默认的构造函数，加载因子为默认的0.75f
    public HashMap() {
        this.loadFactor = DEFAULT_LOAD_FACTOR; // all other fields defaulted
    }
```

我们在一般情况下都是用这个无参构造器的，这就证明当我们平时new了一个HashMap时，底层只是设置了一个加载因子的值为默认的0.75f

## 2. 指定哈希桶容量的构造函数

```java
    public HashMap(int initialCapacity) {
        this(initialCapacity, DEFAULT_LOAD_FACTOR);
    }
```

其实调用的是下面一个构造函数，不过这里在指定哈希表的容量的同时，也指定了加载因子为默认值

## 3. 指定哈希桶容量和加载因子的构造函数

```java
	//指定加载因子的构造函数还是用的比较少的
    public HashMap(int initialCapacity, float loadFactor) {
    	//初始化容量不能为负数
        if (initialCapacity < 0)
            throw new IllegalArgumentException("Illegal initial capacity: " +
                                               initialCapacity);
		//初始化容量不能超过2的30次方
        if (initialCapacity > MAXIMUM_CAPACITY)
            initialCapacity = MAXIMUM_CAPACITY;
        if (loadFactor <= 0 || Float.isNaN(loadFactor))
            throw new IllegalArgumentException("Illegal load factor: " +
                                               loadFactor);

		//设置加载因子的值
        this.loadFactor = loadFactor;
		//设置哈希表的阈值，将哈希桶的容量暂时存放在哈希表的阈值
        this.threshold = tableSizeFor(initialCapacity);
    }
```

在上面对容量做了一些边界处理，然后设置了加载因子的值和哈希表的阈值，这里在设置阈值时，首先会调用tableSizeFor函数对容量做一些处理

```java
    static final int tableSizeFor(int cap) {
        int n = cap - 1;
        n |= n >>> 1;
        n |= n >>> 2;
        n |= n >>> 4;
        n |= n >>> 8;
        n |= n >>> 16;
        return (n < 0) ? 1 : (n >= MAXIMUM_CAPACITY) ? MAXIMUM_CAPACITY : n + 1;
    }
```

上面具体实现就不再深究，最后的结果就是能够返回刚好大于cap的2的n次方，当扩容时也会调用这个函数，这就保证了哈希桶的容量永远都是2的n次方，也正是因为这个前提下，接下来的取下标的操作能够通过 `hash&(table.length-1)`来替换`hash%(table.length)`。

这时候你也许就会有疑问了，我明明传入的是哈希桶的容量，怎么最后却赋值给了阈值呢？这其实是因为在构造器中，并没有对哈希桶table进行初始化，初始化的工作交给了扩容函数。当第一次put时，会调用扩容函数，将阈值赋值给哈希桶的容量，接着对哈希桶table进行初始化，然后根据公式设置重新设置阈值，大概流程是这样，后续还会提到！

## 4. 批量添加元素的构造函数

```java
    public HashMap(Map<? extends K, ? extends V> m) {
        this.loadFactor = DEFAULT_LOAD_FACTOR;
        putMapEntries(m, false);
    }
```

这个构造函数比较特殊，其作用就是在构造一个新的哈希表的同时加入指定map所有的元素。这里也是首先设置了加载因子为默认值，然后调用putMapEntries方法来进行批量增加元素，注意这里的第二个参数为false。

```java
    //将map表的所有元素加入到当前表中，当前Map初始化时evict为false，其它情况为true
    final void putMapEntries(Map<? extends K, ? extends V> m, boolean evict) {
        int s = m.size();
        if (s > 0) {
            if (table == null) { // pre-size
            	//求出需要的容量。因为实际使用的长度=容量*加载因子
            	//+1是因为小数相除，基本都不会是整数，容量大小不能为小数的，
                //后面转换为int，多余的小数就要被丢掉，
            	//所以+1，例如，map实际长度22，22/0.75=29.3,所需要的容量肯定为30
            	//如果刚刚好除得整数呢，除得整数的话，容量大小多1也没什么影响
                float ft = ((float)s / loadFactor) + 1.0F;
                int t = ((ft < (float)MAXIMUM_CAPACITY) ?
                         (int)ft : MAXIMUM_CAPACITY);
			    //将哈希桶的容量存在阈值中
                if (t > threshold)
                    threshold = tableSizeFor(t);
            }
			//当前表还没有初始化，所以不会进行扩容
            else if (s > threshold)
                resize();

            //遍历
            for (Map.Entry<? extends K, ? extends V> e : m.entrySet()) {
                K key = e.getKey();
                V value = e.getValue();
                //添加
                putVal(hash(key), key, value, false, evict);
            }
        }
    }

```

假设这里要批量增加的数据不为0，通过上面分析我们知道，构造器并没有构造哈希桶table，所以这里的table为null。接着就是跟上面的构造函数一样，将要创建的哈希桶的容量暂时存在阈值中。

这里有个问题值得一提，刚开始我一直以为ft是阈值，所以一直搞不明白为什么要用` s/loadFactor` ,因为公式是：`threshold = s*loadFactor`才对。后来才想通，这里的ft并不是阈值，而是哈希桶的容量，因为最后并不是设置阈值，而是将容量的值暂存在阈值中。

接着会遍历m依次将元素添加到当前哈希表中，这里涉及到了两个操作：遍历和添加，这里不进行展开讲，在后文会详细进行分析。

## 5. 小结

这里我们重新梳理下当new一个HashMap时，内部的实际工作：

- 无参数构造HashMap，如`HashMap<Integer,Integer> map = new HashMap<>()`,只设置了加载因子为默认值
- 指定哈希桶容量构造HashMap,如`HashMap<Integer,Integer> map = new HashMap<>(5)`,内部的工作就是设置加载因子为默认值。暂存容量的值到阈值中，其值为刚好大于容量的2的n次方，这里的阈值为8。
- 同时指定容量和加载因子，这个构造器一般很少使用
- 在构造的同时批量加载数据。如`HashMap<Integer,Integer> newMap = new HashMap<>(map)`,根据map的size和公式算出哈希桶的容量，然后将容量暂存到阈值中，最后遍历map，将map中的元素添加到newMap中

> 可以发现在构造的时候并没有构造哈希桶table的实例，所以将哈希桶的容量都先暂存在阈值threshold中

# 四、添加

put操作其实包括了哈希表的增、改两个操作。当添加的元素key存在时，就会修改value的值，不存在则添加这个元素。

## 1. 添加(改)一个元素

在实际上我们的操作很简单，就一行代码搞定

```java
map.put(key,value);

```

所以我们直接看put的操作

> HashMap#put

```java
    public V put(K key, V value) {
        //先计算key的hash值
        return putVal(hash(key), key, value, false, true);
    }

```

这里涉及到了hash函数来求当前key的hash值，我们来看看

> HashMap#hash

```java
    static final int hash(Object key) {
        int h;
		//当前key为null，则hash指为0，否则返回扰动函数干扰后的hashCode值
        return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
    }

```

这个函数被称之为扰动函数，可以发现这个函数并没有简单的返回了key的hashCode的值，而是进行了干扰，干扰的细节不进行分析，其原理就是将key的hashCode值的高位变相的添加到低位去，然后增加key的hash值的随机性来减少hash冲突。为什么要将高位添加到低位呢？这是因为在HashMap中取桶下标的方式是通过 `hash&(桶.size-1)`来替代模操作，而位操作的时候hashCode只有低位参与位运算。

讲完hash，我们回到putVal方法上（在批量增加元素的构造函数中遍历添加也会调用这个函数）

```java
    final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
                   boolean evict) {        
        Node<K,V>[] tab; Node<K,V> p; int n, i;
		//如果当前的哈希桶是空的，则表示当前为首次初始化
        if ((tab = table) == null || (n = tab.length) == 0)
			//扩容操作
            n = (tab = resize()).length;

		//没有发生哈希碰撞，直接构建一个新的结点，然后放在指定的位置
        if ((p = tab[i = (n - 1) & hash]) == null)
            tab[i] = newNode(hash, key, value, null);
		//发生哈希冲突，链地址法解决哈希冲突
        else {
			
            Node<K,V> e; K k;
			//如果key的哈希值相同，key也相同，则进行覆盖value操作
            if (p.hash == hash &&
                ((k = p.key) == key || (key != null && key.equals(k))))
                e = p;
			//红黑树操作
            else if (p instanceof TreeNode)
                e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);

			//不是覆盖，则在链表末端插入一个新的结点
            else {
                for (int binCount = 0; ; ++binCount) {
				   //链表末端
                    if ((e = p.next) == null) {
						//插入一个新的结点
                        p.next = newNode(hash, key, value, null);
						//如果链表结点数>=8，则转化成红黑树
                        if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st
                            treeifyBin(tab, hash);
                        break;
                    }
					//如果找到要覆盖的结点
                    if (e.hash == hash &&
                        ((k = e.key) == key || (key != null && key.equals(k))))
                        break;
                    p = e;
                }
            }

			//value覆盖操作，如果e不为null，则需要覆盖value
            if (e != null) { 
				//覆盖结点值，并返回原来结点的value
                V oldValue = e.value;
                if (!onlyIfAbsent || oldValue == null)
                    e.value = value;
				//空实现的函数，如果是LinkedHashMap会重写该方法
                afterNodeAccess(e);
                return oldValue;
            }
        }
		//走到这，表示是添加操作

		//修改次数+1
        ++modCount;
		//更新size，判断是否需要扩容
        if (++size > threshold)
            resize();
		//空实现的函数，如果是LinkedHashMap会重写该方法
        afterNodeInsertion(evict);
        return null;
    }

```

这个方法干的事情很多，其职责就是：

- 首次初始化需要进行扩容操作
- 判断是否发生哈希冲突
  - 没有，则直接构造一个新的节点，然后放在哈希桶对应下标的位置当作链表的头结点
  - 发生哈希冲突，采用链地址法解决冲突
    - 如果头节点的key等于要添加的key，表示是修改操作，则进行覆盖value操作
    - 否则，如果是头结点是红黑树，则进行红黑树的添加操作
    - 否则，表示当前链表节点少于8，则对链表进行遍历，如果遍历中途出现节点key等于要添加的元素的key时候，表示是修改操作，跳出遍历；否则遍历到链表末端插入一个新的节点，插入后判断当前链表节点数是否达到8，达到8则进行转换成红黑树的操作
    - 进行覆盖value操作，覆盖节点值，返回原来节点的值。
- 如果是添加操作，则更新当前哈希表的大小，然后判断是否需要扩容，最后返回null

所以如果是添加操作则返回null，如果是修改操作则返回原来的value。另外扩容操作将在下文进行解析。

## 2. 批量添加元素

实际使用也很简单，也是一行代码搞定,下面的map和oldMap都是HashMap类型，并且两个key和value类型一致

> 实际使用

```java
map.putAll(oldMap)

```

然后我们看看HashMap中的putAll方法

> HashMap#putAll

```java
    //批量增加数据
    public void putAll(Map<? extends K, ? extends V> m) {
        putMapEntries(m, true);
    }

```

调用了putMapEntries，emmmm，怎么感觉这个函数似曾相识！没错，在批量添加元素的构造函数中，也调用了这个putMapEntries，不同的是在构造中传入的第二个参数是false,而在putAll中传入的是true.我们还是再次看看这个putMapEntries方法。

```java
    final void putMapEntries(Map<? extends K, ? extends V> m, boolean evict) {
        int s = m.size();
        if (s > 0) {
			//如果当前表是空的
            if (table == null) { // pre-size
                float ft = ((float)s / loadFactor) + 1.0F;
                int t = ((ft < (float)MAXIMUM_CAPACITY) ?
                         (int)ft : MAXIMUM_CAPACITY);
                if (t > threshold)
                    threshold = tableSizeFor(t);
            }
			//如果当前表已经初始化，并且m的元素数量大于阈值，则进行扩容
            else if (s > threshold)
                resize();
            //添加元素
            for (Map.Entry<? extends K, ? extends V> e : m.entrySet()) {
                K key = e.getKey();
                V value = e.getValue();
                putVal(hash(key), key, value, false, evict);
            }
        }
    }

```

所以这个批量添加元素的函数和批量添加元素的构造函数有什么区别吗？当调用putAll前，没有初始化（构造后直接调用putAll）确实没什么区别。但是当putAll前，已经调用过put或者putAll时，就不一样了，这时候因为已经初始化，所以table并不为null，故还要判断添加的元素个数是否需要进行扩容（扩容操作后续解析），然后才遍历oldMap添加元素。

# 五、扩容

扩容操作可谓是HashMap的精髓，在了解这个操作前，我们首先需要知道这个神秘的扩容究竟在什么场合下会出现。

## 1. 触发扩容的情况

从上面的分析中，我们得知触发扩容有三种情况：

**1.首次初始化，有可能是第一个put操作或者第一个putAll操作，也有可能是使用批量添加元素的构造函数**

**2.已经初始化，putAll批量添加元素，增加元素的总个数大于阈值**

**3.已经初始化，putVal添加一个节点后，节点个数大于阈值**

> 注：putVal包括了添加一个元素和批量添加元素的情况，因为批量添加元素也会调用putVal

然后接着看具体扩容方法

## 2. 扩容方法

由于扩容方法太长，这里将分解成两部分进行讲解

```java
    final Node<K,V>[] resize() {
    	//当前哈希桶
        Node<K,V>[] oldTab = table;
		//当前桶的容量，如果首次初始化，则当前桶的容量为0
        int oldCap = (oldTab == null) ? 0 : oldTab.length;
		//当前的阈值
        int oldThr = threshold;
		//扩容后新的容量和新的阈值
        int newCap, newThr = 0;
        
        
		//1.构造新的哈希桶
		.....
		
		//2.合并哈希桶
		.....
        return newTab;
    }

```

### 2.1 构造新的哈希桶

> HashMap#resize

```java
	    .....
		//1.如果当前桶的容量大于0，则是触发的情况2或情况3
        if (oldCap > 0) {
            //边界处理
            if (oldCap >= MAXIMUM_CAPACITY) {
                threshold = Integer.MAX_VALUE;
                return oldTab;
            }
			//否则，则设置新的容量为当前的容量的两倍
            else if ((newCap = oldCap << 1) < MAXIMUM_CAPACITY &&
                     oldCap >= DEFAULT_INITIAL_CAPACITY)
                //如果当前的容量达到16的话，新的阈值也等于旧的阈值的2倍    
                newThr = oldThr << 1; // double threshold
        }

		//2.当前的阈值大于0，只能是1情况并且使用了指定容量的构造函数
        else if (oldThr > 0) 
            newCap = oldThr;
		//3.还是情况1，并且使用的是无参数的构造函数
        else {
			//新的容量为默认容量16
            newCap = DEFAULT_INITIAL_CAPACITY;
			//新的阈值=默认容量*默认加载因子，即新的阈值为16*0.75=12
            newThr = (int)(DEFAULT_LOAD_FACTOR * DEFAULT_INITIAL_CAPACITY);
        }

		//前面没有对新的阈值进行赋值
        if (newThr == 0) {
            float ft = (float)newCap * loadFactor;
            newThr = (newCap < MAXIMUM_CAPACITY && ft < (float)MAXIMUM_CAPACITY ?
                      (int)ft : Integer.MAX_VALUE);
        }
		//更新操作
        threshold = newThr;
        @SuppressWarnings({"rawtypes","unchecked"})
         Node<K,V>[] newTab = (Node<K,V>[])new Node[newCap];
        table = newTab;
		
		//如果以前的哈希桶有元素,合并哈希桶
		.....


```

构造新的哈希桶，首先要得得到新的容量才能构造，并且在构造的同时还得设置阈值。所以需要根据不同的情况设置新的容量和阈值，其整体流程如下：

- 如果当前桶的容量大于0，表示已经初始化，则应该是触发扩容的情况2和情况3，则设置新的容量为当前容量的两倍，如果当前容量大于16，则设置新的阈值为当前阈值的两倍（什么情况下在已经初始化后，当前容量还是小于16呢？答案是如果是使用指定了容量的构造函数这种情况）
- 否则，当前阈值大于0，表明是初始化并且使用了指定容量的构造器，将暂存在threshold的哈希桶的容量取出来，即新的容量等于当前阈值。
- 否则，表明是初始化并且没有指定容量，则设置新的容量为默认值16，新的阈值为12（新的阈值=默认容量*默认加载因子）
- 如果当前的阈值为0，则表明上述没有对新的阈值进行赋值，则新的阈值等于新的容量*加载因子
- 更新全局变量阈值threshold，接着根据新的容量构建新的哈希桶，并且赋值给全局变量table

在这里也验证了上面构造函数中提到的设置阈值threshold，其实只是暂存容量的说法。

### 2.2 合并哈希桶

```java
		 if (oldTab != null) {
			//遍历当前哈希桶
            for (int j = 0; j < oldCap; ++j) {
			    //当前的结点
                Node<K,V> e;
			    //当前哈希桶中有元素，赋值给e
                if ((e = oldTab[j]) != null) {
					//将当前哈希桶的结点置为null，便于gc
                    oldTab[j] = null;
					//如果当前链表只有一个元素（没有发生哈希碰撞）
                    if (e.next == null)
					  //则将这个元素放进新的哈希桶中
                        newTab[e.hash & (newCap - 1)] = e;
                    else if (e instanceof TreeNode)
						//如果发生哈希碰撞且结点数超过8个，转化成红黑树的情况
                        ((TreeNode<K,V>)e).split(this, newTab, j, oldCap);
					
				 //发生哈希碰撞
                    else { 
                        Node<K,V> loHead = null, loTail = null;
                        Node<K,V> hiHead = null, hiTail = null;
                        Node<K,V> next;
                        do {
                            next = e.next;
						  //等于0：当前结点的哈希值小于oldCap,故放在low位链表中
                            if ((e.hash & oldCap) == 0) {
                                if (loTail == null)
                                    loHead = e;
                                else
                                    loTail.next = e;
                                loTail = e;
                            }
						 //当前结点的哈希值大于oldCap，故放在high位链表中
                            else {
                                if (hiTail == null)
                                    hiHead = e;
                                else
                                    hiTail.next = e;
                                hiTail = e;
                            }
                        } while ((e = next) != null);
						//低位链表存放在原位置
                        if (loTail != null) {
                            loTail.next = null;
                            newTab[j] = loHead;
                        }
						//高位链表存放在新位置
                        if (hiTail != null) {
                            hiTail.next = null;
                            newTab[j + oldCap] = hiHead;
                        }
                    }
                }
            }
        }

```

从上面的一开始的判断我们可以得知，如果在已经初始化并且当前哈希表有元素的情况下就会进行合并哈希桶的操作，操作的过程如下：

- 遍历当前哈希桶
- 如果当前哈希桶的下标，即链表头节点有元素，则赋值给e
  - 首先为了便于GC需要将当前链表的头节点置为null
  - 如果当前链表只有一个节点，则表示这个链表之前并没有发生哈希冲突，所以直接位操作取下标放到新的哈希桶上（这里为什么不用判断头节点的hash值与原哈希桶容量的大小关系呢？因为是容量翻倍，如果当前链表只有一个节点，位操作取下标（模运算）后依旧会是一个节点，所以不管大小，最后的链表都只会是一个节点）
  - 否则，如果头结点是红黑树，则进行红黑树的的合并操作
  - 否则，当前链表节点小于8，则需要根据每个节点的hash值来放入到低位链表或高位链表
    - 遍历当前链表，利用位运算`e.hash & oldCap`来判断当前节点与当前容量的大小关系，如果`e.hash & oldCap=0`，则表示当前节点的hash值小于当前容量，故放入低位链表；否则，放入高位链表
    - 遍历结束，将低位链表放回原位置，将高位链表放在新位置，`新位置 = 原位置+ 当前容量（oldCap）`

我们发现这里又运用了位操作，这么做的原因就是为了提升扩容的效率。讲到这扩容就分析完了，让我们接着下一个操作！

# 六、查询

在HashMap中提供了几种查询操作，get,containsKey,containsValue,还有Java8新增的getOrDefault，接下来我们一个个进行分析

## 1. get

> HashMap#get

```java
    public V get(Object key) {
        Node<K,V> e;
        return (e = getNode(hash(key), key)) == null ? null : e.value;
    }

```

get查询几乎和put操作形影不离，从上面我们也可以发现当查询不到的时候返回null,查询到就返回value。这里实际上调用了getNode方法来进行查询

```java
    final Node<K,V> getNode(int hash, Object key) {
        Node<K,V>[] tab; Node<K,V> first, e; int n; K k;
		
        if ((tab = table) != null && (n = tab.length) > 0 &&
            (first = tab[(n - 1) & hash]) != null) {
            //链表头为查找的节点
            if (first.hash == hash && 
                ((k = first.key) == key || (key != null && key.equals(k))))
                return first;
			//链表和红黑树查找
            if ((e = first.next) != null) {
                if (first instanceof TreeNode)
                    return ((TreeNode<K,V>)first).getTreeNode(hash, key);
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

getNode的方法也很简单，流程如下：

- 根据key来通过位操作取下标，获得当前key的链表
- 如果链表头是查询的节点，则直接返回该节点
- 如果头结点是红黑树，则进行红黑树查询操作
- 否则，遍历链表，返回要查询的节点
- 如果查询不到，返回null

## 2. getOrDefault

这个方法是Java8新增的，笔者使用过一次之后，对它可谓是爱不释手，因为有个这个函数，省去了一开始的判空操作。比如有这么一个需求，我们需要计算一个int数组各个数字出现的次数。

```java
//常规操作
for(int num:nums){
    if(map.get(num) == null){
        map.put(num,0);
    }
    map.put(num,map.get(num)+1);
}

//getOrDefault
for(int num:nums){
    map.put(num,map.getOrDefault(num,0)+1);
}

```

是不是顿时爱上了这个getOrDefault，其实其内部实现也是很简单的，让我们看看其操作

```java
    public V getOrDefault(Object key, V defaultValue) {
        Node<K,V> e;
        return (e = getNode(hash(key), key)) == null ? defaultValue : e.value;
    }

```

有没有发现其实就是跟get方法是一样的，只不过在这里帮我们实现了在实际上的判空操作。所以当查询不到，返回默认值defaultValue，查询到了就返回value。

## 3. containsKey

这个方法在平时也是经常会使用到

```java
    public boolean containsKey(Object key) {
        return getNode(hash(key), key) != null;
    }

```

可以发现其内部实现其实就是跟get的实现是一样的，一样是调用了getNodet,不同的是两者的返回类型不一样。

## 4. containsValue

```java
    public boolean containsValue(Object value) {
        Node<K,V>[] tab; V v;
        if ((tab = table) != null && size > 0) {
            for (int i = 0; i < tab.length; ++i) {
                for (Node<K,V> e = tab[i]; e != null; e = e.next) {
                    if ((v = e.value) == value ||
                        (value != null && value.equals(v)))
                        return true;
                }
            }
        }
        return false;
    }

```

containsValue的实现，需要遍历哈希桶的每一个链表，然后与节点上的value匹配，如果找到value，则返回true，否则返回false。

# 七、删除

在HashMap提供了两个删除操作，都是remove，不过一个只需提供key，一个是需要提供key和value，我们在实际使用上前者应该用的比较多

## 1. 根据key删除

```java
    public V remove(Object key) {
        Node<K,V> e;
        return (e = removeNode(hash(key), key, null, false, true)) == null ?
            null : e.value;
    }

```

从上面发现当删除成功会返回删除的value，删除失败，则会返回null。通过调用removeNode来进行删除操作，这里传入的matchValue是为false，如果是true的话，则key和value都相等才能删除

```java
    final Node<K,V> removeNode(int hash, Object key, Object value,
                               boolean matchValue, boolean movable) {
        Node<K,V>[] tab; Node<K,V> p; int n, index;

		//当前哈希表不为空，并且该key对应的index的链表有结点
        if ((tab = table) != null && (n = tab.length) > 0 &&
            (p = tab[index = (n - 1) & hash]) != null) {
            //node为待删除的结点
            Node<K,V> node = null, e; K k; V v;
			//如果链表头就是要删除的结点
            if (p.hash == hash &&
                ((k = p.key) == key || (key != null && key.equals(k))))
                node = p;
            else if ((e = p.next) != null) {
				//红黑树
                if (p instanceof TreeNode)
                    node = ((TreeNode<K,V>)p).getTreeNode(hash, key);
				//链表的情况
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
			//有待删除的节点，并且matchValue为false或者删除的值相等
            if (node != null && (!matchValue || (v = node.value) == value ||
                                 (value != null && value.equals(v)))) {

				//红黑树情况，跳过
                if (node instanceof TreeNode)
                    ((TreeNode<K,V>)node).removeTreeNode(this, tab, movable);
				//链表头为待删除的节点,则更新链表头
                else if (node == p)
                    tab[index] = node.next;

				//待删除的节点不是链表头，则直接删除该节点
                else
                    p.next = node.next;
				//操作次数加1
                ++modCount;
				//更新哈希表的size
                --size;
				//LinkedHashMape回调函数
                afterNodeRemoval(node);
				//返回删除的节点
                return node;
            }
        }
        return null;
    }

```

这里的删除操作与添加增加有点类似，大概流程如下：

- 首先根据key找到哈希桶上对应下标的链表，然后找待删除的节点
- 如果待删除的节点为链表头，则直接更新链表头
- 否则，如果链表头为红黑树，则进行红黑树的删除操作
- 否则，遍历链表，找到待删除的节点，直接删除该节点
- 如果删除成功，更新操作次数和哈希表的容量，返回删除的节点
- 如果删除失败，则返回null

## 2. 根据key和value删除

```java
    public boolean remove(Object key, Object value) {
        return removeNode(hash(key), key, value, true, true) != null;
    }

```

这个删除操作与根据key删除的操作没多大区别，不同的是这里调用的removeNode中传入matchValue为true，表示只有key和value都匹配才能删除节点，并且返回类型不一致。

# 八、遍历

HashMap的遍历有很多种，这里列举了三种常见的遍历方法

## 1. for-each遍历entrySet

上面我们在批量添加元素时，在HashMap的内部，也是通过这种方式来遍历的

```java
            for (Map.Entry<? extends K, ? extends V> e : m.entrySet()) {
                K key = e.getKey();
                V value = e.getValue();
                putVal(hash(key), key, value, false, evict);
            }

```

所以我们之前看entrySet方法

> HashMap#entrySet

```java
    public Set<Map.Entry<K,V>> entrySet() {
        Set<Map.Entry<K,V>> es;
        return (es = entrySet) == null ? (entrySet = new EntrySet()) : es;
    }

```

由于是第一次调用该方法，会直接构造这个EntrySet,这个EntrySet是HashMap的内部类

> HashMap.EntrySet

```java
    final class EntrySet extends AbstractSet<Map.Entry<K,V>> {
        public final int size()                 { return size; }
        public final void clear()               { HashMap.this.clear(); }

		//获取iterator
        public final Iterator<Map.Entry<K,V>> iterator() {
            return new EntryIterator();
        }

		//最终调用了getNode
        public final boolean contains(Object o) {
            if (!(o instanceof Map.Entry))
                return false;
            Map.Entry<?,?> e = (Map.Entry<?,?>) o;
            Object key = e.getKey();
            Node<K,V> candidate = getNode(hash(key), key);
            return candidate != null && candidate.equals(e);
        }

		//最终调用了removeNode方法
        public final boolean remove(Object o) {
            if (o instanceof Map.Entry) {
                Map.Entry<?,?> e = (Map.Entry<?,?>) o;
                Object key = e.getKey();
                Object value = e.getValue();
                return removeNode(hash(key), key, value, true, true) != null;
            }
            return false;
        }
        public final Spliterator<Map.Entry<K,V>> spliterator() {
            return new EntrySpliterator<>(HashMap.this, 0, -1, 0, 0);
        }

		//for-each遍历entrySet
        public final void forEach(Consumer<? super Map.Entry<K,V>> action) {
            Node<K,V>[] tab;
            if (action == null)
                throw new NullPointerException();
            if (size > 0 && (tab = table) != null) {
                int mc = modCount;
                for (int i = 0; i < tab.length; ++i) {
                    for (Node<K,V> e = tab[i]; e != null; e = e.next)
                        action.accept(e);
                }
                if (modCount != mc)
                    throw new ConcurrentModificationException();
            }
        }
    }

```

在这个类中我们可以发现很多操作，不过这些操作很多都是直接调用HashMap的方法来实现的，然后forEach方法就是for-each遍历的实现，通过这个方法，我们大概可以得出for-each在这个类中就是会遍历哈希桶上的每个链表，然后返回链表上的节点。并且其原理其实是调用了Iterator.next,所以我们可以继续看下一个遍历方法来了解下EntrySet的Iterator

## 2. 使用Iterator迭代

> 实际使用

```java
Map<Integer, Integer> map = new HashMap<Integer, Integer>();
Iterator<Map.Entry<Integer, Integer>> iterator = map.entrySet().iterator();
while (iterator.hasNext()) {
	Map.Entry<Integer, Integer> entry = iterator.next();
	System.out.println("Key = " + entry.getKey() + ", Value = " + entry.getValue());
}

```

通过Iterator迭代，其实是调用了EntrySet的iterator方法

> HashMap.EntrySet#iterator

```java
public final Iterator<Map.Entry<K,V>> iterator() {
       return new EntryIterator();
}

```

可以发现这里其实只是返回了一个EntryIterator的对象，所以我们需要看看这个EntryIterator.

> EntryIterator

```java
final class EntryIterator extends HashIterator
    implements Iterator<Map.Entry<K,V>> {
    public final Map.Entry<K,V> next() { return nextNode(); }
}

```

在实际使用中我们通过iterator.next的方式获取到当前节点，而在底层其实调用了父类HashIterator的nextNode方法，所以我们看看父类HashIterator

> HashIterator

```java
    abstract class HashIterator {
        Node<K,V> next;        // next entry to return
        Node<K,V> current;     // current entry
        int expectedModCount;  // for fast-fail
        int index;             // current slot

        HashIterator() {
			//线程不安全，保存modCount
            expectedModCount = modCount;
            Node<K,V>[] t = table;
            current = next = null;
            index = 0;
			//next初始化时，指向哈希桶上第一个不为null的链表头
            if (t != null && size > 0) { // advance to first entry
                do {} while (index < t.length && (next = t[index++]) == null);
            }
        }

        public final boolean hasNext() {
            return next != null;
        }

 		
        final Node<K,V> nextNode() {
            Node<K,V>[] t;
            Node<K,V> e = next;
            if (modCount != expectedModCount)
                throw new ConcurrentModificationException();
            if (e == null)
                throw new NoSuchElementException();

			//依次取链表的下一个节点
            if ((next = (current = e).next) == null && (t = table) != null) {
				//如果当前链表节点遍历完了，则取哈希桶的下一个不为null的链表头
                do {} while (index < t.length && (next = t[index++]) == null);
            }
            return e;
        }

        public final void remove() {
            Node<K,V> p = current;
            if (p == null)
                throw new IllegalStateException();
            if (modCount != expectedModCount)
                throw new ConcurrentModificationException();
            current = null;
            K key = p.key;
			//最终利用removeNode删除节点
            removeNode(hash(key), key, null, false, false);
            expectedModCount = modCount;
        }
    }

```

从上面我们会发现，在调用map.entrySet().iterator()的时候，实际上会构造一个HashIterator对象，并赋值哈希桶上第一个不为null的链表头给next，iterator.hasNext的操作其实就是判断当前的next是否为null,在nextNode中next会被赋值下一个节点，返回的是当前的节点。

## 3. for-each遍历keySet和values

在实际中，有时候我们会遍历key和values的集合，一般情况下都是调用map的keySet和values

> 实际使用

```java
for (Integer key : map.keySet()) {
	System.out.println("Key = " + key);
}
for (Integer value : map.values()) {
	System.out.println("Value = " + value);
}

```

接着我们看看其内部实现

> HashMap

```java
    public Set<K> keySet() {
        Set<K> ks;
        return (ks = keySet) == null ? (keySet = new KeySet()) : ks;
    }

```

```java
    final class KeySet extends AbstractSet<K> {
        public final int size()                 { return size; }
        public final void clear()               { HashMap.this.clear(); }
        public final Iterator<K> iterator()     { return new KeyIterator(); }
        public final boolean contains(Object o) { return containsKey(o); }
        public final boolean remove(Object key) {
            return removeNode(hash(key), key, null, false, true) != null;
        }
        public final Spliterator<K> spliterator() {
            return new KeySpliterator<>(HashMap.this, 0, -1, 0, 0);
        }
        public final void forEach(Consumer<? super K> action) {
            Node<K,V>[] tab;
            if (action == null)
                throw new NullPointerException();
            if (size > 0 && (tab = table) != null) {
                int mc = modCount;
                for (int i = 0; i < tab.length; ++i) {
                    for (Node<K,V> e = tab[i]; e != null; e = e.next)
                        action.accept(e.key);
                }
                if (modCount != mc)
                    throw new ConcurrentModificationException();
            }
        }
    }

```

看来上面的代码是不是跟entrySet的实现很类似，forEach方法也是一致的，不一样的是iterator方法返回的是KeyIterator类似的

> KeyIterator

```java
    final class KeyIterator extends HashIterator
        implements Iterator<K> {
        public final K next() { return nextNode().key; }
    }

```

结果这个迭代器跟entrySet的迭代器基本是一样的，只不过这里返回的是nextNode的key。而values的遍历其实跟entrySet和keySet基本是一样的，最终都是HashIterator中的实现。有兴趣的同学可以自行阅读相关源码，这里不再进行分析。

## 4. 小结

从遍历的方法我们也可以发现，HashMap的遍历是无序的，其顺序是哈希桶从左往右，链表从上往下依次进行遍历的

# 总结

在Java8中HashMap的底层数据结构是数组，称之为哈希桶。每个桶里放的是链表，链表中的节点就是哈希表的元素。当添加元素时，采用链地址法解决哈希冲突，如果链表的节点数超过8个就会将当前链表转换成红黑树，来提高插入和查询效率。由于哈希桶是数组，所以存在扩容问题。当哈希表的容量达到阈值时或者初始化的时候，就会发生扩容。另外哈希表在实现过程中用了很多位运算替代常规操作来提高效率。

> 参考博客：
>
> - [面试必备：HashMap源码解析（JDK8）](https://juejin.im/post/599652796fb9a0249975a318#heading-14)
> - [一文读懂HashMap](https://www.jianshu.com/p/ee0de4c99f87)



