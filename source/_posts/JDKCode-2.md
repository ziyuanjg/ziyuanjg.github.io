---
title: LinkedHashMap源码阅读笔记（基于jdk1.8）
date: 2017-06-19 14:29:18
tags: JDKCode
---

　　LinkedHashMap是HashMap的子类，很多地方都是直接引用HashMap中的方法，所以需要注意的地方并不多。关键的点就是几个重写的方法：
## Entry
　　Entry是继承与Node类，也就是LinkedHashMap与HashMap的根本区别所在，Node是链表形式，只有next与下一个元素进行连接，而Entry的链表有before和after两个连接点。
	
	static class Entry<K,V> extends HashMap.Node<K,V> {
        Entry<K,V> before, after;
        Entry(int hash, K key, V value, Node<K,V> next) {
            super(hash, key, value, next);
        }
    }
<!-- more -->
## 链表节点
　　区别是将节点变成Entry，并且按照链表方式将元素有序连接

	Node<K,V> newNode(int hash, K key, V value, Node<K,V> e) {
        LinkedHashMap.Entry<K,V> p =
            new LinkedHashMap.Entry<K,V>(hash, key, value, e);
        linkNodeLast(p);
        return p;
    }

## 红黑树节点 
　　红黑树节点类型并没有改变，也只是按照链表方式连接
	
	TreeNode<K,V> newTreeNode(int hash, K key, V value, Node<K,V> next) {
        TreeNode<K,V> p = new TreeNode<K,V>(hash, key, value, next);
        linkNodeLast(p);
        return p;
    }

　　HashMap中为LinkedHashMap留下了三个预留方法：

### 第一个比较好理解，在删除元素e的时候将e前后的元素相连。

	void afterNodeRemoval(Node<K,V> e) { // unlink
        LinkedHashMap.Entry<K,V> p =
            (LinkedHashMap.Entry<K,V>)e, b = p.before, a = p.after;
        p.before = p.after = null;
        if (b == null)
            head = a;
        else
            b.after = a;
        if (a == null)
            tail = b;
        else
            a.before = b;
    }

### 这个方法表面上看是判断是否删除链表中第一个元素，但是实际上是为重写LRU而准备的。
	
	void afterNodeInsertion(boolean evict) { // possibly remove eldest
        LinkedHashMap.Entry<K,V> first;
        if (evict && (first = head) != null && removeEldestEntry(first)) {
            K key = first.key;
            removeNode(hash(key), key, null, false, true);
        }
    }

关键就是下面这个方法，在LinkedHashMap中默认返回是false，当需要实现LRU算法的时候继承LinkedHashMap的子类重写这个方法即可，LRU内容较多，稍后会单开一贴记录。

	protected boolean removeEldestEntry(Map.Entry<K,V> eldest) {
        return false;
    }

### 第三个方法主要是将元素移到链表的最后一位，关键参数是accessOrder，这个参数只有在public LinkedHashMap(int initialCapacity, float loadFactor,boolean accessOrder) 中可以手动设置为true，其余时候都默认为false，所以这个方法实际用到的地方比较少。

	void afterNodeAccess(Node<K,V> e) { // move node to last
        LinkedHashMap.Entry<K,V> last;
        if (accessOrder && (last = tail) != e) {
            LinkedHashMap.Entry<K,V> p =
                (LinkedHashMap.Entry<K,V>)e, b = p.before, a = p.after;
            p.after = null;
            if (b == null)
                head = a;
            else
                b.after = a;
            if (a != null)
                a.before = b;
            else
                last = b;
            if (last == null)
                head = p;
            else {
                p.before = last;
                last.after = p;
            }
            tail = p;
            ++modCount;
        }
    }

accessOrder这个参数的实际意义是控制链表get的时候是按照插入顺序还是访问顺序，比如下面的例子：

	Map<String,String> m = new LinkedHashMap(16,0.75F,true);
        m.put("1", "a");
        m.put("2", "b");
        m.put("3", "c");
        m.put("4", "d");
        m.get("1");
        m.get("2");
        m.forEach((x,y)->{System.out.println(y);});

这时的输出是cdab，如果accessOrder设置为false则输出abcd。
 
　　这种链表形式其实在调用get方法的时候没有什么差别，存储方式也没有什么差别，差别在于forEach这个方法。HashMap中想要遍历所有元素只能通过三种set集合的迭代器，但是LinkedHashMap自身实现了forEach方法，而且在put的时候队元素进行了排序，所以可以直接对自身进行遍历。