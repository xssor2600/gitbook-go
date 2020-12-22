#### MPG模型

Goroutine协程是Go提供用户态线程，都是跑在内核级线程之上，当开发者创建了很多goroutine,并且他们都是跑在同一个内核线程P之上，就需要有调度器维护这些goroutine确保所有goroutine都能使用，尽可能公平使用cpu资源。

> 调度器有4个重要组成结构:M,G,P,Sched.

Golang1.1后协程调度模型使用MPG模型。通过引入逻辑Processor来解决GM模型的几个问题。

![image](https://s3.ax1x.com/2020/12/22/rrhgER.png)



##### 基础组成

通过列表分别说明MPG职责：

| 标识              | 数据结构                                 | 数量                                            | 意义                   |
| ----------------- | ---------------------------------------- | ----------------------------------------------- | ---------------------- |
| G (**Goroutine**) | `runtime.g`运行的函数指针，stack上下文   | 每次`go func`都代表一个G无限制                  | 代表一个用户代码执行流 |
| P (**Processor**) | `runtime.p` p er-P的cache,runq和free g等 | 默认为机器核数，可通过`GOMAXPROCES`环境变量调整 | 表示执行所需要的资源   |
| M(**Machine**)    | `runtime.m`对应一个由clone创建的线程     | 比P多，一般不会太多                             | 代表执行者底层OS线程   |

>其中P表示工作底层线程处理协程任务需要的资源（CPU资源），同时也是执行 Go 代码的最大的并行度。



##### 整体调度流程

<img src="https://s3.ax1x.com/2020/12/22/rs1FTx.png" alt="image" style="zoom:50%;" />

从图中可以大概看到	`MPG`之间的交互流程。

1. 不再是单独全局的`runq`,每个P有自己的runq,新的G优先放入自己的runq。满了之后批量放入全局runq。优先从自己的runq中获取G来执行。
2. 实现了`work stealing`,当某个P的runq没有可运行的G时，可以从全局或者其他P获取.
3. 当G因为网络或者锁切换了，那么G和M分离，M通过P调度执行新的G
4. 当M因为系统调用阻塞或者cgo运行一段时间，sysmon协程会将P和M分离，由其他M来结合P进行调度处理。



>**其中G（goroutine）可以通过`go func(){}`来创建，并将G存入P中的runqueue,那么M和P是如何创建的？**

P的创建是在`main`函数启动时候，通过runtime.schedinit创建。

M的创建没有入口直接给开发者创建，底层是通过runtime来创建的：

```go
func newm(fn func(), _p_ *p) {
	mp := allocm(_p_, fn)
	mp.nextp.set(_p_)
	mp.sigmask = initSigmask
	if gp := getg(); gp != nil && gp.m != nil && (gp.m.lockedExt != 0 || gp.m.incgo) && GOOS != "plan9" {
		// We're on a locked M or a thread that may have been
		// started by C. The kernel state of this thread may
		// be strange (the user may have locked it for that
		// purpose). We don't want to clone that into another
		// thread. Instead, ask a known-good thread to create
		// the thread for us.
		//
		// This is disabled on Plan 9. See golang.org/issue/22227.
		//
		// TODO: This may be unnecessary on Windows, which
		// doesn't model thread creation off fork.
		lock(&newmHandoff.lock)
		if newmHandoff.haveTemplateThread == 0 {
			throw("on a locked thread with no template thread")
		}
		mp.schedlink = newmHandoff.newm
		newmHandoff.newm.set(mp)
		if newmHandoff.waiting {
			newmHandoff.waiting = false
			notewakeup(&newmHandoff.wake)
		}
		unlock(&newmHandoff.lock)
		return
	}
	newm1(mp)
}
```



- **Goroutine状态图**

  作为与开发者最近的协程，是我们关注的程序处理逻辑，MPG模型主要的也是协调处理如何分配资源执行处理这些协程。当协程执行流处理不同任务的时候，是`有状态的`，因为需要被中断调度和恢复，必然要保存现场数据，所以是有状态的。

  ![image](https://s3.ax1x.com/2020/12/22/rs15Ax.png)

| G状态      |  值  | 说明                                                         |
| :--------- | :--: | ------------------------------------------------------------ |
| Gidle      |  0   | 刚刚分配，还没有被初始化                                     |
| Grunnable  |  1   | 表示在runqueue上，还没有被运行（`go func(){} | runtime.Gosched() `) |
| Grunning   |  2   | go协程可能在执行代码，不在runqueue上，与M，P已绑定           |
| Gsyscall   |  3   | go协程在执行系统调用，没执行go代码，没有在runqueue上，与M绑定（`readFile`...) |
| Gwaiting   |  4   | go协程被阻塞（IO，GC，chan阻塞，锁等等）不在runqueue上，但是存在某个地方，比如channel,锁排队中(`time.Sleep|conn.Read()..`) |
| Gdead      |  6   | 协程没有在有效执行处理，也许执行完了，或者在free list中。    |
| Gcopystack |  8   | 栈正在复制，此时没有go代码执行，也不在runqueue上。           |
| Gscan      | 0x10 | 入runnable，running，syscall,waiting状态结合，表示GC正在扫描在这个G的栈。 |

