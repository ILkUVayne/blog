---
title: GMP-调度策略
date: 2024-04-10 14:30:55
categories:
- golang
tags:
- runtime
- gmp
- golang
---

## 前言

> golang version: 1.21

golang在初始化完成之后，就会正式进入协程调度阶段。本文主要分析golang的3种协程调度策略。

## 主动调度

主动调度即当前的g主动放弃CPU执行权，并将其放入到全局可运行队列中。它还是可以继续运行的，只是我们主动放弃了，等待下次被调度执行。

### Gosched

在用户代码中通过调用runtime.Gosched()函数发生主动调度。Gosched函数会调用mcall从当前g的栈切换到m的g0栈执行gosched_m函数

/src/runtime/proc.go:336

~~~go
func Gosched() {
    // checkTimeouts 为空实现
    checkTimeouts()
    // 切换用户栈到g0栈，并执行gosched_m函数
    mcall(gosched_m)
}
~~~

### gosched_m

gosched_m调用goschedImpl挂起用户协程g

/src/runtime/proc.go:3764

~~~go
func gosched_m(gp *g) {
    // 如果跟踪已启用,开启跟踪调度
    if traceEnabled() {
        traceGoSched()
    }
    // gp为准备挂起的当前协程，即调用runtime.Gosched()函数所在的协程g
    goschedImpl(gp)
}
~~~

### goschedImpl

主要逻辑：

1. 修改挂起协程（主动调度的g）的状态为_Grunnable
2. 解除gp和当前m的绑定关系
3. 将gp放入全局可执行队列中去
4. 开启一轮新的调度

/src/runtime/proc.go:3748

~~~go
func goschedImpl(gp *g) {
    // 获取准备挂起g(gp)的状态，它的状态为目前处于_Grunning，因为它正在运行
    status := readgstatus(gp)
    // 检查gp的状态是否合法，如果不为_Grunning状态说明状态出现了异常
    if status&^_Gscan != _Grunning {
        dumpgstatus(gp)
        throw("bad g status")
    }
    // 将gp（用户协程）的状态从_Grunning改变为_Grunnable
    casgstatus(gp, _Grunning, _Grunnable)
    // 解除gp和当前m的绑定关系
    dropg()
    lock(&sched.lock)
    // 将gp放入全局可执行队列中去
    globrunqput(gp)
    unlock(&sched.lock)
    // 开启一轮新的调度
    schedule()
}
~~~

## 被动调度

被动调度即一个g被迫让出CPU执行权。有哪些情况会让出CPU使用权？我们常见的有channel发生阻塞、网络IO阻塞、执行垃圾回收而暂停用户程序执行等。为了提供CPU的使用率，与其发生阻塞导致CPU干等不如让出CPU给其他可以执行的程序使用。

1. 其他协程调用gopark函数，发生被动调度，并维护协程状态
2. 当达成唤醒条件时，调用goready函数，使被阻塞的g重新恢复为可运行状态_Grunnable，并放入p的本地可执行队列中去等待执行
3. 会调用wakep函数尝试唤醒（新建）一个m执行任务

### gopark

发生被动调度时（如channel发生阻塞），会调用gopark函数，使g触发被动调度,该函数会调用mcall函数切换到g0栈，并调用park_m函数

/src/runtime/proc.go:381

~~~go
func gopark(unlockf func(*g, unsafe.Pointer) bool, lock unsafe.Pointer, reason waitReason, traceReason traceBlockReason, traceskip int) {
    if reason != waitReasonSleep {
        checkTimeouts() // timeouts may expire while two goroutines keep the scheduler busy
    }
    mp := acquirem()
    gp := mp.curg
    status := readgstatus(gp)
    if status != _Grunning && status != _Gscanrunning {
        throw("gopark: bad g status")
    }
    mp.waitlock = lock
    mp.waitunlockf = unlockf
    gp.waitreason = reason
    mp.waitTraceBlockReason = traceReason
    mp.waitTraceSkip = traceskip
    releasem(mp)
    // can't do anything that might move the G between Ms here.
    // 切换到g0栈上执行park_m函数
    mcall(park_m)
}
~~~

### park_m

1. 修改gp（当前被动调度的用户协程）状态从_Grunning到_Gwaiting
2. 解绑用户协程和m的绑定关系
3. 如果unlockf函数存在，尝试执行，若返回false，则恢复用户协程状态并执行
4. unlockf函数不存在或者返回true，直接开启一轮新的调度

/src/runtime/proc.go:3721

