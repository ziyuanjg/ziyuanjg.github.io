---
title: HashMap源码阅读笔记（基于jdk1.8）
date: 2017-06-19 14:29:15
tags: JDKCode
---

## HashMap概述
　　HashMap是基于Map接口的一个非同步实现，此实现提供key-value形式的数据映射，支持null值。
<!-- more -->
HashMap的常量和重要变量如下：

|常量名|含义|
|:--|:--|
|DEFAULT_INITIAL_CAPACITY = 16	|Node数组的默认长度|
|MAXIMUM_CAPACITY = 1073741824	|Node数组的最大长度|
|DEFAULT_LOAD_FACTOR = 0.75F 	|负载因子，调控控件与冲突率的因数|
|TREEIFY_THRESHOLD = 8 			|链表转换为树的阈值，超过这个长度的链表会被转换为红黑树|
|UNTREEIFY_THRESHOLD = 6 		|当进行resize操作时，小于这个长度的树会被转换为链表|
|MIN_TREEIFY_CAPACITY = 64 		|链表被转换成树形的最小容量，如果没有达到这个容量只会执行resize进行扩容|
|Node<K, V>[] table 			|储存元素的数组|
|Set<Map.Entry<K, V>> entrySet 	|set数组，用于迭代元素|
|int size 						|存放元素的个数，但不等于数组的长度|
|int modCount 					|每次扩容和更改map结构的计数器|
|int threshold 					|临界值，当实际大小（容量*负载因子）超过临界值的时候，会进行扩容|
|float loadFactor 				|负载因子，默认为0.75F|

## HashMap实现原理
　　在jdk1.8中，HashMap是采用数组+链表+红黑树的形式实现。如下图：
![](/images/mybatis_29.png)

