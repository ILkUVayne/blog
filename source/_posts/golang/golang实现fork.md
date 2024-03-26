---
title: golang实现fork
date: 2024-03-26 15:23:56
categories:
- golang
tags:
- golang
- multi-process
- syscall
---

## 前言

golang默认屏蔽了线程和进程的概念，采用协程的方式实现并发。如想要在golang中实现如C语言类似的多进程，需要一些技巧。golang实现多进程的方式有很多，本文采用系统调用的方式，尽量还原C语言风格多进程

## 实现fork函数

golang没有提供官方的fork函数，但是提供了较为底层的系统调用Syscall，可以实现我们想要的fork功能

~~~go
func fork() int {
    id, _, _ := syscall.Syscall(syscall.SYS_FORK, 0, 0, 0)
    return int(id)
}
~~~

编写代码测试fork函数

~~~go
// proc.go
func main() {
    var childPid int
    
    a := 1
    if childPid = fork(); childPid == 0 {
        a++
        fmt.Printf("In child: childPid = %d, a = %d\r\n", childPid, a)
    } else {
        a--
        fmt.Printf("In parent: childPid = %d, a = %d\r\n", childPid, a)
    }
}
~~~

执行结果如下：

从执行结果能看出，达到了我们想要的效果

~~~bash
$ go run proc.go
In child: childPid = 0, a = 2
In parent: childPid = 13740, a = 0
~~~

## 封装wait4函数

进程调用 os.Exit() 退出执行后，被设置为僵死状态，这时父进程可以通过 wait4() 系统调用查询子进程是否终结，之后再进行最后的操作，彻底删除进程所占用的内存资源。 wait4() 系统调用由 linux 内核实现，linux 系统通常提供了 wait()、waitpid()、wait3()、wait4() 这四个函数，四个函数的参数不同，语义也有细微的差别，但是都返回关于终止进程的状态信息。本文主要对wait4()进行封装，对于前三个函数的区别可以查看 [Linux内核学习笔记（4）-- wait、waitpid、wait3 和 wait4](https://www.cnblogs.com/tongye/p/9558320.html)

### 函数实现

pid

- pid < -1 等待进程组 ID 为 pid 绝对值的进程组中的任何子进程。
- pid = -1 等待任何子进程. 
- pid = 0 等待进程组 ID 与当前进程相同的任何子进程（也就是等待同一个进程组中的任何子进程）.
- pid > 0 等待任何子进程 ID 为 pid 的子进程，只要指定的子进程还没有结束，wait4() 就会一直等下去.

options 

- syscall.WNOHANG 如果没有任何已经结束了的子进程，则马上返回，不等待
- syscall.WUNTRACED 如果子进程进入暂停执行的情况，则马上返回，但结束状态不予理会

返回值

- return 0 如果设置了 WNOHANG，而调用 wait4() 时，没有发现已退出的子进程可收集，则返回0.
- return > 0 正常返回时，wait4() 返回收集到的子进程的PID.
- return -1 如果调用出错，则返回 -1，这时error 会被设置为相应的值以指示错误所在。（当 pid 所指示的子进程不存在，或此进程存在， 但不是调用进程的子进程， wait4() 就会返回出错，这时 error 被设置为 ECHILD）

~~~go
func wait4(pid int, options int) (int, error) {
    var status syscall.WaitStatus
    id, err := syscall.Wait4(pid, &status, options, nil)
    return id, err
}
~~~

## 测试示例

编写测试代码:

main函数先调用child()函数，然后循环调用wait4函数，判断子进程是否执行完毕并退出循环，未执行完则sum++

~~~go
var cPid int

func main() {
    child()
    sum := 0
    for {
        pid, _ := wait4(-1, syscall.WNOHANG)
        if pid != 0 && pid != -1 {
            // 判断执行完成的子进程是否为cPid，如果有多个子进程执行任务，可以这样判断
            if pid == cPid {
                // 实际的代码中，这里会执行一些清扫收尾工作
                fmt.Println("child process done")
                break
            }
        }
        // 这里演示代码循环是否阻塞，实际的代码中，可以继续执行主进程的相关逻辑
        sum++
    }
    fmt.Println("sum: ", sum)
}

// fork子进程，并使用sleep模拟处理逻辑
func child() {
    var childPid int
    
    if childPid = fork(); childPid == 0 {
        time.Sleep(1 * time.Second)
        fmt.Println("In child: child process success")
        os.Exit(0)
    } else {
        cPid = childPid
        fmt.Printf("In parent: childPid = %d\r\n", childPid)
    }
}
~~~

执行结果：

~~~bash
$ go run proc.go
In parent: childPid = 23652
In child: child process success
child process done
sum:  5604417
~~~

从执行结果能看出，wait4函数并不会阻塞代码运行，因为 sum !=1 ,sum的值可以是大于1的任意整数。在实际的代码实现上，我们可以使用wait4函数来检查子进程的完成情况，并在子进程结束后进行相应的清扫收尾工作，同时不影响主进程的代码执行。