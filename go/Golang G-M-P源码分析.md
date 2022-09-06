# Golang G-M-P源码分析

* 关于GMP讲解的文章已经多的一批了，但总结一下前人们的成果，也能加深一下印象；

* 分析底层代码需要熟悉 **汇编和操作系统**，然而俺都不熟悉，只能跟着前人的步伐磕磕绊绊的走下去（前人的遣词造句都很精妙了，我就直接拿过来）；
* golang版本是1.17
* 引用
  
  https://draveness.me/golang/docs/part3-runtime/ch06-concurrency/golang-goroutine/
  
  https://strikefreedom.top/archives/high-performance-implementation-of-goroutine-pool#toc-head-5
  
  https://learnku.com/articles/41728
  
  https://www.jianshu.com/p/665aca7af949

## 线程的三种实现模型

内核级线程模型、用户级线程模型和两级线程模型（也称混合型线程模型），它们之间最大的差异就在于用户线程与内核调度实体（KSE，Kernel Scheduling Entity）之间的对应关系上。而所谓的内核调度实体 KSE 就是指可以被操作系统内核调度器调度的对象实体。简单来说 KSE 就是内核级线程，是操作系统内核的最小调度单元，也就是我们写代码的时候通俗理解上的线程了。

（记得之前老板问进程和线程的区别：自己的回答是，进程是资源分配的最小单元，线程是cup最小调度单元；老板的回答，进程独享一块内存空间，线程共享进程的内存空间）

### 用户级线程模型

用户线程与内核线程 KSE 是多对一（N : 1）的映射模型，多个用户线程的一般从属于单个进程并且多线程的调度是由用户自己的线程库来完成，线程的创建、销毁以及多线程之间的协调等操作都是由用户自己的线程库来负责而无须借助系统调用来实现。一个进程中所有创建的线程都只和同一个 KSE 在运行时动态绑定，也就是说，操作系统只知道用户进程而对其中的线程是无感知的，内核的所有调度都是基于用户进程。许多语言实现的 协程库 基本上都属于这种方式（比如 python 的 gevent）。

* 优点：由于线程调度是在用户层面完成的，也就是相较于内核调度不需要让 CPU 在用户态和内核态之间切换，这种实现方式相比内核级线程可以做的很轻量级，对系统资源的消耗会小很多，因此可以创建的线程数量与上下文切换所花费的代价也会小得多。
* 缺点：但该模型有个原罪：并不能做到真正意义上的并发，假设在某个用户进程上的某个用户线程因为一个阻塞调用（比如 I/O 阻塞）而被 CPU 给中断（抢占式调度）了，那么该进程内的所有线程都被阻塞（因为单个用户进程内的线程自调度是没有 CPU 时钟中断的，从而没有轮转调度），整个进程被挂起。即便是多 CPU 的机器，也无济于事

因为在用户级线程模型下，一个 CPU 关联运行的是整个用户进程，进程内的子线程绑定到 CPU 执行是由用户进程调度的，内部线程对 CPU 是不可见的，此时可以理解为 CPU 的调度单位是用户进程。所以很多的协程库会把自己一些阻塞的操作重新封装为完全的非阻塞形式（系统调用阻塞不知道咋让出），然后在以前要阻塞的点上，主动让出自己，并通过某种方式通知或唤醒其他待执行的用户线程在该 KSE 上运行，从而避免了内核调度器由于 KSE 阻塞而做上下文切换，这样整个进程也不会被阻塞了。

### 内核级线程模型

用户线程与内核线程 KSE 是一对一（1 : 1）的映射模型，也就是每一个用户线程绑定一个实际的内核线程，而线程的调度则完全交付给操作系统内核去做，应用程序对线程的创建、终止以及同步都基于内核提供的系统调用来完成，大部分编程语言的线程库(比如 Java 的 java.lang.Thread、C++11 的 std::thread 等等)都是对操作系统的线程（内核级线程）的一层封装，创建出来的每个线程与一个独立的 KSE 静态绑定，因此其调度完全由操作系统内核调度器去做，也就是说，一个进程里创建出来的多个线程每一个都绑定一个 KSE。这种模型的优势和劣势同样明显：

