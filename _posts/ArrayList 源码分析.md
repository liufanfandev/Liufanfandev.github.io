---
layout: post
title: Java 集合之ArrayList 源码分析
categories: 源码
description: 
keywords: 源码，集合, Java
---
### 简介
* ArrayList 是一个数组队列，相当于动态数组。继承于AbstractList，实现了List,RandomAccess, Cloneable, java.io.Serializable这些接口。 

* ArrayList 允许所有的 list 操作，并允许存储所有的元素，包括 null。  

* ArrayList 通过改变数组的大小来存储更多的元素，该类和 Vector 特别像，主要区别是 ArrayList 在多线程中是不安全的。

 
### 源码解读  
#### 构造函数
```
    private static final long serialVersionUID = 8683452581122892189L;

    /**
     * 默认的初始容量
     */
    private static final int DEFAULT_CAPACITY = 10;

    /**
     * 初始化默认为空的数组
     */
    private static final Object[] EMPTY_ELEMENTDATA = {};

    /**
     * 用来标记第一个元素添加的时候怎么扩容
     */
    private static final Object[] DEFAULTCAPACITY_EMPTY_ELEMENTDATA = {};

    /**
     * 保存ArrayList中数据的数组
     */
    transient Object[] elementData; // non-private to simplify nested class access

    /**
     *  ArrayList中实际数据的数量
     */
    private int size;

    /**
     * ArrayList带容量大小的构造函数。
     *
     */
    public ArrayList(int initialCapacity) {
        if (initialCapacity > 0) {
            this.elementData = new Object[initialCapacity];
        } else if (initialCapacity == 0) {
            this.elementData = EMPTY_ELEMENTDATA;
        } else {
            throw new IllegalArgumentException("Illegal Capacity: "+
                                               initialCapacity);
        }
    }

    /**
     * 没有构造参数的时候默认初始化一个大小为 10 的数组
     */
    public ArrayList() {
        this.elementData = DEFAULTCAPACITY_EMPTY_ELEMENTDATA;
    }

    /**
     * 创建一个包含 collection 的ArrayList
     */
    public ArrayList(Collection<? extends E> c) {
        elementData = c.toArray();
        if ((size = elementData.length) != 0) {
            // c.toArray might (incorrectly) not return Object[] (see 6260652)
            if (elementData.getClass() != Object[].class)
                elementData = Arrays.copyOf(elementData, size, Object[].class);
        } else {
            this.elementData = EMPTY_ELEMENTDATA;
        }
    }

    /**
     * 将当前容量值设为实际的元素个数
     */
    public void trimToSize() {
        modCount++;
        if (size < elementData.length) {
            elementData = (size == 0)
              ? EMPTY_ELEMENTDATA
              : Arrays.copyOf(elementData, size);
        }
    }
```  
构造函数 ArrayList(Collection<? extends E> c) 中：  
* 先用 Collection.toArray() 转成数组，然后赋值给 elementData。
* 如果存在类型不相等的情况，则用 Arrays.copyOf() 来重新赋值。Arrays.copyOf() 根据 class 类型来生成新数组。  

#### 增加元素方法
* **add(E e)**  
```
    public boolean add(E e) {
        ensureCapacityInternal(size + 1);  // Increments modCount!!
        elementData[size++] = e;
        return true;
    }
    
    private void ensureCapacityInternal(int minCapacity) {
        ensureExplicitCapacity(calculateCapacity(elementData, minCapacity));
    }
    
     private static int calculateCapacity(Object[] elementData, int minCapacity) {
        if (elementData == DEFAULTCAPACITY_EMPTY_ELEMENTDATA) {
            return Math.max(DEFAULT_CAPACITY, minCapacity);
        }
        return minCapacity;
    }
    
    private void ensureExplicitCapacity(int minCapacity) {
        modCount++;

        // overflow-conscious code
        if (minCapacity - elementData.length > 0)
            grow(minCapacity);
    }
    
    private void grow(int minCapacity) {
        // overflow-conscious code
        int oldCapacity = elementData.length;
        int newCapacity = oldCapacity + (oldCapacity >> 1);
        if (newCapacity - minCapacity < 0)
            newCapacity = minCapacity;
        if (newCapacity - MAX_ARRAY_SIZE > 0)
            newCapacity = hugeCapacity(minCapacity);
        // minCapacity is usually close to size, so this is a win:
        elementData = Arrays.copyOf(elementData, newCapacity);
    }

    private static int hugeCapacity(int minCapacity) {
        if (minCapacity < 0) // overflow
            throw new OutOfMemoryError();
        return (minCapacity > MAX_ARRAY_SIZE) ?
            Integer.MAX_VALUE :
            MAX_ARRAY_SIZE;
    }
    
    private static final int MAX_ARRAY_SIZE = Integer.MAX_VALUE - 8;
```  
 
