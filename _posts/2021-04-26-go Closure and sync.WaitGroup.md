---
layout: post
title: "go Closure and sync.WaitGroup"
---

之前看 mit6.824 第二节课时 RPC And Threads ，有讲到下面这块代码，涉及到 closure 和 waitGroup，所以简单记下。

	func ConcurrentMutex(url string, fetcher Fetcher, f *fetchState) {
		f.mu.Lock()
		already := f.fetch[url]
		f.fetched[url] = true
		f.mu.Unlock()
		
		if already {
			return
		}
		
		urls, err := fetcher.Fetch(url)
		if err != nil {
			return
		}
		var done sync.WaitGroup
		for _, u := range urls {
			done.Add(1)
			go func (u string ) {
				defer done.Done()
				ConcurrentMutex(u, fetcher, f)
			}
		}
		done.Wait()
		return
	}
	
### closure

wiki 中定义：a closure is a record storing a function together with an environment.The environment is a mapping associating each free variable of the function (variables that are used locally, but defined in an enclosing scope) with the value or reference to which the name was bound when the closure was created.

简单来说就是 函数 + 被捕获的外部变量

上面代码 for 循环，内部函数引用了外部函数 `u` 这个变量，go compiler 将会从 heap 上为 `u` 分配内存，即使外部函数返回了，变量依然在 heap 上，inner function 仍然可以访问到这个变量，变量的释放依赖垃圾回收。

闭包是一个函数值，引用了函数题之外的变量。一旦 inner function 引用了 inner function 之外的变量，这些变量就和 inner function 绑定在一起。

	package main

	import "fmt"

	func adder() func(int) int {
		sum := 0
		return func(x int) int {
			sum += x
			return sum
		}
	}

	func main() {
		pos, neg := adder(), adder()
		for i := 0; i < 10; i++ {
			fmt.Println(
				pos(i),
				neg(-2*i),
			)
		}
	}

adder 函数返回一个闭包，执行结果：

	0 0
	1 -2
	3 -6
	6 -12
	10 -20
	15 -30
	21 -42
	28 -56
	36 -72
	45 -90
	
用闭包实现一个 fibonacci 数列

	package main

	import "fmt"

	func fibonacci() func() int {
		i := 0
		j := 1
		return func() int {
			temp := i
			sum := i + j
			i = j
			j = sum
			return temp
		}
	}

	func main() {
		f := fibonacci()
		for i := 0; i < 10; i++ {
			fmt.Println(f())
		}
	}

### sync.WaitGroup

文章开头的代码中，要等待当前 url 下所有的 url 都爬取结束才能返回。

*A WaitGroup waits for a collection of goroutines to finish*

#### 用法

* sync.WaitGroup 初始化
* wg.Add(delta) 设置 worker 线程数量
* 线程结束后调用 wg.Done()
* main 主线程调用 wg.Wait() 被阻塞，等待所有的 worker 都执行结束后返回。

#### waitGroup 实现

	type WaitGroup struct {
		noCopy noCopy

		// 64-bit value: high 32 bits are counter, low 32 bits are waiter count.
		// 64-bit atomic operations require 64-bit alignment, but 32-bit
		// compilers do not ensure it. So we allocate 12 bytes and then use
		// the aligned 8 bytes in them as state, and the other 4 as storage
		// for the sema.
		state1 [3]uint32
	}
	
	// state returns pointers to the state and sema fields stored within wg.state1.
	func (wg *WaitGroup) state() (statep *uint64, semap *uint32) {
		if uintptr(unsafe.Pointer(&wg.state1))%8 == 0 {
			return (*uint64)(unsafe.Pointer(&wg.state1)), &wg.state1[2]
		} else {
			return (*uint64)(unsafe.Pointer(&wg.state1[1])), &wg.state1[0]
		}
	}

`noCopy` vet 检查，如果有变量的 copy 则报错

	wg := sync.WaitGroup{}
	w := wg
	
执行 go vet main.go 会报错

三个变量

* `statep *uint64 ` 如果是 64 位操作系统，高 32 位是 counter 计数器，低 32 位是 waiter 计数器。
* `sema` 信号量


