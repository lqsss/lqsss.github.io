---
title: ArrayList源码分析(JDK1.8)
tag: 
- JDK
- Data struct
- List
categories: Source code
---

基于JDK1.8 ArrList源码分析
<!--more-->
## 成员变量
```java
    //序列化版本
    private static final long serialVersionUID = 8683452581122892189L;

    //默认容量为10
    private static final int DEFAULT_CAPACITY = 10;

    //EMPTY_ELEMENTDATA 默认空数组（构造函数无参）
    private static final Object[] EMPTY_ELEMENTDATA = {};

    //DEFAULTCAPACITY_EMPTY_ELEMENTDATA （构造函数有参，但为0）
    private static final Object[] DEFAULTCAPACITY_EMPTY_ELEMENTDATA = {};

    //实际存储数据的数组
    transient Object[] elementData; 

    //elementData大小
    private int size;
```

## 构造函数
```java
public ArrayList(int initialCapacity) {
        if (initialCapacity > 0) {
        //新建一个Object[]存储
            this.elementData = new Object[initialCapacity];
        } else if (initialCapacity == 0) {
            this.elementData = EMPTY_ELEMENTDATA;
        } else {
            throw new IllegalArgumentException("Illegal Capacity: "+initialCapacity);
        }
    }

    public ArrayList() {
        //默认无参构造函数
        this.elementData = DEFAULTCAPACITY_EMPTY_ELEMENTDATA;
    }


    public ArrayList(Collection<? extends E> c) {
        elementData = c.toArray();
        if ((size = elementData.length) != 0) {
            // c.toArray might (incorrectly) not return Object[] (see 6260652)
            if (elementData.getClass() != Object[].class)
                elementData = Arrays.copyOf(elementData, size, Object[].class);
        } else {
            // replace with empty array.
            this.elementData = EMPTY_ELEMENTDATA;
        }
    }
```

这里用到了Arrays.copyOf()
> Arrays.copyOf：public static int[] copyOf(int[] original,int newLength)
浅拷贝

```java
public static int[] copyOf(int[] original, int newLength) {  
        int[] copy = new int[newLength];  
        System.arraycopy(original, 0, copy, 0,  
                          Math.min(original.length, newLength));  
        return copy;  
    }  
```
在其内部创建了一个新的数组，然后调用arrayCopy()向其复制内容，返回出去。 

## 添加元素
1. add(E e) 
```java
    public boolean add(E e) {
        //首先确认是否扩容，增加modCount
        ensureCapacityInternal(size + 1);  // Increments modCount!!
        //内置数组赋值、size++
        elementData[size++] = e;
        return true;
    }
    
    private void ensureCapacityInternal(int minCapacity) {
        //如果是开始创建的空数组,minCapacity更新
        if (elementData == DEFAULTCAPACITY_EMPTY_ELEMENTDATA) {
            minCapacity = Math.max(DEFAULT_CAPACITY, minCapacity);
        }
        //modCount++,判断是否需要扩容
        ensureExplicitCapacity(minCapacity);
    }
    
    private void ensureExplicitCapacity(int minCapacity) {
        modCount++;

        // overflow-conscious code
        //如果所需的minCapacity比当前的长度长
        if (minCapacity - elementData.length > 0)
            grow(minCapacity);
    }
    
    private void grow(int minCapacity) {
        // overflow-conscious code
        int oldCapacity = elementData.length;
        //扩容1.5倍
        int newCapacity = oldCapacity + (oldCapacity >> 1);
        //如果新扩容的长度还是不够
        if (newCapacity - minCapacity < 0)
        //将新扩容长度设为最低所需minCapacity
            newCapacity = minCapacity;
        if (newCapacity - MAX_ARRAY_SIZE > 0)
        //新的容量
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
```

```java
//最大容量Integer.MAX_VALUE - 8，避免一些机器内存溢出
private static final int MAX_ARRAY_SIZE = Integer.MAX_VALUE - 8;
```

2. add(int index, E element)
```java
    public void add(int index, E element) {
        //检查越界
        rangeCheckForAdd(index);

        ensureCapacityInternal(size + 1);  // Increments modCount!!
        //调用native拷贝
        System.arraycopy(elementData, index, elementData, index+1,
                         size - index);
        elementData[index] = element;
        size++;
    }
    
    //如果越界，抛出异常
    private void rangeCheckForAdd(int index) {
        if (index > size || index < 0)
            throw new IndexOutOfBoundsException(outOfBoundsMsg(index));
    }
```

