---
title: GMP-调度器初始化
date: 2024-04-03 13:08:32
categories:
- golang
tags:
- runtime
- gmp
- golang
---

## 前言

> golang version: 1.21

golang在正式开始协程调度之前，要先进行一系列的初始化工作，本文主要分析runtime.rt0_go函数中调度器初始化的大致流程

## 初始化

/src/runtime/asm_amd64.s:347

~~~plan9_x86
// 调用runtime.args
CALL	runtime·args(SB)
// 调用osinit函数，获取CPU的核数，存放在全局变量ncpu中，供后面的调度时使用
CALL	runtime·osinit(SB)
// 调用schedinit进行调度的初始化
CALL	runtime·schedinit(SB)
~~~

上面部分代码展示了runtime.rt0_go函数中调度器初始化的过程，runtime.args和runtime.osinit函数较为简单，主要对一些参数和全局变量进行设置，runtime.schedinit函数即为实际的调度器初始化函数

## runtime.args

函数 args 整理命令⾏参数

~~~go
// argc argv 为全局变量
// /src/runtime/runtime1.go:54
var (
    argc int32
    argv **byte
)

// /src/runtime/runtime1.go:66
func args(c int32, v **byte) {
    argc = c
    argv = v
    sysargs(c, v)
}
~~~

## runtime.osinit

/src/runtime/os_linux.go:346

~~~go

func osinit() {
    // 获取CPU的核数，存放在全局变量ncpu中，供后面的调度时使用
    ncpu = getproccount()
    // 获取physHugePageSize大小,存放在全局变量physHugePageSize中，供后面的调度时使用
    // 获取操作系统重一个物理大页的大小，是2的幂次，
    // 在虚拟内存的管理场景下，使用大页，可以使得大大减少虚拟地址映射表的加载量
    // 减少内核需要加载的页表数量，提高性能
    physHugePageSize = getHugePageSize()
    if iscgo {
        // 初始化cgo参数
        sigdelset(&sigsetAllExiting, 32)
        sigdelset(&sigsetAllExiting, 33)
        sigdelset(&sigsetAllExiting, 34)
    }
    // 架构初始化,在源码中没有看到具体的实现，由编译器注入
    osArchInit()
}

func osArchInit() {}
~~~

## runtime.schedinit

调度器的初始化:

1. 初始化各种锁
2. STW(stop the world)
3. 初始化各种资源,包括堆栈空间、内存分配器（基于tcmalloc的内存分配器）、m0、信号处理以及gc等等的初始化工作
4. 使用osinit函数设置的cpu核数ncpu,创建并初始化所有p
5. STW(start the world)

/src/runtime/proc.go:694

~~~go
func schedinit() {
    // 初始化各种锁
    lockInit(&sched.lock, lockRankSched)
	
    // ...
	
    // 获取当前g,当前只有g0,所以获取的是g0
    gp := getg()
	
    //...
	
    // 设置调度框架最大允许申请的 m对象 个数
    sched.maxmcount = 10000
	
    // stw，停止gc工作
    worldStopped()
    // 各种资源初始化
    moduledataverify()
    //初始化栈空间和栈内存分配
    stackinit()
    // 内存分配器初始化
    mallocinit()
	
    // ...
	
    //获取GODEBUG环境变量，进行cpu参数初始化
    cpuinit(godebug)       // must run before alginit
    
    // ...
    
    // 初始化m0,gp为g0,所以gp.m即为g0.m，g0.m也就是m0
    mcommoninit(gp.m, -1)
	
    // ...

    lock(&sched.lock)
    sched.lastpoll.Store(nanotime())
    // 通过 CPU 核心数和 GOMAXPROCS 环境变量确定 P 的数量
    procs := ncpu
    // GOMAXPROCS大于0时，procs = GOMAXPROCS
    if n, ok := atoi32(gogetenv("GOMAXPROCS")); ok && n > 0 {
        procs = n
    }
    // 调整p的数量，创建并初始化所有的p，所有的p保存在全局变量allp中
    // 完成所有p的创建和初始化
    if procresize(procs) != nil {
        throw("unknown runnable goroutine during bootstrap")
    }
    unlock(&sched.lock)
	
    // stw结束，调度器可以继续执行调度任务
    worldStarted()

    // ...
}
~~~

