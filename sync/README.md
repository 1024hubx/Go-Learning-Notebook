# Go 语言中的同步 #

## 为什么需要同步 ##

虽然Go语言中提倡[用通讯的方式共享数据](https://en.wikipedia.org/wiki/Communicating_sequential_processes)，但是也支持主流的共享内存的方式来共享数据。

一旦数据被多个执行单元共享，那么就很可能会产生争用和冲突的情况(竞态条件，race condition)，而这往往会破坏共享数据的一致性。

考虑如下的[例子1](https://play.golang.org/p/PzVD2U4HP1h):

```
package main

import (
	"fmt"
	"time"
)

func main() {
	count := 0
	worker := func() {
		for i := 0; i < 100; i++ {
			count++
		}
	}

	for i := 0; i < 1000; i++ {
		go worker()
	}

	time.Sleep(1 * time.Second)
	fmt.Println(count)
}
```

在正常的情况下，count的预期值应该是100000，但实际情况往往小于该值，原因既是第12行代码出现了竞态条件。

**注意**：在go playground中运行代码不会出现上述情况，原因可能为在那个环境下只提供了一个CPU供使用。

使用 **go tool compile -N -S main.go >main.S** 命令获取上述代码的汇编代码：

```
	0x002a 00042 (main.go:12)	PCDATA	$2, $2
	0x002a 00042 (main.go:12)	MOVQ	"".&count+8(SP), AX
	0x002f 00047 (main.go:12)	PCDATA	$2, $0
	0x002f 00047 (main.go:12)	MOVQ	(AX), AX
	0x0032 00050 (main.go:12)	PCDATA	$2, $3
	0x0032 00050 (main.go:12)	MOVQ	"".&count+8(SP), CX
	0x0037 00055 (main.go:12)	INCQ	AX
	0x003a 00058 (main.go:12)	PCDATA	$2, $0
	0x003a 00058 (main.go:12)	MOVQ	AX, (CX)
	0x003d 00061 (main.go:12)	JMP	63
```

可以看到第12行 **count++** 可能会被编译为以上代码(去除了优化)，在如下的执行顺序中会出现错误：

1. goroutine1 启动, 并执行到 0x002a 处, 此时 AX 的值为 0
2. goroutine2 启动，同样执行到 0x002a 处，此时 AX 的值为 0
3. goroutine1 继续运行并执行到 0x003a 处，此时 count 被赋值为 1
4. goroutine2 继续运行并执行到 0x003a 处，此时 count 被赋值为 1

在如上所述的执行流程下，最终会因为竞态条件而获得错误的结果。所以当一个执行单元想要访问共享资源的时候必须进行同步，避免多个执行单元同时操作同一个共享资源，或者避免执行同一个代码块(往往是因为同样的代码块会访问同样的共享资源)。

go语言中的sync包提供了多种同步工具，包括互斥量(mutex)，条件变量(cond)等。

## 互斥量 ##

互斥量是最常用的同步工具，sync包提供了Mutex类型来对应。一个互斥量可以用来保护一个临界区，它可以确保同一时刻只有一个goroutine执行于该临界区之内。

为了应对读多写少的情况，进一步提升效率，go语言中提供了互斥锁和读写锁两种实现。

**互斥锁的使用方式**

将例子1的代码使用互斥锁修改为[例子2](https://play.golang.org/p/Oo11c8rKNHM):

```
package main

import (
	"fmt"
	"sync"
	"time"
)

func main() {
	count := 0
	var l sync.Mutex

	worker := func() {
		for i := 0; i < 100; i++ {
			func() {
				l.Lock()
				count++
				l.Unlock()
			}()
		}
	}

	for i := 0; i < 1000; i++ {
		go worker()
	}

	time.Sleep(1 * time.Second)
	fmt.Println(count)
}
```

**读写锁的使用方式**

相对于互斥锁，读写锁进一步提供了Rlock/RUnlock来实现对读锁的请求/释放。

### 注意事项 ###

在使用互斥锁的过程中，有以下的一些注意事项：

1. go语言的互斥锁是不可重入的，重入的情况会导致死锁
2. 请求/释放必须配对，不然也会导致死锁的情况，必要时可以使用 defer 语句来确保释放锁
3. 不能在没有请求到锁的时候进行释放，会导致 panic
4. 互斥锁不能进行复制
5. 在使用读写锁的时候，有如下规则：

| 锁的类型 | 读锁 | 写锁 |
|:--|:--|:--|
| 读锁 | 不互斥 | 互斥 |
| 写锁 | 互斥 | 互斥 |

6. 必要时可以将 sync.Mutex 嵌入结构体

将例子1的代码使用互斥锁修改为[例子3](https://play.golang.org/p/tZE8WQdBpdu):

```
package main

import (
	"fmt"
	"sync"
	"time"
)

type Count struct {
	sync.Mutex
	count int
}

func (c *Count) Add() {
	c.Lock()
	c.count++
	c.Unlock()
}

func (c *Count) Value() int {
	c.Lock()
	defer c.Unlock()
	
	return c.count
}

func main() {
	count := Count{}

	worker := func() {
		for i := 0; i < 100; i++ {
			count.Add()
		}
	}

	for i := 0; i < 1000; i++ {
		go worker()
	}

	time.Sleep(1 * time.Second)
	fmt.Println(count.Value())
}
```

但是这种使用方式需要小心，如果该类型被导出的包外的话可能会导致意想不到的错误，包外的代码可能错误的使用了请求/释放操作。

## 条件变量 ##

前面介绍到可以使用互斥量来保护共享资源，那么如何协调要访问共享资源的执行单元呢？答案是条件变量，即sync.Cond，条件变量是基于互斥量的。

考虑如下的[例子4](https://play.golang.org/p/fpLM7-cblzU)，生产者向一个限制最大长度的缓冲中填入数据，消费者从缓冲中获取数据:

```
package main

import (
	"container/list"
	"fmt"
	"sync"
	"time"
)

// Channel is example for sync.Cond
type Channel struct {
	l    list.List
	cond *sync.Cond
	lock sync.Mutex
}

// NewChannel returns Channel instance
func NewChannel() *Channel {
	r := &Channel{}
	r.cond = sync.NewCond(&(r.lock))
	return r
}

// Push value to channel
func (c *Channel) Push(x int) {
	c.lock.Lock()
	defer c.lock.Unlock()

	for c.l.Len() == 10 {
		c.cond.Wait()
	}
	c.l.PushBack(x)
	if c.l.Len() == 1 {
		c.cond.Broadcast()
	}

	return
}

// Get value from channel
func (c *Channel) Get() int {
	c.lock.Lock()
	defer c.lock.Unlock()

	for c.l.Len() == 0 {
		c.cond.Wait()
	}
	e := c.l.Front()
	c.l.Remove(e)
	if c.l.Len() == 9 {
		c.cond.Signal()
	}
	return e.Value.(int)
}

func main() {
	c := NewChannel()
	go func() {
		for i := 0; i < 100; i++ {
			c.Push(i)
		}
	}()

	for i := 0; i < 1; i++ {
		go func(i int) {
			for {
				fmt.Println(i, ": ", c.Get())
			}
		}(i)
	}

	time.Sleep(2 * time.Second)
}
```

当生产者填入数据时首先检查缓冲是否已满，如果是则等待缓冲中有空闲空间再填入；胜消费者获取数据时首先检查是否有数据，如果没有则等待生产者填入数据。

如果不用条件变量，则需要生产者/消费者不断的去循环判断条件是否满足，这会浪费CPU时间，而使用条件变量的则可以避免自旋。

那么如何使用条件变量呢？

1. 使用一个 Locker 初始化条件变量
2. 当条件不满足时调用 Wait 等待信号
3. 被唤醒后重复检查条件是否满足
4. 如果满足则进行后续操作，并且在必要情况下调用 Signal/Broadcast 唤醒等待的条件变量

一个简单的例子代码如下:

```
var lock sync.Mutex
cond := sync.NewCond(&lock)

func f1() {
    lock.Lock()
    defer lock.Unlock()

    for needwait {
        cond.Wait()
    }

    // do somethind
    if neednotify {
        cond.Signal()  or cond.Broadcast()
    }
}
```

使用条件变量的过程中有以下点需要注意:

1. 条件变量使用的代码区域需要使用 lock 进行保护
2. 一般需要循环的判断是否满足等待条件, 因为可能有多个执行单元被唤醒/或者有多个资源状态/或者可能发生[虚假唤醒](https://en.wikipedia.org/wiki/Spurious_wakeup), 醒来的执行单元需要重复判断条件
3. Signal只会唤醒一个，Broadcast唤醒多个。如果不确定该使用哪个则统一使用Broadcast
4. 如果更注重性能，可以先执行Unlock操作再进行Signal/Broadcast

## 原子操作 ##

通过互斥锁我们可以确保代码是串行执行的，但是仍然可能被中断(不能确保原子性)。sync/atomic 提供了原子操作，这是由CPU提供的芯片级别的支持，在执行过程中是不允许中断的。

原子操作可以消除竞态条件，并能够保证并发安全性。它的执行速度要快于其他的同步工具，但是只支持一些简单的操作。

一个简单的自增以及自减2的[例子5](https://play.golang.org/p/jEYJ1dCXYde)如下:

```
package main

import (
	"fmt"
	"sync/atomic"
	"time"
)

func main() {
	c := int32(0)

	for i := 0; i < 100; i++ {
		go func(i int) {
			atomic.AddInt32(&c, 1)
		}(i)
	}

	v := uint32(100)
	for i := 0; i < 40; i++ {
		go func() {
			atomic.AddUint32(&v, ^uint32(1))
		}()
	}

	time.Sleep(2 * time.Second)
	fmt.Println(c)
	fmt.Println(v)
}
```

除了加法，原子操作还支持CAS，Load，Store以及Swap等操作。例如，可以通过CAS实现一个简单的乐观锁，[例子6](https://play.golang.org/p/YlH7zqnHDek):

```
package main

import (
	"fmt"
	"sync/atomic"
	"time"
)

func main() {
	c := int32(0)

	go func() {
		for {
			if atomic.CompareAndSwapInt32(&c, 1, 100) {
				break
			}
			fmt.Println("Waiting...")
			time.Sleep(100 * time.Millisecond)
		}
	}()

	go func() {
		time.Sleep(300 * time.Millisecond)
		atomic.StoreInt32(&c, 1)
	}()

	time.Sleep(2 * time.Second)
	fmt.Println(c)
}
```

## 其他 ##

### WaitGroup ###

在之前的例子代码中，我们都是使用 time.Sleep 来确保子协程已经执行完成，其实这不是一个标准的正确做法。sync包提供了 WaitGroup 来实现等待某种操作完成的功能:

将例子1的代码使用WaitGroup重构如下, [例子7](https://play.golang.org/p/Qt6YvwW7tf8):

```
package main

import (
	"fmt"
	"sync"
)

func main() {
	count := 0
	var wg sync.WaitGroup

	worker := func() {
		defer wg.Done()
		for i := 0; i < 100; i++ {
			count++
		}
	}

	for i := 0; i < 1000; i++ {
		wg.Add(1)
		go worker()
	}

	wg.Wait()
	fmt.Println(count)
}
```

在使用 WaitGroup 的过程中需要注意以下几点:

1. WaitGroup中内部有一个计数器, 每次调用Done函数会将值减1，如果值小于0会触发panic
2. WaitGroup是一个结构体，如果发生复制则会产生意想不到的效果，应尽量避免复制
3. 不能在调用第一次Add之前调用Wait，会触发panic
4. 调用Add的代码和调用Wait的代码应该在同一个goroutine中

以下是一个简单的封装工具来方便使用:

```
import (
	"sync"
)

// WaitGroupWrapper is a help struct for sync.WaitGroup
type WaitGroupWrapper struct {
	sync.WaitGroup
}

// Wrap will handle sync.WaitGroup Add and Done
func (w *WaitGroupWrapper) Wrap(cb func()) {
	w.Add(1)
	go func() {
		defer w.Done()
		cb()
	}()
}
```

### Once ###

很多时候我们需要确保某个函数只被执行一次，常见的如初始化数据库连接池，sync.Once 可以帮助我们实现这个要求, [例子8](https://play.golang.org/p/ExKMTc0bJoB)。

```
package main

import (
	"fmt"
	"sync"
)

func main() {

	var wg sync.WaitGroup
	var once sync.Once

	for i := 0; i < 1000; i++ {
		wg.Add(1)
		go func(i int) {
			defer wg.Done()

			once.Do(func() {
				fmt.Println(i)
			})
		}(i)
	}

	wg.Wait()
}
```

sync.Once 的Do函数需要一个函数变量作为参数，它只会执行首次被调用时传递的那个函数。

### 临时对象池 ###

sync.Pool 提供了一个临时对象池的实现，可以作为一种缓存结构来使用。在 logrus 包中使用了临时对象池，代码如下:

```
var bufferPool *sync.Pool

func init() {
	bufferPool = &sync.Pool{
		New: func() interface{} {
			return new(bytes.Buffer)
		},
	}
}

buffer = bufferPool.Get().(*bytes.Buffer)
buffer.Reset()
defer bufferPool.Put(buffer)
```

1. sync.Pool是一个结构体，初始化时需要为New字段指定一个 func() interface{} 类型的函数
2. 通过Get函数从对象池中获取对象，并进行一定的初始化(如果需要的话)
3. 通过Put函数将对象放回对象池缓存

### 并发安全字典 ###

go语言内建了数据类型map，但是它并不是线程安全的，即在同一时刻让在不同的goroutine中操作map变量是不安全的行为。

sync.Map是Go语言官方发布的标准的并发安全字典，提供了一些常见的键值存取操作方法，并保证了这些操作的并发安全。

一个简单的[例子9](https://play.golang.org/p/sogn0SGTQ1V):

```
package main

import (
	"fmt"
	"sync"
)

func main() {
	var wg sync.WaitGroup
	wg.Add(2)

	var m sync.Map

	go func() {
		defer wg.Done()

		for i := 0; i < 40; {
			v, ok := m.Load(i)
			if !ok {
				continue
			}

			i++
			fmt.Println(v.(int))
		}
	}()
	go func() {
		defer wg.Done()

		for i := 0; i < 40; i++ {
			m.Store(i, i)
		}
	}()


	wg.Wait()
}
```

在使用 sync.Map 的过程中需要注意以下几点:

1. 键的类型不能是函数类型，字段类型和切片类型，因为并发安全字典的内部是使用原生字典进行存储的