**协程只是一个执行流, 并不是运行实体。运行执行线程的实体是M（machine）。**



###### 核心调度

> golang的调度职责在理解了MPG模型后，实质上调度过程就是：
> 为需要执行的Go代码（G）寻找执行者（M）以及执行的准许和资源P。

并没有一个实质的调度器实体，调度是需要发生调度时由m执行`runtime.schedule`方法进行的。

什么时候会发生协程的调度？

- Channel,mutex等sync锁原语操作发生协程阻塞
- time.sleep
- 网络操作暂时未ready
- Gc work
- 主动yield
- 运行超时或者syscall超时
- ...



在通过`newm`接口只是给新创建的M分配了一个空闲的P.通过`acquirep`来将P实例与当前创建的M进行绑定分配。

```go
 else if(m != &runtime·m0) {
	acquirep(m->nextp);
	m->nextp = nil;
}
schedule();
```

为当前M装配上P，nextp就是newm分配的空闲的P，只是到这个时候才真正拿到手罢了。没有P，M是无法执行goroutine的。

工作线程M开始处理的流程：

```go
// One round of scheduler: find a runnable goroutine and execute it.
// Never returns.
//runtime.proc.schedule
func schedule() {
  var gp *g
  if gp == nil {
		gp, inheritTime = runqget(_g_.m.p.ptr())
		// We can see gp != nil here even if the M is spinning,
		// if checkTimers added a local goroutine via goready.
	}
  if gp == nil {
		gp, inheritTime = findrunnable() // blocks until work is available
	}
  // If about to schedule a not-normal goroutine (a GCworker or tracereader),
	// wake a P if there is one.
	if tryWakeP {
		if atomic.Load(&sched.npidle) != 0 && atomic.Load(&sched.nmspinning) == 0 {
			wakep()
		}
	}
  execute(gp, inheritTime)  // 执行G
}
```

1. runqget, 地鼠(M)试图从自己的小车(P)取出一块砖(G)，当然结果可能失败，也就是这个地鼠的小车已经空了，没有砖了。
2. findrunnable, 如果地鼠自己的小车中没有砖，那也不能闲着不干活是吧，所以地鼠就会试图跑去工场仓库（全局`global runqueue`)取一块砖来处理,若是全局G队列也没有任务，则尝试`work stealing`从其他M中获取G处理。若是实在没有G处理，则M进行线程休眠。
3. wakep，M有太多G处理不过来，则唤醒其他M来进行帮忙处理G。
4. execute就是M通过P来执行G流程。



> runtime.park和runtime.gosched两种方式区别？

- goroutine调用park后，这个goroutine就会被设置位waiting状态，放弃cpu。被park的goroutine处于waiting状态，并且与当前P进行解绑。
- runtime·gosched函数也可以让当前goroutine放弃cpu，gosched是将goroutine设置为runnable状态，然后放入到调度器全局等待队列。



> sysmon协程：抢占

在MPG模型引入之后，对P（资源票据）的管理与控制即日益重要，因为P的数量默认是机器CPU核心数，P数量影响了GO代码的协程数（一个P可以有runqueue队列缓存256个G），如果没有机制来对占用P超时处理，则会影响G调度。

`sysmon`协程是在**go runtime初始化之后**, 执行用户编写的代码之前, 由runtime启动的不与任何P绑定, 直接由一个M执行的协程. 类似于linux中的执行一些系统任务的`内核线程`. 



> go的main函数执行时候，主goroutine是如何启动的？

在asm_amd64.s文件中的汇编代码_rt0_amd64就是整个启动过程：

```go
CALL	runtime·args(SB)
CALL	runtime·osinit(SB)
CALL	runtime·hashinit(SB)
CALL	runtime·schedinit(SB)  // 初始化goroutine调度器,创建sysmon

// create a new goroutine to start program
PUSHQ	$runtime·main·f(SB)		// entry
PUSHQ	$0			// arg size
CALL	runtime·newproc(SB)
POPQ	AX
POPQ	AX

// start this M
CALL	runtime·mstart(SB)


```

启动过程做了调度器初始化`runtime·schedinit`后，调用runtime·newproc创建出第一个goroutine，这个goroutine将执行的函数是`runtime·main`,**第一个goroutine也就是所谓的主goroutine**。

**当然任何一个Go程序的入口都是从这个goroutine开始的。最后调用的runtime·mstart就是真正的执行上一步创建的主goroutine。**





- Sysmon执行间隔

  可认为是**10ms执行一次**. (初始运行间隔为20us, sysmon运行1ms后逐渐翻倍, 最终每10ms运行一次. 如果有发生过抢占成功, 则又恢复成初始20us的运行间隔, 如此循环)



