---
title: golang 程序（初始化）入口分析
date: 2024-04-01 14:51:11
categories:
- golang
tags:
- golang
- runtime
- plan9
- gmp
---

> 版本：go 1.21
> 平台：amd64 linux
> 调试工具：gdb 、 dlv

## 前言

> 本文展示代码为关键代码，建议结合完整源代码对照学习

本文尝试从源码的角度分析golang程序（初始化）入口，了解golang启动过程中大概都做了哪些事情。

## Hello world

使用golang编写一个hello world

entry.go

~~~go
func main() {
    println("Hello world!")
}
~~~

编译成可执行文件

~~~bash
# -gcflags "-N -l" 参数关闭编译器代码优化和函数内联，避免断点和单步执⾏⽆法准确对应源码⾏，避免⼩函数和局部变量被优化掉
go build -gcflags "-N -l" -o entry entry.go
~~~

## gdb调试entry

> tips: 设置断点后，若发现存在多个断点（即可能存在多个同名函数等）时，可使用 (gdb) i b 命令查看每个断点信息

gdb调试，设置断点

~~~bash
gdb entry 
...
(gdb) info file
Local exec file:
        
        Entry point: 0x455e40
...        
        
# Entry point: 0x455e40 即为入口点
# 设置断点

(gdb) b *0x455e40
Breakpoint 1 at 0x455e40: file /usr/local/go_src/21/go/src/runtime/rt0_linux_amd64.s, line 8.

# /usr/local/go_src/21/go/src/runtime 为我安装的golang源码路径
# rt0_linux_amd64.s 即为该断点的汇编代码路径（不同的平台的实际汇编文件会不一样）
# 用你熟悉的方式查看rt0_linux_amd64.s文件
~~~

### rt0_linux_amd64.s

查看 rt0_linux_amd64.s 中第8行的代码

~~~plan9_x86
#include "textflag.h"

TEXT _rt0_amd64_linux(SB),NOSPLIT,$-8
        JMP     _rt0_amd64(SB)

TEXT _rt0_amd64_linux_lib(SB),NOSPLIT,$0
        JMP     _rt0_amd64_lib(SB)
~~~

发现汇编代码中函数 _rt0_amd64_linux 实际跳转的是 _rt0_amd64(SB) 函数，继续设置断点

~~~bash
(gdb) b _rt0_amd64
Breakpoint 2 at 0x454200: file /usr/local/go_src/21/go/src/runtime/asm_amd64.s, line 16.
# 发现该函数在/usr/local/go_src/21/go/src/runtime/asm_amd64.s 16行
~~~

### asm_amd64.s

查看 asm_amd64.s 16行代码

~~~plan9_x86
TEXT _rt0_amd64(SB),NOSPLIT,$-8
	MOVQ	0(SP), DI	// argc
	LEAQ	8(SP), SI	// argv
	JMP	runtime·rt0_go(SB)
    //...
~~~

该函数设置了arg后，跳转到了runtime·rt0_go(SB)函数，设置断点

~~~bash
# 注意汇编函数runtime·rt0_go(SB)的"中点(·)"需要改成"下点(.)"，才能正常设置断点
(gdb) b runtime.rt0_go
Breakpoint 3 at 0x454220: file /usr/local/go_src/21/go/src/runtime/asm_amd64.s, line 161.
# 找到runtime.rt0_go方法的在同文件161行
~~~

阅读源码可发现runtime.rt0_go函数即为go程序的实际入口（初始化起点）

## 初始化步骤分析

> 使用dlv 工具调式代码，方便了解执行步骤

通过上面的调试过程，我们发现rt0_go函数即为go程序的实际入口，该函数主要功能是完成Go程序启动时的所有初始化工作

开始debug

~~~bash
# 添加断点runtime.rt0_go
dlv debug
Type 'help' for list of commands.
(dlv) b runtime.rt0_go
Breakpoint 1 set at 0x463680 for runtime.rt0_go() /usr/local/go_src/21/go/src/runtime/asm_amd64.s:161
~~~

### copy参数

copy argc和argv到AX和BX寄存器

/src/runtime/asm_amd64.s:161

