---
layout:     post
title:      ArrayList和Vector的区别
subtitle:   ArrayList和Vector的区别
date:       2018-07-29
author:     XINGXING
header-img: img/post-bg-ios9-web.jpg
catalog: true
tags:
    - Java
---

>
>ArrayList和Vector的区别
> 





**这里说下ArrayList和Vector的区别**       
先看下java util包的类结构:  
![](https://ws1.sinaimg.cn/large/006tNbRwly1fxp2rgsnfej318s0q20tn.jpg)


**这两者的区别表现在两方面:**      
第一:线程安全性，Vector是线程安全的，而ArrayList不是线程安全的   
第二:扩容: ArrayList在底层数组的容量不够用时在原来的基础上扩容0.5倍，而Vector则扩容1倍;  


**线程安全性**   
我们打开java.util.ArrayList和java.util.Vector源代码;  
我们看下两者的add方法，可见Vector的Add方法使用了synchronized修饰，实现了线程安全，而ArrayList却没有，
```java
    /**
     * ArrayList的Add方法
     * Appends the specified element to the end of this list.
     *
     * @param e element to be appended to this list
     * @return <tt>true</tt> (as specified by {@link Collection#add})
     */
    public boolean add(E e) {
        ensureCapacityInternal(size + 1);  // Increments modCount!!
        elementData[size++] = e;
        return true;
    }
//------------------------------------------------------------------------------------------
    /**
     * Vector 的add方法
     * Appends the specified element to the end of this Vector.
     *
     * @param e element to be appended to this Vector
     * @return {@code true} (as specified by {@link Collection#add})
     * @since 1.2
     */
    public synchronized boolean add(E e) {
        modCount++;
        ensureCapacityHelper(elementCount + 1);
        elementData[elementCount++] = e;
        return true;
    }    
    
```
再看下两者的remove方法，Vector依然使用了synchronized来修饰保证了线程的安全性；
```java

    /**
     * ArrayList的remove方法
     * Removes the element at the specified position in this list.
     * Shifts any subsequent elements to the left (subtracts one from their
     * indices).
     *
     * @param index the index of the element to be removed
     * @return the element that was removed from the list
     * @throws IndexOutOfBoundsException {@inheritDoc}
     */
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
    
//------------------------------------------------------------------------------------------    
    
    /** Vector的remove方法
     * Removes the element at the specified position in this Vector.
     * Shifts any subsequent elements to the left (subtracts one from their
     * indices).  Returns the element that was removed from the Vector.
     *
     * @throws ArrayIndexOutOfBoundsException if the index is out of range
     *         ({@code index < 0 || index >= size()})
     * @param index the index of the element to be removed
     * @return element that was removed
     * @since 1.2
     */
    public synchronized E remove(int index) {
        modCount++;
        if (index >= elementCount)
            throw new ArrayIndexOutOfBoundsException(index);
        E oldValue = elementData(index);

        int numMoved = elementCount - index - 1;
        if (numMoved > 0)
            System.arraycopy(elementData, index+1, elementData, index,
                             numMoved);
        elementData[--elementCount] = null; // Let gc do its work

        return oldValue;
    }

```

再看下两者的扩容方法，如下代码所示:  
```java
    /**
     * ArrayList的扩容方法
     * Increases the capacity to ensure that it can hold at least the
     * number of elements specified by the minimum capacity argument.
     *
     * @param minCapacity the desired minimum capacity
     */
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
  
//------------------------------------------------------------------------------------------    
    
    //Vector的扩容方法
    private void grow(int minCapacity) {
        // overflow-conscious code
        int oldCapacity = elementData.length;
        int newCapacity = oldCapacity + ((capacityIncrement > 0) ?
                                         capacityIncrement : oldCapacity);
        if (newCapacity - minCapacity < 0)
            newCapacity = minCapacity;
        if (newCapacity - MAX_ARRAY_SIZE > 0)
            newCapacity = hugeCapacity(minCapacity);
        elementData = Arrays.copyOf(elementData, newCapacity);
    }
```

**总结**  
无一例外，Vector只要是关键性的操作，方法前面都加了synchronized关键字，来保证线程的安全性。当执行synchronized修饰的方法前，系统会对该方法加一把锁，方法执行完成后释放锁，加锁和释放锁的这个过程，在系统中是有开销的;  
和ArrayList和Vector一样，同样的类似关系的类还有HashMap和HashTable，StringBuilder和StringBuffer，后者是前者线程安全版本的实现;