* 优点：优势是实现简单，直接借助操作系统内核的线程以及调度器，所以 CPU 可以快速切换调度线程，于是多个线程可以同时运行，因此相较于用户级线程模型它真正做到了并行处理；
* 缺点：但它的劣势是，由于直接借助了操作系统内核来创建、销毁和以及多个线程之间的上下文切换和调度，因此资源成本大幅上涨，且对性能影响很大。

### 两级线程模型

两级线程模型是博采众长之后的产物，充分吸收前两种线程模型的优点且尽量规避它们的缺点。在此模型下，用户线程与内核 KSE 是多对多（N : M）的映射模型：首先，区别于用户级线程模型，两级线程模型中的一个进程可以与多个内核线程 KSE 关联，也就是说一个进程内的多个线程可以分别绑定一个自己的 KSE，这点和内核级线程模型相似；其次，又区别于内核级线程模型，它的进程里的线程并不与 KSE 唯一绑定，而是可以多个用户线程映射到同一个 KSE，当某个 KSE 因为其绑定的线程的阻塞操作被内核调度出 CPU 时，其关联的进程中其余用户线程可以重新与其他 KSE 绑定运行。所以，两级线程模型既不是用户级线程模型那种完全靠自己调度的也不是内核级线程模型完全靠操作系统调度的，而是中间态（自身调度与系统调度协同工作），因为这种模型的高度复杂性，操作系统内核开发者一般不会使用，所以更多时候是作为第三方库的形式出现，而 Go 语言中的 runtime 调度器就是采用的这种实现方案，实现了 Goroutine 与 KSE 之间的动态关联，不过 Go 语言的实现更加高级和优雅；该模型为何被称为两级？即用户调度器实现用户线程到 KSE 的『调度』，内核调度器实现 KSE 到 CPU 上的『调度』。

## go调度器设计原理

今天的 Go 语言调度器有着优异的性能，但是如果我们回头看 Go 语言的 0.x 版本的调度器会发现最初的调度器不仅实现非常简陋，也无法支撑高并发的服务。调度器经过几个大版本的迭代才有今天的优异性能，历史上几个不同版本的调度器引入了不同的改进，也存在着不同的缺陷:

#### 单线程调度器 ·0.x

- 只包含 40 多行代码；
- 程序中只能存在一个活跃线程，由 G-M 模型组成；

#### 多线程调度器 ·1.0

- 允许运行多线程的程序；
- 全局锁导致竞争严重；

#### 任务窃取调度器 ·1.1

- 引入了处理器 P，构成了目前的 **G-M-P** 模型；
- 在处理器 P 的基础上实现了基于**工作窃取**的调度器；
- 在某些情况下，Goroutine 不会让出线程，进而造成饥饿问题；
- 时间过长的垃圾回收（Stop-the-world，STW）会导致程序长时间无法工作；

#### 抢占式调度器 ·1.2 ~ 至今

- 基于协作的抢占式调度器 - 1.2 ~ 1.13
  - 通过编译器在函数调用时插入**抢占检查**指令，在函数调用时检查当前 Goroutine 是否发起了抢占请求，实现基于协作的抢占式调度；
  - Goroutine 可能会因为垃圾回收和循环长时间占用资源导致程序暂停；
- 基于信号的抢占式调度器 - 1.14 ~ 至今
  - 实现**基于信号的真抢占式调度**；
  - 垃圾回收在扫描栈时会触发抢占调度；
  - 抢占的时间点不够多，还不能覆盖全部的边缘情况；

#### 非均匀存储访问调度器 · 提案

- 对运行时的各种资源进行分区；
- 实现非常复杂，到今天还没有提上日程；

