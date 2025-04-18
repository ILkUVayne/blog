---
title: GMP-创建协程
date: 2024-04-07 14:55:01
categories:
- golang
tags:
- runtime
- gmp
- golang
- goroutine
---

## 前言

> golang version: 1.21

本文主要分析golang创建协程的实现方式和过程，无论是golang初始化过程中创建的runtime.main协程还是代码中使用go关键字创建的协程都是调用相同的函数实现协程创建。

## 创建协程

### golang初始化

/src/runtime/asm_amd64.s:354

~~~plan9_x86
CALL	runtime·newproc(SB)
~~~

golang初始化过程中创建的runtime.main协程实际调用runtime.newproc函数

### go关键字

~~~go
func main() {
    go func() {
        println("Hello world!")
    }()
}
~~~

编译上述代码：

~~~bash
$ go tool compile -S main.go
main.main STEXT size=40 args=0x0 locals=0x10 funcid=0x0 align=0x0
        0x0000 00000 (/home/luoyang/code/notes/go/project/go_assembly/main.go:3)        TEXT    main.main(SB), ABIInternal, $16-0
        0x0000 00000 (/home/luoyang/code/notes/go/project/go_assembly/main.go:3)        CMPQ    SP, 16(R14)
        0x0004 00004 (/home/luoyang/code/notes/go/project/go_assembly/main.go:3)        PCDATA  $0, $-2
        0x0004 00004 (/home/luoyang/code/notes/go/project/go_assembly/main.go:3)        JLS     33
        0x0006 00006 (/home/luoyang/code/notes/go/project/go_assembly/main.go:3)        PCDATA  $0, $-1
        0x0006 00006 (/home/luoyang/code/notes/go/project/go_assembly/main.go:3)        PUSHQ   BP
        0x0007 00007 (/home/luoyang/code/notes/go/project/go_assembly/main.go:3)        MOVQ    SP, BP
        0x000a 00010 (/home/luoyang/code/notes/go/project/go_assembly/main.go:3)        SUBQ    $8, SP
        0x000e 00014 (/home/luoyang/code/notes/go/project/go_assembly/main.go:3)        FUNCDATA        $0, gclocals·g2BeySu+wFnoycgXfElmcg==(SB)
        0x000e 00014 (/home/luoyang/code/notes/go/project/go_assembly/main.go:3)        FUNCDATA        $1, gclocals·g2BeySu+wFnoycgXfElmcg==(SB)
        0x000e 00014 (/home/luoyang/code/notes/go/project/go_assembly/main.go:13)       LEAQ    main.main.func1·f(SB), AX
        0x0015 00021 (/home/luoyang/code/notes/go/project/go_assembly/main.go:13)       PCDATA  $1, $0
        0x0015 00021 (/home/luoyang/code/notes/go/project/go_assembly/main.go:13)       CALL    runtime.newproc(SB)
        0x001a 00026 (/home/luoyang/code/notes/go/project/go_assembly/main.go:16)       ADDQ    $8, SP
        0x001e 00030 (/home/luoyang/code/notes/go/project/go_assembly/main.go:16)       POPQ    BP
        0x001f 00031 (/home/luoyang/code/notes/go/project/go_assembly/main.go:16)       NOP
        0x0020 00032 (/home/luoyang/code/notes/go/project/go_assembly/main.go:16)       RET
        ...
~~~

从编译结果能发现，go关键字最终会也被编译为 **CALL    runtime.newproc(SB)** ，即调用runtime.newproc函数创建协程

## runtime.newproc

newproc函数是协程创建的入口函数

主要逻辑：

1. newproc函数获取当前正在运行的g，获取调用方 PC/IP 寄存器值
2. 切换 g0 系统栈调用newproc1函数创建 Goroutine 对象
3. 将newproc1函数创建的可执行g依据优先顺序P.runnext、P.runq、sched.runq存放到任务队列
4. mainStarted == true 时(main函数已经开始执行)，尝试唤醒一个m执行g

/src/runtime/proc.go:4477

