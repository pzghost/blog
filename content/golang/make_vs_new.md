---
title: "make 和 new"
isCJKLanguage: true
date: 2023-06-02T17:11:16+08:00
#lastmod: 2022-04-28T15:13:38+08:00
#categories: [CSI]
#tags: [kubernetes, csi, storage, volume]
draft: false
enableDisqus : true
enableMathJax: false
toc: true
disableToC: false
disableAutoCollapse: true
---

> **Have a nice day! :heart:**

make 和 new 都是 Go 语言中用于创建变量的内建函数，但它们的作用和用途是不同的。

# make 函数

- make 用于创建切片、映射和通道等引用类型的数据结构。
- make 函数返回一个已初始化且可用的数据结构。
- make 函数的语法：make(T, args...)，其中 T 是切片、映射或通道的类型，args 是传递给 make 函数的参数。
- 示例：
```go
// 创建一个切片，长度为 5，容量为 10
s := make([]int, 5, 10)

// 创建一个映射
m := make(map[string]int)

// 创建一个带缓冲的通道
ch := make(chan int, 100)

```

# new 函数

- new 用于创建值类型的变量，并返回一个指向该变量的指针。
- new 函数为变量分配内存，并将其初始化为零值。
- new 函数的语法：new(T)，其中 T 是值类型的类型。
- 示例：
```go
// 创建一个整数变量的指针
p := new(int)

// 创建一个结构体变量的指针
q := new(Person)

```

# 注意

- make 只能用于创建切片、映射和通道等引用类型的数据结构，而不能用于创建值类型的变量。
- new 只能用于创建值类型的变量，并返回指向该变量的指针。
- make 和 new 返回的都是已经初始化的数据结构或变量，而不是零值。
- make 不能用于初始化 interface{} 和 Function 类型的变量，因为 interface{} 是一个空接口，它可以表示任意类型的值。

综上所述，make 用于创建引用类型的数据结构，而 new 用于创建值类型的变量并返回指针。具体选择使用哪个函数取决于你要创建的变量的类型和需求。


# 常见的引用类型

- 切片（Slice）：切片是对数组的抽象，它提供了对数组部分或全部元素的动态访问和操作。切片是引用类型，当切片被赋值给另一个变量或作为参数传递时，它们会共享相同的底层数组。
- 映射（Map）：映射是一种键值对的无序集合。它提供了快速的查找功能，通过键可以获取对应的值。映射是引用类型，当映射被赋值给另一个变量或作为参数传递时，它们会共享相同的底层数据。
- 通道（Channel）：通道用于在 goroutine 之间进行通信和同步。通道允许一个 goroutine 发送数据到通道，而另一个 goroutine 可以从通道中接收数据。通道是引用类型，当通道被赋值给另一个变量或作为参数传递时，它们会共享相同的通道。
- 函数（Function）：函数是一种可调用的代码块，它可以接收参数并返回结果。函数类型是引用类型，可以将函数赋值给变量，并将函数作为参数传递给其他函数。
- 接口（Interface）：接口是一组方法的抽象，它定义了一系列的行为。接口类型是引用类型，可以将实现了接口的类型赋值给接口变量。

需要注意的是，引用类型在赋值或传递时会共享底层的数据，而不是进行值的拷贝。这与值类型不同，值类型在赋值或传递时会进行复制操作。

# 参考
[draveness 大神](https://draveness.me/golang/docs/part2-foundation/ch05-keyword/golang-make-and-new/)