除了多线程、任务窃取和抢占式调度器之外，Go 语言社区目前还有一个非均匀存储访问（Non-uniform memory access，NUMA）调度器的提案。在这一节中，我们将依次介绍不同版本调度器的实现原理以及未来可能会实现的调度器提案。

俺只关心G-M-P的调度原理，之前的版本不管它。

## G-P-M 模型概述

每一个 OS 线程都有一个固定大小的内存块(**一般会是 2MB**)来做栈，这个栈会用来存储当前正在被调用或挂起(指在调用其它函数时)的函数的内部变量。这个固定大小的栈同时很大又很小。因为 2MB 的栈对于一个小小的 goroutine 来说是很大的内存浪费，而对于一些复杂的任务（如深度嵌套的递归）来说又显得太小。因此，Go 语言做了它自己的『线程』。

#### G

* G: Goroutine，每个 Goroutine 对应一个 G 结构体，G 存储 Goroutine 的运行堆栈、状态以及任务函数，可重用。G 并非执行体，每个 G 需要绑定到 P 才能被调度执行。
* 在 Go 语言中，每一个 **goroutine** 是一个独立的执行单元，相较于每个 OS 线程固定分配 2M 内存的模式，goroutine 的栈采取了**动态扩容**方式， 初始时仅为 **2KB**，随着任务执行按需增长，最大可达**1GB**（64 位机器最大是 1G，32 位机器最大是 256M），且完全由 golang 自己的调度器 Go Scheduler 来调度。此外，GC 还会周期性地将不再使用的内存回收，收缩栈空间。Go 的创造者大概对 goroutine 的定位就是屠龙刀，因为他们不仅让 goroutine 作为 golang 并发编程的最核心组件（开发者的程序都是基于 goroutine 运行的）而且 golang 中的许多标准库的实现也到处能见到 goroutine 的身影，比如 net/http 这个包，甚至语言本身的组件 **runtime** 运行时和 **GC** 垃圾回收器都是运行在 goroutine 上的，作者对 goroutine 的厚望可见一斑。

#### P

* P: Processor，表示逻辑处理器， 对 G 来说，P 相当于 CPU 核，G 只有绑定到 P才能被调度。对 M 来说，P 提供了相关的执行环境(Context)，如内存分配状态(mcache)，任务队列(G)等，P 的数量决定了系统内最大可并行的 G 的数量（前提：物理 CPU 核数 >= P 的数量），P 的数量由用户设置的 **GOMAXPROCS** 决定，但是不论 GOMAXPROCS 设置为多大，P 的数量最大为 **256**。

#### M

* M: Machine，OS 线程抽象，代表着真正执行计算的资源，在绑定有效的 P 后，进入 schedule 循环；而 schedule 循环的机制大致是从 Global 队列、P 的 Local 队列以及 wait 队列和全局队列中获取 G，切换到 G 的执行栈上并执行 G 的函数，调用 goexit 做清理工作并回到 M，如此反复。M 并不保留 G 状态，这是 G 可以跨 M 调度的基础，M 的数量是不定的，由 Go Runtime 调整，为了防止创建过多 OS 线程导致系统调度不过来，目前默认最大限制为 10000 个。

任何用户线程最终肯定都是要交由 OS 线程来执行的，goroutine也不例外，但是 G 并不直接绑定 OS 线程运行，而是由 Goroutine Scheduler 中的 **P** - Logical Processor （逻辑处理器）来作为两者的『中介』，P 可以看作是一个抽象的资源或者一个上下文，一个 P 绑定一个 OS 线程，在 golang 的实现里把 OS 线程抽象成一个数据结构：**M**，G 实际上是由 M 通过 P 来进行调度运行的，但是在 G 的层面来看，P 提供了 G 运行所需的一切资源和环境，因此在 G 看来 P 就是运行它的 “CPU”，由 G、P、M 这三种由 Go 抽象出来的实现，最终形成了 Go 调度器的基本结构

## 关键数据结构

#### G 结构体定义和各个状态

