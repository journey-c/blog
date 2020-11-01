---
title:  sync.pool源码阅读
categories:
- 语言
- Golang
tags:
- 源码
- Golang
---

阅读项目代码的时候发现很多地方用到了golang的sync.pool，所以好奇golang的sync.pool底层实现是什么样的，有哪些优化。
本文是基于go1.13.8，做讲解。

<!-- more -->

# 介绍

Pool翻译过来就是池子，主要功能就是: 需要使用某个Object的时候可以从Pool获取，使用完毕再归还，从而减少创建和销毁Object的开销。而本文讲的就是golang中的Pool源码实现。

# 用法

**千万不要想当然的认为put进去的Object和get出来的Object有什么关系，Pool存的Object在GC时会都清理掉**

```
package main

import (
	"fmt"
	"sync"
)

type Book struct {
	Name string
	Info map[string]string
}

func NewBook() interface{} {
	return &Book{
		Name: "",
		Info: make(map[string]string),
	}
}

func main() {
	// 创建pool并定义创建object的函数
	bookPool := sync.Pool{New:NewBook}

	// 从pool获取object
	a := bookPool.Get().(*Book)
	a.Name = "go"
	a.Info["a"] = "b"

	fmt.Println(a)

	// 放回pool
	bookPool.Put(a)
}
```

# 结构图

![](/images/pool.png)

# 实现细节

- Pool实现源码是这两个文件go/src/sync/pool.go, go/src/sync/poolqueue.go

## 数据结构——从下往上讲一下Pool底层存储是如何实现

### eface

```go
// 存储元素的结构体，类型指针和值指针
type eface struct {
        typ, val unsafe.Pointer
}
```
Pool底层用eface来存储单个Object, 包括typ指针: Object的类型，val指针: Object的值

### poolDequeue 

poolDequeue是一个无锁、固定大小的单生产端多消费端的环形队列，单一producer可以在头部push和pop(可能和传统队列头部只能push的定义不同)，多consumer可以在尾部pop

1. headTail:

```
[hhhhhhhh hhhhhhhh hhhhhhhh hhhhhhhh tttttttt tttttttt tttttttt tttttttt] 
1. headTail表示下标，高32位表示头下标，低32位表示尾下标，poolDequeue定义了，head tail的pack和unpack函数方便转化，
	实际用的时候都会mod ( len(vals) - 1 ) 来防止溢出
2. head和tail永远只用32位表示，溢出后会从0开始，这也满足循环队列的设计
3. 队列为空的条件  tail == head
4. 队列满的条件    (tail+uint32(len(d.vals)))&(1<<dequeueBits-1) == head tail加上队列长度和head相等(实际上就是队列已有的空间都有值了,满了)
```

2. vals:

1) poolDequeue是被poolChain使用，poolChain使用poolDequeue时
	a) 初始化vals长度为8，vals长度必须是2的幂
	b) 当队列满时，vals长度*2，最大扩展到 dequeueLimit = (1 << 32) / 4 = (1 << 30)，之后就不会扩展了

2) 为什么vals长度必须是2的幂
	这是因为go的内存管理策略是将内存分为2的幂大小的链表，申请2的幂大小的内存可以有效减小分配内存的开销

3) 为什么dequeueLimit是(1 << 32) / 4 = 1 << 30
	a) dequeueLimit 必须是2的幂(上边解释过)
	b) head和tail都是32位，最大是1 << 31，如果都用的话，head和tail就是无符号整型，无符号整型使用的时候会有很多上溢的错误，这类错误是不容易检测的，所以相比之下还不如用31位有符号整型，有错就报出来，结论参考https://stackoverrun.com/cn/q/10770747