~~~go
func newproc(fn *funcval) {
    // 程序运行时使用 go func 创建协程时，获取正在运行的g
    // 初始运行是main goroutine，gp就是m0中的g0
    gp := getg()
    // 获取调用方 PC/IP 寄存器值
    // 将调用newproc时由call指令压栈的函数返回地址，初始时，pc的值是
    // CALL runtime.newproc(SB)指令后面的指令的地址
    pc := getcallerpc()
    // systemstack表示切换到g0栈运行，初始时执行到这里的时候已经在g0栈，
    // 所以直接调用newproc1
    systemstack(func() {
        // 传递的参数包括 fn 函数入口地址, argp 参数起始地址, gp（g0），调用方 pc（goroutine）
        newg := newproc1(fn, gp, pc)
        
        pp := getg().m.p.ptr()
        // 将这里新创建的 g 放入 p 的本地队列或直接放入全局队列
        // true 表示放入执行队列的下一个(P.runnext)，false 表示放入队尾
        // 任务队列分为三级，按优先级从⾼到低分别是 P.runnext、P.runq、Sched.runq
        runqput(pp, newg, true)
        
        // 唤醒一个m来运行g,初始时不会执行，因为mainStarted为false,即runtime包中的main函数还未执行
        // runtime.main函数执行后mainStarted会设置为true proc.go:166
        // mainStarted==true 就会调用wakep()，尝试唤醒（创建）一个m来执行任务（充分利用多核优势）
        // wakep()并不能百分百唤醒(创建)一个m，例如当前没有空闲的p时
        if mainStarted {
            wakep()
        }
    })
}

type funcval struct {
    fn uintptr
    // 变长大小，fn 的数据在应在 fn 之后
}

// getcallerpc 返回它调用方的调用方程序计数器 PC program conter
//go:noescape
func getcallerpc() uintptr
~~~

### runqput函数

runqput函数

主要步骤：

1. 如果next==true,尝试优先将g放入p.runnext 作为下一个优先执行任务,若p.runnext不为空，则进行交换，老的值放入pp.runq队尾
2. 如果next==false,尝试优先将g放入pp.runq本地可运行队列队尾
3. 如果本地队列已满（本地队列长度256 runtime2.go:643），将当前P中前len(p)/2加上当前g一起放到全局可运行队列sched.runq中去

/src/runtime/proc.go:6200

~~~go
func runqput(pp *p, gp *g, next bool) {
    if randomizeScheduler && next && fastrandn(2) == 0 {
        next = false
    }
    // 如果可能（next==true），直接将g放在p.runnext，作为下一个优先执行任务
    if next {
    retryNext:
        // 对pp.runnext进行备份
        oldnext := pp.runnext
        // 通过cas操作，将gp和oldnext进行交换
        if !pp.runnext.cas(oldnext, guintptr(unsafe.Pointer(gp))) {
            goto retryNext
        }
        // 如果oldnext为0，说明pp.runnext之前没有g,现在已放入完毕，直接返回
        if oldnext == 0 {
            return
        }
        // Kick the old runnext out to the regular run queue.
        // 将之前的g赋值给gp,下面会将gp放入pp的本地队列
        gp = oldnext.ptr()
    }

retry:
    // h为本地队列的队头
    h := atomic.LoadAcq(&pp.runqhead) // load-acquire, synchronize with consumers
    // t为本地队列的队尾
    t := pp.runqtail
    // t-h小于队列的长度，即本地队列还没有满，放到本地队列的尾部
    if t-h < uint32(len(pp.runq)) {
        pp.runq[t%uint32(len(pp.runq))].set(gp)
        atomic.StoreRel(&pp.runqtail, t+1) // store-release, makes the item available for consumption
        return
    }
    // 走到这里说明本地队列满了，放到全局队列， 放入到全局队列并不是一个，而是将当前P中前len(p)/2个G批量放入到global queue中
    if runqputslow(pp, gp, h, t) {
        return
    }
    // the queue is not full, now the put above must succeed
    // 走到这里说明往全局队列中没有放成功，没有成功的原因是，本地队列没有满，所以进一步重试，
    // 尝试放入本地队列
    goto retry
}
~~~

runqputslow函数

将gp和本地可运行队列中的一批g放到全局队列中

/src/runtime/proc.go:6235

