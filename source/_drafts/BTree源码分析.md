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

## 创建

### new

btree包提供两种创建B树的函数

New函数用于创建一颗B树，degree是这棵树的度，从NewWithFreeList函数能看出，degree必须大于1。 例如New(2)则会创建一个2-3-4树，每个结点有1-3个元素，有2-4个孩子

NewWithFreeList函数创建颗B树时，需要传出一个FreeList，这个FreeList可以在多颗树中复用结点（New函数则会在内部new一个FreeList，不会共享）

~~~go
const (
    DefaultFreeListSize = 32
)

func New(degree int) *BTree {
    return NewWithFreeList(degree, NewFreeList(DefaultFreeListSize))
}

func NewWithFreeList(degree int, f *FreeList) *BTree {
    if degree <= 1 {
        panic("bad degree")
    }
    return &BTree{
        degree: degree,
        cow:    &copyOnWriteContext{freelist: f},
    }
}

// NewFreeList函数创建一个新的结点复用列表，size是列表的容量大小
func NewFreeList(size int) *FreeList {
    return &FreeList{freelist: make([]*node, 0, size)}
}
~~~

### cow

cow通过写时复制，降低内存的开销，多颗BTree可以共享节点node

copyOnWriteContext结构

~~~go
type copyOnWriteContext struct {
    freelist *FreeList
}

// 通过缓存，能够对节点进行复用，降低GC开销
type FreeList struct {
    mu       sync.Mutex
    freelist []*node
}

// 获取（创建）node
func (f *FreeList) newNode() (n *node) {
    f.mu.Lock()
    index := len(f.freelist) - 1
    // 小于0说明没有多余的node（已经被用完），new一个新的
    if index < 0 {
        f.mu.Unlock()
        return new(node)
    }
    // 用可复用的node，直接获取并返回
    n = f.freelist[index]
    f.freelist[index] = nil
    f.freelist = f.freelist[:index]
    f.mu.Unlock()
    return
}

// 尝试释放node到freelist，成功返回ture
func (f *FreeList) freeNode(n *node) (out bool) {
    f.mu.Lock()
    // freelist未满时，放入用于下次复用
    if len(f.freelist) < cap(f.freelist) {
        f.freelist = append(f.freelist, n)
        out = true
    }
    f.mu.Unlock()
    return
}
~~~

申请新的node时，会优先尝试从freelist中获取可复用的node

~~~go
func (c *copyOnWriteContext) newNode() (n *node) {
    n = c.freelist.newNode()
    n.cow = c
    return
}
~~~

删除node时也尝试释放到freelist

~~~go
func (c *copyOnWriteContext) freeNode(n *node) freeType {
    if n.cow == c {
        // clear to allow GC
        n.items.truncate(0)
        n.children.truncate(0)
        n.cow = nil
		// 尝试释放到freelist
        if c.freelist.freeNode(n) {
            return ftStored
        } else {
            // freelist已满，node会被GC回收
            return ftFreelistFull
        }
    } else {
        return ftNotOwned
    }
}
~~~

## 克隆

通过Clone操作，可以获取两颗一模一样的BTree,这两颗BTree共享节点信息，实际只保存了一份节点，节省空间

~~~go
func (t *BTree) Clone() (t2 *BTree) {
    // 创建两个cow对象
    cow1, cow2 := *t.cow, *t.cow
    // 将被克隆的BTree赋值给新的BTree，复用结点，节省内存
    out := *t
    // 更新旧BTree的cow
    t.cow = &cow1
    // 更新新BTree的cow
    out.cow = &cow2
    return &out
}
~~~

## 插入

// todo