```go
type poolDequeue struct {
	headTail uint64

	vals []eface
}

// poolDequeue成员函数
// 这里的删除操作，是将指针置空，然后让GC来回收内存空间
unpack     将headTail分解为head和tail
pack       将head和tail组合成headTail
pushHead   添加元素到队首
popHead    获取并删除队首元素
popTail    获取并删除队尾元素
PushHead   添加元素到队首
PopHead    获取并删除队首元素
PopTail    获取并删除队尾元素
```

### poolChainElt

链表的一个节点 Node

```
type poolChainElt struct {
	poolDequeue

	// next and prev link to the adjacent poolChainElts in this
	// poolChain.
	//
	// next is written atomically by the producer and read
	// atomically by the consumer. It only transitions from nil to
	// non-nil.
	//
	// prev is written atomically by the consumer and read
	// atomically by the producer. It only transitions from
	// non-nil to nil.
	next, prev *poolChainElt
}
```

### poolChain

poolChain 是动态版的poolDequeue
head(poolDequeue)[prev] --> <--- [next](...)[prev] ---> <---[next]tail(poolDequeue)
动态的队列，队列每个节点又是一个环形队列(poolDequeue)

```
type poolChain struct {
	// 头指针，只能单一producer操作(push, pop)
	head *poolChainElt

	// 尾指针，可以被多个consumer pop，必须是原子操作
	tail *poolChainElt
}

// poolChain成员函数
func (c *poolChain) pushHead(val interface{})
	1. 如果head为nil，说明队列现在是空的，那么新建一个节点，将head和tail都指向这个节点
	2. 将val push到head的环形队列中，如果push成功了，可以返回了
	3. 如果没push成功，则说明head的环形队列满了，就再创建一个两倍head大小的节点[最大(1 << 32) / 4]，
		将新节点作为head，并且处理好新head和旧head的next，prev关系
	4. 将val push到head的环形队列中

func (c *poolChain) popHead()
	1. 先在head环形队列中popHead试试，如果空了，当前节点就没用了，就删掉当前节点，去prev节点并且把prev节点作为新head再取一值递归下去，
		能取到就返回，取不到说明队列空了
func (c *poolChain) popTail()
	1. 如果tail为nil，说明队列是空的，直接返回
	2. 如果tail非nil，就取取试试，有东西就返回
	3. 如果没取出来东西，那么说明tail节点没存东西了，递归去prev节点环形队列中popTail，并且把prev节点作为tail，能取到就返回，取不到就是空了
```

### poolLocal
1. poolLocal是每个调度器(P)存Object的结构体
2. private是每个调度器私有的，shared是所有调度器公有的，每个调度器pop时的逻辑是: 先看private，没有在看自己的shared，再没有就去其他调度器的shared偷，再没有才是空
3. pad是防止伪共享，参考https://www.cnblogs.com/cyfonly/p/5800758.html
```go
type poolLocal struct {
	poolLocalInternal

	// Prevents false sharing on widespread platforms with
	// 128 mod (cache line size) = 0 .
	pad [128 - unsafe.Sizeof(poolLocalInternal{})%128]byte
}

// Local per-P Pool appendix.
// 当前调度器的内部资源
type poolLocalInternal struct {
	// 当前调度器的私有资源
	private interface{} // Can be used only by the respective P.
	// 所有调度器的公有资源
	shared  poolChain   // Local P can pushHead/popHead; any P can popTail.
}
```

## 主要函数

### Put

Put adds x to the pool.
1. 首先关闭竞争检测，然后会将当前goroutine固定到一个调度器(P)上，且不允许抢占
2. 从Pool的local中取出来当前goroutine固定到那个调度器(P)对应的poolLocal, 没有就新建
3. 先判断这个当前调度器(P)专属poolLocal，私有空间是不是空的，如果是把x放到私有空间，并把x置nil
4. 判断x是否为nil，如果不为空说明私有空间满了，就push到该调度器专属poolLocal的shared head
5. 允许抢占，开启竞争检测

