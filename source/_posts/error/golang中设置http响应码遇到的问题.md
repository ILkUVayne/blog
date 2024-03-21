---
title: golang中设置http响应码遇到的问题
date: 2024-03-21 14:07:31
categories:
- error
tags:
- golang
- http
---

## 问题描述

> 当前golang版本1.21

调用http.ResponseWriter.WriteHeader()设置httpCode,由于调用时机的不同，会导致意料之外的错误

## 代码示例

### 1. 示例一

~~~go
package main

import (
    "log"
    "net/http"
)
func main() {
    http.HandleFunc("/ping1", handler1)
    log.Fatal(http.ListenAndServe("localhost:8080", nil))
}

func handler1(w http.ResponseWriter, r *http.Request) {
    w.Header().Set("Content-Type", "application/json; charset=utf-8")
    w.WriteHeader(http.StatusFailedDependency)
    _, err := w.Write([]byte("request /ping1"))
    if err != nil {
        return
    }
}
~~~

示例一代码为正常情况，Content-Type header头以及httpCode http.StatusFailedDependency能够正常响应

### 2. 示例二

~~~go
package main

import (
    "log"
    "net/http"
)

func main() {
    http.HandleFunc("/ping1", handler2)
    log.Fatal(http.ListenAndServe("localhost:8080", nil))
}

func handler2(w http.ResponseWriter, r *http.Request) {
    w.WriteHeader(http.StatusFailedDependency)
    w.Header().Set("Content-Type", "application/json; charset=utf-8")
    _, err := w.Write([]byte("request /ping2"))
    if err != nil {
        return
    }
}
~~~

示例二的结果就出现了异常，httpCode http.StatusFailedDependency能正常响应，但是后面设置的Content-Type并没有生效

### 3. 示例三

~~~go
package main

import (
    "log"
    "net/http"
)

func main() {
    http.HandleFunc("/ping1", handler3)
    log.Fatal(http.ListenAndServe("localhost:8080", nil))
}

func handler3(w http.ResponseWriter, r *http.Request) {
    w.Header().Set("Content-Type", "application/json; charset=utf-8")
    _, err := w.Write([]byte("request /ping3"))
    if err != nil {
        return
    }
    w.WriteHeader(http.StatusFailedDependency)
}
~~~

示例三的httpCode以及Content-Type都能够整个设置并响应，但是会有错误日志**http: superfluous response.WriteHeader call from main.handler3**

## 造成原因代码

主要在于w.WriteHeader方法的实现方式，代码位于net/http/server.go中

### 1. WriteHeader方法

WriteHeader方法会判断w.wroteHeader的值是否为true,这个值默认为false,当调用WriteHeader或Write方法后会被设置为true。所以重复调用WriteHeader方法或者在Write方法后调用WriteHeader方法都会抛出异常

~~~go
func (w *response) WriteHeader(code int) {
    //...
    if w.wroteHeader {
        caller := relevantCaller()
        w.conn.server.logf("http: superfluous response.WriteHeader call from %s (%s:%d)", caller.Function, path.Base(caller.File), caller.Line)
        return
    }
    //...

    w.wroteHeader = true
    w.status = code

    if w.calledHeader && w.cw.header == nil {
        w.cw.header = w.handlerHeader.Clone()
    }

    //...
}
~~~

### 2. Write方法

Write方法会判断w.wroteHeader的值是否为true,不为true时，会调用WriteHeader方法设置httpCode为200

~~~go
func (w *response) write(lenData int, dataB []byte, dataS string) (n int, err error) {
    //...
    if !w.wroteHeader {
        w.WriteHeader(StatusOK)
    }
    //...
}
~~~

### 3. Header方法

~~~go
func (w *response) Header() Header {
    if w.cw.header == nil && w.wroteHeader && !w.cw.wroteHeader {
        // Accessing the header between logically writing it
        // and physically writing it means we need to allocate
        // a clone to snapshot the logically written state.
        w.cw.header = w.handlerHeader.Clone()
    }
    w.calledHeader = true
    return w.handlerHeader
}
~~~

## 结论

### 1. 示例三代码报错原因

从源码中可以看出，write和WriteHeader方法都会设置w.wroteHeader = true（write方法调用WriteHeader设置），如果调用write后再调用WriteHeader就会抛出错误了，需要注意的是例如fmt.Fprintf()、xml.NewEncoder(w).Encode()等方法，也会调用write方法，故要设置httpCode也需要再这些方法之前调用

### 2. 示例二代码设置header头失效原因

从WriteHeader方法中可以看出，header实际的响应头存放在w.cw.header，而非w.handlerHeader，根据不同的header和httpCode设置顺序有两种情况

#### 2.1 先设置header，再设置httpCode能够正常使用

这种情况下，(w.cw.header == nil && w.wroteHeader && !w.cw.wroteHeader)恒等于false，故实际设置的是w.handlerHeader中的值，最后再调用WriteHeader方法时，赋值给w.cw.header，所以能够正常使用

#### 2.2 先设置httpCode，再设置header会失效

这种情况下，(w.cw.header == nil && w.wroteHeader && !w.cw.wroteHeader)的值有两种情况

- 第一种情况
(w.cw.header == nil)为真既实际响应header未设置，先进行赋值操作，初始化w.cw.header (w.cw.header = w.handlerHeader.Clone()),然后返回w.handlerHeader

- 第二种情况
若不等于nil，则直接返回w.handlerHeader

两种情况的后续set操作都是对w.handlerHeader进行设置，均不能影响到w.cw.header，所以设置的header失效了