### mcommoninit函数

初始化m0

设置m0的id字段，每个m有唯一的id标识，并将当前的m插入到全局保存m的链表对象allm中。这个链表是一个单链表，allm始终指向链表头

/src/runtime/proc.go:822

~~~go
func mcommoninit(mp *m, id int64) {
    // 获取当前g -> g0
    gp := getg()
    
    // g0 stack won't make sense for user (and is not necessary unwindable).
    if gp != gp.m.g0 {
        callers(1, mp.createstack[:])
    }
    
    lock(&sched.lock)
    
    // 设置m的id字段
    if id >= 0 {
        mp.id = id
    } else {
        mp.id = mReserveID()
    }
    // m0堆栈信息及其他属性初始化
    // ...
	
    // allm保存了所有的m对象，它是以链表的形式存储的，通过头插法的方式将当前的
    // m(mp)插入到全局链表对象allm中
    mp.alllink = allm
	
    // 更新allm的头节点为当前的m对象mp
    atomicstorep(unsafe.Pointer(&allm), unsafe.Pointer(mp))
    unlock(&sched.lock)
    
    // ...
}

func mReserveID() int64 {
    assertLockHeld(&sched.lock)
    // sched.mnext溢出了
    if sched.mnext+1 < sched.mnext {
        throw("runtime: thread ID overflow")
    }
    id := sched.mnext
    sched.mnext++
    // 检查当前存活的m数量是否超过10000
    checkmcount()
    return id
}
~~~

### procresize函数

完成所有p的创建和初始化,该函数会在golang初始化和程序主动调用GOMAXPROCS函数时被调用

主要步骤：
1. 计算当前真正p的数量nprocs，初始化保存所有p的全局变量allp,allp为一个切片，它里面保存的对象为*p类型，利用make初始化allp.
2. 循环创建nprocs个p对象并初始化它，并把*p对象保存到全局变量allp切片中.
3. 将m0和allp[0]互相绑定，并将allp[0]状态设置为_Prunning
4. 将allp[1:nprocs]中的p放入到全局变量sched.pidle空闲队列中

/src/runtime/proc.go:5183