```go
func (p *Pool) Put(x interface{}) {
	// 如果put进来的值为空直接返回
	if x == nil {
		return
	}
	// 关闭竞争检测
	if race.Enabled {
		if fastrand()%4 == 0 {
			// Randomly drop x on floor.
			return
		}
		race.ReleaseMerge(poolRaceAddr(x))
		race.Disable()
	}
	// 
	l, _ := p.pin()
	if l.private == nil {
		l.private = x
		x = nil
	}
	if x != nil {
		l.shared.pushHead(x)
	}
	runtime_procUnpin()
	if race.Enabled {
		race.Enable()
	}
}
```

把当前的goroutine固定到调度器(P)，不允许抢占, 返回该调度器(P)对应的poolLocal和调度器(P)ID
运行时调度器的三个重要组成部分 — 线程 M、Goroutine G 和调度器 P(负责调度)

判断pid是否小于[]poolLocal的长度，小于的话就在取出poolLocal[P]返回，否则就去执行pinSlow函数
Caller must call runtime_procUnpin() when done with the pool.

```go
func (p *Pool) pin() (*poolLocal, int) {
	// 关闭抢占，等这个goroutine工作完，其他goroutine才能获得时间片工作
	pid := runtime_procPin()
	// In pinSlow we store to local and then to localSize, here we load in opposite order.
	// Since we've disabled preemption, GC cannot happen in between.
	// Thus here we must observe local at least as large localSize.
	// We can observe a newer/larger local, it is fine (we must observe its zero-initialized-ness).

	s := atomic.LoadUintptr(&p.localSize) // load-acquire
	l := p.local                          // load-consume
	if uintptr(pid) < s {
		return indexLocal(l, pid), pid
	}
	return p.pinSlow()
}
```

当goroutine固定到的调度器(P)没有poolLocal时，pins() 函数就会调用pinSlow() 来重新固定到其他调度器(P)，
如果新固定到的调度器(P)还是没有poolLocal，就给该调度器创建一个poolLocal放到Pool的local中

1. 打开抢占并且pool加锁然后关闭抢占，这里如果不先打开抢占的话，其他goroutine如果之前获得锁了，但不能运行，当前goroutine在获取锁，就会死锁
2. 如果判断pid和len([]poolLocal)的关系，小于就返回[PID]poolLocal
3. 如果此Pool的[]poolLocal是空的，就把Pool加到allPools中
4. 获得当前cpu的数量，创建一个cpu数量大小的[]poolLocal

```go
func (p *Pool) pinSlow() (*poolLocal, int) {
	runtime_procUnpin()
	allPoolsMu.Lock()
	defer allPoolsMu.Unlock()
	pid := runtime_procPin()
	// poolCleanup won't be called while we are pinned.
	s := p.localSize
	l := p.local
	if uintptr(pid) < s {
		return indexLocal(l, pid), pid
	}
	if p.local == nil {
		allPools = append(allPools, p)
	}
	// If GOMAXPROCS changes between GCs, we re-allocate the array and lose the old one.
	size := runtime.GOMAXPROCS(0)
	local := make([]poolLocal, size)
	atomic.StorePointer(&p.local, unsafe.Pointer(&local[0])) // store-release
	atomic.StoreUintptr(&p.localSize, uintptr(size))         // store-release
	return &local[pid], pid
}
```

### Get

从Pool中获取对象，然后返回，如果Pool为空的就用New来创建
不要假设Put进来的对象和Get得到的对象有什么关系

1. 关掉竞争检测
2. 将goroutine固定到一个调度器(P), 并获取他的poolLocal和PID
3. 判断该调度器(P)的poolLocal的私有空间是不是空的，如果是空的，就从该调度器(P)的poolLocal shared空间头
	pop一下看有没有
4. 如果没有，就说明该调度器(P)自己的poolLocal没有对象了，就调用getSlow