1. 首先调用 ensureCapacityInternal(int minCapacity) 函数
2. 判断 ArrayList 如果使用默认构造函数生成的，则取默认初始值 10 和 minCapacity 中较大的。  
3. modCount 用来记录ArrayList修改的次数，是在 AbstractList 中定义的。
4. grow() 方法扩容默认是扩容 1.5 倍，如果扩容后还达不到要求则采用 size+1 之后的值
5. 如果 size+1 的值大于 MAX_ARRAY_SIZE,就取值 Integer.MAX_VALUE，反之 取值  MAX_ARRAY_SIZE  

* **add(int index, E element)**  
```
    public void add(int index, E element) {
        rangeCheckForAdd(index);

        ensureCapacityInternal(size + 1);  // Increments modCount!!
        System.arraycopy(elementData, index, elementData, index + 1,
                         size - index);
        elementData[index] = element;
        size++;
    }
    
     private void rangeCheckForAdd(int index) {
        if (index > size || index < 0)
            throw new IndexOutOfBoundsException(outOfBoundsMsg(index));
    }
```  

1. 和 add(E e) 方法相比，需要先校验索引的位置是否越界。  
2. 然后扩容
3. System.arraycopy(Object src,  int  srcPos, Object dest, int destPos,  int length)  定义是：将数组 src 从下标为 srcPos 开始拷贝，一直拷贝 length 个元素到 dest 数组中，在 dest 数组中从 destPos 开始加入先的 srcPos 数组元素。相当于将 src 集合中的 [srcPos,srcPos+length] 这些元素添加到集合 dest 中去，起始位置为 destPos  
4. 接着赋值元素，调整 size 大小
* **addAll(Collection<? extends E> c)**  
```
    public boolean addAll(Collection<? extends E> c) {
        Object[] a = c.toArray();
        int numNew = a.length;
        ensureCapacityInternal(size + numNew);  // Increments modCount
        System.arraycopy(a, 0, elementData, size, numNew);
        size += numNew;
        return numNew != 0;
    }
```  

有了前面两个 add() 方法的分析，理解改方法并不难，值得分析的地方是 c.toArray() 在转化前并没有判断空，如果传入的 Collection 为空，就会报空指针的异常。测试代码如下：  
```
public class ArrayListTest {

    public static void main(String[] args) {
        ArrayList arrayList = new ArrayList();
        arrayList.add("1");
        arrayList.add("2");
        System.out.println(arrayList.toString());

        ArrayList arrayList1 = null;
        arrayList.addAll(arrayList1);
        System.out.println(arrayList.toString());
    }
}
```  
运行结果如下：  

