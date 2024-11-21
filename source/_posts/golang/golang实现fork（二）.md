---
title: golang实现fork（二）
date: 2024-09-10 15:31:06
categories:
  - golang
tags:
  - golang
  - multi-process
  - syscall
---

## 前言

在多线程的程序中使用 `fork` 是比较危险的。fork的时候只复制当前线程到子进程，假设在fork之前，一个线程对某个互斥锁进行lock持有该锁，这时另一个线程调用fork创建了一个子进程，可是在子进程视野中持有那个锁的线程“消失”了，对于子进程来说，这个锁被“永久”上锁了，最终导致死锁了。

## 复现

> 实现fork的方式参考：{% post_link golang/golang实现fork %} ， 本文同样使用相同的代码来复现，只修改child方法，其余代码不变

child函数主动触发gc，模拟多线程加锁（gc过程中会有很多加锁的情况），复现多线程下fork死锁。

~~~go
func child() {
    var childPid int
    runtime.GC()
    if childPid = fork(); childPid == 0 {
        fmt.Println("In child: child process success")
        os.Exit(0)
    } else {
        cPid = childPid
        fmt.Printf("In parent: childPid = %d\r\n", childPid)
    }
}
~~~

### 执行

手动触发gc并不会百分百复现死锁（可能锁刚被释放的时候fork），但复现的概率还是比较高的。

正常输出：

~~~bash
go run proc.go
In child: child process success
In parent: childPid = 16875
child process done
sum:  0
~~~

死锁：

~~~bash
go run proc.go
In parent: childPid = 17922
~~~

从输出结果能看出，子进程卡住了。

## 分析

使用`strace`命令分析pid为17922的子进程。

~~~bash
strace -p 17922    
strace: Process 17922 attached
futex(0x5233d8, FUTEX_WAIT_PRIVATE, 2, NULL
~~~

使用`kill -6 17922`中断该子进程，并打印栈信息。

~~~bash
goroutine 1 [running]:
runtime.futex()
        /usr/local/go_src/21/go/src/runtime/sys_linux_amd64.s:557 +0x21 fp=0xc000074c50 sp=0xc000074c48 pc=0x45e841
runtime.futexsleep(0xc00000001e?, 0x43b6cd?, 0xc0000424c8?)
        /usr/local/go_src/21/go/src/runtime/os_linux.go:69 +0x30 fp=0xc000074ca0 sp=0xc000074c50 pc=0x42d3d0
runtime.lock2(0x5233d8)
        /usr/local/go_src/21/go/src/runtime/lock_futex.go:107 +0x73 fp=0xc000074ce0 sp=0xc000074ca0 pc=0x409ef3
runtime.lockWithRank(...)
        /usr/local/go_src/21/go/src/runtime/lockrank_off.go:24
runtime.lock(...)
        /usr/local/go_src/21/go/src/runtime/lock_futex.go:48
runtime.GOMAXPROCS(0x0)
        /usr/local/go_src/21/go/src/runtime/debug.go:21 +0x25 fp=0xc000074d00 sp=0xc000074ce0 pc=0x406245
sync.(*Pool).pinSlow(0x51ebc0)
        /usr/local/go_src/21/go/src/sync/pool.go:229 +0x165 fp=0xc000074d98 sp=0xc000074d00 pc=0x467be5
sync.(*Pool).pin(0x51ebc0)
        /usr/local/go_src/21/go/src/sync/pool.go:209 +0x45 fp=0xc000074db0 sp=0xc000074d98 pc=0x467a65
sync.(*Pool).Get(0x51ebc0)
        /usr/local/go_src/21/go/src/sync/pool.go:131 +0x1c fp=0xc000074de8 sp=0xc000074db0 pc=0x4677bc
fmt.newPrinter()
        /usr/local/go_src/21/go/src/fmt/print.go:152 +0x1e fp=0xc000074e10 sp=0xc000074de8 pc=0x47607e
fmt.Fprintln({0x4b5a18, 0xc00009c008}, {0xc000074ea8, 0x1, 0x1})
        /usr/local/go_src/21/go/src/fmt/print.go:303 +0x30 fp=0xc000074e60 sp=0xc000074e10 pc=0x476490
fmt.Println(...)
        /usr/local/go_src/21/go/src/fmt/print.go:314
main.child()
        /home/luoyang/code/go_demo/src_test/proc.go:32 +0x6c fp=0xc000074ec8 sp=0xc000074e60 pc=0x47cfcc
main.main()
        /home/luoyang/code/go_demo/src_test/proc.go:13 +0x17 fp=0xc000074f40 sp=0xc000074ec8 pc=0x47ce57
runtime.main()
        /usr/local/go_src/21/go/src/runtime/proc.go:267 +0x2bb fp=0xc000074fe0 sp=0xc000074f40 pc=0x43355b
runtime.goexit()
        /usr/local/go_src/21/go/src/runtime/asm_amd64.s:1650 +0x1 fp=0xc000074fe8 sp=0xc000074fe0 pc=0x45cac1
        
        ...
~~~

从`strace`和`kill -6`的结果中都能看出，子进程阻塞在了`futex`，即发生了死锁，而子进程本身的代码并未执行。

## 建议

> golang中的fork死锁可以参考我的笨办法处理方式：[simple-redis/src/networking#checkRdbOrAofExecTimeout](https://github.com/ILkUVayne/simple-redis/blob/master/src/networking.go)

1. 若非必要，应当尽量避免在多线程下使用fork，特别是像golang这种本身就存在gc，即使用户代码中不存在锁，但也可能会因为gc或者内存管理（mallocgc）过程中的加锁操作导致出现死锁的情况。
2. 若是一定要使用，在如c语言这种手动管理内存的语言中，我们应该尽量注意锁的使用并且fork后立即执行exec。如果是go，则只能手动发送中断信号`syscall.Kill(pid, syscall.SIGINT)`，结束死锁的子进程（后面若是发现更好的方式再更新）