~~~go
func runqputslow(pp *p, gp *g, h, t uint32) bool {
    // 存储要移动的g,移动的数量为本地队列的一半+1个，这里的1是为传入的gp分配一个位置
    var batch [len(pp.runq)/2 + 1]*g
    
    // First, grab a batch from local queue.
    n := t - h
    n = n / 2
    // 确保n为本地队列的长度的一半
    if n != uint32(len(pp.runq)/2) {
        throw("runqputslow: queue is not full")
    }
    // 将队头的n个g存储到batch中
    for i := uint32(0); i < n; i++ {
        batch[i] = pp.runq[(h+i)%uint32(len(pp.runq))].ptr()
    }
    // 原子更新pp队列队头的位置，队头向后移动n个位置
    if !atomic.CasRel(&pp.runqhead, h, h+n) { // cas-release, commits consume
        return false
    }
    // 将传入runqputslow的gp放入到batch的末尾
    batch[n] = gp
    // 是否需要要随机化调度，打乱batch中元素的顺序，默认false
    if randomizeScheduler {
        for i := uint32(1); i <= n; i++ {
            j := fastrandn(i + 1)
            batch[i], batch[j] = batch[j], batch[i]
        }
    }
    
    // Link the goroutines.
    // 将batch中的g串起来，构成一个链表，因为batch中有n+1元素
    // 所以这里循环n次，就将n+1个组成了链表结构
    for i := uint32(0); i < n; i++ {
        batch[i].schedlink.set(batch[i+1])
    }
    var q gQueue
    // 将链表的头节点和尾节点加入到q中，方便一次性将batch中的g加入到全局队列
    q.head.set(batch[0])
    q.tail.set(batch[n])
    
    // Now put the batch on global queue.
    // 将g一次性批量放入全局队列
    lock(&sched.lock)
    globrunqputbatch(&q, int32(n+1))
    unlock(&sched.lock)
    return true
}
~~~

globrunqputbatch函数

将一批可运行goroutine放入全局可运行队列

/src/runtime/proc.go:5983

~~~go
func globrunqputbatch(batch *gQueue, n int32) {
    assertLockHeld(&sched.lock)
    // 将batch一次性放入sched.runq中
    sched.runq.pushBackAll(*batch)
    // 更新sched.runq中g的数量
    sched.runqsize += n
    *batch = gQueue{}
}
~~~

pushBackAll函数

将q2链表中的g加入到全局队列中

/src/runtime/proc.go:5983

~~~go
func (q *gQueue) pushBackAll(q2 gQueue) {
    if q2.tail == 0 {
        return
    }
    q2.tail.ptr().schedlink = 0
    // 直接将全局队列的队尾连接到q2的头节点，这样q2就加入了全局g链表中
    if q.tail != 0 {
        q.tail.ptr().schedlink = q2.head
    } else {
        q.head = q2.head
    }
    // 更新全局链表尾节点的位置，指向q2的尾部
    q.tail = q2.tail
}
~~~

### wakep函数

wakep函数尝试唤醒（新建）M执⾏任务，发挥多核优势。若当前不存在空闲的p时，唤醒失败。

/src/runtime/proc.go:2729

~~~go
func wakep() {
    if sched.nmspinning.Load() != 0 || !sched.nmspinning.CompareAndSwap(0, 1) {
        return
    }
	
    // 禁止被抢占
    mp := acquirem()
    
    var pp *p
    lock(&sched.lock)
    pp, _ = pidlegetSpinning(0)
    // 没有空闲的p,唤醒失败，直接返回
    if pp == nil {
        if sched.nmspinning.Add(-1) < 0 {
            throw("wakep: negative nmspinning")
        }
        unlock(&sched.lock)
        // 恢复当前g的m可以被抢占
        releasem(mp)
        return
    }

    unlock(&sched.lock)
    // 尝试唤醒（创建）m，并执行
    startm(pp, true, false)
    
    releasem(mp)
}
~~~

### startm函数

主要逻辑：

1. 如果pp为空就获取缓存的pp
2. 如果没有空闲的m, new一个m并且初始化m, 包括创建g0和gsignal, 新建系统线程，并且在该线程上执行mstart
3. 如果有空闲的m, 唤醒m

/src/runtime/proc.go:2563