~~~go
func park_m(gp *g) {
    // g0.m
    mp := getg().m
    
    if traceEnabled() {
        traceGoPark(mp.waitTraceBlockReason, mp.waitTraceSkip)
    }
    
    // N.B. Not using casGToWaiting here because the waitreason is
    // set by park_m's caller.
    // 修改gp（当前被动调度的用户协程）状态从_Grunning到_Gwaiting
    casgstatus(gp, _Grunning, _Gwaiting)
    // 解绑用户协程和m的绑定关系
    dropg()
    // mp.waitunlockf函数是发生被动调度时，调用gopark函数时传入的unlockf
    // 如果mp.waitunlockf函数返回false，则恢复用户协程执行任务，否则开启一轮新的调度
    if fn := mp.waitunlockf; fn != nil {
        ok := fn(gp, mp.waitlock)
        mp.waitunlockf = nil
        mp.waitlock = nil
        if !ok {
            if traceEnabled() {
                traceGoUnpark(gp, 2)
            }
            // 将状态重新修改为_Grunnable，并执行
            casgstatus(gp, _Gwaiting, _Grunnable)
            execute(gp, true) // Schedule it back, never returns.
        }
    }
    // 开启一轮新的调度
    schedule()
}
~~~

### goready

goready完成的功能是将一个g状态修改为_Grunnable，并将它加入待运行队列，等待被调度程序运行

/src/runtime/proc.go:407

~~~go
func goready(gp *g, traceskip int) {
    // 切换到g0栈，并执行ready函数
    systemstack(func() {
        ready(gp, traceskip, true)
    })
}
~~~

### ready

1. 修改gp状态为_Grunnable（可运行状态）
2. 将gp放入本地可运行队列中等待执行
3. 调用wakep函数尝试唤醒（创建）一个新的m执行任务

/src/runtime/proc.go:890

~~~go
func ready(gp *g, traceskip int, next bool) {
    if traceEnabled() {
        traceGoUnpark(gp, traceskip)
    }
    // 获取gp状态
    status := readgstatus(gp)
    
    // Mark runnable.
    mp := acquirem() // disable preemption because it can be holding p in a local var
    // 检查gp的状态是不是_Gwaiting，如果不是说明gp的状态出现了异常
    if status&^_Gscan != _Gwaiting {
        dumpgstatus(gp)
        throw("bad g->status in ready")
    }
    
    // status is Gwaiting or Gscanwaiting, make Grunnable and put on runq
    // 设置gp状态为_Grunnable（可运行状态）
    casgstatus(gp, _Gwaiting, _Grunnable)
    // 将gp放入本地可运行队列中，如果本地队列已满，会放入全局队列
    runqput(mp.p.ptr(), gp, next)
    // 尝试唤醒一个新的m执行任务
    wakep()
    releasem(mp)
}
~~~

## 抢占调度

### sysmon

为了确保每个g都有机会被调度执行，保证调度的公平性，在初始化的时候会启动一个特殊的线程来执行监控任务（sysmon函数），sysmon在runtime.main函数被启动(proc.go:169)。

下面看sysmon具体处理逻辑。sysmon函数是一个死循环函数，在第一轮循环的时候休眠20微妙，之后每轮循环中休眠时间加倍，直到最大休眠时间达到10毫秒。在循环中会检查是否有准备就绪的网络，并将其放入到全局队列中，也会进行抢占处理,按时间强制执行gc以及触发定时gc等操作。这里主要关注抢占处理retake函数，具体怎么抢占都是由该函数实现的。

/src/runtime/proc.go:5515

~~~go
func sysmon() {
	
    // ...
    
    for {
        if idle == 0 { // start with 20us sleep...
            delay = 20
        } else if idle > 50 { // start doubling the sleep after 1ms...
            delay *= 2
        }
        if delay > 10*1000 { // up to 10ms
            delay = 10 * 1000
        }
        usleep(delay)
        
        now := nanotime()
        // gc等操作
        if debug.schedtrace <= 0 && (sched.gcwaiting.Load() || sched.npidle.Load() == gomaxprocs) {
            // ...
        }
        
        // ...
		
        // poll network if not polled for more than 10ms
        lastpoll := sched.lastpoll.Load()
        // 如果超过10ms没有进行netpoll,强制进行一次netpoll，如果获取到了可运行的g,插入g的全局列表
        if netpollinited() && lastpoll != 0 && lastpoll+10*1000*1000 < now {
            sched.lastpoll.CompareAndSwap(lastpoll, now)
            list := netpoll(0) // non-blocking - returns list of goroutines
            if !list.empty() {
                incidlelocked(-1)
                // 把找到的可运行的g列表插入到全局队列
                injectglist(&list)
                incidlelocked(1)
            }
        }
        
        // ...
		
        // retake P's blocked in syscalls
        // and preempt long running G's
        // 重新获取在系统调用中阻塞的p并抢占长时间运行的g
        if retake(now) != 0 {
            idle = 0
        } else {
            idle++
        }
        // check if we need to force a GC
        // 检查是否需要启动垃圾回收程序(定时gc)
        if t := (gcTrigger{kind: gcTriggerTime, now: now}); t.test() && forcegc.idle.Load() {
            // ...
        }
		
        // ...
    }
}
~~~

