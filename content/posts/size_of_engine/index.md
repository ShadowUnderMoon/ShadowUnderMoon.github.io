---
title: "java Size Of Engine"
author: "爱吃芒果"
description:
date: "2025-06-22T20:20:36+08:00"
image:
math:
license:
hidden: false
comments: true
draft: false
tags:
    - "Java"
categories:
    - "Java"
---



ehcache3提供了限制缓存容量的选择，如果堆上存储的容量超过了指定的大小，则会驱逐缓存中的元素，直到有空间可以容纳为止。

自己实现的sizeOfEngine应该实现ehcache3提供的引擎接口。

```java
public interface SizeOfEngine {

  /**
   * Size of the objects on heap including the overhead
   *
   * @param key key to be sized
   * @param holder value holder to be sized
   * @return size of the objects on heap including the overhead
   * @throws LimitExceededException if a configured limit is breached
   */
  <K, V> long sizeof(K key, Store.ValueHolder<V> holder) throws LimitExceededException;

}
```

ValueHolder包含value以及有关的元数据，比如创建时间、过期时间等。

```java
public class DefaultSizeOfEngine implements org.ehcache.core.spi.store.heap.SizeOfEngine {

  private final long maxObjectGraphSize;
  private final long maxObjectSize;
  private final SizeOf sizeOf;
  private final long chmTreeBinOffset;
  private final long onHeapKeyOffset;

  public DefaultSizeOfEngine(long maxObjectGraphSize, long maxObjectSize) {
    this.maxObjectGraphSize = maxObjectGraphSize;
    this.maxObjectSize = maxObjectSize;
    this.sizeOf = SizeOf.newInstance(new SizeOfFilterSource(true).getFilters());
    this.onHeapKeyOffset = sizeOf.deepSizeOf(new CopiedOnHeapKey<>(new Object(), new IdentityCopier<>()));
    this.chmTreeBinOffset = sizeOf.deepSizeOf(ConcurrentHashMap.FAKE_TREE_BIN);
  }

  @Override
  public <K, V> long sizeof(K key, Store.ValueHolder<V> holder) throws org.ehcache.core.spi.store.heap.LimitExceededException {
    try {
      return sizeOf.deepSizeOf(new EhcacheVisitorListener(maxObjectGraphSize, maxObjectSize), key, holder) + this.chmTreeBinOffset + this.onHeapKeyOffset;
    } catch (VisitorListenerException e) {
      throw new org.ehcache.core.spi.store.heap.LimitExceededException(e.getMessage());
    }
  }

}
```

我们在计算对象占用时，经常需要忽略一些对象，ehcache3提供了Filter接口来排除对象。

```java

/**
 * Filters all the sizing operation performed by a SizeOfEngine instance
 *
 * @author Alex Snaps
 */
public interface Filter {

    /**
     * Adds the class to the ignore list. Can be strict, or include subtypes
     *
     * @param clazz  the class to ignore
     * @param strict true if to be ignored strictly, or false to include sub-classes
     */
  	// 忽略类
    void ignoreInstancesOf(final Class clazz, final boolean strict);

    /**
     * Adds a field to the ignore list. When that field is walked to by the SizeOfEngine, it won't navigate the graph further
     *
     * @param field the field to stop navigating the graph at
     */
  	// 忽略字段
    void ignoreField(final Field field);

}

```



```java
/**
 * Filter to filter types or fields of object graphs passed to a SizeOf engine
 *
 * @author Chris Dennis
 * @see org.ehcache.sizeof.SizeOf
 */
public interface SizeOfFilter {

    /**
     * Returns the fields to walk and measure for a type
     *
     * @param klazz  the type
     * @param fields the fields already "qualified"
     * @return the filtered Set
     */
    Collection<Field> filterFields(Class<?> klazz, Collection<Field> fields);

    /**
     * Checks whether the type needs to be filtered
     *
     * @param klazz the type
     * @return true, if to be filtered out
     */
    boolean filterClass(Class<?> klazz);
}
```



```java
/**
 * Will Cache already visited types
 */
private class CachingSizeOfVisitor implements ObjectGraphWalker.Visitor {
    private final WeakIdentityConcurrentMap<Class<?>, Long> cache = new WeakIdentityConcurrentMap<>();

    public long visit(final Object object) {
        Class<?> klazz = object.getClass();
        Long cachedSize = cache.get(klazz);
        if (cachedSize == null) {
            if (klazz.isArray()) {
                return sizeOf(object);
            } else {
                long size = sizeOf(object);
                cache.put(klazz, size);
                return size;
            }
        } else {
            return cachedSize;
        }
    }
}
```



实现里有一个非常常用的数据结构 IdentityHashMap
