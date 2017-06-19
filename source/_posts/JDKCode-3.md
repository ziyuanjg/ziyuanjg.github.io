---
title: ArrayList源码阅读笔记(基于JDk1.8)
date: 2017-06-19 14:29:28
tags: JDKCode
---

## 概述
　　ArrayList是基于数组实现的一个有序的列表，允许添加null值.除了List接口中的方法外，还实现了一些操作数组元素和大小的方法，具体下面会列举。每个ArrayList实例都有一个元素数，即size属性，这个属性标注了在实例中包含了多少个元素，这些元素存储在elementData这个数组中，当实例的容量不足时，会调用grow方法进行扩容，通常情况下每次扩大一半容量，但是有两种特殊情况，一种是手动指定的容量大于当前容量的1.5倍时会按照指定容量扩容，另一种是当前容量的1.5倍大于MAX_ARRAY_SIZE这个值的时候会扩容至Integer.MAX_VALUE或MAX_ARRAY_SIZE。

<!-- more -->
## 关键常量
|常量名|含义|
|:--|:--|
|private static final int DEFAULT_CAPACITY = 10							|当没有其他参数影响数组大小时的默认数组大小|
|private static final Object[] EMPTY_ELEMENTDATA = {}					|如果elementData用这个变量初始化，则DEFAULT_CAPACITY不会参与数组大小的运算|
|private static final Object[] DEFAULTCAPACITY_EMPTY_ELEMENTDATA = {}	|如果elementData用这个变量初始化，则DEFAULT_CAPACITY会参与数组大小的运算,只有ArrayList()中有调用|
|transient Object[] elementData 										|实际存储数据的数组|
|private int size 														|数组大小|
|private static final int MAX_ARRAY_SIZE = Integer.MAX_VALUE - 8 		|之所以减8是因为不同的vm有不同的保留字会占用一部分容量|


## 主要方法
### 构造方法
	
	public ArrayList() {
        this.elementData = DEFAULTCAPACITY_EMPTY_ELEMENTDATA;
    }
	public ArrayList(int initialCapacity) {       //指定容量
        if (initialCapacity > 0) {        		//指定容量>0，则按照指定的容量构建数组
            this.elementData = new Object[initialCapacity];
        } else if (initialCapacity == 0) {        //指定容量=0，则构造默认空数组
            this.elementData = EMPTY_ELEMENTDATA;
        } else {
            throw new IllegalArgumentException("Illegal Capacity: "+initialCapacity);
        }
    }
	public ArrayList(Collection<? extends E> c) {      //通过一个集合构造数组
        elementData = c.toArray();
        if ((size = elementData.length) != 0) {        //参数集合不为空，则将实例构造成Object[]
            if (elementData.getClass() != Object[].class)
                elementData = Arrays.copyOf(elementData, size, Object[].class);
        } else {        							   //参数集合为空，则将实例构造成默认空数组
            this.elementData = EMPTY_ELEMENTDATA;
        }
    }

### 新增
	
	public boolean add(E e) {
        ensureCapacityInternal(size + 1);      //扩容操作
        elementData[size++] = e;
        return true;
    }
	public void add(int index, E element) {　　//添加元素到指定位置
        rangeCheckForAdd(index);    		   //检查index是否越界

        ensureCapacityInternal(size + 1); 
        System.arraycopy(elementData, index, elementData, index + 1,size - index);
        elementData[index] = element;
        size++;
    }
    public boolean addAll(Collection<? extends E> c) {    //将参数集合添加至实例
        Object[] a = c.toArray();
        int numNew = a.length;
        ensureCapacityInternal(size + numNew);  
        System.arraycopy(a, 0, elementData, size, numNew);
        size += numNew;
        return numNew != 0;
    }
	 public boolean addAll(int index, Collection<? extends E> c) {    //将参数集合添加至实例指定位置
        rangeCheckForAdd(index);    			//检查index是否越界

        Object[] a = c.toArray();
        int numNew = a.length;
        ensureCapacityInternal(size + numNew);  // Increments modCount

        int numMoved = size - index;        	//这里要注意，插入前会将原数组从index位置分开
        if (numMoved > 0)
            System.arraycopy(elementData, index, elementData, index + numNew,numMoved);

        System.arraycopy(a, 0, elementData, index, numNew);
        size += numNew;
        return numNew != 0;
    }

### 覆盖
	
	public E set(int index, E element) {        
		//新增会将index位置之后的元素依次后移，而覆盖会将index位置的元素替换，对其他位置的元素没有影响
        rangeCheck(index);

        E oldValue = elementData(index);
        elementData[index] = element;
        return oldValue;
    }

### 读取

	public E get(int index) {
        rangeCheck(index);

        return elementData(index);
    }

### 删除

	public E remove(int index) {        
        rangeCheck(index);

        modCount++;
        E oldValue = elementData(index);

        int numMoved = size - index - 1;
        if (numMoved > 0)
            System.arraycopy(elementData, index+1, elementData, index,        //删除元素之后会将index之后的元素依次向前一位
                             numMoved);
        elementData[--size] = null; // clear to let GC do its work

        return oldValue;
    }
	public boolean remove(Object o) {
        if (o == null) {
            for (int index = 0; index < size; index++)        //遍历数组直到找到指定元素并删除
                if (elementData[index] == null) {
                    fastRemove(index);
                    return true;
                }
        } else {
            for (int index = 0; index < size; index++)        //遍历数组直到找到指定元素并删除
                if (o.equals(elementData[index])) {
                    fastRemove(index);
                    return true;
                }
        }
        return false;
    }

### 清空

	public void clear() {
	    modCount++;

	    for (int i = 0; i < size; i++)
	        elementData[i] = null;

	    size = 0;
	}   

### 扩容

	private void ensureCapacityInternal(int minCapacity) {
        if (elementData == DEFAULTCAPACITY_EMPTY_ELEMENTDATA) {    //当使用第一种构造方法时，第一次执行扩容方法会进入这个分支，将容量扩展为默认大小（10）
            minCapacity = Math.max(DEFAULT_CAPACITY, minCapacity);
        }
        ensureExplicitCapacity(minCapacity);
    }

    private void ensureExplicitCapacity(int minCapacity) {
        modCount++;
        if (minCapacity - elementData.length > 0)    			   //如果参数长度大于数组长度则会扩容
            grow(minCapacity);
    }
	private void grow(int minCapacity) {    
        int oldCapacity = elementData.length;
        int newCapacity = oldCapacity + (oldCapacity >> 1);        //默认每次增长为1.5倍
        if (newCapacity - minCapacity < 0)        				   //特例1：参数大于数组长度的1.5倍，此时按照参数扩容
            newCapacity = minCapacity;
        if (newCapacity - MAX_ARRAY_SIZE > 0)    				   //特例2：数组长度的1.5倍大于MAX_ARRAY_SIZE，此时将数组扩容至MAX_ARRAY_SIZE 或Integer.MAX_VALUE
            newCapacity = hugeCapacity(minCapacity);
        elementData = Arrays.copyOf(elementData, newCapacity);
    }

### Fail-Fast机制
ArrayList的迭代器也具备快速失败机制，具体是通过checkForComodification()进行控制：

	final void checkForComodification() {
        if (modCount != expectedModCount)
            throw new ConcurrentModificationException();
    }

如果并发线程对实例进行增删操作，则迭代器会抛出异常以防止在不确定的时间发生某种行为带来的未知风险。