~~~go
func startm(pp *p, spinning, lockheld bool) {
    // 禁止抢占
    mp := acquirem()
	
    // ...
	
    if pp == nil {
		
        // ...
		
        // pp == nil 尝试获取空闲p
        pp, _ = pidleget(0)
        // 没有空闲p,唤醒（创建）m失败，直接返回
        if pp == nil {
			
            // ...
			
            // 恢复当前g的m可以被抢占
            releasem(mp)
            return
        }
    }
    // 获取休眠闲置的m
    nmp := mget()
    // 如果没有休眠闲置的m，就新建
    if nmp == nil {
        id := mReserveID()
        unlock(&sched.lock)
        // 默认启动函数
        var fn func()
		
        // ...
		
        // 创建m
        newm(fn, pp, id)
        
        // ...
		
        // 恢复当前g的m可以被抢占
        releasem(mp)
        return
    }
	
    // ...
	
    // The caller incremented nmspinning, so set m.spinning in the new M.
    // 设置自旋状态和暂存p
    nmp.spinning = spinning
    // 设置将要与m绑定的p,调用notewakeup后，stopm的唤醒逻辑被执行，即调用acquirep(gp.m.nextp.ptr())绑定nmp.nextp到m上
    nmp.nextp.set(pp)
    // 唤醒m
    notewakeup(&nmp.park)
    releasem(mp)
}
~~~

### newm函数

主要逻辑：

1. new一个m并且初始化m, 包括创建g0和gsignal
2. 初始化一些参数
3. 新建一个系统线程并且执行mstart

/src/runtime/proc.go:2385

~~~go
func newm(fn func(), pp *p, id int64) {
    acquirem()
    // 创建m对象
    mp := allocm(pp, fn, id)
    // 暂存p
    mp.nextp.set(pp)
    mp.sigmask = initSigmask
    if gp := getg(); gp != nil && gp.m != nil && (gp.m.lockedExt != 0 || gp.m.incgo) && GOOS != "plan9" {
        // ...
    }
    // 关联真正的 os thread
    // 分配一个系统线程，且完成 g0上的栈分配
    // 传入 mstart 函数，让线程执行 mstart
    newm1(mp)
    releasem(getg().m)
}
~~~

allocm函数

创建一个新的m

/src/runtime/proc.go:1889

~~~go
func allocm(pp *p, fn func(), id int64) *m {
    allocmLock.rlock()
	
    acquirem()
    
    gp := getg()
    if gp.m.p == 0 {
        acquirep(pp) // temporarily borrow p for mallocs in this function
    }
	
    // 释放等待释放的M列表
    // mexit的时候会加到freem, m.gsignal会在那时候释放，这个结构
    // 因为m是又new创建的，可以由gc释放
    if sched.freem != nil {
        // ...
    }
    // 创建新的M
    mp := new(m)
    // 设置启动函数
    mp.mstartfn = fn
    // M初始化
    mcommoninit(mp, id)
	
    // 创建g0
    // 从这里结合rt0_go初始化汇编代码可以发现，m0的g0是在rt0_go由汇编代码初始化
    // 而main函数启动后，其他m的g0是由newm函数调用allocm函数创建的
    if iscgo || mStackIsSystemAllocated() {
        mp.g0 = malg(-1)
    } else {
        mp.g0 = malg(8192 * sys.StackGuardMultiplier)
    }
    mp.g0.m = mp
    
    if pp == gp.m.p.ptr() {
        releasep()
    }
    
    releasem(gp.m)
    allocmLock.runlock()
    return mp
}
~~~

### newm1函数

/src/runtime/proc.go:2434

~~~go
func newm1(mp *m) {
    // cgo处理
    if iscgo {
        // ...
    }
    execLock.rlock() // Prevent process clone.
    // 创建系统线程
    // 分配一个系统线程，且完成 g0上的栈分配
    // 传入 mstart 函数，让线程执行 mstart
    newosproc(mp)
    execLock.runlock()
}
~~~

### newosproc函数

调用clone函数，创建系统线程，分配g0，并执行 mstart函数，启动m,开始任务调度

/src/runtime/os_linux.go:166