```go
/// runtime/runtime2.go 关键字段
type g struct {
    stack       stack   // g自己的栈

    m            *m      // 执行当前g的m
    sched        gobuf   // 保存了g的现场，goroutine切换时通过它来恢复
    atomicstatus uint32  // g的状态Gidle,Grunnable,Grunning,Gsyscall,Gwaiting,Gdead
    goid         int64
    schedlink    guintptr // 下一个g, g链表

    preempt       bool //抢占标记

    lockedm        muintptr // 锁定的M,g中断恢复指定M执行
    gopc           uintptr  // 创建该goroutine的指令地址
    startpc        uintptr  // goroutine 函数的指令地址
}	
```

结构体 `runtime.g` 的 `atomicstatus` 字段存储了当前 Goroutine 的状态。除了几个已经不被使用的以及与 GC 相关的状态之外，Goroutine 可能处于以下 9 种状态

| 状态          | 描述                                                         |
| ------------- | ------------------------------------------------------------ |
| `_Gidle`      | 刚刚被分配并且还没有被初始化                                 |
| `_Grunnable`  | 没有执行代码，没有栈的所有权，存储在运行队列中               |
| `_Grunning`   | 可以执行代码，拥有栈的所有权，被赋予了内核线程 M 和处理器 P  |
| `_Gsyscall`   | 正在执行系统调用，拥有栈的所有权，没有执行用户代码，被赋予了内核线程 M 但是不在运行队列上 |
| `_Gwaiting`   | 由于运行时而被阻塞，没有执行用户代码并且不在运行队列上，但是可能存在于 Channel 的等待队列上 |
| `_Gdead`      | 没有被使用，没有执行代码，可能有分配的栈                     |
| `_Gcopystack` | 栈正在被拷贝，没有执行代码，不在运行队列上                   |
| `_Gpreempted` | 由于抢占而被阻塞，没有执行用户代码并且不在运行队列上，等待唤醒 |
| `_Gscan`      | GC 正在扫描栈空间，没有执行代码，可以与其他状态同时存在      |

#### M: runtime.m

```go
/// runtime/runtime2.go 关键字段
type m struct {
    g0      *g     // g0, 每个M都有自己独有的g0, g0是持有调度栈的Goroutine
    curg    *g     // 是在当前线程上运行的用户 Goroutine
    
    p             puintptr // 当前用于的p
    nextp         puintptr // 当m被唤醒时，首先拥有这个p
    id            int64
    spinning      bool // 是否处于自旋

    park          note
    alllink       *m // on allm
    schedlink     muintptr // 下一个m, m链表
    mcache        *mcache  // 内存分配
    lockedg       guintptr // 和 G 的lockedm对应
    freelink      *m // on sched.freem

}
```

#### P

```
/// runtime/runtime2.go 关键字段
type p struct {
    id          int32
    status      uint32 // 状态
    link        puintptr // 下一个P, P链表
    m           muintptr // 拥有这个P的M
    mcache      *mcache  

    // P本地runnable状态的G队列
    runqhead uint32
    runqtail uint32
    runq     [256]guintptr
    
    runnext guintptr // 一个比runq优先级更高的runnable G

    // 状态为dead的G链表，在获取G时会从这里面获取
    gFree struct {
        gList
        n int32
    }

    gcBgMarkWorker       guintptr // (atomic)
    gcw gcWork

}
```

`runtime.p` 结构体中的状态 `status` 字段会是以下五种中的一种：

| 状态        | 描述                                                         |
| ----------- | ------------------------------------------------------------ |
| `_Pidle`    | 处理器没有运行用户代码或者调度器，被空闲队列或者改变其状态的结构持有，运行队列为空 |
| `_Prunning` | 被线程 M 持有，并且正在执行用户代码或者调度器                |
| `_Psyscall` | 没有执行用户代码，当前线程陷入系统调用                       |
| `_Pgcstop`  | 被线程 M 持有，当前处理器由于垃圾回收被停止                  |
| `_Pdead`    | 当前处理器已经不被使用                                       |

