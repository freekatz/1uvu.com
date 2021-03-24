---
title: "Go Basic"
date: 2021-03-23T16:29:53+08:00
aliases: []
categories:
 - Programming Language
tags: 
 - Golang
 - Defer
description: 
featured_image:
draft: false
author: 1uvu
---

## defer

这一部分的内容基于：https://tour.go-zh.org/

在 Golang 中，具有流程控制作用的关键字包括：if-else，for，switch，continue，break，goto，defer。

其中 if-else 前可以额外执行一次任意语句；switch 中的 case 不要求条件为常量值，因此可以用来简化多个 if-else 的情况；for 既有 "for" 的作用，也有 "while" 的作用；可以通过为代码添加标签的方式使用 goto 进行跳转；而 defer 是其中一个十分特别的存在。

`defer` 语句后可以执行调用一个函数，这个函数的调用执行会被推迟到外层函数结束返回时，推迟调用的函数**其参数会立即求值**，但是**直到外层函数返回前该函数都不会被调用**。（即传入推迟调用函数的参数会被立即确定，但是调用执行会被推迟到外层函数结束）

### defer 栈

被推迟的函数调用会被压入一个栈中，外层函数结束时按照后进先出的次序来调用执行。

因此，在一个外层函数中，并不建议放置多个 defer 函数调用，迫不得已的情况，需要特别注意 **defer 函数调用的顺序，是自下向上，而其传入参数是自上向下的**。



### defer 的用处

可以使用 defer 完成一些函数中的收尾工作，如：**文件关闭**、**关闭互斥锁**等等。

其中一定需要知道的技巧是通过定义 **defer 匿名函数**的方式来实现**错误恢复 (Recover)**和**异常 (Panic)处理**。

看一下官方博客的例子：[Defer, Panic, and Recover - Go 语言博客 (go-zh.org)](https://blog.go-zh.org/defer-panic-and-recover)

```golang
package main

import "fmt"

func main() {
    f()
    fmt.Println("Returned normally from f.")
}

func f() {
    defer func() {
        if r := recover(); r != nil {
            fmt.Println("Recovered in f", r)
        }
    }()
    fmt.Println("Calling g.")
    g(0)
    fmt.Println("Returned normally from g.")
}

func g(i int) {
    if i > 3 {
        fmt.Println("Panicking!")
        panic(fmt.Sprintf("%v", i))
    }
    defer fmt.Println("Defer in g", i)
    fmt.Println("Printing in g", i)
    g(i + 1)
}
```

此程序输出：

```shell
Calling g.
Printing in g 0
Printing in g 1
Printing in g 2
Printing in g 3
Panicking!
Defer in g 3
Defer in g 2
Defer in g 1
Defer in g 0
Recovered in f 4
Returned normally from f.
```

如果移除 defer 关键词：

```shell
Calling g.
Printing in g 0
Printing in g 1
Printing in g 2
Printing in g 3
Panicking!
Defer in g 3
Defer in g 2
Defer in g 1
Defer in g 0
panic: 4
 
panic PC=0x2a9cd8
[stack trace omitted]
```