3. addAll(Collection<? extends E> c) 
```java
    public boolean addAll(Collection<? extends E> c) {
        Object[] a = c.toArray();
        int numNew = a.length;
        ensureCapacityInternal(size + numNew);// Increments modCount
        System.arraycopy(a, 0, elementData, size, numNew);
        size += numNew;
        return numNew != 0;
    }
```
---
add总结：
1. 判断是否需要扩容，将增加后的size那去判断
2. 如果需要扩容，开始默认是10，每次扩容1.5，如果扩容一半不够，就用增加后的size作为扩容后的容量,需要进行数组复制来增加容量
```java
elementData = Arrays.copyOf(elementData, newCapacity);
```
3. add涉及到具体index时，需要先进行越界判断
4. 最后为elementData赋值新元素，更新size


## 删除元素

1. remove(int index)

```java
    public E remove(int index) {
    //越界判断
        rangeCheck(index);

        modCount++;
        //获取即将删除的数据
        E oldValue = elementData(index);
        //完成删除操作，需要移动的数目
        int numMoved = size - index - 1;
        //数组往前移动
        if (numMoved > 0)
            System.arraycopy(elementData, index+1, elementData, index,numMoved);
        //更新size，null
        elementData[--size] = null; // clear to let GC do its work
        //返回删除元素
        return oldValue;
    }
    
    
    private void rangeCheck(int index) {
        if (index >= size)
            throw new IndexOutOfBoundsException(outOfBoundsMsg(index));
    }
```

2. remove(Object o)

```java
    public boolean remove(Object o) {
    //ArrayList允许为null
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
    
    //不会越界 不用判断，也不需要取出该元素。
    private void fastRemove(int index) {
        modCount++;
        int numMoved = size - index - 1;
        if (numMoved > 0)
            System.arraycopy(elementData, index+1, elementData, index,numMoved);
        elementData[--size] = null; // clear to let GC do its work
    }
```

3. removeAll(Collection<?> c)
```java
    public boolean removeAll(Collection<?> c) {
        //判断参数c是否为空
        Objects.requireNonNull(c);
        //
        return batchRemove(c, false);
    }
    
    public static <T> T requireNonNull(T obj) {
        if (obj == null)
            //抛空指针异常
            throw new NullPointerException();
        return obj;
    }
    
```

4. batchRemove
```java
private boolean batchRemove(Collection<?> c, boolean complement) {
        final Object[] elementData = this.elementData;
        //r就是已经遍历数组的当前下标，w就是删除后新数组的下标
        int r = 0, w = 0;
        boolean modified = false;
        try {
            for (; r < size; r++)
                //遍历数组，如果包含了容器里的数据就删除
                if (c.contains(elementData[r]) == complement)
                    elementData[w++] = elementData[r];
        } finally {
            // Preserve behavioral compatibility with AbstractCollection,
            // even if c.contains() throws.
            //contains()方法可能会抛出异常（null）
            if (r != size) {
            //解决办法就是将已经遍历r下标后的元素覆盖到w后
                System.arraycopy(elementData, r,
                                 elementData, w,
                                 size - r);
                w += size - r;
            }
            //有删除的元素
            if (w != size) {
                // clear to let GC do its work
                for (int i = w; i < size; i++)
                    //删除引用
                    elementData[i] = null;
                modCount += size - w;
                size = w;
                //更新成功
                modified = true;
            }
        }
        return modified;
    }
```

**add、remove操作都会更新modCount**

## 改元素
```java
    public E set(int index, E element) {
        rangeCheck(index);

        E oldValue = elementData(index);
        elementData[index] = element;
        return oldValue;
    }
    
    E elementData(int index) {
        return (E) elementData[index];
    }
```

## 查
```java
    public E get(int index) {
        rangeCheck(index);

        return elementData(index);
    }
```

## 清空
```java
    public void clear() {
        modCount++;

        // clear to let GC do its work
        for (int i = 0; i < size; i++)
            elementData[i] = null;

        size = 0;
    }
```
**clear操作也会更新modCount**

## 包含
```java
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

## iterator
```java
  private class Itr implements Iterator<E> {
        int cursor;       // index of next element to return
        int lastRet = -1; // index of last element returned; -1 if no such
        //判断修改标志位
        int expectedModCount = modCount;

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
            //更新lastRet
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
                //更新expectedModCount
                expectedModCount = modCount;
            } catch (IndexOutOfBoundsException ex) {
                throw new ConcurrentModificationException();
            }
        }

        //Fail-fast
        //判断是否被其他线程修改过,不安全的操作
        final void checkForComodification() {
            if (modCount != expectedModCount)
                throw new ConcurrentModificationException();
        }
    }
```

modCount继承来自AbstractList
expectedModCount判断修改标志位,Fail-fast机制