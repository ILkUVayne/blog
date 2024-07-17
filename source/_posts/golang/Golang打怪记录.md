---
title: Golang打怪记录
date: 2024-07-17 10:51:00
categories:
  - golang
tags:
  - golang
  - notes
---

## 前言

Golang学习使用中遇到的一些问题及技巧记录。

## for循环闭包问题

> 本问题存在于在Go version <= 1.21，1.22版本修复了这个问题

普通for循环：

~~~go
func main() {
    s := []int{1, 2, 3, 4, 5}
    for _, v := range s {
        fmt.Println(&v)
    }
}
~~~

执行结果：

~~~bash
go run test1.go
1 0xc00009a010
2 0xc00009a010
3 0xc00009a010
4 0xc00009a010
5 0xc00009a010
~~~

从结果能发现，for循环中的迭代变量 `v` 的地址始终不变，每次遍历会给该变量重新赋值。

当for循环中存在闭包：

~~~go
func main() {
    var pps []func()
    s := []int{1, 2, 3, 4, 5}
    for _, v := range s {
        pps = append(pps, func() { fmt.Println(v, &v) })
    }
    for _, pp := range pps {
        pp()
    }
}
~~~

执行结果：

~~~bash
go run test1.go
5 0xc000012088
5 0xc000012088
5 0xc000012088
5 0xc000012088
5 0xc000012088
~~~

从结果能发现，切片中的函数执行结果均为5，与我们的实际预期不符，在 `Go version <= 1.21` 时，我们需要在append之前重新赋值一个变量，例如：`v := v` ，就能达到我们想要的预期效果。Go在1.22修复了这个问题，无需额外的编码，for循环在每次循环都会重新生成新的变量而非将迭代的值重新赋值给相同地址的变量。

## 强制实现接口

有如下示例代码：

~~~go
type animal interface {
    eat(food string)
    sleep()
}

type dog struct{}

func (d dog) eat(food string) {
    fmt.Println("eat ", food)
}

func main() {
    dog{}.eat("apple")
}
~~~

此时，接口 `animal` 并未被实现，代码能够正常执行。如果我需要让 `dog` 强制实现 `animal` 接口，需要一些特殊的操作，因为Go的接口实现是松散的，并不能像 `java` 等面向对象语言一样可以显式声明实现某个接口。

实现方式：

~~~go
// ... 

var _ animal = (*dog)(nil)

type dog struct{}

// ...
~~~

添加示例声明代码即可（其他的接口同理），此时再执行 `dog{}.eat("apple")` 方法就会报错：` cannot use (*dog)(nil) (value of type *dog) as animal value in variable declaration: *dog does not implement animal (missing method sleep) ` ，因为此时我们还没有实现 `sleep` 方法。