### retake

- _Prunning

p在_Prunning状态，说明与p关联的m上的g正在被运行，判断g运行时间是否超过10毫秒，如果超过了调用preemptone函数进行抢占。这个抢占是一个异步处理，preemptone设置当前被抢占的g的preempt字段为true,并将stackguard0字段设置为0xfffffade，此时并没有将当前g暂停执行的逻辑。 Go在函数调用之前会设置安全点，这个是编译器为我们插入的代码。在函数调用的时候会检查stackguard0的大小，而决定是否调用runtime.morestack_noctxt函数。morestack_noctxt是汇编编写的，实现如下，它直接调用morestack函数，morestack会将当前调度信息保存起来，然后切换到g0栈执行newstack函数。newstack函数主要完成两项工作：一是检查是否响应sysmon发起的抢占请求，二是检查栈是否需要进行扩容。newstack检查被抢占g的状态，如果处于抢占状态，调用gopreempt_m将被抢占的gp切换出去放入到全局g运行队列。gopreempt_m是goschedImpl的简单包装，真正处理逻辑在goschedImpl函数，逻辑同主动调度

- _Psyscall

p在_Psyscall状态，说明与p关联的m上的g正在进行系统调用。只要下面的三个条件有任何一个满足，就会对处于_Psyscall状态的p进行抢占。

1. p的运行队列中还有等待运行的g，需要及时的让p的本地队列中的g得到调度，而当前的g又处在系统调用中，无法执行其他的g,因此将当前的p抢占过来运行其他的g.
2. 当前没有空闲的p(sched.npidle为0),也没有自旋的m(sched.nmspinning为0)，说明大家都在忙，这时抢占当前正处于系统调用中而实际上用不上的p,分配给其他线程用于调度执行其他的g.
3. 从上一次监控线程留意到p对应的m处于系统调用中到现在已经超过了10毫秒，说明系统调用已花费了比较长的时间，需要进行抢占。使得retake函数的返回值不等于0，让sysmon线程保持20微妙的轮询检查周期，提高监控的实时性。

当上述三个条件有任何一个满足时，会将p的状态从_Psyscall修改为_Pidle，然后调用handoffp函数，将与进入系统调用的m绑定的p分离。handoff会对当前的条件进行检查，如果满足下面的条件则会调用startm函数启动新的工作线程来与当前的p进行关联，执行可运行的g.具体条件有以下几种：

1. pp的本地队列或全局队列中有运行的g,立即启动工作线程对g进行调度
2. 当前处于在垃圾回收标记阶段，启动线程运行gc work
3. 当前没有进行自旋的m也没有空闲的p,启动一个线程m进行自旋
4. 这是最后一个运行的p并且没有人在轮询网络，则需要唤醒另一个m来轮询网络
   最后，如果上述三个条件都满足，说明当前比较空闲，将p放入到P的全局空闲队列中即可。

/src/runtime/proc.go:5672

