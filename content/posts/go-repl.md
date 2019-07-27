+++
date = "2019-07-27"
title = "Golang语言中的REPL库"
slug = "go-repl"
tags = ["repl"]
categories = ["repl"]
+++

### REPL

REPL是`Read-Eval-Print Loop`的缩写，是一种简单的，交互式的编程环境，其中REPL分别指：

Read。获得用户输入
Eval。对输入求值
Print。打印，输出求值的结果
Loop。循环，可以不断的重复Read-Eval-Print

REPL对于学习一门新的编程语言非常有帮助，你可以再这个交互环境里面通过输出快速验证你的理解是不是正确。CPython自带了一个这样的编程环境：

{{< highlight python >}}
❯ python3
Python 3.7.1 (default, Dec 13 2018, 22:28:16)
[Clang 10.0.0 (clang-1000.11.45.5)] on darwin
Type "help", "copyright", "credits" or "license" for more information.
>>> a = 1
>>> a + 2
3
>>> print(a - 1)
0
>>> def b(n):
...     return n + 2
...
>>> b(3)
5
{{< /highlight >}}

### IPython

使用Python开发、DEBUG效率高的一个重要原因是由于IPython这个工具，相信做过Python开发的同学都有体会。IPython是一个基于Python Shell的交互式解释器，可以快速验证代码运行结果是否符合预期。它有以下主要特性：

1. Tab自动补全。可以用Tab对变量、函数、方法等自动补全，比如`import os`后，输入`os.`再按Tab就能列出全部方法，在按Tab就会一个个的选择它们，另外可以输入`os.p`再按Tab会列出以p开头的全部方法等。
2. 能快速获得模块/函数/类的信息，如参数、文档、原始代码等。有时候忘记方法名字或者签名可以直接在IPython里面获得对应信息，甚至可以看到原始代码，这非常方便。
3. 支持Python语法高亮。不用再面对纯白色的一大片代码了。

当然IPython还有很多很多其他的功能，如历史记录、各种Magic函数、autoreload等扩展，这些对于开发和调试都非常有帮助，可以说是Python工程师必备工具。

开始学习Golang后，对于这种REPL编程环境非常渴望，因为它对于初学者友好且对于熟悉Golang非常有帮助。

一番调研，目前Golang世界中共有6个可用的REPL解释环境：3个终端使用，3个在线编辑，本文分别介绍它们。

### gomacro