　　其中链表的实现如下：

	static class Node<K,V> implements Map.Entry<K,V> {
        final int hash;
        final K key;
        V value;
        Node<K,V> next;

        Node(int hash, K key, V value, Node<K,V> next) {
            this.hash = hash;
            this.key = key;
            this.value = value;
            this.next = next;
        }

        public final K getKey()        { return key; }
        public final V getValue()      { return value; }
        public final String toString() { return key + "=" + value; }

        public final int hashCode() {
            return Objects.hashCode(key) ^ Objects.hashCode(value);
        }

        public final V setValue(V newValue) {
            V oldValue = value;
            value = newValue;
            return oldValue;
        }

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

　　可以看到，node中包含一个next变量，这个就是链表的关键点，hash结果相同的元素就是通过这个next进行关联的。

　　接下来看看红黑树的实现：

	static final class TreeNode<K,V> extends LinkedHashMap.Entry<K,V> {
        TreeNode<K,V> parent;  // red-black tree links
        TreeNode<K,V> left;
        TreeNode<K,V> right;
        TreeNode<K,V> prev;    // needed to unlink next upon deletion
        boolean red;
        TreeNode(int hash, K key, V val, Node<K,V> next) {
            super(hash, key, val, next);
        }
　　　......
	}

　　红黑树比链表多了四个变量，parent父节点、left左节点、right右节点、prev上一个同级节点，红黑树内容较多，有兴趣的可以自行百度，不在赘述。

　　在来说说hash算法，HashMap中使用的算法如下

	static final int hash(Object key) {
        int h;
        return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
    }

　　这个hash先将key右移了16位，然后与key进行异或，这里还涉及到后面put方法中的另一次&操作，

	tab[i = (n - 1) & hash]

　　tab既是table，n是map集合的容量大小，hash是上面方法的返回值。因为通常声明map集合时不会指定大小，或者初始化的时候就创建一个容量很大的map对象，所以这个通过容量大小与key值进行hash的算法在开始的时候只会对低位进行计算，虽然容量的2进制高位一开始都是0，但是key的2进制高位通常是有值的，因此先在hash方法中将key的hashCode右移16位在与自身异或，使得高位也可以参与hash，更大程度上减少了碰撞率。

　　构造方法如下：

	public HashMap() 											//默认构造方法
	public HashMap(int initialCapacity)						  //参数为初始大小
	public HashMap(int initialCapacity, float loadFactor)		//参数为初始大小，负载因子

　　这里又涉及到另一个算法：

	static final int tableSizeFor(int cap) {
        int n = cap - 1;
        n |= n >>> 1;
        n |= n >>> 2;
        n |= n >>> 4;
        n |= n >>> 8;
        n |= n >>> 16;
        return (n < 0) ? 1 : (n >= MAXIMUM_CAPACITY) ? MAXIMUM_CAPACITY : n + 1;
    }

　　这个算法很有意思，通过给定的大小cap，计算大于等于cap的最小的2的幂数。连续5次右移运算乍一看没有什么意思，但仔细一想2进制都是0和1啊，这就有问题了，第一次右移一位，就表示但凡是1的位置右边的一位都变成了1，第二次右移两位，上次已经把有1的位置都变成连续两个1了，是不是感觉很神奇，如此下来5次运算正好将int的32位都转了个遍，以最高的一个1的位置为基准将后面所有位数都变为1，然后在进行n+1，不就变成了2的幂数。这里还有一点要注意的是第一行的cap-1，这是因为如果cap本身就是2的幂数，会出现结果是cap的2倍的情况，会浪费空间。

## HashMap的存取
### 存储

	final V putVal(int hash, K key, V value, boolean onlyIfAbsent, boolean evict) {    				   //put方法的实际执行者
			//hash，key的hash值；onlyIfAbsent，是否改变原有值；evict，LinkedHashMap回调时才会用到。
            Node<K,V>[] tab; 
			Node<K,V> p; 
			int n, i;
            if ((tab = table) == null || (n = tab.length) == 0)
            	n = (tab = resize()).length;    														  //table为空或长度为0时，对table进行初始化，分配内存
         　 if ((p = tab[i = (n - 1) & hash]) == null)
                tab[i] = newNode(hash, key, value, null);    											 //当put的key在map中不存在时，直接new一个Node存入table。
        	else {    																					//当key在map中存在时，进入此分支。
                　Node<K,V> e; K k;
                if (p.hash == hash && ((k = p.key) == key || (key != null && key.equals(k))))   		  //此处的p是table中的第n-1个元素，每一次必检查此元素的key是否与put的key相同，如果相同则替换value
                    e = p;
                else if (p instanceof TreeNode)    													   //检查p是否是TreeNode类型。
                    e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
                else {    																				//如果第n-1个元素不符合，则一次遍历table中其余元素，直到找到相应key值。
                	for (int binCount = 0; ; ++binCount) {
                		if ((e = p.next) == null) {    												   //此处主要为了防止hash出现重复，只有上边tab[i = (n - 1) & hash]中存在元素且不是put的元素时才会进入这个分支。
                        	p.next = newNode(hash, key, value, null);
                        	if (binCount >= TREEIFY_THRESHOLD - 1) 									   // 此处判断的意义是当链表长度超过8时，转换为红黑树，在1.8以前是没有的，链表查询的复杂度是O(n)，而红黑树是O(log(n))，但是如果hash结果不均匀会极大的影响性能
                        		treeifyBin(tab, hash);
                			break;
            			}
                		if (e.hash == hash && ((k = e.key) == key || (key != null && key.equals(k))))     //进入此分支标志已找到与put的key值相同的元素，元素变量为e。
                        　　break;
                		p = e;
                	}
            	}
           		if (e != null) { 																	  // 将对应key的value替换为put的value，同时返回旧value，只有存在key时才会返回！
                	V oldValue = e.value;
                	if (!onlyIfAbsent || oldValue == null)
                    	e.value = value;
                	afterNodeAccess(e);    
                	return oldValue;
           		}
        	}
        ++modCount;
        if (++size > threshold)
            resize();    																				//对table进行扩容
        afterNodeInsertion(evict);    
        return null;
    }    

　　从上面的源码可以看出，首先会判断key的hash与map容量-1的与计算值，如果数组中这个位置没有元素则直接插入，反之则在遍历此位置的元素链表，直到在最后插入。这个地方有一个链表的阈值（默认是8），如果一个链表的长度达到了阈值，则调用treefyBin()将链表转换成红黑树。

### 扩容

	final Node<K,V>[] resize() {    								  //扩容方法
        Node<K,V>[] oldTab = table;
        int oldCap = (oldTab == null) ? 0 : oldTab.length;        	//map现在的容量
        int oldThr = threshold;                            
        int newCap, newThr = 0;
        if (oldCap > 0) {
            if (oldCap >= MAXIMUM_CAPACITY) {    					 //如果现在的容量大于等于设定的最大容量则不会进行扩容
                threshold = Integer.MAX_VALUE;
                return oldTab;
            }
            else if ((newCap = oldCap << 1) < MAXIMUM_CAPACITY && oldCap >= DEFAULT_INITIAL_CAPACITY)
                newThr = oldThr << 1;         						//如果现在的容量扩大为2倍依然没有超过最大容量，并且现在的容量大于等于数组的默认长度，将map的容量扩大为2倍
        }
        else if (oldThr > 0)  										//hashMap有一个构造方法会指定初始大小，这时候就会用到这个分支，对map进行初始化
            newCap = oldThr;
        else {               										 //这个分支是默认的初始化参数分支，比如用new hashMap()生成一个对象，就会进入这个分支初始化
            newCap = DEFAULT_INITIAL_CAPACITY;
            newThr = (int)(DEFAULT_LOAD_FACTOR * DEFAULT_INITIAL_CAPACITY);
        }
        if (newThr == 0) {
            float ft = (float)newCap * loadFactor;
            newThr = (newCap < MAXIMUM_CAPACITY && ft < (float)MAXIMUM_CAPACITY ? (int)ft : Integer.MAX_VALUE);
        }
        threshold = newThr;
        @SuppressWarnings({"rawtypes","unchecked"})
        Node<K,V>[] newTab = (Node<K,V>[])new Node[newCap];
        table = newTab;
        if (oldTab != null) {        								 //扩容操作，将oldTab的元素依次添加到newTab
            for (int j = 0; j < oldCap; ++j) {
                Node<K,V> e;
                if ((e = oldTab[j]) != null) {
                    oldTab[j] = null;
                    if (e.next == null)
                        newTab[e.hash & (newCap - 1)] = e;
                    else if (e instanceof TreeNode)
                        ((TreeNode<K,V>)e).split(this, newTab, j, oldCap);
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

　　在这个方法里有几个比较有意思的地方，首先是最大容量MAXIMUN_CAPACITY，这个值实际就是int的最大值，个人推测应该是为了配合put时的hash算法，因为计算key值hash的算法返回值是int型，如果容量超过int的阈值，在进行与运算时碰撞率会增大很多倍。然后是扩容的方式，这里是new了一个数组！这也是HashMap不安全的关键之一。当超过一个线程对一个HashMap对象进行put的时候如果触发了resize方法，后执行的那一个会把之前执行的结果覆盖掉！也就是先put的值不会真的存入map中。

### 链表转换为红黑树

	final void treeifyBin(Node<K,V>[] tab, int hash) {    				 //将链表转换为红黑树
        int n, index; Node<K,V> e;
        if (tab == null || (n = tab.length) < MIN_TREEIFY_CAPACITY)    	//如果map的容量小于64（默认值），会调用resize扩容，不会转换为红黑树
            resize();
        else if ((e = tab[index = (n - 1) & hash]) != null) {
            TreeNode<K,V> hd = null, tl = null;
            do {
                TreeNode<K,V> p = replacementTreeNode(e, null);    		//Node转换为TreeNode
                if (tl == null)
                    hd = p;
                else {
                    p.prev = tl;
                    tl.next = p;
                }
                tl = p;
            } while ((e = e.next) != null);
            if ((tab[index] = hd) != null)    
                hd.treeify(tab);        								   //调用TreeNode的树排序方法
        }
    }

　　这里需要注意的是当map容量小于64时，就算链表超过了8也不会转换为红黑树。

### Map克隆

	final void putMapEntries(Map<? extends K, ? extends V> m, boolean evict) {    //主要用于clone方法和putAll方法和一个构造方法HashMap(Map<? extends K, ? extends V> m)。
        int s = m.size();
        if (s > 0) {    
            if (table == null) {     											 // 如果是一个new的map，即table变量还没有初始化，先通过参数的map.size和负载因子计算所需的map大小。
                float ft = ((float)s / loadFactor) + 1.0F;
                int t = ((ft < (float)MAXIMUM_CAPACITY) ?
                         (int)ft : MAXIMUM_CAPACITY);
                if (t > threshold)    											//如果计算出的大小大于map的实际存储容量，则重新计算存储容量
                    threshold = tableSizeFor(t);
            }
            else if (s > threshold)    										   //如果参数map大小大于实际存储容量，则将map容量变为2倍
                resize();
            for (Map.Entry<? extends K, ? extends V> e : m.entrySet()) {    	  //遍历赋值
                K key = e.getKey();
                V value = e.getValue();
                putVal(hash(key), key, value, false, evict);
            }
        }
    }

### 读取

	final Node<K,V> getNode(int hash, Object key) {    							//get的实际实现方法
        Node<K,V>[] tab;
		Node<K,V> first, e; 
		int n; K k;
        if ((tab = table) != null && (n = tab.length) > 0 && (first = tab[(n - 1) & hash]) != null) {          //通过hash定位到一个桶，并且有元素存在
            if (first.hash == hash &&  ((k = first.key) == key || (key != null && key.equals(k))))        	 //总是检查桶中的第一个元素
                return first;
            if ((e = first.next) != null) {    									//如果第一个元素不符合，则继续搜索
                if (first instanceof TreeNode)    								 //如果桶中是红黑树，进入此分支，实际执行的是TreeNode中的find方法，从根节点开始向下搜寻
                    return ((TreeNode<K,V>)first).getTreeNode(hash, key);
                do {        													   //如果是链表结构则依次遍历
                    if (e.hash == hash && ((k = e.key) == key || (key != null && key.equals(k))))
                        return e;
                } while ((e = e.next) != null);
            }
        }
        return null;
    }

　　看过put方法后再来看get就容易很多了，首先用put中的hash算法定位到数组中的位置，如果有元素且key值相同直接返回，如果有元素且key值不同那就要向下遍历了，如果是链表就一直遍历next，如果是红黑树则调用getTreeNode方法查找节点。

### 迭代器

	abstract class HashIterator {    											  //KeySet和EntrySet的迭代器的父类
        Node<K,V> next;        													// 下一个元素
        Node<K,V> current;     													// 删除方法会调用
        int expectedModCount;  													// 就是hashMap中的modCount
        int index;             													// 元素序号

        HashIterator() {    													   //为上面四个变量赋初始值
            expectedModCount = modCount;
            Node<K,V>[] t = table;
            current = next = null;
            index = 0;
            if (t != null && size > 0) { // advance to first entry
                do {} while (index < t.length && (next = t[index++]) == null);
            }
        }
		.......
	}

　　HashIterator是KerSet和EntrySet的父类，这个迭代器中有一个属性是expectedModCount要特别注意，这个变量等价于HashMap对象中的modCount，如果在迭代器没有执行完的期间对map进行增删，会抛出异常阻止继续迭代，代码如下：

	final Node<K,V> nextNode() {    											  //获取下一个元素
        Node<K,V>[] t;
        Node<K,V> e = next;
        if (modCount != expectedModCount)    									 //在迭代器执行期间只能用下面的remove删除元素，否则会抛出异常，不能增加元素
            throw new ConcurrentModificationException();
            if (e == null)    													//如果没有下一个会抛出异常
                throw new NoSuchElementException();
            if ((next = (current = e).next) == null && (t = table) != null) {     //如果next没有取到值，通过index继续向下遍历，直到取到元素为止
                do {} while (index < t.length && (next = t[index++]) == null);
            }
            return e;
        }

　　不过迭代器虽然不允许添加元素，但是给了一个删除的方法：

	public final void remove() {    											  //删除方法
            Node<K,V> p = current;
            if (p == null)
                throw new IllegalStateException();
            if (modCount != expectedModCount)
                throw new ConcurrentModificationException();
            current = null;
            K key = p.key;
            removeNode(hash(key), key, null, false, false);
            expectedModCount = modCount;    									  //关键语句，这里做了一个同步，否则与正常的删除没有区别，这个同步避免了上面的异常抛出
        }

　　与普通的删除方法就差在最后这一行上，这里同步了迭代器和HashMap对象的操作数，便不会抛出异常。