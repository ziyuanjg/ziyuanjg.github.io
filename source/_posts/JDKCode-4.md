---
title: LinkedList源码阅读笔记（基于JDK1.8）
date: 2017-06-19 14:29:30
tags: JDKCode
---

　　LinkedList是List接口的一个有序链表实现，存储节点是内部类Node，Node中有两个属性prev和next，负责连接前后两个元素。由于不是使用数组进行存储，所以查询需要遍历链表一半的元素（后面会解释），但是因为插入的时候只需要查询插入位置的元素，然后修改前后两个元素的对应属性即可，所以插入效率相对ArrayList较高（针对add(int index, E element)方法，add(E e)直接插入链表最后）。
<!-- more -->

### 添加

	public void addFirst(E e) {				   //添加元素到列表头
        linkFirst(e);
    }
	public void addLast(E e) {					//添加元素到列表尾
        linkLast(e);
    }
 	public boolean add(E e) {					//添加元素到列表头，会返回添加是否成功
        linkLast(e);
        return true;
    }
	public void add(int index, E element) {	   //添加元素到index位置
        checkPositionIndex(index);				//检查index是否越界

        if (index == size)						//如果index等于链表长度则插入到链表尾
            linkLast(element);
        else									  //否则插入到链表头
            linkBefore(element, node(index));
    }
	public boolean addAll(int index, Collection<? extends E> c) {	//在index位置插入集合c的全部元素
        checkPositionIndex(index);				//检查index是否越界

        Object[] a = c.toArray();
        int numNew = a.length;
        if (numNew == 0)
            return false;

        Node<E> pred, succ;
        if (index == size) {					//如果index为链表长度，则插入位置为链表尾
            succ = null;
            pred = last;
        } else {								//否则将记录index位置的前一个元素，为之后连接链表准备
            succ = node(index);
            pred = succ.prev;
        }

        for (Object o : a) {					//遍历参数集合，将元素依次插入实例集合
            @SuppressWarnings("unchecked") E e = (E) o;
            Node<E> newNode = new Node<>(pred, e, null);    
            if (pred == null)
                first = newNode;
            else
                pred.next = newNode;
            pred = newNode;
        }
　　　　//插入完毕，将原集合index之后的部分连接到链表尾
        if (succ == null) {
            last = pred;
        } else {
            pred.next = succ;
            succ.prev = pred;
        }

        size += numNew;
        modCount++;
        return true;
    }
    public E set(int index, E element) {		//修改index位置元素为element
        checkElementIndex(index);
        Node<E> x = node(index);
        E oldVal = x.item;
        x.item = element;
        return oldVal;
    }

### 取值

	public E getFirst() {    					 //获取链表头元素
        final Node<E> f = first;
        if (f == null)
            throw new NoSuchElementException();
        return f.item;
    }
    public E getLast() {    					 //获取链表尾元素
        final Node<E> l = last;
        if (l == null)
            throw new NoSuchElementException();
        return l.item;
    }
    public E get(int index) {   			 	//获取链表index位置的元素
        checkElementIndex(index);
        return node(index).item;
    }
    Node<E> node(int index) {        			//获取特定位置元素的实现方法
        // assert isElementIndex(index);

        if (index < (size >> 1)) {    		   //如果index小于链表长度的一半则从前向后遍历前一半链表
            Node<E> x = first;
            for (int i = 0; i < index; i++)
                x = x.next;
            return x;
        } else {    							//如果index大于等于链表长度的一半则从后向前遍历后一半链表
            Node<E> x = last;
            for (int i = size - 1; i > index; i--)
                x = x.prev;
            return x;
        }
    }