~~~plan9_x86
MOVQ	DI, AX		// argc
MOVQ	SI, BX		// argv
# 向下移动SP寄存器
SUBQ	$(5*8), SP		// 3args 2auto
ANDQ	$~15, SP
# 把argc和argv分别赋值到SP+24 SP+32
MOVQ	AX, 24(SP)
MOVQ	BX, 32(SP)
~~~

### 初始化g0

g0 m0是全局变量，在执行rt0_go时已被初始化

/src/runtime/proc.go:113

~~~go
var (
    m0           m
    g0           g
    mcache0      *mcache
    raceprocctx0 uintptr
    raceFiniLock mutex
)
~~~

next到初始化g0位置

~~~bash
(dlv) n
> runtime.rt0_go() /usr/local/go_src/21/go/src/runtime/asm_amd64.s:170 (PC: 0x463698)
Warning: debugging optimized function
   165:         MOVQ    AX, 24(SP)
   166:         MOVQ    BX, 32(SP)
   167:
   168:         // create istack out of the given (operating system) stack.
   169:         // _cgo_init may update stackguard.
=> 170:         MOVQ    $runtime·g0(SB), DI
   171:         LEAQ    (-64*1024)(SP), BX
   172:         MOVQ    BX, g_stackguard0(DI)
   173:         MOVQ    BX, g_stackguard1(DI)
   174:         MOVQ    BX, (g_stack+stack_lo)(DI)
   175:         MOVQ    SP, (g_stack+stack_hi)(DI)
~~~

代码分析

/src/runtime/asm_amd64.s:170

~~~plan9_x86
// 将g0地址赋值给DI寄存器（前面已经分析了，g0是全局变量，已被初始化分配了地址）
MOVQ	$runtime·g0(SB), DI
// SP向下移动64*1024字节，并获取地址，赋值给BX寄存器
LEAQ	(-64*1024)(SP), BX
// BX = *SP-64*1024
// g0.stackguard0 = BX
MOVQ	BX, g_stackguard0(DI)
// g0.stackguard1 = BX
MOVQ	BX, g_stackguard1(DI)
// g0.stack.lo = BX
MOVQ	BX, (g_stack+stack_lo)(DI)
// g0.stack.hi = SP
MOVQ	SP, (g_stack+stack_hi)(DI)
~~~

查看g0值，验证

~~~bash
(dlv) p g0
runtime.g {
        #// 140728952698464 - 140728952632928 = 65,536 = 64*1024
        stack: runtime.stack {lo: 140728952632928, hi: 140728952698464},
        stackguard0: 140728952632928,
        stackguard1: 140728952632928,
        _panic: *runtime._panic nil,
        _defer: *runtime._defer nil,
        m: *runtime.m nil,
        sched: runtime.gobuf {sp: 0, pc: 0, g: 0, ctxt: unsafe.Pointer(0x0), ret: 0, lr: 0, bp: 0},
        #// ...
        gcAssistBytes: 0,}
~~~

### 设置线程的本地存储

/src/runtime/asm_amd64.s:258

~~~plan9_x86
    LEAQ	runtime·m0+m_tls(SB), DI
    CALL	runtime·settls(SB)
    
    // store through it, to make sure it works
    get_tls(BX)
    MOVQ	$0x123, g(BX)
    MOVQ	runtime·m0+m_tls(SB), AX
    CMPQ	AX, $0x123
    JEQ 2(PC)
    CALL	runtime·abort(SB)
ok:
    // set the per-goroutine and per-mach "registers"
    get_tls(BX)
    LEAQ	runtime·g0(SB), CX
    MOVQ	CX, g(BX)
    LEAQ	runtime·m0(SB), AX
    
    // save m->g0 = g0
    MOVQ	CX, m_g0(AX)
    // save m0 to g0->m
    MOVQ	AX, g_m(CX)
~~~

#### 设置前，g0和m0的值

g0

~~~bash
(dlv) p g0
runtime.g {
        // ...
        m: *runtime.m nil,
        sched: runtime.gobuf {sp: 0, pc: 0, g: 0, ctxt: unsafe.Pointer(0x0), ret: 0, lr: 0, bp: 0},
        // ...
        gcAssistBytes: 0,}
        