~~~go
func newosproc(mp *m) {
    stk := unsafe.Pointer(mp.g0.stack.hi)
    /*
    * note: strace gets confused if we use CLONE_PTRACE here.
    */
    if false {
        print("newosproc stk=", stk, " m=", mp, " g=", mp.g0, " clone=", abi.FuncPCABI0(clone), " id=", mp.id, " ostk=", &mp, "\n")
    }
    
    // Disable signals during clone, so that the new thread starts
    // with signals disabled. It will enable them in minit.
    var oset sigset
    sigprocmask(_SIG_SETMASK, &sigset_all, &oset)
    ret := retryOnEAGAIN(func() int32 {
        // 调用clone方法，创建Linux系统线程
        // 与C的clone系统条用不同，go重写clone
        // 以amd64架构为例，实现代码位于：/usr/local/go_src/21/go/src/runtime/sys_linux_amd64.s：563
        // int32 clone(int32 flags, void *stk, M *mp, G *gp, void (*fn)(void))
        r := clone(cloneFlags, stk, unsafe.Pointer(mp), unsafe.Pointer(mp.g0), unsafe.Pointer(abi.FuncPCABI0(mstart)))
        // clone returns positive TID, negative errno.
        // We don't care about the TID.
        if r >= 0 {
            return 0
        }
        return -r
    })
    sigprocmask(_SIG_SETMASK, &oset, nil)
    
    if ret != 0 {
        print("runtime: failed to create new OS thread (have ", mcount(), " already; errno=", ret, ")\n")
        if ret == _EAGAIN {
            println("runtime: may need to increase max user processes (ulimit -u)")
        }
        throw("newosproc")
    }
}
~~~

## runtime.newproc1

newproc1函数为实际创建协程的函数

主要逻辑：

1. 从当前g.m.p(如果是初始化时，g0.m0.p)本地队列p.gFree中获取g，本地为空，从全局队列sched.gFree中获取,都没有时，调用malg新创建一个栈大小为2KB的g
2. 对获取到的g进行一些初始化（或者重置）操作
3. 将 g 更换为 _Grunnable 状态，分配唯一id

/src/runtime/proc.go:4495