##### Add 实现

	// Typically this means the calls to Add should execute before the statement
	// creating the goroutine or other event to be waited for.
	// If a WaitGroup is reused to wait for several independent sets of events,
	// new Add calls must happen after all previous Wait calls have returned.
	func (wg *WaitGroup) Add(delta int) {
		statep, semap := wg.state()
		if race.Enabled {
			_ = *statep // trigger nil deref early
			if delta < 0 {
				// Synchronize decrements with Wait.
				race.ReleaseMerge(unsafe.Pointer(wg))
			}
			race.Disable()
			defer race.Enable()
		}
		// add delta to WaitGroup counter
		state := atomic.AddUint64(statep, uint64(delta)<<32)
		v := int32(state >> 32)
		w := uint32(state)
		if race.Enabled && delta > 0 && v == int32(delta) {
			// The first increment must be synchronized with Wait.
			// Need to model this as a read, because there can be
			// several concurrent wg.counter transitions from 0.
			race.Read(unsafe.Pointer(semap))
		}
		// 如果 counter 变为 0，触发 panic
		if v < 0 {
			panic("sync: negative WaitGroup counter")
		}
		// 如果已经有 goroutine 在 wait 了，但是又调用 add 会触发 panic
		if w != 0 && delta > 0 && v == int32(delta) {
			panic("sync: WaitGroup misuse: Add called concurrently with Wait")
		}
		// 正常情况下，没有 waiter 或者任务未全部完成
		if v > 0 || w == 0 {
			return
		}
		
		
		// This goroutine has set counter to 0 when waiters > 0.
		// Now there can't be concurrent mutations of state:
		// - Adds must not happen concurrently with Wait,
		// - Wait does not increment waiters if it sees counter == 0.
		// Still do a cheap sanity check to detect WaitGroup misuse.
		// 此时 v== 0 waiter > 0  此时如果数据还在变动，done 调用 触发 panic
		if *statep != state {
			panic("sync: WaitGroup misuse: Add called concurrently with Wait")
		}
		// Reset waiters count to 0.
		// Counter 变为 0 ，通知所有阻塞在 wait 上的 goroutine
		*statep = 0
		for ; w != 0; w-- {
			runtime_Semrelease(semap, false, 0)
		}
	}

* counter 变为负数会触发 panic
* wait 如果要调用 add，必须要等待所有的 wait 都返回才能重新 add （所有的任务完成，reset waiter count to 0，所有的 wait goroutine 被通知到 返回之后） 否则会触发 panic
	
##### Done 实现

	// Done decrements the WaitGroup counter by one.
	func (wg *WaitGroup) Done() {
		wg.Add(-1)
	}
	
##### wait 实现

	// Wait blocks until the WaitGroup counter is zero.
	func (wg *WaitGroup) Wait() {
		statep, semap := wg.state()
		if race.Enabled {
			_ = *statep // trigger nil deref early
			race.Disable()
		}
		for {
			state := atomic.LoadUint64(statep)
			v := int32(state >> 32)
			w := uint32(state)
			// 如果 worker counter 为 0 则结束等待 返回
			if v == 0 {
				// Counter is 0, no need to wait.
				if race.Enabled {
					race.Enable()
					race.Acquire(unsafe.Pointer(wg))
				}
				return
			}
			// 增加 waiter 数量
			// Increment waiters count.
			if atomic.CompareAndSwapUint64(statep, state, state+1) {
				if race.Enabled && w == 0 {
					// Wait must be synchronized with the first Add.
					// Need to model this is as a write to race with the read in Add.
					// As a consequence, can do the write only for the first waiter,
					// otherwise concurrent Waits will race with each other.
					race.Write(unsafe.Pointer(semap))
				}
				runtime_Semacquire(semap)
				// 如果信号来了，但是 statep 还未被设置成0，说明 wait 之后还有 goroutine 再调用 add, 触发 panic
				if *statep != 0 {
					panic("sync: WaitGroup is reused before previous Wait has returned")
				}
				if race.Enabled {
					race.Enable()
					race.Acquire(unsafe.Pointer(wg))
				}
				return
			}
		}
	}

此处增加 worker 数量没有使用锁，而是使用 CAS。

> 比较并交换(compare and swap, CAS)，是原子操作的一种，可用于在多线程编程中实现不被打断的数据交换操作，从而避免多线程同时改写某一数据时由于执行顺序不确定性以及中断的不可预知性产生的数据不一致问题。 该操作通过将内存中的值与指定数据进行比较，当数值一样时将内存中的数据替换为新的值。

	int cas(long *addr, long old, long new)
	{
    	/* Executes atomically. */
    	if(*addr != old)
        	return 0;
    	*addr = new;
    	return 1;
	}

>在使用上，通常会记录下某块内存中的旧值，通过对旧值进行一系列的操作后得到新值，然后通过CAS操作将新值与旧值进行交换。如果这块内存的值在这期间内没被修改过，则旧值会与内存中的数据相同，这时CAS操作将会成功执行 使内存中的数据变为新值。如果内存中的值在这期间内被修改过，则一般[2]来说旧值会与内存中的数据不同，这时CAS操作将会失败，新值将不会被写入内存。

如果没有修改成功则再次 for 循环尝试写入。

直至写入成功，调用 runtime_Semacquire 阻塞等待信号量。函数返回。


### 参考
 
 [维基百科 CAS](https://zh.wikipedia.org/wiki/%E6%AF%94%E8%BE%83%E5%B9%B6%E4%BA%A4%E6%8D%A2)
 
[https://segmentfault.com/a/1190000002963465](https://segmentfault.com/a/1190000002963465)

[https://tour.go-zh.org/moretypes/25](https://tour.go-zh.org/moretypes/25)