---
title: "Go Structures Priority Queue"
date: 2018-02-03T22:55:20+08:00
draft: false
---
本文基于项目[go-datastruct](https://github.com/Workiva/go-datastructures)中的代码进行解读，如有不足，敬请留言指教。  

## 基本介绍  
这此讲解的是优先队列(priority_queue)，从整个接口提供和功能实现上一篇的队列(queue)差距不大，无非就是在入队时做根据规则做了一次排序。优先队列算法实现使用堆排序算法实现。由于与Queue是实现相似度高，本文会大量引用和对比Queue的实现。

* 注意：用本文用queue表示包模块，P_Queue表示优先队列队列实体，Queue表示队列实体。  

Queue提供以下方法:  

* Put() -- 添加值进入队列，入队操作。
* Get() -- 向队列阻塞获取某一个位置的值，出队操作。
* Peek() -- 获取队列头的值，但不会修改队列。
* Empty() -- 判断队列是否为空。
* Len() -- 获取队列长度。
* Disposed() -- 判断队列是否销毁。
* Dispose() -- 销毁队列，并返回队列里的所有内容。  

以下是全局方法:  

* NewPriorityQueue() -- 生成指定容量队列

对比Queue，所提供的方法更少，但并不缺少基本的出队、入队、销毁操作。

## P_Queue基本数据结构

```
// PriorityQueue is similar to queue except that it takes
// items that implement the Item interface and adds them
// to the queue in priority order.
type PriorityQueue struct {
	waiters         waiters                 //操作同步变量数组
	items           priorityItems           //队列元素数组，interface数组类型
	itemMap         map[Item]struct{}       //队列元素表，用来加速判断队列是否村存在该元素
	lock            sync.Mutex              //队列操作锁
	disposeLock     sync.Mutex              //队列销毁锁
	disposed        bool                    //队列销毁标志
	allowDuplicates bool                    //是否允许队列元素重复标志
}
```

对比Queue，P_Queue多了itemMap、disposedLock、allowDuplicates这三个数据结构，作用意思都比较明显，在下文实现是再具体分析。  

与Queue不同，P_Queue队列元素为Item接口数据，该接口定义的了一个Compare()方法，在堆排序时调用。
```
// Item is an item that can be added to the priority queue.
type Item interface {
	// Compare returns a bool that can be used to determine
	// ordering in the priority queue.  Assuming the queue
	// is in ascending order, this should return > logic.
	// Return 1 to indicate this object is greater than the
	// the other logic, 0 to indicate equality, and -1 to indicate
	// less than other.
	Compare(other Item) int
}
```
对于Compare()方法的返回值定义，返回1表示“调用者“比other大；返回0表示两者相等；返回-1表示调用者比other小。  

举个例子，假设有定义了一个数据结构并实现了Compare()方法
```
type MyInt{
    value int
    ...
}

(i MyInt)Compare(other MyInt) int {
    ...
}

...
a := MyInt{
    value : 1,
}

b := MyInt{
    value : 2,
}
...
```

如果优先队列要实现a比b高，那么在实现Compare方法时则需要这样：
```
(i MyInt)Compare(other MyInt) int {
    ...
    if i.value > other.value {
        return -1
    } else if  i.value < other.value {
        return 1
    } else {
        return 0
    }
}
```
如果a和b优先级需要掉转，则把返回1和-1的位置调换一下就好。  

## 具体实现分析

### 1.NewPriorityQueue() - 初始化队列
该函数实现非常简单，传入队列容量参数和是否允许元素重复参数，通过make初始化items和itemsMap，返回示例。  

### 2.P_Queue.Put() - 插入队列
该函数传入多个Item，执行完返回错误信息。  

进入该函数，首先进行参数长度检查，然后操作加锁、是否正在销毁判定。当检查完成后，会根据初始化时是否允许重复元素时做元素插入，如果不允许元素重复会使用itemsMap判断队列是否已经存在需要插入的元素，然后调用P_Queue.items.push()插入。 

插入所有元素后，通过P_Queue.waiters.get()确认在P_Queue.Put()操作之前是否有P_Queue.Get()操作。如果没有返回的same为空，Put()操作完成；如果same为不空，则存在P_Queue.Get()，并通过same.ready通知P_Queue.Get()已经有元素插入，然后P_Queue.response.Wait()等待P_Queue.Get()操作完成，直到队列的数据为空或所有的P_Queue.Get()已经完成。  

items的数据类型为priorityItems，实现元素的插入(push)、删除(pop)、获取(get)、位置交换(swap)，其实也是实现了golang版的堆的插入与删除，完成了对排序实现的优先队列。
```
type priorityItems []Item

func (items *priorityItems) swap(i, j int) {
	(*items)[i], (*items)[j] = (*items)[j], (*items)[i]
}

func (items *priorityItems) pop() Item {
	size := len(*items)

	// Move last leaf to root, and 'pop' the last item.
	items.swap(size-1, 0)
	item := (*items)[size-1] // Item to return.
	(*items)[size-1], *items = nil, (*items)[:size-1]

	// 'Bubble down' to restore heap property.
	index := 0
	childL, childR := 2*index+1, 2*index+2
	for len(*items) > childL {
		child := childL
		if len(*items) > childR && (*items)[childR].Compare((*items)[childL]) < 0 {
			child = childR
		}

		if (*items)[child].Compare((*items)[index]) < 0 {
			items.swap(index, child)

			index = child
			childL, childR = 2*index+1, 2*index+2
		} else {
			break
		}
	}

	return item
}

func (items *priorityItems) get(number int) []Item {
	returnItems := make([]Item, 0, number)
	for i := 0; i < number; i++ {
		if i >= len(*items) {
			break
		}

		returnItems = append(returnItems, items.pop())
	}

	return returnItems
}

func (items *priorityItems) push(item Item) {
	// Stick the item as the end of the last level.
	*items = append(*items, item)

	// 'Bubble up' to restore heap property.
	index := len(*items) - 1
	parent := int((index - 1) / 2)
	for parent >= 0 && (*items)[parent].Compare(item) > 0 {
		items.swap(index, parent)

		index = parent
		parent = int((index - 1) / 2)
	}
}
```
关于堆和堆排序，可以参考[这里](https://zh.wikipedia.org/wiki/%E5%A0%86_(%E6%95%B0%E6%8D%AE%E7%BB%93%E6%9E%84))和[这里](https://zh.wikipedia.org/wiki/%E5%A0%86%E6%8E%92%E5%BA%8F)。

## 3.P_Queue.Get() - 获取队列数据
该函数传入一个参数，表示获取多少个数据，执行完成后返回一个元素切片和错误信息。  

同样的，进入该函数，首先进行参数长度检查，然后操作加锁、是否正在销毁判定。当检查完成后，定义了一个匿名函数，作用就是传入队列元素切片，并删除itemsMap中的这些元素切片。  

随后判断队列长度，如果长度不为0，则直接调用P_Queue.items.get()获取队列元素，然后调用匿名函数删除出队的元素。需要注意的是，如果队列长度比所需获取的长度要短，则返回却元素切片长度为队列长度，容量为所需获取的长度，所以需要用len()判断实际返回了多少元素。如果队列长度为0，则解锁队列操作，并生成一个新的same，通过P_Queue.waiters.put()传入并通过<-same.ready阻塞等待P_Queue.Put()插入数据或队列销毁和通知，当same.ready结束等待，需要先判断队列是否进入销毁状态，然后调用P_Queue.items.get()获取队列元素，最后调用sema.response.Done()通知P_Queue.Put()完成操作。

在使用该函数时需要注意该函数很有可能会有阻塞操作，并且没有实现超时机制。

## 4.P_Queue.Peek() P_Qeueu.Empty() P_Qeueu.Len() P_Queue.Disposed() P_Queue.Dispose()
这些函数的实现比较简单并且与Queue的实现相差无几，这里不作说明。

## 总结
总体来讲，P_Queue接口实现的思路与Queue的相差并不大，无非就是比Queue少了一些如超时获取、并行消费的功能，增加了队列元素优先级。