~~~go
func procresize(nprocs int32) *p {
    assertLockHeld(&sched.lock)
    assertWorldStopped()
    // gomaxprocs int32 runtime2.go:1142
    // 初始时gomaxprocs的值为0
    old := gomaxprocs
    if old < 0 || nprocs <= 0 {
        throw("procresize: invalid arg")
    }
    // ...
	
    // 更新统计信息
    // ...
    
    maskWords := (nprocs + 31) / 32
	
    // allp []*p runtime2.go:1153
    // 初始时，allp数组为空，len(allp)==0,nprocs大于len(allp)
    if nprocs > int32(len(allp)) {
        lock(&allpLock)
        if nprocs <= int32(cap(allp)) {
            // 调整allp的len为nprocs， allp的cap不变
            allp = allp[:nprocs]
        } else {
            // 初始化时会创建新的切片保存p
            nallp := make([]*p, nprocs)
            // 将旧切片allp中的p拷贝到新的切片nallp中
            copy(nallp, allp[:cap(allp)])
            // 将allp替换为新创建的切片nallp
            allp = nallp
        }
        // 调整idlepMask、timerpMask
		
        // ...
		
        unlock(&allpLock)
    }
    
    // initialize new P's
    // 开始时，old为0，即创建初始化所有的p,保存到allp中
    for i := old; i < nprocs; i++ {
        pp := allp[i]
        // pp 存在时，直接复用，不存在时候创建
        if pp == nil {
            // 创建P
            pp = new(p)
        }
        // 对pp字段进行初始化设置
        // proc.go:5052
        pp.init(i)
        // 将pp保存到allp[i]
        atomicstorep(unsafe.Pointer(&allp[i]), unsafe.Pointer(pp))
    }
    
    // g0
    gp := getg()
    // 初始时m都还未绑定p,不会进入到这个分支中，程序启动之后，在设置GOMAXPROCS有可能进入下面的分支
    // func GOMAXPROCS(n int) int {} debug.go:16
    // 调用GOMAXPROCS方法修改p数量时，会调用startTheWorldGC()，该方法会调用procresize
	
    // 初始化的时候会进入else分支
    if gp.m.p != 0 && gp.m.p.ptr().id < nprocs {
        // continue to use the current P
        gp.m.p.ptr().status = _Prunning
        gp.m.p.ptr().mcache.prepareForSweep()
    } else {
        // 但初始化的时候g0.m也就是m0还是未绑定p,即gp.m.p == 0 所以不会走到这个if里面
        if gp.m.p != 0 {
            if traceEnabled() {
                traceGoSched()
                traceProcStop(gp.m.p.ptr())
            }
            gp.m.p.ptr().m = 0
        }
        // 清理m0.p
        gp.m.p = 0
        // 已经根据nprocs初始化了全局p队列allp
        // 从全局p队列allp中取第一个p与m0进行绑定
        pp := allp[0]
        pp.m = 0
        // 设置p的状态为_Pidle
        pp.status = _Pidle
        // 将p就是allp[0]与m0互相绑定，并将p的状态修改为_Prunning
        acquirep(pp)
        if traceEnabled() {
            traceGoStart()
        }
    }
    
    // g.m.p is now set, so we no longer need mcache0 for bootstrapping.
    // 这是一个全局变量，/src/runtime/proc.go:116
    // 在调用mallocinit（初始化内存分配器）的时候被初始化，用于在初始化p（pp.init(i)）的时候，设置pp.mcache
    // p初始化完成后，不再需要mcache0
    mcache0 = nil
	
    // 将多余的p进行销毁，即 allp[nprocs:]
    for i := nprocs; i < old; i++ {
        pp := allp[i]
        pp.destroy()
    }
    
    // Trim allp.
    if int32(len(allp)) != nprocs {
        lock(&allpLock)
        allp = allp[:nprocs]
        idlepMask = idlepMask[:maskWords]
        timerpMask = timerpMask[:maskWords]
        unlock(&allpLock)
    }
    
    var runnablePs *p
    // 将除allp[0]（与m0绑定的p）之外的所有的p放入到空闲链表中
    for i := nprocs - 1; i >= 0; i-- {
        pp := allp[i]
        // 如果当前的p是allp[0],跳过，因为它在前面已经绑定到m0上了
        if gp.m.p.ptr() == pp {
            continue
        }
        // 设置p的状态为_Pidle
        pp.status = _Pidle
        // runqhead uint32
        // runqtail uint32
        // runq     [256]guintptr
        // 判断当前的p的本地队列中是不是没有g，初始时p中是没有g的
        // func runqempty(pp *p) bool {} proc.go:6169
        if runqempty(pp) {
            // p的本地队列没有g，将其放入到全局空闲p队列（sched.pidle）中
            pidleput(pp, now)
        } else {
            // 从全局的空闲m队列（sched.midle）中拿出一个m,将这个m绑定到p上
            pp.m.set(mget())
            // 将所有可运行的p通过p.link串联起来，构成了一个链表结构
            // 例如 allp[1].link --> allp[2]  allp[2].link-->allp[3]
            pp.link.set(runnablePs)
            runnablePs = pp
        }
    }
    // 设置p的协程窃据列表
    stealOrder.reset(uint32(nprocs))
    var int32p *int32 = &gomaxprocs // make compiler check that gomaxprocs is an int32
    // gomaxprocs设置为nprocs
    atomic.Store((*uint32)(unsafe.Pointer(int32p)), uint32(nprocs))
    if old != nprocs {
        // Notify the limiter that the amount of procs has changed.
        gcCPULimiter.resetCapacity(now, nprocs)
    }
    // 返回可运行p链表头节点
    return runnablePs
}
~~~

## 总结

调度器初始化过程：

1. 调用函数 args,整理命令⾏参数
2. 调用函数 osinit,初始化CPU的核数、操作系统重一个物理大页的大小、架构初始化
3. 调用函数 schedinit,设置最大线程数m的数量为10000，初始化m0结构,根据cpu的核数或者设置的GOMAXPROCS完成对应数量p对象的创建和初始化,完成m0和p(allp[0])的互相绑定,至此，g0与m0, m0与p的互相绑定