~~~go
func newproc1(fn *funcval, callergp *g, callerpc uintptr) *g {
    if fn == nil {
        fatal("go of nil func value")
    }
    
    // 获取m,因为是在系统栈运行所以此时的 g.m 为 g0.m,并设置不可抢占
    // 初始时，获取的m为m0,即g0.m0
    mp := acquirem() // disable preemption because we hold M and P in local vars.
    // 获得 p,即g0.m.p
    // 初始时，获取的p为g0.m0.p,即allp[0]
    pp := mp.p.ptr()
    // 从本地队列pp中获取一个g,如果本地队列中没有g,从全局队列中获取
    newg := gfget(pp)
    // 如果从pp的本地队列和全局队列中都没有获取到g,则新创建一个g
    // 初始化阶段，gfget函数是不可能找到 g 的
    // 也可能运行中本来就已经耗尽了
    if newg == nil {
        // 创建一个拥有 stackMin 大小的栈的 g  
        // stackMin = 2KB
        // stackMin = 2048 stack.go:75
        newg = malg(stackMin)
        // 将新创建的 g 从 _Gidle 更新为 _Gdead 状态
        casgstatus(newg, _Gidle, _Gdead)
        // 将 Gdead 状态的 g 添加到 allgs，这样 GC 不会扫描未初始化的栈
        // func allgadd(gp *g) {} proc.go:562
        allgadd(newg) // publishes with a g->status of Gdead so GC scanner doesn't look at uninitialized stack.
    }
    if newg.stack.hi == 0 {
        throw("newproc1: newg missing stack")
    }
    
    if readgstatus(newg) != _Gdead {
        throw("newproc1: new g is not Gdead")
    }
    // 计算运行空间大小，对齐
    totalSize := uintptr(4*goarch.PtrSize + sys.MinFrameSize) // extra space in case of reads slightly beyond frame
    totalSize = alignUp(totalSize, sys.StackAlign)
    // 确定 sp 和参数入栈位置
    sp := newg.stack.hi - totalSize
    spArg := sp
    if usesLR {
        // caller's LR
        *(*uintptr)(unsafe.Pointer(sp)) = 0
        prepGoExitFrame(sp)
        spArg += sys.MinFrameSize
    }
    // 清理、创建并初始化的 g 的运行现场
    memclrNoHeapPointers(unsafe.Pointer(&newg.sched), unsafe.Sizeof(newg.sched))
    // 初始化 g 的基本状态
    // newg.sched g的sched字段是gobuf类型，它保存的是goroutine的调度信息，重点就是保存几个关键寄存器的值
    newg.sched.sp = sp
    newg.stktopsp = sp
    newg.sched.pc = abi.FuncPCABI0(goexit) + sys.PCQuantum // +PCQuantum so that previous instruction is in same function
    newg.sched.g = guintptr(unsafe.Pointer(newg))
    // gostartcallfn调整sched成员和newg的栈,核心处理调用的是gostartcall函数。
    // gostartcall函数将栈顶寄存器SP向下移动一个指针的位置，然后将goexit+1即goexit的第二条指令。
    // 然后将buf.pc即newg.sched.pc重新设为fn(runtime.main函数)。
    // 相当于将goexit放到newg的栈顶，伪造成newg是被goeixt函数调用的，当newg中的fn函数执行完成之后，返回到goexit继续执行，做一些清理的操作。
    gostartcallfn(&newg.sched, fn)
    newg.parentGoid = callergp.goid
    newg.gopc = callerpc
    newg.ancestors = saveAncestors(callergp)
    newg.startpc = fn.fn
	
    // ...
	
    // 现在将 g 更换为 _Grunnable 状态
    casgstatus(newg, _Gdead, _Grunnable)
    gcController.addScannableStack(pp, int64(newg.stack.hi-newg.stack.lo))
    // 分配 唯一goid
    if pp.goidcache == pp.goidcacheend {
        // Sched.goidgen is the last allocated id,
        // this batch must be [sched.goidgen+1, sched.goidgen+GoidCacheBatch].
        // At startup sched.goidgen=0, so main goroutine receives goid=1.
        
        // Sched.goidgen 为最后一个分配的 id，相当于一个全局计数器
        // 这一批必须为 [sched.goidgen+1, sched.goidgen+GoidCacheBatch].
        // 启动时 sched.goidgen=0, 因此主 Goroutine 的 goid 为 1
        pp.goidcache = sched.goidgen.Add(_GoidCacheBatch)
        pp.goidcache -= _GoidCacheBatch - 1
        pp.goidcacheend = pp.goidcache + _GoidCacheBatch
    }
    newg.goid = pp.goidcache
    pp.goidcache++
    
    // ...
	
    // 恢复m可抢占
    releasem(mp)
    
    return newg
}
~~~

### acquirem函数

获取当前g绑定的m

/src/runtime/runtime1.go:572

~~~go
//go:nosplit
func acquirem() *m {
    gp := getg()
    gp.m.locks++
    return gp.m
}
~~~

### gfget、gfput函数

gfget函数

主要逻辑：

1. 尝试从本地空闲队列中获取g，为空时，尝试从全局空闲队列中获取转移一批g到本地队列，最多转移32个
2. 从本地空闲队列获取g，检查栈信息，并做相应设置后，返回g

/src/runtime/proc.go:4667