~~~go
func retake(now int64) uint32 {
    n := 0
    lock(&allpLock)
    // 遍历所有的p,因为m运行g需要绑定一个p,检查每个与p绑定的m中运行的g是否需要被抢占
    for i := 0; i < len(allp); i++ {
        pp := allp[i]
        if pp == nil {
            // 如果procresize已经增长了allp但还没有创建新的Ps，就会发生这种情况。
            continue
        }
        // &pp.sysmontick用于监控线程(sysmon)记录被监控的p的运行时间和系统调用时间
        // 以及运行g时的计数器值和进行系统调用时计数器值
        pd := &pp.sysmontick
        // 获取p的状态保存到s中
        s := pp.status
        sysretake := false
        // 如果当前的p正在运行或者处于系统调用中，需要检查是否需要进行抢占
        if s == _Prunning || s == _Psyscall {
            // Preempt G if it's running for too long.
            // 每进行一次调度，pp.schedtick会+1，也就是每切换一个g运行，schedtick会+1
            t := int64(pp.schedtick)
            // 如果pd.schedtick和t不等，也就是pd.schedtick和_p_.schedtick不等
            // 说明p上有进行过调度操作，即切换过g运行，重新更新pd的值schedtick和schedwhen
            // 为下一次判断做准备
            if int64(pd.schedtick) != t {
                pd.schedtick = uint32(t)
                pd.schedwhen = now
            } else if pd.schedwhen+forcePreemptNS <= now {
                // 走到这里说明pd.schedtick与t相等，说明从pd.schedwhen到now这段时间
                // 没有发生过调度，也就是在这段时间，同一个g一直在运行，检查这个g运行的时间
                // 是否超过了10毫秒，就是schedwhen+forcePreemptNS比当前时间小，说明已经超过了

                // 对与pp绑定的m中运行的g进行抢占
                // 对于 syscall 的情况，preemptone() 不工作，因为 M 没有与 P 绑定，
                // 因为在发起系统调用时，会将m和p解除绑定 
                // 代码实现 proc.go:4045 reentersyscall函数 该函数会在系统调用之前执行
                preemptone(pp)
                // In case of syscall, preemptone() doesn't
                // work, because there is no M wired to P.
                sysretake = true
            }
        }
        // p处在系统调用中，需要检查是否进行抢占
        if s == _Psyscall {
            // Retake P from syscall if it's there for more than 1 sysmon tick (at least 20us).
            // pp.syscalltick 用于记录系统调用的次数，在完成系统调用之后加 1
            t := int64(pp.syscalltick)
            // 如果sysretake为false并且p的pd中记录的系统调用tick和当前p的系统调用tick不等
            // 说明进行过调度，不用进行抢占，直接更新pd的值
            if !sysretake && int64(pd.syscalltick) != t {
                pd.syscalltick = uint32(t)
                pd.syscallwhen = now
                continue
            }
            // 同时满足下面3个条件，不进行抢占：
            // 1. pp的本地运行队列和runnext都没有g
            // 2. 当前进行自旋的m数量+当前空闲的p的数量之和大于0
            // 3. 进入系统调用的时间到现在还不够10毫秒
            if runqempty(pp) && sched.nmspinning.Load()+sched.npidle.Load() > 0 && pd.syscallwhen+10*1000*1000 > now {
                continue
            }
            // Drop allpLock so we can take sched.lock.
            unlock(&allpLock)
            incidlelocked(-1)
            // 将pp的状态从_Psyscall修改为_Pidle
            if atomic.Cas(&pp.status, s, _Pidle) {
                if traceEnabled() {
                    traceGoSysBlock(pp)
                    traceProcStop(pp)
                }
                // n记录处于在系统调用中的p并且需要被抢占的总数
                n++
                // 系统调用tick+1
                pp.syscalltick++
                // 将与进入系统调用的m绑定的p分离
                handoffp(pp)
            }
            incidlelocked(1)
            lock(&allpLock)
        }
    }
    unlock(&allpLock)
    return uint32(n)
}
~~~

### preemptone

分析retake函数能发现，发现运行时间过长g时，是通过调用preemptone实现抢占，该方法是一个异步操作，需要编译器的帮助实现真正的抢占

preemptone主要做了3件事：

1. 设置抢占标志gp.preempt为true
2. 将gp.stackguard0设置为stackPreempt,stackPreempt是一个很大的值，用于判断是否抢占
3. 检查是否支持异步抢占，如果不支持，则只能等待发生函数调用时（此时是协作式调度，是否能够抢占gp不能的到保证，取决于gp是否会有函数调用，例如：`go func (for {}){}()` 这种情况下则会一直阻塞），检查gp并抢占。如果支持，直接向当前P绑定的M发送SIGURG信号，M产生中断并执行回调，即将gp抢占

/src/runtime/proc.go:5769

~~~go
func preemptone(pp *p) bool {
    mp := pp.m.ptr()
    if mp == nil || mp == getg().m {
        return false
    }
    gp := mp.curg
    if gp == nil || gp == mp.g0 {
        return false
    }
    // 设置gp的抢占标识为true,表示该g被抢占
    gp.preempt = true
    
    // Every call in a goroutine checks for stack overflow by
    // comparing the current stack pointer to gp->stackguard0.
    // Setting gp->stackguard0 to StackPreempt folds
    // preemption into the normal stack overflow check.
    // 每次调用会检测栈溢出情况，gp.stackguard0 = stackPreempt即表示为栈已溢出
    // 因为当前方法只修改了抢占表示并未将g暂停执行的逻辑
    // 这么做便于在栈溢出检查时，发现该g已被抢占，执行真正的抢占逻辑
    gp.stackguard0 = stackPreempt
    
    // Request an async preemption of this P.
    // 检查是否支持异步抢占，如果支持，直接向当前P绑定的M发送SIGURG信号
    // 请求该 P 的异步抢占
    if preemptMSupported && debug.asyncpreemptoff == 0 {
        pp.preempt = true
        preemptM(mp)
    }
    
    return true
}
~~~

