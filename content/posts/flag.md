+++
date = "2019-07-20T10:10:10"
title = "Go语言标准库 - flag(命令行参数解析)"
slug = "flag"
tags = ["stdlib", "flag"]
categories = ["stdlib"]
+++

Go语言内置了非常多具有常用功能的内置包，它们被称为标准库。我会在博客中以每篇文章一个标准库的方式记录我对标准库的理解和学习笔记。

今天是第一篇：flag包，这个包是用来做命令行参数解析的。开发者经常会写一些命令行程序，所以参数解析是常见的需求。

定义参数可以用`flag.String()`、`flag.Int()`、`flag.Int()`等，定义方式有2种：

{{< highlight go >}}
// 1. flag.XXX。其中XXX可以是Int、String和Bool等，其结果是相应类型的指针：
var q = flag.Bool("q", false, "Exit")  // 相当于 var q *bool
// 2. flag.XXXVar。就是在类型后面加`Var`，其实是把flag绑定到对应类型的变量上:
var h bool
flag.BoolVar(&h, "h", false, "Show help")
{{< /highlight >}}

### 解析

定义完全部参数后可以通过调用`flag.Parse()`进行解析。命令行语法有如下三种形式：

{{< highlight bash >}}
-flag // 只支持bool类型
-flag=x  // 任意类型
-flag x // 只支持非bool类型
{{< /highlight >}}

解析之后，flag的值可以直接使用：如果使用的是flag自身，它们是指针；如果绑定到了某个变量，它们是值：

{{< highlight go >}}
fmt.Println("q is ", *q)
fmt.Println("h is ", h)
{{< /highlight >}}

### 一个简单的例子

用一个简单的例子先感受下：

{{< highlight go >}}
package main

import (
	"flag"
	"fmt"
	"os"
)

var (
	n int
    h bool
	q *bool
    s string
)

func init() {
	q = flag.Bool("q", false, "Exit")
	flag.BoolVar(&h, "h", false, "Show help")
	flag.IntVar(&n, "n", 0, "Set number")
	flag.StringVar(&s, "s", "Default string", "Set String")
}

func main() {
    flag.Parse()

    if h {
        flag.Usage()
    } else {
		if *q {
			fmt.Println("q is ", *q)
			os.Exit(0)
		}
		fmt.Println("Number is ", n)
		fmt.Println("String is ", s)
	}
}
{{< /highlight >}}

定义flag通常放在初始化函数init里面，在main函数中首先用`flag.Parse`解析参数。然后根据命令行的输入决定怎么显示：

{{< highlight bash >}}
❯ go run simple.go  # 默认输出
Number is  0
String is  Default string
❯ go run simple.go -h  # 输出帮助信息
Usage of /var/folders/x6/vg82csf90dl3mnnqtbb32d580000gn/T/go-build470860288/b001/exe/simple:
  -h	Show help
  -n int
    	Set number
  -q	Exit
  -s string
    	Set String (default "Default string")
❯ go run simple.go -q
q is  true
❯ go run simple.go -q false  # 错误用法，false没有生效
q is  true
❯ go run simple.go -q=false  # 正确的用法
Number is  0
String is  Default string
❯ go run simple.go -n 10  # 指定n的值
Number is  10
String is  Default string
❯ go run simple.go -s abc  # 指定s的值
Number is  0
String is  abc
❯ go run simple.go -n 10 -s abc  # 同时指定n和s的值
Number is  10
String is  abc
❯ go run simple.go -n 10 -s abc -q  # 程序逻辑中，q为真时直接打印退出了，忽略了另外2个参数
q is  true
{{< /highlight >}}

flag在一般的情况下是够用的了。

### 代码地址

完整代码可以在[这个地址](https://github.com/golang-dev/strconv.code/blob/master/stdlib/flag)找到。

### 延伸阅读

1. https://golang.org/pkg/flag/
2. https://books.studygolang.com/The-Golang-Standard-Library-by-Example/chapter13/13.1.html
