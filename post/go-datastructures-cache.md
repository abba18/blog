---
title: "Go Datastructures详解 Cache"
date: 2018-01-10T21:54:18+08:00
draft: False
---
本文基于项目[go-datastruct](https://github.com/Workiva/go-datastructures)中的代码进行解读，如有不足，敬请留言指教。  

## cache包 基本介绍
cache包在项目介绍中其实并没有出现，实际上该cache包并非是memory-cache那一类算法的实现（我一开始就以为是这类功能的实现），而是一个简单的缺页置换算法的实现（Cache is a bounded-size in-memory cache of sized items with a configurable eviction policy），默认配置下可以认为是[LRU算法](https://en.wikipedia.org/wiki/Cache_replacement_policies#Least_Recently_Used_(LRU))的实现。  

所以现在来看下cache包对外提供的功能接口:
```
type Cache interface {
    // Get retrieves items from the cache by key.
    // If an item for a particular key is not found, its position in the result will be nil.
    Get(keys ...string) []Item

    // Put adds an item to the cache.
    Put(key string, item Item)

    // Remove clears items with the given keys from the cache
    Remove(keys ...string)

    // Size returns the size of all items currently in the cache.
    Size() uint64
}
```
这里提供了查（Get）、增（Put）、删（Remove）以及获取已使用内存大小（Size）四个接口。  

## cache包的基本使用
cache包的使用非常简单，只需要调用cache.New()并添加容量参数即可完成实例的生成，剩下的就是如同Cache interface所定义的方法调用即可。  
在完成New()函数的调用后，cache的置换策略默认为LUR。而cache本身提供两种置换策略，另外一种策略为[FIFO](https://en.wikipedia.org/wiki/Cache_replacement_policies#Last_In_First_Out_(FIFO))，可通过调用EvictionPolicy()函数切换策略。策略的定义如下：
```
const (
    // LeastRecentlyAdded indicates a least-recently-added eviction policy.
    LeastRecentlyAdded Policy = iota
    // LeastRecentlyUsed indicates a least-recently-used eviction policy.
    LeastRecentlyUsed
)
```

## cache的基本数据结构组成
```
type cache struct {
    sync.Mutex                                  // Lock for synchronizing Get, Put, Remove
    cap          uint64                         // Capacity bound
    size         uint64                         // Cumulative size
    items        map[string]*cached             // Map from keys to cached items
    keyList      *list.List                     // List of cached items in order of increasing evictability
    recordAdd    func(key string) *list.Element // Function called to indicate that an item with the given key was added
    recordAccess func(key string) *list.Element // Function called to indicate that an item with the given key was accessed
}
```
从上述结构体可以看出，主要的数据结构有:  

* sync.Mutex - 访问锁  
* cap - cache最大容量
* size - 当前使用容量
* items - 键值节点映射表，保存内存键值与数据节点映射关系
* keyList - 键值节点表，一个双向链表，cache操作时会根据淘汰策略节点产生顺序变化
* recordAdd - 数据增加时的操作函数
* recordAccess - 数据访问时的操作函数

其中items成员中键值对的值包含两个部分，一部分是数据实体Item接口类型，一部分是键值节点，是KeyList双向链表中的一部分。双向链表实际上操作的节点其实就是对应的键值，链表内容也就是键值字符串。
```
// A tuple tracking a cached item and a reference to its node in the eviction list
type cached struct {
    item    Item
    element *list.Element
}
```

## 具体实现分析
算法的实现本身主要还是依赖于标准库中的list-双向链表，不管使用的是哪种淘汰策略，若触发淘汰--也就是当内存容量不足时（调用New函数时指定的大小），就会移除链表的尾节点。因为所完成的算法类型不多，所以实现也比较简单。

### 1.初始化-New()
初始化的实现非常简单，指定容量，初始化链表以及数据表，同时指定默认淘汰策略。同时，也提供自定义cache初始化的函数参数， 只要传入CacheOption类型的函数即可。
```
// CacheOption configures a cache.
type CacheOption func(*cache)

// New returns a cache with the requested options configured.
// The cache consumes memory bounded by a fixed capacity,
// plus tracking overhead linear in the number of items.
func New(capacity uint64, options ...CacheOption) Cache {
    c := &cache{
        cap:     capacity,
        keyList: list.New(),
        items:   map[string]*cached{},
    }
    // Default LRU eviction policy
    EvictionPolicy(LeastRecentlyUsed)(c)

    for _, option := range options {
        option(c)
    }

    return c
}
```

在指定默认淘汰策略时调用的了EvictionPolicy()函数，这个函数可以根据需配置的策略返回一个CacheOption类型的函数。从EvictionPolicy()的函数实现上看，其实就是定义了cache.recordAccess()和cache.recordAdd()的实现方法，可选的实现方法分别cache.noop()和cache.record()两种方法。 

其中cache.noop()返回一个空值，什么都没做;而cache.record()所做的就仅仅是查询传入的值在内存(cache)中是否存在，如果存在则将对应的链表节点移动至链表头，否则生成链表节点并以键值为链表内容插入链表头。  

所以看回EvictionPolicy()函数的实现，可以发现实现过程非常简单粗暴
```
// EvictionPolicy sets the eviction policy to be used to make room for new items.
// If not provided, default is LeastRecentlyUsed.
func EvictionPolicy(policy Policy) CacheOption {
    return func(c *cache) {
        switch policy {
        case LeastRecentlyAdded:
            c.recordAccess = c.noop
            c.recordAdd = c.record
        case LeastRecentlyUsed:
            c.recordAccess = c.record
            c.recordAdd = c.noop
        }
    }
}
```

### 2.获取数据-Cache.Get()
Get方法主要就是查询传入键值在数据表中是否存在，支持传入多参数，若键值存在则会记录数据的大小，否则为空值，最后以Item数组的形式返回。在查询数据存在时会调用recordAccess()来处理淘汰策略的问题，对与FIFO,recordAccess()就是cache.noop()，而LUR则是cache.record()。
```
func (c *cache) Get(keys ...string) []Item {
    c.Lock()
    defer c.Unlock()

    items := make([]Item, len(keys))
    for i, key := range keys {
        cached := c.items[key]
        if cached == nil {
            items[i] = nil
        } else {
            c.recordAccess(key)
            items[i] = cached.item
        }
    }

	return items
}
```

### 3.数据插入-Cache.Put()
Put方法则是插入一个新的数据，需要传入键值以及实现Item接口的数据，在插入数据时首先会通过cache.remove()判断该键值是否存在，如果存在则会先清除该键值以及对应的数据并增加可用空间，然后调用cache.ensureCapacity()来判断当前cache的可用容量能否存入该数据，如果容量不足则会清除链表的尾节点，增加可用空间直到剩余空间能存入该数据为止，最后就是调用cache.recordAdd()以及cache.recordAccess()来处理淘汰策略，最终添加数据。
```
func (c *cache) Put(key string, item Item) {
    c.Lock()
    defer c.Unlock()

    // Remove the item currently with this key (if any)
    c.remove(key)

    // Make sure there's room to add this item
    c.ensureCapacity(item.Size())

    // Actually add the new item
    cached := &cached{item: item}
    cached.setElementIfNotNil(c.recordAdd(key))
    cached.setElementIfNotNil(c.recordAccess(key))
    c.items[key] = cached
    c.size += item.Size()
}
```

### 4.数据删除-Cache.Remove()
Remove的话就很简单了，就只是将传入的数据数组逐个调用cache.remove(),就更Put方法的调用的一样。

## 总结
cache可以说是Go Datastructures里代码比较简单易读的一个包了，同时也感觉实现比较简单粗暴。不过如果要继续添加新的淘汰策略也比较简单，只要实现了数据添加以及数据访问时对链表的操作策略，然后修改EvictionPolicy()，添加相应recordAcces()函数和recordAdd()函数的赋值，这样就完成了一种淘汰策略的编写。 

数据的定义也比较灵活，只需重新定义对应的结构，实现Item接口定义的方法即数据大小的求值，就能方便接入，降低耦合。