[gomacro](https://github.com/cosmos72/gomacro)是一个具有REPL，Eval，泛型和类似Lisp宏的交互式Go解释器和调试器。

先安装它：

{{< highlight bash >}}
❯ go get -u github.com/cosmos72/gomacro
{{< /highlight >}}


体验一下：

{{< highlight bash >}}
❯ gomacro
...   // 省略一些输出
gomacro> a := 1
gomacro> a
1	// int
gomacro> import "fmt"
gomacro> fmt.Println(a)
1
2	// int
<nil>	// error
gomacro> func Add(a, b int) int {
. . . .    return a + b
. . . .    }
gomacro> Add(1, 2)
3	// int
gomacro> type Comment struct {
. . . .    A int
. . . .    }
gomacro> c := Comment{}
gomacro> c.A
0	// int
gomacro> :inspect fmt.Printf  // 全部支持功能可以输入`:help`回车看到
fmt.Printf	= 0x5084730	// func(string, ...interface {}) (int, error)  // 查看方法签名
// type ? for inspector help
gomacro> :debug Add(1, 2)  // 进入DEBUG模式
// stopped at repl.go:1:1 IP=0, call depth=1. type ? for debugger help
func Add(a, b int) int {
^^^
debug> print a  // 打印a的值和类型
1	// int
debug> vars  // 查看本地变量
// ----------
a	= 1	// int
b	= 2	// int
debug> ?  // 查看帮助信息
// debugger commands:
backtrace       show call stack
env [NAME]      show available functions, variables and constants
                in current scope, or from imported package NAME
?               show this help
help            show this help
inspect EXPR    inspect expression interactively
kill   [EXPR]   terminate execution with panic(EXPR)
print   EXPR    print expression, statement or declaration
list            show current source code
continue        resume normal execution
finish          run until the end of current function
next            execute a single statement, skipping functions
step            execute a single statement, entering functions
vars            show local variables
// abbreviations are allowed if unambiguous. enter repeats last command.
{{< /highlight >}}

总体上的体验就是一个支持基本功能的REPL，支持Tab自动补全(例如输入`fmt.Print`按Tab会在`fmt.Print`、`fmt.Printf`和`fmt.Println`之前切换)、调试和简单的查看函数签名，但是不支持语法高亮。

gomacro对于基本的快速验证代码运行结果是够的，但是不能获取获取源代码，看不了对应实现文档。

### go-pry

[go-pry](https://github.com/d4l3k/go-pry)对自己的描述是「一个Go的交互式REPL，可让你在任何执行点放入代码」。官网有一个动态图非常好的说明它的效果：

![](https://camo.githubusercontent.com/4acbd203127f5c6f4460655f556d432c8aed4cd9/68747470733a2f2f692e696d6775722e636f6d2f483868467a50562e676966)

先安装它：

{{< highlight bash >}}
❯ go get github.com/d4l3k/go-pry
❯ go install -i github.com/d4l3k/go-pry
{{< /highlight >}}

我把它理解为类似于Python里面的IPDB（IPython Debugger），是一个代码命令行调试工具，这个PEPL功能非常有限。体验它一下就能理解了：

{{< highlight bash >}}
❯ go-pry -i="fmt"  # 启动后自动导入了fmt包

From /var/folders/x6/vg82csf90dl3mnnqtbb32d580000gn/T/pry130526050/main.go @ line 9 :

     4:
     5:   "fmt"
     6: )
     7: func main() {
     8:
 =>  9:   pry.Pry()
    10: }
    11:

[2] go-pry> a := 1
=> 1
[3] go-pry> a + 2
=> 3
[4] go-pry> fmt.Println(a)
1
=> []interface {}{2, interface {}(nil)}
[5] go-pry> func Add (a, b int) int { return a + b}
Error:  1:13: expected '(', found Add <nil>
[6] go-pry> type Comment struct {
Error:  1:30: expected ';', found '(' <nil>
{{< /highlight >}}

在使用中时可以感受到，它只支持基本的表达式和函数调用，在输出的过程中会有补全的提示（但是不能自动补全）和函数签名信息。所以它只适合在特定的环境里面调试代码。举个例子，下面这个程序会把终端输入的参数转成整数：

{{< highlight go >}}
package main

import (
    "fmt"
    "os"
    "strconv"
)

func main() {
    args := os.Args
    for i := 1; i < len(args); i++ {
        ret, err := strconv.Atoi(args[i])
        if err != nil {
            fmt.Println(err.Error())
        } else {
            fmt.Println(ret)
        }
    }
}
{{< /highlight >}}

试一下：

{{< highlight bash >}}
❯ go run example.go 1 2 4
1
2
4

❯ go run example.go 1 3 b
1
3
strconv.Atoi: parsing "b": invalid syntax
{{< /highlight >}}

其实作为开发者一眼就能看出来"b"是不能转换的。借用go-pry可以在对应的位置直接插入（就像项目描述说的那样「可让你在任何执行点放入代码」）：

{{< highlight go >}}
package main

import (
    "fmt"
    "os"
    "strconv"

    "github.com/d4l3k/go-pry/pry"
)

func main() {
    args := os.Args
    for i := 1; i < len(args); i++ {
        ret, err := strconv.Atoi(args[i])
        pry.Pry() // 加在了这里，每次循环都会「断点」停下来
        if err != nil {
            fmt.Println(err.Error())
        } else {
            fmt.Println(ret)
        }
    }
}
{{< /highlight >}}

这次我们再看：

{{< highlight bash >}}
❯ go-pry run exampleWithPry.go 1 3 b

From /Users/xiaoxi/strconv.code/repl/goPry/exampleWithPry.go @ line 15 :

    10:
    11: func main() {
    12:   args := os.Args
    13:   for i := 1; i < len(args); i++ {
    14:     ret, err := strconv.Atoi(args[i])
 => 15:     pry.Pry()
    16:     if err != nil {
    17:       fmt.Println(err.Error())
    18:     } else {
    19:       fmt.Println(ret)
    20:     }

[13] go-pry> ret
=> 1
[14] go-pry> err
=> <nil>
[15] go-pry> args
=> []string{"/var/folders/x6/vg82csf90dl3mnnqtbb32d580000gn/T/go-build624971651/b001/exe/exampleWithPry", "1", "3", "b"}
[16] go-pry>  // Ctrl+D 退出当前循环
1
[16] go-pry>  // 继续退出一次循环
3
From /Users/xiaoxi/strconv.code/repl/goPry/exampleWithPry.go @ line 15 :
    10:
    11: func main() { []string
    12:   args := os.Args
    13:   for i := 1; i < len(args); i++ {
    14:     ret, err := strconv.Atoi(args[i])
 => 15:     pry.Pry()
    16:     if err != nil {
    17:       fmt.Println(err.Error())
    18:     } else {
    19:       fmt.Println(ret)
    20:     }

[16] go-pry> ret
=> 0
[17] go-pry> err
=> strconv.NumError{Func:"Atoi", Num:"b", Err:(*errors.errorString)(0xc000064010)}
[18] go-pry> args[i]
=> "b"
[19] go-pry> strconv.Atoi("b")
=> []interface {}{0, (*strconv.NumError)(0xc0003c7b30)}
{{< /highlight >}}

这样就定位到抛错的这一次循环中的各个本地环境变量，知道上下文是什么可以在这个交互环境下试验，就很容易知道问题出在哪里了

### gore

[gore](https://github.com/motemen/gore)是另外一个Go REPL，支持行编辑、自动补全等特性。官网有一个动态图可以体验到它：

![](https://github.com/motemen/gore/raw/master/doc/screencast.gif)

{{< highlight bash >}}
❯ GO111MODULE=off go get -u github.com/motemen/gore/cmd/gore
# 如果希望支持自动补全和更好的输出效果需要安装：
GO111MODULE=off go get -u github.com/mdempsky/gocode
GO111MODULE=off go get -u github.com/k0kubun/pp # 或者用github.com/davecgh/go-spew/spew
{{< /highlight >}}

然后体验它：

{{< highlight bash >}}
❯ gore
gore version 0.4.1  :help for help
gore> a := 1
1
gore> a + 2
3
gore> func Add(a, b int) int { return a + b }
gore> Add(1, 2)
3
gore> type Comment struct {
.....     B string
..... }
gore> c := Comment{}
main.Comment{
  B: "",
}
gore> c.B
""
gore> :import fmt
gore> fmt.Println("Hello")
Hello
6
nil
gore> fmt.Println("Hello")  // 竟然有BUG！！ 多执行一次多重复输出一行
Hello
Hello
6
nil
gore> :help
    :import <package>     import a package
    :type <expr>          print the type of expression
    :print                print current source
    :write [<file>]       write out current source
    :clear                clear the codes
    :doc <expr or pkg>    show documentation
    :help                 show this help
    :quit                 quit the session
gore> :type Add
func(a int, b int) int
gore> :type a
int
gore> :print Add
package main

import (
    "github.com/k0kubun/pp"
    "fmt"
)

func __gore_p(xx ...interface{}) {
    for _, x := range xx {
        pp.Println(x)
    }
}
func main() {
    a := 1
    _ = Add(1, 2)
    type Comment struct{ B string }
    c := Comment{}
    _, _ = fmt.Println("Hello")
    _, _ = fmt.Println("Hello")
}

func Add(a, b int) int { return a + b }

gore> :doc fmt.Println
func Println(a ...interface{}) (n int, err error)
    Println formats using the default formats for its operands and writes to
    standard output. Spaces are always added between operands and a newline is
    appended. It returns the number of bytes written and any write error
    encountered.
{{< /highlight >}}

可以感受到，虽然gore不支持写代码时高亮，但是输出的结果带了颜色还算体验好一些。另外它支持自动补全、查看签名、类型和对应文档，还能看在gore交互环境中的函数的源码(用`:print`)，不过导入包的方法略奇怪(要用`:import`)，且有fmt模块打印相关的BUG(有点不应该了)。

但总体上gore是上述三个库中效果最好的REPL了！

### Repl.it

接着介绍Web端的REPL工具。`Repl.it`是一个云端的在线编辑器和IDE，它支持全部主流的编程语言，包括[Go](https://repl.it/languages/go)。在这个Web端的页面里面可以编辑代码并执行，Web端也会输出结果。它支持显示补全列表和语法高亮，且支持静态检查，直接在对应行显示错误原因。另外可以生成短连接方便分享出去，非常适合快速验证、教学和演示等用途。

### lgo/gophernotes

[lgo](https://github.com/yunabe/lgo)和[gophernotes](https://github.com/gopherdata/gophernotes)都是Golang语言的Jupyter Notebook内核，实现了在Jupyter Notebook(原来的IPython Notebook)上编写Golang代码并执行。

Jupyter Notebook是一个交互式笔记本应用，是一个Web应用，算法和数据分析工程师用的比较多。

lgo和gophernotes具体安装就不演示了，个人觉得还是在终端REPL更好用。有兴趣的可以看一下gophernotes项目下的动态图看效果：

![](https://github.com/gopherdata/gophernotes/raw/master/files/jupyter.gif)

### 代码地址

完整代码可以在[这个地址](https://github.com/golang-dev/strconv.code/blob/master/repl)找到。
