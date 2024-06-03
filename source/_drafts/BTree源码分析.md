---
title: BTree源码分析
categories:
- algorithm
tags:
- algorithm
- btree
- golang
---

## 前言

> btree包GitHub地址：https://github.com/google/btree

本文通过分析Google开源的btree包，来了解golang版本的btree源码实现。

## 数据结构

### BTree结构

~~~go
type BTree struct {
    degree int
    length int
    root   *node
    cow    *copyOnWriteContext
}
~~~

- degree

BTree的度

- length

BTree中有多少个元素

- root

BTree的根结点

- cow

cow字段用于辅助优化作用，用于在多颗BTree中达到节点node复用的作用，减少gc开销。

### node结构

~~~go
type node struct {
    items    items
    children children
    cow      *copyOnWriteContext
}

type items []Item

type Item interface {
    Less(than Item) bool
}

type children []*node
~~~

- items

items存放结点的元素信息，它是一个Item切片。

- children

children存放子结点信息，它是一个node指针切片。

- Item

Item是一个实现了Less方法的接口类型，用于容纳任意实现了Less方法的元素类型，便于后续的创建、查找和删除等操作。

## 插入

// todo