![image](https://note.youdao.com/yws/api/personal/file/27DDB33B9C224F8D92E4CBDB7F3EC7BD?method=download&shareKey=32f52057de8d08414e21b2c90c9ef7fb)  

#### 删除元素方法
* **remove(Object o)**  
```
    public boolean remove(Object o) {
        if (o == null) {
            for (int index = 0; index < size; index++)
                if (elementData[index] == null) {
                    fastRemove(index);
                    return true;
                }
        } else {
            for (int index = 0; index < size; index++)
                if (o.equals(elementData[index])) {
                    fastRemove(index);
                    return true;
                }
        }
        return false;
    }

    private void fastRemove(int index) {
        modCount++;
        int numMoved = size - index - 1;
        if (numMoved > 0)
            System.arraycopy(elementData, index+1, elementData, index,
                             numMoved);
        elementData[--size] = null; // clear to let GC do its work
    }
```  
删除方法也比较容易理解，无论元素是否为空，都是通过循环的方式来快速删除元素（省掉了越界检查的步骤）。编写测试程序，如果list 中耗油相同的元素，只会删除 list 中的第一个元素。测试程序如下：  
```
public class ArrayListTest {

    public static void main(String[] args) {
        ArrayList arrayList = new ArrayList();
        arrayList.add("1");
        arrayList.add("2");
        arrayList.add(null);
        arrayList.add("3");
        arrayList.add(null);
        arrayList.add("1");
        System.out.println(arrayList.toString());
        arrayList.remove(null);
        System.out.println(arrayList.toString());
       
        ArrayList arrayList1 = new ArrayList();
        arrayList1.add("1");
        arrayList1.add("2");
        arrayList1.add(null);
        arrayList1.add("3");
        arrayList1.add(null);
        arrayList1.add("1");
        System.out.println(arrayList1.toString());
        arrayList1.remove("1");
        System.out.println(arrayList1.toString());
    }
}  
```
测试结果如下：  
![image](https://note.youdao.com/yws/api/personal/file/D1B54A074E364BB58698F616DE46333C?method=download&shareKey=20ee41a61bc3bb907f00120fcd2e855a)

* **remove(int index)**  
```
    public E remove(int index) {
        rangeCheck(index);

        modCount++;
        E oldValue = elementData(index);

        int numMoved = size - index - 1;
        if (numMoved > 0)
            System.arraycopy(elementData, index+1, elementData, index,
                             numMoved);
        elementData[--size] = null; // clear to let GC do its work

        return oldValue;
    }
```  
* **removeAll(Collection<?> c)**
```
    public boolean removeAll(Collection<?> c) {
        Objects.requireNonNull(c);
        return batchRemove(c, false);
    } 
    
    private boolean batchRemove(Collection<?> c, boolean complement) {
        final Object[] elementData = this.elementData;
        int r = 0, w = 0;
        boolean modified = false;
        try {
            for (; r < size; r++)
                if (c.contains(elementData[r]) == complement)
                    elementData[w++] = elementData[r];
        } finally {
            // Preserve behavioral compatibility with AbstractCollection,
            // even if c.contains() throws.
            if (r != size) {
                System.arraycopy(elementData, r,
                                 elementData, w,
                                 size - r);
                w += size - r;
            }
            if (w != size) {
                // clear to let GC do its work
                for (int i = w; i < size; i++)
                    elementData[i] = null;
                modCount += size - w;
                size = w;
                modified = true;
            }
        }
        return modified;
    }
    
    public boolean contains(Object o) {
        return indexOf(o) >= 0;
    }
    
     public int indexOf(Object o) {
        if (o == null) {
            for (int i = 0; i < size; i++)
                if (elementData[i]==null)
                    return i;
        } else {
            for (int i = 0; i < size; i++)
                if (o.equals(elementData[i]))
                    return i;
        }
        return -1;
    }
```  
1. 对传入的集合 C 进行判断是否为空  
2. 定义局部变量elementData、r、w、modified， elementData用来重新指向成员变量elementData，用来存储最终过滤后的元素，w用来纪录过滤之后集合中元素的个数，modified用来返回这次是否有修改集合中的元素  
3. 循环遍历集合，发现不是需要移除的元素，则存入 elementData 中。  
4. 如果发生异常，会出现 r != size 的情况，则将索引 r 之后的数组赋值到新的 elementData 中。  
5. w != size 说明数组中有元素被移除了，将数组用 null 补齐置空，等待 GC 回收。  
#### 修改元素方法
* **removeAll(Collection<?> c)**  
```
    public E set(int index, E element) {
        rangeCheck(index);

        E oldValue = elementData(index);
        elementData[index] = element;
        return oldValue;
    }
```  
首先判断是否越界，然后根据索引取出旧值，赋予新值，返回旧值  

#### 查询元素方法
* **get(int index)** 
```
    public E get(int index) {
        rangeCheck(index);

        return elementData(index);
    }
```
查询元素的方法比较简单，首先判断越界，然后就可以取出元素的值  

#### 清空元素方法
* **clear()** 
```
public void clear() {
        modCount++;

        // clear to let GC do its work
        for (int i = 0; i < size; i++)
            elementData[i] = null;

        size = 0;
    }
```
该方法是用一个 for 循环将所有的元素置空，然后改变 size 的大小  

#### 遍历集合的元素（迭代器）
* **iterator()**  
```
    public Iterator<E> iterator() {
        return new Itr();
    }


    private class Itr implements Iterator<E> {
        int cursor;       // index of next element to return
        int lastRet = -1; // index of last element returned; -1 if no such
        int expectedModCount = modCount;

        Itr() {}

        public boolean hasNext() {
            return cursor != size;
        }

        @SuppressWarnings("unchecked")
        public E next() {
            checkForComodification();
            int i = cursor;
            if (i >= size)
                throw new NoSuchElementException();
            Object[] elementData = ArrayList.this.elementData;
            if (i >= elementData.length)
                throw new ConcurrentModificationException();
            cursor = i + 1;
            return (E) elementData[lastRet = i];
        }

        public void remove() {
            if (lastRet < 0)
                throw new IllegalStateException();
            checkForComodification();

            try {
                ArrayList.this.remove(lastRet);
                cursor = lastRet;
                lastRet = -1;
                expectedModCount = modCount;
            } catch (IndexOutOfBoundsException ex) {
                throw new ConcurrentModificationException();
            }
        }

        //lambda 中使用
        @Override
        @SuppressWarnings("unchecked")
        public void forEachRemaining(Consumer<? super E> consumer) {
            Objects.requireNonNull(consumer);
            final int size = ArrayList.this.size;
            int i = cursor;
            if (i >= size) {
                return;
            }
            final Object[] elementData = ArrayList.this.elementData;
            if (i >= elementData.length) {
                throw new ConcurrentModificationException();
            }
            while (i != size && modCount == expectedModCount) {
                consumer.accept((E) elementData[i++]);
            }
            // update once at end of iteration to reduce heap write traffic
            cursor = i;
            lastRet = i - 1;
            checkForComodification();
        }

        final void checkForComodification() {
            if (modCount != expectedModCount)
                throw new ConcurrentModificationException();
        }
    }
```  
1. 获取 ArrayList 的迭代器时，会生成一个 Itr 对象。 
2. Itr 类中游标 cursor 用来记录下一个元素的位置，lastRet 用来记录上一次操作的元素的位置。  
3. expectedModCount 的设计和 fail-fast 事件有关，在实例化一个 Itr 对象的时候，会将 modCount 赋于 expectedModCount，如果两个线程操作一个集合，线程 A 中用迭代器遍历集合，线程 B 对集合进程 add() 和 remove() 操作， modCount 的值改变，在调用checkForComodification() 方法时会报 ConcurrentModificationException() 异常，所以说 ArrayList 是线程不安全的。如果有线程安全的需求可以使用 CopyOnWriteArrayList 来解决，CopyOnWriteArrayList 内部在进行 remove() 和 add() 操作时都用 ReentrantLock 来进行锁操作。

* **listIterator()**  
```
    public ListIterator<E> listIterator() {
        return new ListItr(0);
    }
    
    private class ListItr extends Itr implements ListIterator<E> {
        ListItr(int index) {
            super();
            cursor = index;
        }

        public boolean hasPrevious() {
            return cursor != 0;
        }

        public int nextIndex() {
            return cursor;
        }

        public int previousIndex() {
            return cursor - 1;
        }

        @SuppressWarnings("unchecked")
        public E previous() {
            checkForComodification();
            int i = cursor - 1;
            if (i < 0)
                throw new NoSuchElementException();
            Object[] elementData = ArrayList.this.elementData;
            if (i >= elementData.length)
                throw new ConcurrentModificationException();
            cursor = i;
            return (E) elementData[lastRet = i];
        }

        public void set(E e) {
            if (lastRet < 0)
                throw new IllegalStateException();
            checkForComodification();

            try {
                ArrayList.this.set(lastRet, e);
            } catch (IndexOutOfBoundsException ex) {
                throw new ConcurrentModificationException();
            }
        }

        public void add(E e) {
            checkForComodification();

            try {
                int i = cursor;
                ArrayList.this.add(i, e);
                cursor = i + 1;
                lastRet = -1;
                expectedModCount = modCount;
            } catch (IndexOutOfBoundsException ex) {
                throw new ConcurrentModificationException();
            }
        }
    }
```  

ListIterator 继承于 Itr,相当于 Itr 的加强版，提供了 previous()、set(E e) 和 add(E e) 等方法。  

### 总结  
1. **elementData[] 数组被 transient 修饰**  
在 ArrayList 中的 elementData 这个数组的长度是变长的,java 在扩容的时候,有一个扩容因子,也就是说这个数组的长度是大于等于 ArrayList 的长度的,我们不希望在序列化的时候将其中的空元素也序列化到磁盘中去,所以需要手动的序列化数组对象,所以使用了 transient 来禁止自动序列化这个数组。源码中 writeObject() 和 readObject()  就是支持序列化的方法。 
2. **Lambda 表达式**  
可以使用 Lambda 表达式来遍历 ArrayList 的操作，测试代码如下： 
```
package com.liufanfandev.springboot_test;

import java.util.ArrayList;
import java.util.Iterator;

/**
 * Created by xi on 2019/12/5.
 */
public class ArrayListTest {

    public static void main(String[] args) {
        /**
         * 测试 Lambda 表达式
         */
        ArrayList<String> arrayList = new ArrayList();
        arrayList.add("1");
        arrayList.add("2");
        arrayList.add(null);
        arrayList.add("3");
        arrayList.add(null);
        arrayList.add("1");
        System.out.println(arrayList.toString());

        Iterator<String> iterator = arrayList.iterator();
        arrayList.forEach(String->System.out.print(String+" "));
        //iterator.forEachRemaining(String -> System.out.print(String + " "));
        System.out.println();
        System.out.println("----我是分割线-----");

        arrayList.forEach(String->System.out.print(String+" "));
        //iterator.forEachRemaining(String -> System.out.print(String+" "));
        System.out.println();
        System.out.println("----到我这才结束----");

    }
}

``` 
运行结果如下：  
![image](https://note.youdao.com/yws/api/personal/file/FBD436716A0D4A2E9FBB113499C5099E?method=download&shareKey=ff1fb862b2da7116b69920df2d57d607)  
注释掉 forEach() 的代码，去掉 forEachRemaining() 的注释，运行结果如下：  

![image](https://note.youdao.com/yws/api/personal/file/6D776326BFE142B09AAD95C346617563?method=download&shareKey=74ccc49efae8614469344f6499a2423d)   

3. **Iterable() 和Iterator()**  
Iterable 是一个包含迭代器 Iterator() 的接口，集合类都实现该接口。  
Iterator 是遍历集合的方法，之所以出现 Iterator 是为了解决遍历集合时，从不同类型的集合类中抽象出来，从而避免我们在操作集合的时候必须要根据集合内部结构来选择我们应该如何遍历。  
如果 Collection 直接实现 Iterator 这个接口的时候，则当我们new 一个新的对象的时候，这个对象中就包含了当前迭代位置的数据（指针），当这个对象在不同的方法或者类中传递的时候，当前传递的对象的迭代的位置是不可预知的，那么我们在调用next（）方法的时候也就不知道是指到那一个元素。如果其中加上了一个reset（）方法呢？用来重置当前迭代的位置这样Collection也只能同时存在一个当前迭代位置的对象。所以不能直接选择实现 Iterator。 实现Iterable ，里面的方法 Iterator() 可以在同一个对象每次调用的时候都产生一个新的 Iterator 对象。这样多个迭代器就不会互相干扰了。