~~~go
func gfget(pp *p) *g {
retry:
    // 本地队列为空且全局队列不为空时，尝试从全局队列中获取转移一批g到本地队列
    if pp.gFree.empty() && (!sched.gFree.stack.empty() || !sched.gFree.noStack.empty()) {
        lock(&sched.gFree.lock)
        // 最多转移32个
        for pp.gFree.n < 32 {
            // 首选带堆栈的
            gp := sched.gFree.stack.pop()
            if gp == nil {
                gp = sched.gFree.noStack.pop()
                if gp == nil {
                    break
                }
            }
            // 从全局队列中取出，放入本地pp的队列中
            sched.gFree.n--
            pp.gFree.push(gp)
            pp.gFree.n++
        }
        unlock(&sched.gFree.lock)
        // 跳转retry，再次判断本地队列是否为空
        goto retry
    }
    // 尝试从本地队列获取g
    gp := pp.gFree.pop()
    // 不存在空闲g，返回nil
    if gp == nil {
        return nil
    }
    // 成功获取g
    pp.gFree.n--
    // 检查 g 的栈 （stack）
    if gp.stack.lo != 0 && gp.stack.hi-gp.stack.lo != uintptr(startingStackSize) {
        systemstack(func() {
            stackfree(gp.stack)
            gp.stack.lo = 0
            gp.stack.hi = 0
            gp.stackguard0 = 0
        })
    }
    if gp.stack.lo == 0 {
        // 分配新栈
        systemstack(func() {
            gp.stack = stackalloc(startingStackSize)
        })
        gp.stackguard0 = gp.stack.lo + stackGuard
    } else {
        if raceenabled {
            racemalloc(unsafe.Pointer(gp.stack.lo), gp.stack.hi-gp.stack.lo)
        }
        if msanenabled {
            msanmalloc(unsafe.Pointer(gp.stack.lo), gp.stack.hi-gp.stack.lo)
        }
        if asanenabled {
            asanunpoison(unsafe.Pointer(gp.stack.lo), gp.stack.hi-gp.stack.lo)
        }
    }
    return gp
}
~~~

gfput函数

当goroutine执行完毕，调度器相关函数会将g放回p空闲队列，实现复用

主要逻辑：

1. 优先放入本地空闲队列
2. 本地空闲队列长度大于64时，转移部分到全局空闲列表，本地空闲队列长度小于32时，停止转移

/src/runtime/proc.go:4624

~~~go
func gfput(pp *p, gp *g) {
    // ...
    
    pp.gFree.push(gp)
    pp.gFree.n++
    // 如果本地队列太长（大于64）时，将一个批转移到全局列表
    if pp.gFree.n >= 64 {
        var (
            inc      int32
            stackQ   gQueue
            noStackQ gQueue
        )
        // 本地队列长度小于32时，停止转移
        for pp.gFree.n >= 32 {
            gp := pp.gFree.pop()
            pp.gFree.n--
            if gp.stack.lo == 0 {
                noStackQ.push(gp)
            } else {
                stackQ.push(gp)
            }
            inc++
        }
        lock(&sched.gFree.lock)
        sched.gFree.noStack.pushAll(noStackQ)
        sched.gFree.stack.pushAll(stackQ)
        sched.gFree.n += inc
        unlock(&sched.gFree.lock)
    }
}
~~~

### malg函数

创建新的g对象，并分配stacksize大小的栈空间

/src/runtime/proc.go:4458

~~~go
func malg(stacksize int32) *g {
    newg := new(g)
    if stacksize >= 0 {
        stacksize = round2(stackSystem + stacksize)
        // 切换到g0栈，分配g的栈空间
        systemstack(func() {
            newg.stack = stackalloc(uint32(stacksize))
        })
        newg.stackguard0 = newg.stack.lo + stackGuard
        newg.stackguard1 = ^uintptr(0)
        *(*uintptr)(unsafe.Pointer(newg.stack.lo)) = 0
    }
    return newg
}
~~~

## 总结

newproc方法主要实现的功能：

1. 获取当前g,获取调用方 PC/IP 寄存器值,切换g0栈调用newproc1方法创建执行fn的g
2. 首先尝试从g0.m.p的本地空闲队列p.gFree中获取g,若不存在，则从全局空闲队列sched.gFree中获取，还是不存在时，调用malg创建g
3. 对获取到的g进行一些初始化（或者重置）操作
4. 将 g 更换为 _Grunnable 状态，分配唯一id
5. 将创建的可运行g放入p可运行队列（P.runnext、P.runq）或者全局可运行队列（sched.runq）中去，依据优先级从高到低尝试放入队列
6. mainStarted == true 时(main函数已经开始执行)，则调用wakep()尝试唤醒其他M/P执行任务，充分发挥多核优势
7. wakep会从全局空闲p sched.pidle中获取p,如果不存在则直接返回
8. 存在空闲p，尝试从空闲的m队列sched.midle中获取一个来绑定p执行新的g
9. 如果不存在空闲的m,则调用newm函数创建新的m并创建m.g0,绑定p执行新的g