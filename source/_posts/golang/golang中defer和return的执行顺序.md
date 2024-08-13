---
title: golang中defer和return的执行顺序
date: 2024-08-12 18:51:21
categories:
  - golang
tags:
  - golang
---

## 前言

go 的 defer 语句是用来延迟执行函数的，用于处理一些函数的扫尾工作，例如关闭文件句柄等，那defer与return之间哪个先执行呢？

## 省流结果

对于return语句来说，它的执行过程分为两部：

1. 赋值：先执行赋值操作（或者说执行return语句中的表达式并赋值给返回值），若未命名返回值，则会创建一个临时变量并赋值
2. 返回：携带返回值返回到调用函数

对于defer语句，在return赋值步骤之后但在返回动作之前执行，多个defer语句会以栈（后进先出，后声明的语句先执行）的方式顺序执行

## 示例

### 未命名返回值的函数

~~~go
func test() int {
    var i int
    defer func() {
        i++
        fmt.Println("defer1", i)
    }()
    defer func() {
        i++
        fmt.Println("defer2", i)
    }()
    return i
}

func main() {
    fmt.Println("return:", test())
}
~~~

结果：

~~~bash
defer2 1
defer1 2
return: 0
~~~

分析：

test函数的返回值是一个未命名的int值，从结果能看到，返回的0，而不是2，执行顺序如下：

1. 先执行return赋值（执行表达式）：由于未命名返回变量，会执行一个类似创建一个临时变量作为保存 return 值，假设临时变量未tmp，此时tmp=0
2. 按顺序执行defer语句：defer2执行i++，i=1，defer1执行i++，i=2
3. 执行返回动作：此时返回值为tmp，所以最终返回0

### 命名返回值的函数

~~~go
func test() (i int) {
    defer func() {
        i++
        fmt.Println("defer1", i)
    }()
    defer func() {
        i++
        fmt.Println("defer2", i)
    }()
    return i
}

func main() {
    fmt.Println("return:", test())
}
~~~

结果：

~~~bash
defer2 1
defer1 2
return: 2
~~~

分析：

test函数的返回值是一个命名为i的int值，从结果能看到，返回的2，执行顺序如下：

1. 先执行return赋值（执行表达式）：此时函数的返回值是i，即 i = i ，此时i=0
2. 按顺序执行defer语句：defer2执行i++，i=1，defer1执行i++，i=2
3. 执行返回动作：此时返回值为i，被两个defer语句依次加1，所以最终返回值为2
