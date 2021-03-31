---
title: "Go Function Closure"
date: 2021-03-24T22:00:42+08:00
aliases: []
categories:
 - Programming Language
tags: 
 - Golang
 - Function Closure
description: 
featured_image:
draft: false
author: 1uvu
---

在介绍 Golang 中的函数闭包之前，需要了解到 Golang 中的函数也是值，可以由变量来定义声明，因此也可以像其他值一样传递、作为参数或返回值。

## 什么是函数闭包

而当一个函数作为变量在另一个函数中定义声明或是返回，并且引用了函数体之外的变量时，就发生了函数闭包。也就是说，闭包是一个**函数值**，它**引用了其函数体之外的变量**，并可以**访问并修改**其引用的变量，这个函数与其引用的函数体外的变量**绑定**到一起了。

在 Go 中，**函数字面都是闭包**。

## 函数闭包的实例

这是官方 tour 中使用函数闭包的实现斐波那契数列的实例，可以由此简单地理解下函数闭包的使用。

```go
package main

import "fmt"

func fibonacci() func() int {
	i := 0
	j := 1
	return func() int {
		res := i
		i, j = j, i + j
		return res
	}
}

func main() {
	f := fibonacci()
	for i := 0; i < 10; i++ {
		fmt.Println(f())
	}
}
```

## 函数闭包的好处

以下均是个人理解：

总的来说，函数闭包适合用在对一个函数**频繁调用求值**的情况，闭包函数**与变量相关联**，可以**保存变量的中间状态**，向主函数**隐藏了中间变量的定义**，避免了变量的**泛滥**和**误用**。

函数闭包的关键在于，闭包函数对于**中间变量定义的隐藏**和其**中间状态的保持**。

如上面的斐波那契数列例子，如不使用闭包，除了定义 `var i, j int` 这两个中间变量来计算数列值，为了程序语义那么可能还需额外定义一个变量 `var cur int` 来保存当前的数列值，在主动跳出循环后才可继续使用数列的值。这时，在当前作用域中，就多出了三个只在当前代码段使用的变量。这会间接地导致导致**变量泛滥和误用**。

## 函数闭包注意事项

一、由于中间状态的保持，所以需要额外注意当前闭包处于哪种状态，避免误用风险，更好地方式是，**每次需要使用闭包函数的时候就定义新的闭包**。

二、当闭包函数**引用了全局作用域的变量**时，需要特别小心，因为其他代码对于全局变量的访问均会影响闭包的下一次状态，如：

```go
package main

import "fmt"

var add int = 1

func adder() func(int) int {
	sum := 0
	return func(x int) int {
		sum += x + add
		return sum
	}
}

func main() {
	pos := adder()
	for i := 0; i < 10; i++ {
		// 把 add++ 注释掉试一试
		add++
		fmt.Println(pos(i), add)
	}
}
```

经过良好设计的闭包函数，其**执行的上下文环境应该是完全封闭的**，因此，想要完全避免这个问题，就是**不在闭包函数中引用全局变量**。

三、前面提到闭包函数与其所引用的变量绑定起来，但是这个绑定并非是瞬时的。再看个例子：

```go
package main

import "fmt"

func foo(x int) []func() {
    var fs []func()
    values := []int{1, 2, 3, 4}
    for _, val := range values {
        fs = append(fs, func() {
            fmt.Println(x+val)
        })
    }
    return fs
}

func main() {
    fs := foo(6)
    for _, f := range fs {
        f()
    }
    // 会输出四个 10
}
```

也就是说，在声明一个闭包函数时，并不会直接将当前的环境状态绑定给它，在调用闭包函数时，它才会获取最新的外部环境。在这个例子中，`val` 的最后最新值为 `4`，所以 `fs` 中的闭包函数所绑定的 `val` 在调用时都会使用这一个值。

这也称为**闭包的延迟绑定**。而在并发程序中，如使用了 **goroutine** 等，在异步或并发执行，同样会触发延迟绑定，需要特别注意：**Go Routine的匿名函数的延迟绑定本质就是闭包的延迟绑定**。

个人认为，想要彻底规避延迟绑定问题是不可能的，只有在代码书写上面下一些功夫，如：

- 使用时才定义闭包
- 尽量少用闭包

总之一句话：

> 在Go中，**函数字面都是闭包**，需要尽量保证**函数内引用变量的生命周期**与**函数的活动时间**相同。

日后有新的认识再来补充，over~
