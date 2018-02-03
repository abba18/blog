---
title: "Go Structures详解 Queue -- queue"
date: 2018-01-28T14:02:57+08:00
draft: false
---
本文基于项目[go-datastruct](https://github.com/Workiva/go-datastructures)中的代码进行解读，如有不足，敬请留言指教。  

## queue包介绍
queue包从有3个部分，分别是队列、优先队列和RingBuffer的实现，这次解析的是queue。从官方的提供的测试文件来来看，queue在使用场景上还可以对标channel，相当于无锁channel的实现，性能上而且性能还不错。  

* 注意：用本文用queue表示包模块，Queue表示队列实体

Queue提供以下的方法:  

* Put() -- 添加值进入队列，入队操作。
* Get() -- 向队列阻塞获取某一个位置的值，出队操作。
* Poll() -- 在预定时间内阻塞获取某一个位置的值，出队操作。
* Peek() -- 获取队列头的值，但不会修改队列。
* TakeUntil() -- 根据传入的值判定函数，获取一组数组，出队操作。注意当判定函数只要返回了false，出队就会停止，也就是说并不能判定整个队列，Until的意思也在此。
* Empty() -- 判断队列是否为空。
* Len() -- 获取队列长度。
* Disposed() -- 判断队列是否销毁。
* Dispose() -- 销毁队列，并返回队列里的所有内容。

以下是全局方法:  

* New() -- 生成指定容量的队列。
* ExecuteInParallel() -- 传入一个队列和处理函数，该函数会使用传入的处理函数并行消费完队列里的所有值，当该函数执行完后，队列也将变得不可用。  

## Queue基本数据结构

```
// Queue is the struct responsible for tracking the state
// of the queue.
type Queue struct {
    waiters  waiters        //操作同步变量数组
    items    items          //队列值数组，interface数组类型
    lock     sync.Mutex     //队列操作锁
    disposed bool           //销毁标志
}
```
数据结构比较简单，唯一需要说明的就是waiters结构，这个结构主要负责在边界操作时需要做一些“行为通知”。例如当队列为空时，执行了一个Get()操作，此时当前routine会阻塞，当另外一个routine成功执行了一个Put()操作后，之前routine的Get()操作就会通过waiters知道队列插入了新的数据，从而完成操作。在队列销毁时也会使用到waiters，在后续中再详细分析。  

waiter实际上是一个数组，长度与当队列为空时有多少Get()、Poll()操作在执行相关。数据结构如下:  
```
type waiters []*sema

type sema struct {
    ready    chan bool          //空队列操作消息通知，true或false没有特定意义
    response *sync.WaitGroup    //空队列操作等待
}

func newSema() *sema {
    return &sema{
        ready:    make(chan bool, 1),
        response: &sync.WaitGroup{},
    }
}
```

对于waiters，items，都实现了各自的获取的和插入等方法，为队列的实现提供底层方法，结合具体实现分析讲解。

## 具体实现分析

### 1.New() - 初始化
new()实现非常简单，传入队列的容量，通过make初始化items，然后返回Queue实例。  
### 2.ExecuteInParallel() - 并行消费
该函数主要功能是通过传入的处理函数，快速消费队列里的所有内容，高效的并行实现主要根据当前逻辑核心数来维持routine数量在一个稍微合理的数目，减少routine的切换所带来的性能损失（其实感觉过于理想，没有考虑外部情况）。  

函数执行时，会获取队列长度和逻辑核心数，最终会产生逻辑核心数-1个routine来执行处理函数，同时在routine执行前会使用WaitGroup(wg)来等待所有routine执行完成。在routine执行的过程中通过atomic.AddInt64()来标记和确认正在消费的队列节点序列号（其实就是数组下标）。若当前的序列号大于等于队列长度，则表明队列节点已经完全消费完了，routine结束。最终wg.Wait()结束，队列显示调用方法，销毁自身。  

并行消费的核心代码如下:
```
var wg sync.WaitGroup
wg.Add(numCPU)
items := q.items

for i := 0; i < numCPU; i++ {
    go func() {
        for {
            index := atomic.AddInt64(&done, 1)
            if index >= int64(todo) {
                wg.Done()
                break
            }

            fn(items[index])
            items[index] = 0
        }
    }()
}
wg.Wait()
```

### 3.Queue.Put() - 插入队列
该函数需要传入需要入队的数据，interface类型，执行完成后返回操作错误信息。  

该函数除了除了常规的操作加锁、销毁判定和将数据加入Queue.items，还会通过Qeueu.waiters.get()判断该队列是否正在被获取数据（这里的获取数据指的是调用Queue.Poll()并且队列为空），随后调用sema.response.Add(1)为获取数据的操作做准备，然后通过sema.ready <- true通知Queue.Poll()有数据加入，最后sema.respones.Wait()等待Queue.Poll()完成数据获取调用sema.respones.Done()。  

Queue.waiter的通信代码实现如下:
```
for {
    sema := q.waiters.get()
    if sema == nil {
        break
    }
    sema.response.Add(1)
    select {
    case sema.ready <- true:
        sema.response.Wait()
    default:
        // This semaphore timed out.
    }
    if len(q.items) == 0 {
        break
    }
}

```
select代码断有个细节，sema.ready channel容量只有1，而且有default分支，从注释上也可以看出，只要执行了default分支，就说明获取数据操作(调用Queue.Poll())超时，也就是说sema.ready已满就会执行default分支。所以当Qeueu.Poll()获取超时，会向sema.ready填入数据，“通知”Queue.Put()，sema已无意义。  

### 4.Queue.Get() - 获取队列数据
该函数需要传入一个参数，表示所需获取数据的长度，返回数据数组和操作错误信息。  

若队列为空，函数会阻塞，直到有新的数据入队才会返回，无论插入了多少的数据，都会返回传入参数长度的数组;若队列不为空，则该函数会立即返回传入参数长度的数组，无论队列长度是多少，如果队列长度小于传入参数，返回的数组超出部分为nil。  

该函数实际上是阻塞（无超时时间）调用Queue.Poll()。
```
func (q *Queue) Get(number int64) ([]interface{}, error) {
    return q.Poll(number, 0)
}
```

### 5.Queue.Poll() - 获取队列数据
该函数需要传入获取数据长度和超时时间，返回数据数组和操作错误信息。  

如果队列为空，函数会在超时时间内组阻塞，当超时或队列有插入，函数就会返回，所一在函数返回后，需要判断error部分，是否为nil。  

调用函数后，会进行常规的操作加锁、销毁判定，队列长度不为0，直接通过操作Queue.items.get()获取数据，并返回。若队列为空，则开始等待超时或等待队列有数据插入。队列长度为0时的处理代码如下：
```
if len(q.items) == 0 {
    sema := newSema()
    q.waiters.put(sema)
    q.lock.Unlock()

    var timeoutC <-chan time.Time
    if timeout > 0 {
        timeoutC = time.After(timeout)
    }
    select {
    case <-sema.ready:
        // we are now inside the put's lock
        if q.disposed {
            return nil, ErrDisposed
        }
        items = q.items.get(number)
        sema.response.Done()
        return items, nil
    case <-timeoutC:
        // cleanup the sema that was added to waiters
        select {
        case sema.ready <- true:
            // we called this before Put() could
            // Remove sema from waiters.
            q.lock.Lock()
            q.waiters.remove(sema)
            q.lock.Unlock()
        default:
            // Put() got it already, we need to call Done() so Put() can move on
            sema.response.Done()
        }
        return nil, ErrTimeout
    }
}
```

首先会新建sema，并加入Queue.waiters，随后操作解锁，允许队列进行其他操作。随后就是使用select等待sema.ready或超时回应，由于select代码块没有default分支，所以必定是这两个channel的其中一个接收到数据。  

* 如果sema.ready收到了数据，要么有数据插入队列，要么队列要销毁，所以可以先通过Queue.disposed判断队列是否销毁。若队列销毁则直接返回;若则取队列数据，调用sema.response.Done()“通知”Queue.Put()数据获取结束，并返回数据。  
* 如果已经超时，那么就要清除sema的影响。sema.ready的容量只有1，如果能向sema.ready发送数据，那就说明队列并没有调用Put()方法插入数据，那么就需要操作加锁防止sema被其他操作（如Queue.Put()）获取，然后调用Qeueu.waiters.remove移除sema;如果无法向sema.ready发送数据，则会进入default分支，那么说明Queue.Put()已经获取了sema，只需调用sema.response.Done()表明操作结束即可。  

### 6.Queue.Peek() - 获取队列头
该函数会返回队列头节点数据和操作错误信息。  

该操作不会对队列造成影响，而且会立即返回。本质上是就获取Queue.items[0]的数据值。  

### 7.Queue.TakeUntil() - 根据判断函数获取数据
该函数需要传入一个判定函数，返回数据数组和操作错误信息。  

调用判定函数需要传入队列数据，如果判定函数判定该数据符合需求，就返回True，否则返回False。而且该函数也是非阻塞型，就算是队列为空也会直接返回。  

TakeUntil()除了常规的操作加锁和销毁判定，就是直接是调用Queue.items.getUntil()，在getUntil()中遍历队列数据，如果判定函数返回的false，那么遍历立即结束。也就是说，如果队首数据不符合判定函数，那么无论队列里还有多少符合判定函数，能获取数组长度也只能为0。所以在使用该函数时除了判断返回的error值以外，还需判断数组长度是否为0。  

### 8.Queue.Dispose() - 队列销毁
该函数就是销毁数据结构里的所有数据，并给标志变量置位。然后遍历Queue.waiters获取sema并通过想向sema.ready发送数据让Queue.Poll()结束等待，最后晴空数组变量，获取队列里所有数据，并返回。  

### 9.Qeueu.Empty() Qeueu.Len() Queue.Disposed()
这些函数实现非常简单，意思也很明显，所有不做细说。

## 总结
总体来讲，是一个中规中举的队列实现，实现比较直白易懂，而且对标channel的测试来讲，性能不错。有机会在实际项目中使用再详细测试。