#### 协作式抢占调度

编译一个简单的go程序

~~~go
package main

func main() {
    println("Hello world!")
}
~~~

编译出汇编代码

~~~bash
$ go tool compile -S entry.go 
main.main STEXT size=55 args=0x0 locals=0x18 funcid=0x0 align=0x0
        0x0000 00000 (/home/luoyang/code/notes/go/src_code/entry.go:3)  TEXT    main.main(SB), ABIInternal, $24-0
        0x0000 00000 (/home/luoyang/code/notes/go/src_code/entry.go:3)  CMPQ    SP, 16(R14)
        0x0004 00004 (/home/luoyang/code/notes/go/src_code/entry.go:3)  PCDATA  $0, $-2
        0x0004 00004 (/home/luoyang/code/notes/go/src_code/entry.go:3)  JLS     48
        0x0006 00006 (/home/luoyang/code/notes/go/src_code/entry.go:3)  PCDATA  $0, $-1
        0x0006 00006 (/home/luoyang/code/notes/go/src_code/entry.go:3)  PUSHQ   BP
        0x0007 00007 (/home/luoyang/code/notes/go/src_code/entry.go:3)  MOVQ    SP, BP
        0x000a 00010 (/home/luoyang/code/notes/go/src_code/entry.go:3)  SUBQ    $16, SP
        0x000e 00014 (/home/luoyang/code/notes/go/src_code/entry.go:3)  FUNCDATA        $0, gclocals·g2BeySu+wFnoycgXfElmcg==(SB)
        0x000e 00014 (/home/luoyang/code/notes/go/src_code/entry.go:3)  FUNCDATA        $1, gclocals·g2BeySu+wFnoycgXfElmcg==(SB)
        0x000e 00014 (/home/luoyang/code/notes/go/src_code/entry.go:4)  PCDATA  $1, $0
        0x000e 00014 (/home/luoyang/code/notes/go/src_code/entry.go:4)  CALL    runtime.printlock(SB)
        0x0013 00019 (/home/luoyang/code/notes/go/src_code/entry.go:4)  LEAQ    go:string."Hello world!\n"(SB), AX
        0x001a 00026 (/home/luoyang/code/notes/go/src_code/entry.go:4)  MOVL    $13, BX
        0x001f 00031 (/home/luoyang/code/notes/go/src_code/entry.go:4)  NOP
        0x0020 00032 (/home/luoyang/code/notes/go/src_code/entry.go:4)  CALL    runtime.printstring(SB)
        0x0025 00037 (/home/luoyang/code/notes/go/src_code/entry.go:4)  CALL    runtime.printunlock(SB)
        0x002a 00042 (/home/luoyang/code/notes/go/src_code/entry.go:5)  ADDQ    $16, SP
        0x002e 00046 (/home/luoyang/code/notes/go/src_code/entry.go:5)  POPQ    BP
        0x002f 00047 (/home/luoyang/code/notes/go/src_code/entry.go:5)  RET
        0x0030 00048 (/home/luoyang/code/notes/go/src_code/entry.go:5)  NOP
        0x0030 00048 (/home/luoyang/code/notes/go/src_code/entry.go:3)  PCDATA  $1, $-1
        0x0030 00048 (/home/luoyang/code/notes/go/src_code/entry.go:3)  PCDATA  $0, $-2
        0x0030 00048 (/home/luoyang/code/notes/go/src_code/entry.go:3)  CALL    runtime.morestack_noctxt(SB)
        0x0035 00053 (/home/luoyang/code/notes/go/src_code/entry.go:3)  PCDATA  $0, $-1
        0x0035 00053 (/home/luoyang/code/notes/go/src_code/entry.go:3)  JMP     0
// ...
~~~

Go在函数调用的时候会设置安全点，这个是编译器为我们插入的代码。在函数调用的时候会检查stackguard0的大小，而决定是否调用runtime.morestack_noctxt函数。morestack_noctxt是汇编编写的，实现如下，它直接调用morestack函数，morestack会将当前调度信息保存起来，然后切换到g0栈执行newstack函数。

