+++
date = "2019-07-22"
title = "Golang第三方命令行工具 - spf13/cobra和urfave/cli"
slug = "cli"
tags = ["cli", "cobra"]
categories = ["cli"]
+++

虽然Go语言标准库里面有[flag包](https://strconv.com/posts/flag/)可以作为命令行解析工具，但是flag包的用法有限，举例几个场景：

1. 子命令。flag虽然也可以支持子命令，如`hugo server`，但是写起来非常麻烦。对于三级甚至更多级别的子命令更不支持。
2. --开头参数。flag只支持带一个减号的参数，例如`hugo --debug`这种支持很有限。
3. 同时支持1/2个开头的参数。例如hugo，支持`-h`和`--help`，都是显示帮助信息。

所以在实际项目中可以引入第三方命令行工具更好的支持这些场景。目前Go语言世界中最流行的工具库包含`spf13/cobra`和`urfave/cli`。

### spf13/cobra

[spf13/cobra](https://github.com/spf13/cobra)既是一个用来创建强大的现代CLI命令行的golang库，也是一个生成程序应用和命令行文件的工具。Docker、Kubernetes、Hugo、 Etcd等程序都使用了它构建命令。

首先安装它：

{{< highlight bash >}}
❯ go get -u github.com/spf13/cobra/cobra
{{< /highlight >}}

先感受下生成程序应用和命令行的用法，也就是直接生成个命令行架子：

{{< highlight bash >}}
❯ mkdir -p ~/strconv.code/cli/src/cobraDemo
❯ cd ~/strconv.code/cli/src/cobraDemo
❯ cobra init --pkg-name cobraDemo  # 创建一个叫做cobraDemo的应用，目前官网文档该没有更新，旧的`cobra init cobraDemo`已经不能用了
❯ tree .  # 在当前目录下会创建一个cmd(里面只包含root.go)和一个main.go
.
├── LICENSE
├── cmd
│   └── root.go
└── main.go

1 directory, 3 file
# 由于指定了包名cobraDemo，且其不在默认的GOROOT或者GOPATH目录下，所以需要让GOPATH加上当前项目的源代码目录
❯ export GOPATH=$GOPATH:/Users/dongwm/strconv.code/cli
❯ go run main.go
A longer description that spans multiple lines and likely contains
examples and usage of using your application. For example:

Cobra is a CLI library for Go that empowers applications.
This application is a tool to generate the needed files
to quickly create a Cobra application.
{{< /highlight >}}

这样可以看到默认的输出了，不过目前还没有子命令。代码架构是这样的(为了让结构清晰，去掉了一部分代码)：

{{< highlight go >}}
// main.go
package main

import "cobraDemo/cmd"

func main() {
  cmd.Execute()
}
// cmd/root.go
package cmd

import (
  "fmt"
  "os"
  "github.com/spf13/cobra"
)

var rootCmd = &cobra.Command{
  Use:   "cobraDemo",
  Short: "A brief description of your application",
  Long: `A longer description...`
}

func init() {
  cobra.OnInitialize(initConfig)  // viper是cobra集成的配置文件读取的库，以后我们会专门说

  rootCmd.PersistentFlags().StringVar(&cfgFile, "config", "", "config file (default is $HOME/.cobraDemo.yaml)") // 添加`--config`参数。它是全局的~


  // Cobra also supports local flags, which will only run
  // when this action is called directly.
  rootCmd.Flags().BoolP("toggle", "t", false, "Help message for toggle") // 添加一个布尔值的参数toggle, 在命令行可以用`-t`或者`--toggle`指定
}

func Execute() {
  if err := rootCmd.Execute(); err != nil {
    fmt.Println(err)
    os.Exit(1)
  }
}
{{< /highlight >}}

接着我们添加一个子命令new:

{{< highlight bash >}}
❯ cobra add new
new created at /Users/dongwm/strconv.code/cli/src/cobraDemo
❯ tree .
.
├── LICENSE
├── cmd
│   ├── new.go
│   └── root.go
└── main.go

1 directory, 4 files
{{< /highlight >}}

可以看到这样就多了`cmd/new.go`文件，去掉注释和空行如下：

{{< highlight go >}}
package cmd

import (
	"fmt"

	"github.com/spf13/cobra"
)
var newCmd = &cobra.Command{
	Use:   "new",
	Short: "A brief description of your command",
	Long: `A longer description...`,
    Run: func(cmd *cobra.Command, args []string) {
           fmt.Println("new called")
    },
}
func init() {
	rootCmd.AddCommand(newCmd)
}
{{< /highlight >}}

添加命令就是在init函数中执行`rootCmd.AddCommand`，而newCmd中比之前看的rootCmd多了一个Run成员，指定执行子命令时要做什么，这样整个命令应用就可以用了。为了让输出更简洁我把Command里面的Long部分改短一些，效果是这样的：

{{< highlight bash >}}
❯ go run main.go --help
A longer description...

Usage:
  cobraDemo [command]

Available Commands:
  help        Help about any command
  new         A brief description of your command

Flags:
      --config string   config file (default is $HOME/.cobraDemo.yaml)
  -h, --help            help for cobraDemo
  -t, --toggle          Help message for toggle

Use "cobraDemo [command] --help" for more information about a command.
❯ go run main.go new --help
A longer description...

Usage:
  cobraDemo new [flags]

Flags:
  -h, --help   help for new

Global Flags:
      --config string   config file (default is $HOME/.cobraDemo.yaml)
❯ go run main.go new
new called
{{< /highlight >}}

可以感受到：

1. `--config`是全局的，根命令和子命令下都包含了这个参数项
2. `-h`(--help)是默认参数项
3. 根命令和子命令可以配置需要的参数项

当然我们可以直接在程序里面集成cobra，就像上面做的代码解析，无非就是用`cobra.Command`创建一个rootCmd，用`rootCmd.AddCommand`添加子命令，在对应代码文件的init函数中定义flag。cobra支持2种定义参数的方法：

1. newCmd.PersistentFlags。全局参数，根命令和子命令下都包含了这个参数项」。可以说定义一次，全局可用。
2. newCmd.Flags。本地参数，只对当前命令下生效，其他子命令下不会继承。

在new子命令下体验定义flag的方法(都用`newCmd.Flags`)：

{{< highlight go >}}
package cmd

import (
	"fmt"

	"github.com/spf13/cobra"
)

var (
	n, a int
	s string
)

var newCmd = &cobra.Command{
	Use:   "new",
	Short: "A brief description of your command",
	Long: `A longer description...`,
	Run: func(cmd *cobra.Command, args []string) {
		fmt.Printf("n is %v\n", n)
		fmt.Printf("a is %v\n", a)
		fmt.Printf("s is %v\n", s)
		q, _ := cmd.Flags().GetBool("q")
		fmt.Printf("q is %v\n", q)
		bbb, _ := cmd.Flags().GetInt("bbb")
		fmt.Printf("bbb is %v\n", bbb)
	},
}

func init() {
	rootCmd.AddCommand(newCmd)

	newCmd.Flags().IntVar(&n, "intf", 0, "Set Int")
	newCmd.Flags().StringVar(&s, "stringf", "sss", "Set String")
	newCmd.Flags().Bool("q", false, "Set Bool")

	newCmd.Flags().IntVarP(&a, "aaa", "a", 1, "Set A")
	newCmd.Flags().IntP("bbb", "b", -1, "Set B")
}
{{< /highlight >}}

和Golang语言内置的flag库用法很像，cobra也支持XXVar这样的方法(XX可以使String、Int、Bool等)，第一个参数用的都是变量内存地址，这样在命令行下传递对应的参数就会改变量的值。另外也支持XXP和XXVarP这样的写法：XXP/XXVarP这样的不仅支持短参数也支持长参数，而XXVar只支持长参数；XXP和XXVarP这2种写法的区别主要是看第一个参数，XXP这样的方式中不需要预先定义变量，把内存地址传进来。

怎么获得解析后的参数项和值呢？就看newCmd的Run，用`fmt.Printf`分别打印了n、a、s(因为定义变量，可以直接找到)，而q和bbb这2个参数是隐式的，需要用`cmd.Flags().GetXX`，关键看参数的值的类型，如q是一个布尔值，所以用`cmd.Flags().GetBool("q")`就能获取对应参数的值了，我们体验下：

{{< highlight bash >}}
❯ go run main.go new -h
A longer description...

Usage:
  cobraDemo new [flags]

Flags:
  -a, --aaa int          Set A (default 1)
  -b, --bbb int          Set B (default -1)
  -h, --help             help for new
      --intf int         Set Int
      --q                Set Bool
      --stringf string   Set String (default "sss")

Global Flags:
      --config string   config file (default is $HOME/.cobraDemo.yaml)
❯ go run main.go new
n is 0
a is 1
s is sss
q is false
bbb is -1
❯ go run main.go new -a 1 --bbb=2 --intf 3 --stringf="abc" --q
n is 3
a is 1
s is abc
q is true
bbb is 2
{{< /highlight >}}

这个库的体验就先到这里了，更多的功能和用法请看官方文档。

### urfave/cli

[urfave/cli](https://github.com/urfave/cli)是一个简单、快速的命令行程序开发框架，使用它的知名项目包含Gogs、Drone、Gitea等。

首先安装它：

{{< highlight bash >}}
❯ go get github.com/urfave/cli
{{< /highlight >}}

先了解下`urfave/cli`中怎么定义Flag

{{< highlight go >}}
package main

import (
	"fmt"
	"github.com/urfave/cli"
	"log"
	"os"
)

var (
	flags []cli.Flag
	host  string
	port  int
)

func init() {
	flags = []cli.Flag{
		cli.StringFlag{
			Name:        "t, host, ip-address",
			Value:       "127.0.0.1",
			Usage:       "Server host",
			Destination: &host,
		},
		cli.IntFlag{
			Name:        "p, port",
			Value:       8000,
			Usage:       "Server port",
			Destination: &port,
		},
	}
}
func main() {
	app := cli.NewApp()
	app.Name = "AppName"
	app.Usage = "Application Usage"
	app.HideVersion = true
	app.Flags = flags
	app.Action = Action

	err := app.Run(os.Args)
	if err != nil {
		log.Fatal(err)
	}
}

func Action(c *cli.Context) error {
	if c.Int("port") < 1024 {
		cli.ShowAppHelp(c)
		return cli.NewExitError("Ports below 1024 is not available", 2)
	}

    fmt.Printf("Listening at: http://%s:%d", host, c.Int("port"))
	return nil
}
{{< /highlight >}}

flags是cli.Flag结构列表，包含主机名和端口2项Flag，每项都指定了Name(可以有多个，用逗号隔开)、Value(默认值)、Usage(帮助信息)、Destination(对应变量的内存地址，因为在命令行内会修改变量)。要注意Flag需要使用对应类型XXFlag（如cli.StringFlag、cli.IntFlag)

在main函数中首先用`cli.NewApp()`创建一个新的app，Name、Usage等项的值在帮助输出中都有体现，HideVersion是用于隐藏版本号，而Action表示「行为」，相当于Parse参数后的最终输出，在这里我把Action独立成了一个函数，里面的逻辑：

1. 如果端口号的值小于1024会先当因帮助信息，再抛错
2. 打印主机和端口的值，这里用了2种方法：hots直接用的是变量；而port用了`c.Int("port")`的方式获取，要注意对应参数项的类型，如果获得host的值需要用`c.String("host")`

我们试一下：

{{< highlight bash >}}
❯ go run use_flags.go
Listening at: http://127.0.0.1:8000
❯ go run use_flags.go -h
NAME:
   AppName - Application Usage

USAGE:
   use_flags [global options] command [command options] [arguments...]

COMMANDS:
     help, h  Shows a list of commands or help for one command

GLOBAL OPTIONS:
   -t value, --host value, --ip-address value  Server host (default: "127.0.0.1")
   -p value, --port value                      Server port (default: 8000)
   --help, -h                                  show help

❯ go run use_flags.go -t 172.16.0.1 --port 8080
Listening at: http://172.16.0.1:8080

❯ go run use_flags.go -t 172.16.0.1 --port 21
NAME:
   AppName - Application Usage

USAGE:
   use_flags [global options] command [command options] [arguments...]

COMMANDS:
     help, h  Shows a list of commands or help for one command

GLOBAL OPTIONS:
   -t value, --host value, --ip-address value  Server host (default: "127.0.0.1")
   -p value, --port value                      Server port (default: 8000)
   --help, -h                                  show help
Ports below 1024 is not available
exit status 2
{{< /highlight >}}

可以看到不同参数下执行效果和输出是不一样的。

命令行参数另外一个重要场景是子命令(Subcommand)，基于官网文档看一下例子：

{{< highlight go >}}
package main

import (
	"fmt"
	"log"
	"os"

	"github.com/urfave/cli"
)

func main() {
	app := cli.NewApp()

	app.Commands = []cli.Command{
		{
			Name:     "add",
			Aliases:  []string{"a"},
			Usage:    "add a task to the list",
			Category: "Add",
			Action: func(c *cli.Context) error {
				fmt.Println("added task: ", c.Args().First())
				return nil
			},
		},
		{
			Name:     "template",
			Aliases:  []string{"t", "tmpl"},
			Usage:    "options for task templates",
			Category: "Template",
			Subcommands: []cli.Command{
				{
					Name:  "add",
					Usage: "add a new template",
					Action: func(c *cli.Context) error {
						fmt.Println("new task template: ", c.Args().First())
						return nil
					},
				},
				{
					Name:  "remove",
					Usage: "remove an existing template",
					Action: func(c *cli.Context) error {
						fmt.Println("removed task template: ", c.Args().First())
						return nil
					},
				},
			},
		},
	}

	app.Name = "AppName"
	app.Usage = "application usage"
	app.Description = "application description"  // 描述
	app.Version = "1.0.1" // 版本

	err := app.Run(os.Args)
	if err != nil {
		log.Fatal(err)
	}
}
{{< /highlight >}}

`urfave/cli`支持分组子命令，这个例子中包含2个子命令，分别是add(在Add组)和template(在Template组)，其中template子命令下也有2个子命令add/remove。子命令支持别名，需要放在Aliases里面，分组需要放在Category里面。

另外这次设置了描述和版本号，会在帮助信息中展示出来，而前一个例子隐藏版本部分信息了。

子命令的行为由Action决定，它的值是一个函数，可以看代码了解。

那么感受一下：

{{< highlight bash >}}
❯ go run use_subcommand.go
NAME:
   AppName - application usage

USAGE:
   use_subcommand [global options] command [command options] [arguments...]

VERSION:
   1.0.1

DESCRIPTION:
   application description

COMMANDS:
     help, h  Shows a list of commands or help for one command

   Add:
     add, a  add a task to the list

   Template:
     template, t, tmpl  options for task templates

GLOBAL OPTIONS:
   --help, -h     show help
   --version, -v  print the version

❯ go run use_subcommand.go add abc
added task:  abc

❯ go run use_subcommand.go template --help
NAME:
   AppName template - options for task templates

USAGE:
   AppName template command [command options] [arguments...]

COMMANDS:
     add     add a new template
     remove  remove an existing template

OPTIONS:
   --help, -h  show help

❯ go run use_subcommand.go template add bcd
new task template:  bcd

❯ go run use_subcommand.go template remove cde
removed task template:  cde

❯ go run use_subcommand.go --version
AppName version 1.0.1
{{< /highlight >}}

### 代码地址

完整代码可以在[这个地址](https://github.com/golang-dev/strconv.code/blob/master/cli)找到。

### 延伸阅读

1. https://github.com/spf13/cobra
2. https://github.com/urfave/cli