### 删除

	public E removeFirst() {    				   //删除链表头元素，类似pop
        final Node<E> f = first;
        if (f == null)
            throw new NoSuchElementException();
        return unlinkFirst(f);
    }
	public E removeLast() {    					//删除链表尾元素
        final Node<E> l = last;
        if (l == null)
            throw new NoSuchElementException();
        return unlinkLast(l);
    }
	private E unlinkFirst(Node<E> f) {    		 //删除链表头元素的实现方法
        // assert f == first && f != null;
        final E element = f.item;
        final Node<E> next = f.next;
        f.item = null;
        f.next = null; 							// 这两个置为空是为了方便GC
        first = next;    						  //第二个元素提置第一位，将原第一位元素踢出链表
        if (next == null)    					  //如果没有next说明删除链表头元素后链表为空，将last置为空
            last = null;
        else    								   //next不为空说明删除后链表还存在元素，将现第一位元素的prev置为空（链表头元素prev为空）
            next.prev = null;
        size--;
        modCount++;
        return element;
    }
	 private E unlinkLast(Node<E> l) {   	 	 //删除链表尾元素的实现方法
        // assert l == last && l != null;
        final E element = l.item;
        final Node<E> prev = l.prev;
        l.item = null;
        l.prev = null; 							//这两个置为空是为了方便GC
        last = prev;    						   //将倒数第二个元素置为链表尾元素，将原链表尾元素踢出链表
        if (prev == null)    					  //如果prev为空则说明删除链表尾元素后链表为空，没有已保存的元素，此时first为空
            first = null;
        else    								   //如果prev不为空，说明删除之后链表还存在元素，此时需将链表尾元素的next元素置为空（链表尾元素next为空）
            prev.next = null;
        size--;
        modCount++;
        return element;
    }
	public boolean remove(Object o) {    		   //删除指定元素实现方法
        if (o == null) {   			 			//遍历空对象
            for (Node<E> x = first; x != null; x = x.next) {
                if (x.item == null) {    		   //如果有符合的元素则删除
                    unlink(x);    
                    return true;
                }
            }
        } else {
            for (Node<E> x = first; x != null; x = x.next) {
                if (o.equals(x.item)) {    		//如果有符合的元素则删除
                    unlink(x);
                    return true;
                }
            }
        }
        return false;
    }
	public E remove(int index) {    			   //删除指定位置的元素
        checkElementIndex(index);
        return unlink(node(index));
    }
	public E remove() {    						//删除链表头元素
        return removeFirst();
    }
	E unlink(Node<E> x) {   					   //删除指定元素实现方法
        // assert x != null;
        final E element = x.item;        		  //参数元素的value，用于返回值
        final Node<E> next = x.next;    		   //参数元素的下一位元素
        final Node<E> prev = x.prev;    		   //参数元素的上一位元素

        if (prev == null) {    					//prev为空说明参数元素是链表头元素，将下一位元素提置链表头
            first = next;
        } else {    							   //将上一位元素和下一位元素相连
            prev.next = next;
            x.prev = null;
        }

        if (next == null) {    					//如果next为空说明参数元素是链表尾元素，将上一位元素置为链表尾
            last = prev;
        } else {    							   //将上一位元素和下一位元素相连
            next.prev = prev;
            x.next = null;
        }

        x.item = null;
        size--;
        modCount++;
        return element;
    }
	public boolean removeLastOccurrence(Object o) {    //找到元素出现的最后位置并删除
        if (o == null) {
            for (Node<E> x = last; x != null; x = x.prev) {
                if (x.item == null) {
                    unlink(x);
                    return true;
                }
            }
        } else {
            for (Node<E> x = last; x != null; x = x.prev) {
                if (o.equals(x.item)) {
                    unlink(x);
                    return true;
                }
            }
        }
        return false;
    }

### 迭代
LinkedList的迭代实现和其他几个集合的实现相比要强大一些，不仅可以向后遍历，也可以向前遍历，而且包含了实现fail-fast的增删改操作。

	public E next() {    								 //获取下一个元素
            checkForComodification();
            if (!hasNext())
                throw new NoSuchElementException();

            lastReturned = next;
            next = next.next;
            nextIndex++;
            return lastReturned.item;
        }
	public E previous() {        						 //获取上一个元素，需要注意的是当调用next然后调用previous时会返回同一个元素，因为调用next的时候nextIndex被加一，而previous中nextIndex减一，所以会取到同一个元素
            checkForComodification();
            if (!hasPrevious())
                throw new NoSuchElementException();

            lastReturned = next = (next == null) ? last : next.prev;
            nextIndex--;
            return lastReturned.item;
        }
	public void remove() {    							//实现了fail-fast的删除
            checkForComodification();
            if (lastReturned == null)
                throw new IllegalStateException();

            Node<E> lastNext = lastReturned.next;
            unlink(lastReturned);
            if (next == lastReturned)
                next = lastNext;
            else
                nextIndex--;
            lastReturned = null;
            expectedModCount++;
        }

        public void set(E e) {    						//实现了fail-fast的修改
            if (lastReturned == null)
                throw new IllegalStateException();
            checkForComodification();
            lastReturned.item = e;
        }

        public void add(E e) {    						//实现了fail-fast的新增
            checkForComodification();
            lastReturned = null;
            if (next == null)
                linkLast(e);
            else
                linkBefore(e, next);
            nextIndex++;
            expectedModCount++;
        }