> 因为不同的架构汇编代码可能不同，需要使用gdb打断点找到实际的代码位置，自行调试查看即可

runtime.morestack_noctxt函数

/src/runtime/asm_amd64.s:599

~~~plan9_x86
TEXT runtime·morestack_noctxt(SB),NOSPLIT,$0
    MOVL	$0, DX
    JMP	runtime·morestack(SB)
    
TEXT runtime·morestack(SB),NOSPLIT|NOFRAME,$0-0
    // Cannot grow scheduler stack (m->g0).
    get_tls(CX)
    MOVQ	g(CX), BX
    MOVQ	g_m(BX), BX
    MOVQ	m_g0(BX), SI
    CMPQ	g(CX), SI
    JNE	3(PC)
    CALL	runtime·badmorestackg0(SB)
    CALL	runtime·abort(SB)
    
    // Cannot grow signal stack (m->gsignal).
    MOVQ	m_gsignal(BX), SI
    CMPQ	g(CX), SI
    JNE	3(PC)
    CALL	runtime·badmorestackgsignal(SB)
    CALL	runtime·abort(SB)
    
    // Called from f.
    // Set m->morebuf to f's caller.
    NOP	SP	// tell vet SP changed - stop checking offsets
    MOVQ	8(SP), AX	// f's caller's PC
    MOVQ	AX, (m_morebuf+gobuf_pc)(BX)
    LEAQ	16(SP), AX	// f's caller's SP
    MOVQ	AX, (m_morebuf+gobuf_sp)(BX)
    get_tls(CX)
    MOVQ	g(CX), SI
    MOVQ	SI, (m_morebuf+gobuf_g)(BX)
    
    // Set g->sched to context in f.
    // 将当前的调度信息保存到g.sched字段中，具体是将PC、SP、
    // BP寄存器的值和当前g的地址值保存到内存中
    MOVQ	0(SP), AX // f's PC
    MOVQ	AX, (g_sched+gobuf_pc)(SI)
    LEAQ	8(SP), AX // f's SP
    MOVQ	AX, (g_sched+gobuf_sp)(SI)
    MOVQ	BP, (g_sched+gobuf_bp)(SI)
    MOVQ	DX, (g_sched+gobuf_ctxt)(SI)
    
    // Call newstack on m->g0's stack.
    // 下面开始从当前的g切换到g0
    // 将m.g0的地址拷贝到BX寄存器中
    MOVQ	m_g0(BX), BX
    // 将寄存器BX中的值拷贝到tls中，即设置tls中的g为g0
    MOVQ	BX, g(CX)
    // 把g0栈的栈顶寄存器的值恢复到CPU的寄存器SP中，顺利的切换到g0栈
    MOVQ	(g_sched+gobuf_sp)(BX), SP
    MOVQ	(g_sched+gobuf_bp)(BX), BP
    // 调用newstack函数
    CALL	runtime·newstack(SB)
    CALL	runtime·abort(SB)	// crash if newstack returns
    RET
~~~

newstack函数

newstack函数主要完成两项工作：一是检查是否响应sysmon发起的抢占请求，二是检查栈是否需要进行扩容，下面只抽取了与响应抢占相关的代码。newstack检查被抢占g的状态，如果处于抢占状态，调用gopreempt_m将被抢占的gp切换出去放入到全局g运行队列。gopreempt_m是goschedImpl的简单包装，真正处理逻辑在goschedImpl函数，此函数在前面的主动调度中已分析过了，详情见前面的分析。

/src/runtime/stack:966

~~~go
func newstack() {
    // g0
    thisg := getg()
    
    // ...
    
    gp := thisg.m.curg
    
    // ...
    
    stackguard0 := atomic.Loaduintptr(&gp.stackguard0)
    
    preempt := stackguard0 == stackPreempt
    // 保守的对用户态代码进行抢占，而非抢占运行时代码
    // 如果正持有锁、分配内存或抢占被禁用，则不发生抢占
    if preempt {
        // 检查被抢占g的状态,如果不是真正处于抢占状态
        if !canPreemptM(thisg.m) {
            // 不发生抢占，继续调度
            gp.stackguard0 = gp.stack.lo + stackGuard
            // 重新进入调度循环
            gogo(&gp.sched) // never return
        }
    }
    
    // ...
	
    // 如果需要被抢占
    if preempt {
        if gp == thisg.m.g0 {
            throw("runtime: preempt g0")
        }
        if thisg.m.p == 0 && thisg.m.locks == 0 {
            throw("runtime: g is running but p is not")
        }
        
        if gp.preemptShrink {
            gp.preemptShrink = false
            shrinkstack(gp)
        }
        // 表现得像是调用了 runtime.Gosched，主动让权
        if gp.preemptStop {
            preemptPark(gp) // never returns
        }
        
        // Act like goroutine called runtime.Gosched.
        gopreempt_m(gp) // never return
    }
    
    // Allocate a bigger segment and move the stack.
    // 栈扩容的逻辑
    
    // ...
	
    casgstatus(gp, _Gcopystack, _Grunning)
    gogo(&gp.sched)
}
~~~

