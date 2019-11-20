---
title: "记一次golang“内存泄漏”问题"
date: 2019-11-03T14:05:39+08:00
draft: false
---

# 记一次golang“内存泄漏”问题

先说结论，这不是并不是真正代码意义上的内存泄漏，而是对golang的内存管理机制的一次重新理解。  

本次使用的golang版本为12.5
## 问题浮现  

问题出现在一次机器内存不足，并且cpu负载间歇峰值，导致整个服务访问缓慢。该服务只是一个无状态的后台http服务，没有使用进程内自定义缓存方法，理论上应该不太容易出现内存泄漏问题。  

那么就开启pprof进行相关分析，奇怪的是，gorontine数量只有10多20个，在内存分析使用`inuse_space`选项分析发现整个内存占用其实不高，按照文档来说，说明进程当前使用的内存其实也不高，但是`top`命令显示进程的`RES`已经占用不低了。不得已只能换一种方式去看，看下能不能先复现问题。内存分析时使用`alloc_space`来查看已经分配内存，然后发现一个excel包的函数分配内存量很高，但其实很正常，本身导出数据量以千以万计算，所以在测试环境直接就拿一些相对极端的参数请求相关的接口。发现内存占用确实是增长，pprof重新分析确实也符合线上的分析情况，说明算是复现问题了。  

## 问题抽象

从目前了解的资料，gc有两种触发条件，一个是定时触发，一个是内存阈值（GCPERCENT）。根据这个两个条件，第一次excel下载产生的内存申请足以由于内存阈值触发了gc了，再不济应该定时触发处理gc，但程序内存一直都没有释放给系统（一个通宵）。根据pprof的分析，代码应该不存在由于引用问题导致内存没有gc，为了验证这个问题，重新抽象了行为来验证，以下是重新抽象后的代码:
```
e.g 1:
func getMemory(i int) []byte {
	max := 1024 * 1024 * 512 * i
	b := make([]byte, max)
	count := 0
	for i := 0; i < max; i++ {
		if count > 10 {
			count = 0
		}
		b[i] = byte(count)
		count++
	}
	return b
}
func calc() {
	a := [2][]byte{}
	for i := 0; i < len(a); i++ {
		time.Sleep(time.Second * 10)
		a[i] = getMemory()
		// getMemory()
	}
}

func main() {
	c := make(chan os.Signal, 1)
	signal.Notify(c, os.Interrupt)
	go func() {
		calc()
	}()
	<-c
}

```
抽象后，`calc()`函数就是实际的业务处理函数，`getMemory()`就是excel生成的数据，执行命令`go build -o test main.go && GODEBUG=gctrace=1 ./test`开启gc跟踪，我们就可以看到整个内存的分配情况:
```
gc 输出： 
gc 1 @10.019s 0%: 0.004+1347+0.006 ms clock, 0.009+0/0.081/1347+0.013 ms cpu, 512->512->512 MB, 513 MB goal, 2 P
len = 536870912
len = 536870912
gc 2 @21.385s 0%: 0.004+5696+0.013 ms clock, 0.009+1.2/1.2/5693+0.027 ms cpu, 1024->1024->1024 MB, 1025 MB goal, 2 P
GC forced
gc 3 @147.085s 0%: 0.015+3.4+0.008 ms clock, 0.030+0/1.2/4.7+0.016 ms cpu, 1024->1024->0 MB, 2048 MB goal, 2 P
scvg0: inuse: 0, idle: 1087, sys: 1087, released: 63, consumed: 1024 (MB)
GC forced
gc 4 @267.090s 0%: 0.050+0.26+0.008 ms clock, 0.10+0/0.075/0.28+0.017 ms cpu, 0->0->0 MB, 5 MB goal, 2 P
scvg1: inuse: 0, idle: 1087, sys: 1087, released: 63, consumed: 1024 (MB)
GC forced
gc 5 @387.092s 0%: 0.011+0.27+0.008 ms clock, 0.022+0/0.068/0.30+0.016 ms cpu, 0->0->0 MB, 4 MB goal, 2 P
GC forced
gc 6 @507.093s 0%: 0.007+0.28+0.007 ms clock, 0.014+0/0.067/0.26+0.014 ms cpu, 0->0->0 MB, 4 MB goal, 2 P
scvg2: 1024 MB released
scvg2: inuse: 0, idle: 1087, sys: 1087, released: 1087, consumed: 0 (MB)
GC forced
gc 7 @627.194s 0%: 0.008+1.4+0.007 ms clock, 0.017+0/0.068/2.0+0.014 ms cpu, 0->0->0 MB, 4 MB goal, 2 P

```
以上的gc输出是程序运行了一个小时后程序的前10分钟的输出（后面的gc数据基本没有变化），从gc输出来看（输出参数含义可看[这里](https://golang.org/pkg/runtime/)），gc1时分配了一次512M的内存，堆留存512M，10s后再次分配内存，堆留存1G内存，此时`calc()`函数处理结束。2分钟后触发gc3，强制gc，此时堆留存已变为0。再过多一段时间后触发scvg，从scvg2可以发现已经将1024M内存归还给系统了，但此时系统的RES依然高达1G。
```
top输出
top - 21:25:27 up  2:17,  1 user,  load average: 0.38, 0.34, 0.92
Tasks: 196 total,   1 running, 195 sleeping,   0 stopped,   0 zombie
%Cpu(s):  0.2 us,  0.7 sy,  0.0 ni, 99.0 id,  0.0 wa,  0.2 hi,  0.0 si,  0.0 st
MiB Mem :   3807.9 total,    174.2 free,   2777.3 used,    856.4 buff/cache
MiB Swap:   4096.0 total,   4017.7 free,     78.2 used.    478.2 avail Mem 

    PID USER      PR  NI    VIRT    RES    SHR S  %CPU  %MEM     TIME+ COMMAND  
   5283 abba      20   0 1186972   1.0g    616 S   0.0  27.8   0:06.61 test    
```

通过这些数据，说明应用并没有出现由于误用导致的内存泄漏问题，而是gc本身处理出现了“问题”。  

## 解决问题
从[刚刚](https://golang.org/pkg/runtime/)但看到gc参数含义中，有一段文档引起了注意：
```
madvdontneed: setting madvdontneed=1 will use MADV_DONTNEED
instead of MADV_FREE on Linux when returning memory to the
kernel. This is less efficient, but causes RSS numbers to drop
more quickly.
```
大意就是配置该变量时，在linux上会切换另一种内存返回方式给内存，该方式会更为低效，但RSS（对应上文RES）会下降得更快。  
这段话恰恰说明了一个问题，那就是默认的内存返还方式并不会快速降低RSS，也就符合了刚刚的测试。那么重新换一个编译运行方式RSS就应该能市RSS快速下降。  
代码不作修改，运行`go build -o test main.go && GODEBUG=gctrace=1,madvdontneed=1 ./test`，重新查看gc输出和top输出。gc输出与top输出在前期并没有什么不同，但在gc输出`scvg2: 1024 MB released`后，系统的RES确实立即下降了，从而也就印证了文档的说法。  


MADV_FREE与MADV_DONTNEED是linux通过[madvise函数](http://man7.org/linux/man-pages/man2/madvise.2.html)分配内存的参数，它们的主要区别是：MADV_DONTNEED会立即降低系统的常驻内存即RSS，而MADV_FREE则是内核回标记这些内存，但不会立即归还，同时还能再次写入，直到出现内存压力。也就是说MADV_FREE可以机内存充足的时候重新