# CopyOnWriteArrayList

## 简介

CopyOnWriteArrayList是JUC封装的一个安全线程操作方法

```java
public class CopyOnWriteArrayList<E>
    implements List<E>, RandomAccess, Cloneable, java.io.Serializable {
    
}
```

CopyOnWriteArrayList读取完全不加锁，写入也不会阻塞读取操作，只有写入和写入之间需要进行同步等待，读操作的性能得到大幅度提升。

## 实现原理

CopyOnWriteArrayList类的所有可变操作(add,set等)是通过底层数组创建新副本来实现的。当需要改动时，不直接操作原有数组对象，而是拷贝一个新副本，修改新副本，最后完成覆盖操作。

## 读写源码解析

1. CopyOnWriteArrayList 读取操作的实现

读取操作没有任何同步控制和锁操作，理由就是内部数组 array 不会发生修改，只会被另外一个 array 替换，因此可以保证数据安全。

```java
    /** The array, accessed only via getArray/setArray. */
    private transient volatile Object[] array;

    public E get(int index) {
        return get(getArray(), index);
    }

    @SuppressWarnings("unchecked")
    private E get(Object[] a, int index) {
        return (E) a[index];
    }

    final Object[] getArray() {
        return array;
    }
```

 2.CopyOnWriteArrayList 写入操作的实现

CopyOnWriteArrayList 写入操作 `add()` 方法在添加集合的时候加了锁，保证同步，避免多线程写的时候会 copy 出多个副本。

```java
    /**
     * Appends the specified element to the end of this list.
     *
     * @param e element to be appended to this list
     * @return {@code true} (as specified by {@link Collection#add})
     */
    public boolean add(E e) {
        final ReentrantLock lock = this.lock;
        lock.lock();  // 加锁
        try {
            Object[] elements = getArray();
            int len = elements.length;
            Object[] newElements = Arrays.copyOf(elements, len + 1);  // 拷贝新数组
            newElements[len] = e;
            setArray(newElements);
            return true;
        } finally {
            lock.unlock();  // 释放锁
        }
    }
```