#### 非协作式（信号）抢占调度

支持异步抢占时，发送抢占信号给当前g->m

~~~go
//...
if preemptMSupported && debug.asyncpreemptoff == 0 {
    pp.preempt = true
    preemptM(mp)
}
//...
~~~

preemptM函数

/src/runtime/signal_unix.go:368

~~~go
func preemptM(mp *m) {
    if GOOS == "darwin" || GOOS == "ios" {
        execLock.rlock()
    }
    
    if mp.signalPending.CompareAndSwap(0, 1) {
        if GOOS == "darwin" || GOOS == "ios" {
            pendingPreemptSignals.Add(1)
        }
        // 给mp发送sigPreempt信号 
        // sigPreempt = _SIGURG
        // _SIGURG = 23
        signalM(mp, sigPreempt)
    }
    
    if GOOS == "darwin" || GOOS == "ios" {
        execLock.runlock()
    }
}
~~~

signalM函数

调用tgkill函数发送信号

/src/runtime/os_linux.go:575

~~~go
func signalM(mp *m, sig int) {
    tgkill(getpid(), int(mp.procid), sig)
}
~~~

tgkill函数

该函数实际是发起一个系统调用，SYS_tgkill系统调用是向一个特定线程发送信号，即向需要发起抢占的m发送抢占信号

/src/runtime/sys_linux_amd64.s:171

~~~plan9_x86
TEXT ·tgkill(SB),NOSPLIT,$0
    MOVQ	tgid+0(FP), DI
    MOVQ	tid+8(FP), SI
    MOVQ	sig+16(FP), DX
    MOVL	$SYS_tgkill, AX
    SYSCALL
    RET
~~~

m初始化，会对信号处理进行初始化

~~~go
// /src/runtime/proc.go:1531
func mstart0() {
    //...
    mstart1()
    
    // ...
    mexit(osStack)
}

// /src/runtime/proc.go:1573
func mstart1() {
    // ...
    if gp.m == &m0 {
        mstartm0()
    }
    //...
    schedule()
}

// /src/runtime/proc.go:1616
func mstartm0() {
    // ...
    initsig(false)
}

// /src/runtime/signal_unix.go:114
// 初始化信号，setsig -> sighandler
// sighandler即收到信号后的处理函数
func initsig(preinit bool) {
    if !preinit {
        signalsOK = true
    }
	
    if (isarchive || islibrary) && !preinit {
        return
    }
    
    for i := uint32(0); i < _NSIG; i++ {
        t := &sigtable[i]
        if t.flags == 0 || t.flags&_SigDefault != 0 {
            continue
        }
        // 此时不需要原子操作，因为此时没有其他运行的 Goroutine
        fwdSig[i] = getsig(i)
        // 检查该信号是否需要设置 sighandler
        // 有些信号不需要使用sighandler函数来进行信号处理
        if !sigInstallGoHandler(i) {
            if fwdSig[i] != _SIG_DFL && fwdSig[i] != _SIG_IGN {
                setsigstack(i)
            } else if fwdSig[i] == _SIG_IGN {
                sigInitIgnored(i)
            }
            continue
        }
        
        handlingSig[i] = 1
        //  通过 setsig 来设置信号对应的动作,即sighandler函数
        setsig(i, abi.FuncPCABIInternal(sighandler))
    }
}
~~~

sighandler函数

当线程 m 接收到信号后，会从用户栈 g 切换到 gsignal 执行信号处理逻辑，即 sighandler 流程，只截取抢占信号处理逻辑

/src/runtime/signal_unix.go:619

~~~go
func sighandler(sig uint32, info *siginfo, ctxt unsafe.Pointer, gp *g) {
    gsignal := getg()
    mp := gsignal.m
    c := &sigctxt{info, ctxt}
    // ...
    // sig == sigPreempt表示当前信号是抢占信号
    if sig == sigPreempt && debug.asyncpreemptoff == 0 && !delayedSignal {
        doSigPreempt(gp, c)
    }
    
    // ...
}
~~~