```go
func (p *Pool) Get() interface{} {
	if race.Enabled {
		race.Disable()
	}
	l, pid := p.pin()
	x := l.private
	l.private = nil
	if x == nil {
		// Try to pop the head of the local shard. We prefer
		// the head over the tail for temporal locality of
		// reuse.
		x, _ = l.shared.popHead()
		if x == nil {
			x = p.getSlow(pid)
		}
	}
	runtime_procUnpin()
	if race.Enabled {
		race.Enable()
		if x != nil {
			race.Acquire(poolRaceAddr(x))
		}
	}
	if x == nil && p.New != nil {
		x = p.New()
	}
	return x
}
```

懒获取函数
1. 取到Pool的localSize和local
2. 然后遍历其他调度器(P)对应的poolLocal，看看能不能从对应poolLocal中的shared tail中取出对象, 如果能取到，直接返回
3. 如果取不到就到victim中查询，有就返回，没有调用New创建一个新的Object返回

```go
func (p *Pool) getSlow(pid int) interface{} {
	// See the comment in pin regarding ordering of the loads.
	size := atomic.LoadUintptr(&p.localSize) // load-acquire
	locals := p.local                        // load-consume
	// Try to steal one element from other procs.
	for i := 0; i < int(size); i++ {
		l := indexLocal(locals, (pid+i+1)%int(size))
		if x, _ := l.shared.popTail(); x != nil {
			return x
		}
	}

	// Try the victim cache. We do this after attempting to steal
	// from all primary caches because we want objects in the
	// victim cache to age out if at all possible.
	size = atomic.LoadUintptr(&p.victimSize)
	if uintptr(pid) >= size {
		return nil
	}
	locals = p.victim
	l := indexLocal(locals, pid)
	if x := l.private; x != nil {
		l.private = nil
		return x
	}
	for i := 0; i < int(size); i++ {
		l := indexLocal(locals, (pid+i)%int(size))
		if x, _ := l.shared.popTail(); x != nil {
			return x
		}
	}

	// Mark the victim cache as empty for future gets don't bother
	// with it.
	atomic.StoreUintptr(&p.victimSize, 0)

	return nil
}
```

## 附录

### pool.dot

```
digraph {
    bgcolor="#C6CFD532";

    node [shape=record, fontsize="8", margin="0.04", height=0.2, color=gray]
    edge [fontname="Inconsolata, Consolas", fontsize=10, arrowhead=normal]

    pool [shape=record,label="{noCopy|<local>local|localSize|<victim>victim|victimSize|New}",xlabel="Pool"]
    poolLocal[shape=record,label="{<poolLocalInternal>poolLocalInternal|pad}",xlabel="poolLocal"]
    poolLocalInternal[shape=record,label="{private|<shared>shared}",xlabel="poolLocalInternal"]
    poolChain[shape=record,label="{<head>head|<tail>tail}",xlabel="poolChain"]
    poolChainElt[shape=record,label="{<poolDequeue>poolDequeue|next|prev}",xlabel="poolChainElt"]
    poolDequeue[shape=record,label="{headTail|<vals>vals}",xlabel="poolDequeue"]
    eface[shape=record,label="{typ|val}",xlabel="eface"]
    victim[shape=record,label="GC的时候，首先把local中每个处理器(P)对应的poolLocal赋给victim，然后清空local，所以victim就是缓存GC前的local",xlabel="victim"]

    pool:local -> poolLocal [label="local指针指向[]poolLocal首地址",rankdir=LR]
    poolLocal:poolLocalInternal -> poolLocalInternal
    poolLocalInternal:shared -> poolChain[label="shared是一个队列"]
    poolChain:head -> poolChainElt[label="head和tail是队列的收尾节点指针"]
    poolChain:tail -> poolChainElt
    poolChainElt:poolDequeue -> poolDequeue[label="poolDequeue是一个环形队列"]
    poolDequeue:vals -> eface[label="eface存储Object的结构体，typ和val是Object的类型和值指针"]
    pool:victim -> victim
}
```