#### schedt : 全局调度器，主要存储了一些空闲的G、M、P

```go
/// runtime/runtime2.go 关键字段
type schedt struct {

    lock mutex

    midle        muintptr // 空闲M链表
    nmidle       int32    // 空闲M数量
    nmidlelocked int32    // 被锁住的M的数量
    mnext        int64    // 已创建M的数量，以及下一个M ID
    maxmcount    int32    // 允许创建最大的M数量
    nmsys        int32    // 不计入死锁的M数量
    nmfreed      int64    // 累计释放M的数量

    pidle      puintptr // 空闲的P链表
    npidle     uint32   // 空闲的P数量

    runq     gQueue // 全局runnable的G队列
    runqsize int32  // 全局runnable的G数量

    // Global cache of dead G's.
    gFree struct {
        lock    mutex
        stack   gList // Gs with stacks
        noStack gList // Gs without stacks
        n       int32
    }

    // freem is the list of m's waiting to be freed when their
    // m.exited is set. Linked through m.freelink.
    freem *m
}
```

## 程序启动

 下面俺以一段代码开启抄袭式分析

```go
func main() {
	fmt.Println("start main ...")
  go hello()
	time.Sleep(time.Second)
}

func hello() {
  fmt.Println("func hello ...")
  go func {
    fmt.Println("func world ...")
  }()
}
// 设置GOMAXPROCS = 4
```

代码文件 **runtime/asm_amd64.s, runtime/proc.go, runtime/runtime2.go**

```go
// set the per-goroutine and per-mach "registers"
	get_tls(BX)
	LEAQ	runtime·g0(SB), CX
	MOVQ	CX, g(BX)
	LEAQ	runtime·m0(SB), AX

	// save m->g0 = g0
	MOVQ	CX, m_g0(AX)
	// save m0 to g0->m
	MOVQ	AX, g_m(CX)

	CLD				// convention is D is always left cleared
	CALL	runtime·check(SB)

	MOVL	16(SP), AX		// copy argc
	MOVL	AX, 0(SP)
	MOVQ	24(SP), AX		// copy argv
	MOVQ	AX, 8(SP)
	CALL	runtime·args(SB)
	CALL	runtime·osinit(SB)
	CALL	runtime·schedinit(SB)

	// create a new goroutine to start program
	MOVQ	$runtime·mainPC(SB), AX		// entry
	PUSHQ	AX
	PUSHQ	$0			// arg size
	CALL	runtime·newproc(SB)
	POPQ	AX
	POPQ	AX

	// start this M
	CALL	runtime·mstart(SB)

	CALL	runtime·abort(SB)	// mstart should never return
	RET

	// Prevent dead-code elimination of debugCallV2, which is
	// intended to be called by debuggers.
	MOVQ	$runtime·debugCallV2<ABIInternal>(SB), AX
	RET
```

> 1.**创建g0**：G0 是每次启动一个 M 都会第一个创建的 goroutine，G0 仅用于负责调度的 G，G0 不指向任何可执行的函数，每个 M 都会有一个自己的 G0。在调度或系统调用时会使用 G0 的栈空间，全局变量的 G0 是 M0 的 G0
>
> 2.**创建m0**：M0 是启动程序后的编号为 0 的主线程，这个 M 对应的实例会在全局变量 runtime.m0 中，不需要在 heap 上分配，M0 负责执行初始化操作和启动第一个 G， 在之后 M0 就和其他的 M 一样了。
>
> 3.**m0.g0 = g0**
>
> 4.**g0.m = m0**
>
> 5.**命令行初始化，os初始化**
>
> 6.**schedinit 调度器初始化**
>
> 7.**newproc 将runtime.main 作为参数创建goroutine**
>
> 8.**mstart**

### schedinit 调度器初始化