doSigPreempt函数

/src/runtime/signal_unix.go:341

~~~go
func doSigPreempt(gp *g, ctxt *sigctxt) {
    // 检查当前 G 是否想被抢占
    // 检查当前 G 被抢占是否安全
    if wantAsyncPreempt(gp) {
        // isAsyncSafePoint 中会把一些不应该抢占的场景过滤掉，具体包括：
        // 1.当前代码在汇编编写的函数中执行
        // 2.代码在 runtime，runtime/internal 或者 reflect 包中执行
        // 3.线程处于内存分配中
        if ok, newpc := isAsyncSafePoint(gp, ctxt.sigpc(), ctxt.sigsp(), ctxt.siglr()); ok {
            // Adjust the PC and inject a call to asyncPreempt.
            ctxt.pushCall(abi.FuncPCABI0(asyncPreempt), newpc)
        }
    }
    
    // 确认抢占
    // preemptGen对已完成的抢占信号的数量进行计数
    gp.m.preemptGen.Add(1)
    gp.m.signalPending.Store(0)
    
    if GOOS == "darwin" || GOOS == "ios" {
        pendingPreemptSignals.Add(-1)
    }
}
~~~

pushCall函数

pushCall 相当于将用户将要执行的下一条代码的地址直接 push 到栈上，并 jmp 到指定的 target 地址去执行代码

/src/runtime/signal_amd64.go:80

~~~go
func (c *sigctxt) pushCall(targetPC, resumePC uintptr) {
    // Make it look like we called target at resumePC.
    sp := uintptr(c.rsp())
    sp -= goarch.PtrSize
    *(*uintptr)(unsafe.Pointer(sp)) = resumePC
    c.set_rsp(uint64(sp))
    c.set_rip(uint64(targetPC))
}
~~~

before:

~~~
-----                     PC = 0x123
local var 1
-----
local var 2
----- <---- SP
~~~

after:

~~~
-----                  PC = targetPC
local var 1
-----
local var 2
-----
prev PC = 0x123
----- <---- SP
~~~

targetPC即传入的asyncPreempt

asyncPreempt函数

asyncPreempt 分为上半部分和下半部分，中间被 asyncPreempt2 隔开。上半部分负责将 goroutine 当前执行现场的所有寄存器都保存到当前的运行栈上。

下半部分负责在 asyncPreempt2 返回后将这些现场恢复出来。

/src/runtime/preempt_amd64.s:7

~~~plan9_x86
TEXT ·asyncPreempt(SB),NOSPLIT|NOFRAME,$0-0
    // 保存现场
    
    ...
    
    CALL ·asyncPreempt2(SB)
    // 恢复现场
    
    ...
    
    RET
~~~

asyncPreempt2函数

/src/runtime/preempt.go:301

~~~go
func asyncPreempt2() {
    gp := getg()
    gp.asyncSafePoint = true
    // 这个 preemptStop 是在 GC 的栈扫描中才会设置为 true
    if gp.preemptStop {
        mcall(preemptPark)
    // 其它抢占全部走这条分支
    } else {
        // 切换g0栈调用gopreempt_m函数
        mcall(gopreempt_m)
    }
    gp.asyncSafePoint = false
}
~~~

gopreempt_m函数

/src/runtime/preempt.go:3784

~~~go
func gopreempt_m(gp *g) {
    if traceEnabled() {
        traceGoPreempt()
    }
    // 将gp即需要抢占的g放到全局队列，实现最终的信号抢占调度
    // 剩下的逻辑同主动调度一样
    goschedImpl(gp)
}
~~~

## 总结

协程的调度策略主要分3种：

- 主动调度

主动调度即当前的g主动放弃CPU执行权，并将其放入到全局可运行队列中，等待被下次调度执行

- 被动调度

被动调度即一个g被迫让出CPU执行权。当达成唤醒条件时，调用goready函数，使被阻塞的g重新恢复为可运行状态_Grunnable，并放入p的本地可执行队列中去等待执行

- 抢占调度

抢占调度分为两种情况：

p在_Psyscall状态，说明与p关联的m上的g正在进行系统调用。当不满足特定条件时，就会对处于_Psyscall状态的p进行抢占。并可能调用startm函数启动新的工作线程来与当前的p进行关联，执行可运行的g。

p在_Prunning状态，说明与p关联的m上的g正在被运行，判断g运行时间是否超过10毫秒，如果超过了调用preemptone函数进行抢占。分为两种抢占调度方式：如果不支持异步抢占，则使用协作式抢占调度。反之则使用信号抢占调度。