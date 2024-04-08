---
title: GMP-启动工作线程M
date: 2024-04-08 15:08:53
categories:
- golang
tags:
- runtime
- gmp
- golang
---

## 前言

> golang version: 1.21

本文分析golang初始化的最后一步，启动一个工作线程M并开启golang的协程调度循环。

## 初始化

/src/runtime/asm_amd64.s:358

~~~plan9_x86
CALL	runtime·mstart(SB)

CALL	runtime·abort(SB)	// mstart should never return
~~~

通过汇编代码能发现，实际调用了runtime.mstart函数启动一个工作线程M，并开始调度循环


## runtime.mstart

mstart函数即为新m的入口点，实际方法使用汇编实现

/src/runtime/proc.go:1520

~~~go
// mstart is the entry-point for new Ms.
// It is written in assembly, uses ABI0, is marked TOPFRAME, and calls mstart0.
func mstart()
~~~

mstart函数的实际实现，调用mstart0方法

/src/runtime/asm_amd64.s:393

~~~plan9_x86
TEXT runtime·mstart(SB),NOSPLIT|TOPFRAME|NOFRAME,$0
    CALL	runtime·mstart0(SB)
    RET // not reached
~~~

## runtime.mstart0

mstart函数即为新m的实际入口点，调用mstart1函数启动m

/src/runtime/proc.go:1530

~~~go
func mstart0() {
    // 获取当前运行的g0
    gp := getg()
    // 初始化g0堆栈
    // 初始时，g0的stack.lo已完成初始化，它不等于0
    osStack := gp.stack.lo == 0
    if osStack {
        size := gp.stack.hi
        if size == 0 {
            size = 8192 * sys.StackGuardMultiplier
        }
        gp.stack.hi = uintptr(noescape(unsafe.Pointer(&size)))
        gp.stack.lo = gp.stack.hi - size + 1024
    }
    // 初始化堆栈保护
    gp.stackguard0 = gp.stack.lo + stackGuard
    gp.stackguard1 = gp.stackguard0
    // 为调度做一些处理工作，并调用schedule开始任务调度
    mstart1()
    
    // Exit this thread.
    // 正常情况下不会走到退出线程逻辑，因为调用调度函数schedule后，该函数不会返回
    if mStackIsSystemAllocated() {
        osStack = true
    }
    // 退出线程
    mexit(osStack)
}
~~~

## runtime.mstart1

为调度做一些处理工作，并调用schedule开始任务调度。该函数不会返回

主要逻辑：