调度器的启动过程是我们平时比较难以接触的过程，不过作为程序启动前的准备工作，理解调度器的启动过程对我们理解调度器的实现原理很有帮助，运行时通过 `runtime.schedinit` 初始化调度器：

```go
///runtime/proc.go
func schedinit() {

    // 获取g0
    _g_ := getg() 
    ...
    // 设置最大 M 数量
    sched.maxmcount = 10000    
    ...    
    // 栈和内存初始化
    stackinit()
    mallocinit()
    ...
    // 初始化当前 M
    mcommoninit(_g_.m)   
    ...    
    //参数和环境初始化
    goargs()
    goenvs()
    ...
 	  lock(&sched.lock)
  	sched.lastpoll = uint64(nanotime())
    // 设置 P 的数量
    procs := ncpu
    // 通过环境变量设置P的数量
    if n, ok := atoi32(gogetenv("GOMAXPROCS")); ok && n > 0 {
        procs = n
    }  
  	// 将m0和p绑定
  	if procresize(procs) != nil {
			throw("unknown runnable goroutine during bootstrap")
		}
		unlock(&sched.lock)
    ...
}
```

> 在调度器初始函数执行的过程中会将 `maxmcount` 设置成 10000，这也就是一个 Go 语言程序能够创建的最大线程数，虽然最多可以创建 10000 个线程，但是可以同时运行的线程还是由 `GOMAXPROCS` 变量控制。
>
> 我们从环境变量 `GOMAXPROCS` 获取了程序能够同时运行的最大处理器数之后就会调用 `runtime.procresize` 更新程序中处理器P的数量，在这时整个程序不会执行任何用户 Goroutine，调度器也会进入锁定状态，`runtime.procresize` 的执行过程如下：
>
> 1. 如果全局变量 `allp` 切片中的处理器数量少于期望数量，会对切片进行扩容；
> 2. 使用 `new` 创建新的处理器结构体并调用  `runtime.p.init`  初始化刚刚扩容的处理器；
> 3. 通过指针将线程 m0 和处理器 `allp[0]` 绑定到一起；
> 4. 调用 `runtime.p.destroy` 释放不再使用的处理器结构；
> 5. 通过截断改变全局变量 `allp` 的长度保证与期望处理器数量相等；
> 6. 将除 `allp[0]` 之外的处理器 P 全部设置成 `_Pidle` 并加入到全局的空闲队列中；
>
> 调用 `runtime.procresize` 是调度器启动的最后一步，在这一步过后调度器会完成相应数量处理器的启动，等待用户创建运行新的 Goroutine 并为 Goroutine 调度处理器资源

### newproc 创建goroutine

`runtime.newproc` 调用 `runtime.newproc1` 函数获取新的 Goroutine 结构体、将其加入处理器P的运行队列并在满足条件时调用 `runtime.wakep` 唤醒新的处理器P执行 Goroutine：

```go
func newproc(siz int32, fn *funcval) {
  // sys.PtrSize = 8, 表示跳过函数指针， 获取第一个参数的地址
	argp := add(unsafe.Pointer(&fn), sys.PtrSize)
	gp := getg() //获取当前g0，程序刚启动，所以获取的是全局的g0
	pc := getcallerpc()
	systemstack(func() {
		newg := newproc1(fn, argp, siz, gp, pc)

		_p_ := getg().m.p.ptr()
		runqput(_p_, newg, true) //将G放入队列...

		if mainStarted { //这里判断是main线程开启的goroutine还是其他goroutine开启的goroutine
			wakep() //如果是main线程，那么会进行一系列的p的操作
		}
	})
}

func newproc1(fn *funcval, argp unsafe.Pointer, narg int32, callergp *g, callerpc uintptr) *g {
  ...
	_g_ := getg() //获取g0
  ...
	_p_ := _g_.m.p.ptr() 	// 获取P
	newg := gfget(_p_) 		// 在P或者sched中获取空闲的G, 在这里也就是主goroutine
	if newg == nil {	 		// 程序刚起动没有可用的goroutine，创建一个新的
		newg = malg(_StackMin)
		casgstatus(newg, _Gidle, _Gdead)
		allgadd(newg) // publishes with a g->status of Gdead so GC scanner doesn't look at uninitialized stack.
	}
	// 将runtime.main地址存储在主goroutine的sched中
  gostartcallfn(&newg.sched, fn)    
  ...
  runqput(_p_, newg, true) // 将runnable的newg放入P中
 	...
	return newg
}

```