# g0.m = nil
~~~

m0

~~~bash
(dlv) p m0
runtime.m {
        g0: *runtime.g nil,
        // ...
        }
# m0.g0 = nil
~~~

#### 设置后，g0和m0的值

g0

~~~bash
(dlv) p g0
runtime.g {
        stack: runtime.stack {lo: 140728952632928, hi: 140728952698464},
        stackguard0: 140728952632928,
        stackguard1: 140728952632928,
        _panic: *runtime._panic nil,
        _defer: *runtime._defer nil,
        m: *runtime.m {
                g0: *(*runtime.g)(0x52fe60),
                // ...
                },
        sched: runtime.gobuf {sp: 0, pc: 0, g: 0, ctxt: unsafe.Pointer(0x0), ret: 0, lr: 0, bp: 0},
        // ...
        }
~~~

m0

~~~bash
(dlv) p m0
runtime.m {
        g0: *runtime.g {
                stack: (*runtime.stack)(0x52fe60),
                stackguard0: 140728952632928,
                stackguard1: 140728952632928,
                _panic: *runtime._panic nil,
                _defer: *runtime._defer nil,
                m: *(*runtime.m)(0x530240),
                // ...
                },
        // ...
        }
~~~

查看前后g0 m0的值能发现，g0中的m与m0中的g已经建立互相的关联引用

### 开始协程调度

/src/runtime/asm_amd64.s:343

~~~plan9_x86
// 下面语句处理操作系统传递过来的参数
// 即方法开始时的参数操作
// 将argc从内存搬到AX存储器中
MOVL	24(SP), AX		// copy argc
// 将argc搬到SP+0的位置，即栈顶位置
MOVL	AX, 0(SP)
// 将argv从内存搬到AX寄存器中
MOVQ	32(SP), AX		// copy argv
// 将argv搬到SP+8的位置
MOVQ	AX, 8(SP)
// 调用runtime.args
CALL	runtime·args(SB)
// 调用osinit函数，获取CPU的核数，存放在全局变量ncpu中，供后面的调度时使用
CALL	runtime·osinit(SB)
// 调用schedinit进行调度的初始化
CALL	runtime·schedinit(SB)

// create a new goroutine to start program
// runtime·mainPC就是runtime.main函数，这里是将runtime·mainPC的地址赋值给AX
MOVQ	$runtime·mainPC(SB), AX		// entry
// 将AX入栈，作为newproc的fn参数
PUSHQ	AX
// 调用runtime.newproc方法 runtime/proc.go:4477
// 创建goroutine,用来运行runtime.main函数
// runtime.main函数中会执行的逻辑：
//    1.新建一个线程执行sysmon函数，sysmon的工作是系统后台监控，通过轮询的方式判断是否需要进行垃圾回收和调度抢占
//    2.启动gc清扫的goroutine
//    3.执行runtime包的初始化，main包以及main包import的所有包的初始化
//    4.最终执行main.mian函数，完成后调用exit(0)系统调用退出进程
CALL	runtime·newproc(SB)
POPQ	AX

// start this M
// 调用runtime.mstart启动工作线程m,进入调度系统
// 在进行一些设置和检测后
// 执行调度方法schedule()，开始协程调度，该函数不会返回
// 这里就会调度刚创建的runtime.main协程，并运行，至此程序初始化完成并启动
CALL	runtime·mstart(SB)

CALL	runtime·abort(SB)	// mstart should never return
RET
~~~

## 总结

通过调试并设置断点，我们发现golang程序的起点并非我们编写的main.main函数，而是runtime/asm_amd64.s.runtime.rt0_go（amd64），该函数才是golang真正（初始化）入口。

整个初始化过程就是go调度器（g0,m0,p）的初始化过程，初始化g0,m的线程本地存储，创建runtime.main协程，主要是调用runtime.proc方法，这个方法会创建实际运行runtime.main方法的协程，初始化协程属性以及p的队列处理，最后启动工作线程m,进入调度系统

系统在初始化后，主要就是围绕schedule()协程调度逻辑，此时runtime.main协程被调度，并最终执行main.main函数，即我们所写的hello world