1. 设置g0的调度信息
2. 初始化m信号堆栈，为m0时设置信号处理函数
3. 执行M启动函数（如果存在），启动函数设置位置：[GMP-创建协程#allocm函数](https://www.llyy.ink/golang/6d9c37961205/#newm%E5%87%BD%E6%95%B0)
4. 绑定p（非m0）,调用schedule函数开始协程调度

/src/runtime/proc.go:1573

~~~go
func mstart1() {
    // 初始时，gp为m0中的g0,其他情况下gp也是各个m的g0
    gp := getg()
    
    if gp != gp.m.g0 {
        throw("bad runtime·mstart")
    }
	
    // gp.sched保存了goroutine的调度信息
    // getcallerpc()获取mstart1执行完的返回地址
    // getcallersp()获取调用mstart1时的栈顶地址
    // 保存g0调度信息
    gp.sched.g = guintptr(unsafe.Pointer(gp))
    gp.sched.pc = getcallerpc()
    gp.sched.sp = getcallersp()
    
    asminit()
    // 初始化m,主要设置线程的备用信号堆栈和掩码
    minit()
	
    // 如果当前的m是m0,执行mstartm0操作
    if gp.m == &m0 {
        // 对于初始m,需要设置系统信号量的处理函数
        mstartm0()
    }
    // 执行启动函数
    // 对于m0是没有mstartfn函数
    // 对其他m如果有起始任务函数，则需要执行，比如runtime.main函数中创建sysmon时 /src/runtime/proc.go:170
    // newm函数创建新m时，会设置m0.mstartfn
    // 具体是newm函数调用allocm创建m对象时，设置mstartfn
    if fn := gp.m.mstartfn; fn != nil {
        fn()
    }
    // 如果当前g的m不是m0,它现在还没有p，需要获取一个p
    // m0在调度器初始化中（调用procresize函数初始化p时）已经绑定了allp[0]，所以不用关心m0
    if gp.m != &m0 {
        // 完成gp.m和p的互相绑定
        acquirep(gp.m.nextp.ptr())
        gp.m.nextp = 0
    }
    // 调用调度函数schedule开始协程调度，该函数不会返回
    schedule()
}
~~~

### asminit函数

由汇编实现，amd64架构下是一个空函数

/src/runtime/asm_amd64.s:389

~~~plan9_x86
TEXT runtime·asminit(SB),NOSPLIT,$0-0
	// No per-thread init.
	RET
~~~

### acquirep函数

M 与 P 的绑定过程只是简单的将 P 链表中的 P ，保存到 M 中的 P 指针上。 绑定前，P 的状态一定是 _Pidle，绑定后 P 的状态一定为 _Prunning

/src/runtime/proc.go:5326

~~~go
func acquirep(pp *p) {
    // Do the part that isn't allowed to have write barriers.
    wirep(pp)
	
    pp.mcache.prepareForSweep()
    
    if traceEnabled() {
        traceProcStart()
    }
}

func wirep(pp *p) {
    gp := getg()
    
    if gp.m.p != 0 {
        throw("wirep: already in go")
    }
    // 检查 m 是否正常，并检查要获取的 p 的状态
    if pp.m != 0 || pp.status != _Pidle {
        id := int64(0)
        if pp.m != 0 {
            id = pp.m.ptr().id
        }
        print("wirep: p->m=", pp.m, "(", id, ") p->status=", pp.status, "\n")
        throw("wirep: invalid p state")
    }
    // 将 p 绑定到 m，p 和 m 互相引用
    // gp.m.p = pp
    gp.m.p.set(pp)
    // pp.m = gp.m
    pp.m.set(gp.m)
    pp.status = _Prunning
}
~~~

## m的暂止和复始

stopm函数

当 M 需要被暂止时，可能（因为还有其他暂止 M 的方法）会执行该调用。 此调用会将 M 进行暂止，并阻塞到它被复始时。这一过程就是工作线程的暂止和复始。

/src/runtime/proc.go:2520

~~~go
func stopm() {
    gp := getg()
    
    // ...
	
    lock(&sched.lock)
    // 将 m 放回到空闲列表中，因为我们马上就要暂止了
    mput(gp.m)
    unlock(&sched.lock)
    // 在此阻塞，直到被唤醒
    mPark()
    // 此时已经被唤醒，说明有任务要执行
    // 立即 acquire P
    acquirep(gp.m.nextp.ptr())
    gp.m.nextp = 0
}

func mPark() {
    gp := getg()
    // 暂止当前的 M，在此阻塞，直到被唤醒，等待被其他工作线程唤醒
    // 调用notewakeup唤醒m
    // 如调用startm函数时，sched.midle空闲队列存在m,则取出并调用notewakeup唤醒m proc.go:2649
    notesleep(&gp.m.park)
    // 清除暂止的 note
    noteclear(&gp.m.park)
}

func notesleep(n *note) {
    gp := getg()
    // 必须在g0上执行
    if gp != gp.m.g0 {
        throw("notesleep not on g0")
    }
    ns := int64(-1)
    if *cgo_yield != nil {
        // Sleep for an arbitrary-but-moderate interval to poll libc interceptors.
        ns = 10e6
    }
    for atomic.Load(key32(&n.key)) == 0 {
        gp.m.blocked = true
        // 休眠时间不超过ns，ns<0表示永远休眠
        // futexsleep调用c系统调用futex实现线程阻塞
        // 超过休眠时间时间还未被唤醒（ns>0），则会被定时任务唤醒
        futexsleep(key32(&n.key), 0, ns)
        if *cgo_yield != nil {
            asmcgocall(*cgo_yield, nil)
        }
        gp.m.blocked = false
    }
}
~~~

startm函数分析可以查看： [GMP-创建协程#startm函数](https://www.llyy.ink/golang/6d9c37961205/#startm%E5%87%BD%E6%95%B0)

## 总结

mstart函数是在go程序初始化完成后（或者新m启动时）的入口函数，主要是检查（初始化）m的堆栈信息，设置信号，g0的调度信息。完成后最终调用schedule()函数，正式开始程序的协程任务调度