### mstart 启动主gorutine

在第7步，newproc为runtime.main 创建了main goroutine，main goroutine运行runtime.main，runtime.main会调用我们自己写的main函数

```go
//runtime.main
func main() {
    ...
    //  64位系统 栈的最大空间为 1G, 32为系统 为 250M
    if sys.PtrSize == 8 {
        maxstacksize = 1000000000
    } else {
        maxstacksize = 250000000
    }
    ...    
    fn := main_main // 这就是我们代码main包的main函数
    fn() // 运行我们的main函数    
    ...   
}

func mstart() // 空实现，会调用mstart0
func mstart0() { //最后会调用到mstart1
	_g_ := getg()
  ...
	mstart1()
	...
	mexit(osStack)
}

func mstart1() {
	_g_ := getg()

	if _g_ != _g_.m.g0 { //如果获取的g不是g0的话，直接挂掉
		throw("bad runtime·mstart")
	}
	...
	schedule()
}

```

开始调度执行(大量g0的操作)

```go
func schedule() {
    _g_ := getg() //g0
    ...
top:
    pp := _g_.m.p.ptr()   
    ...
    var gp *g    
    ...    
    // 从sched或者P获取G,启动阶段至此一共产生了两个G:g0和main的G。g0不会存在sched和P中，所以这里获取的是main的G
    if gp == nil {
        if _g_.m.p.ptr().schedtick%61 == 0 && sched.runqsize > 0 {
            lock(&sched.lock)
            gp = globrunqget(_g_.m.p.ptr(), 1)
            unlock(&sched.lock)
        }
    }
    if gp == nil {
        gp, inheritTime = runqget(_g_.m.p.ptr())
    }
    if gp == nil {
        gp, inheritTime = findrunnable() // blocks until work is available
    }
		...
    execute(gp, inheritTime)
}


func execute(gp *g, inheritTime bool) {
    ...
    // 在上面我们已经将runtime.main的地址存在gp.sched中。这里就调用runtime.main。
    gogo(&gp.sched)
}
```

### 启动总结

> 通过汇编创建全局的g0，m0，将g0和m0关联起来
>
> schedinit调度器初始化，根据GOMAXPROCS初始化P，将m0和allp[0] （P）绑定
>
> 为runtime.main 生成G，放到 P的队列中
>
> 最后进入schedule，调用过程主要依靠g0，如果主线程不阻塞，执行完就退出了。

## G-P-M 调度分析

> 上面只是启动过程，中间涉及的函数调用省略了很多实现细节，接下来介绍 **go func** 开启的调度，会详细分析上面函数的实现细节；
>
> 我们在main函数中 go func 启动了一个goroutine hello，编译器会调用runtime.newproc函数

### G 创建

```go
func newproc(siz int32, fn *funcval) {
  // sys.PtrSize = 8, 表示跳过函数指针， 获取第一个参数的地址
	argp := add(unsafe.Pointer(&fn), sys.PtrSize)
	gp := getg() //获取当前g0，main函数启动的hello，所以此时g0是全局的g0；world是hello启动的g0是和执行word线程的M关联的g0
	pc := getcallerpc()
	systemstack(func() {
		newg := newproc1(fn, argp, siz, gp, pc)

		_p_ := getg().m.p.ptr()
		runqput(_p_, newg, true) //将G放入队列...

		if mainStarted { //这里判断是main线程开启的goroutine还是其他goroutine开启的goroutine
			wakep() //如果是main线程，那么会进行一系列的p的操作
		}
	})
}
```

