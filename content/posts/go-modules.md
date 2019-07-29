+++
date = "2019-07-28"
title = "Golang中的包管理工具 - Go Modules"
slug = "go-modules"
categories = ["modules"]
+++

### GO包管理的前世今生

先来了解下Go包管理的发展历程

#### GOPATH

在Go1.5之前使用GOROOT和GOPATH这2个系统环境变量来决定包的位置，对于开发者主要使用GOPATH。GOPATH 解决了第三方源码依赖的问题，看一下我本机`$GOPATH/src`下的目录：

{{< highlight bash >}}
❯ tree -L 1 $GOPATH/src/
/Users/dongwm/src/
├── github.com
├── golang.org
├── google.golang.org
└── gopkg.in

5 directories, 0 files
{{< /highlight >}}

如果执行`go get github.com/UserA/ProjectB`，就会把对应的内容里下载到`$GOPATH/src/github.com/UserA/ProjectB`目录。如果在代码中引用这个依赖，需要这么写：

{{< highlight go >}}
import "github.com/UserA/ProjectB"
{{< /highlight >}}

go会去到$GOPATH下面找这个路径，就能找到这个包了。

一开始Golang这么设计是有原因的，因为Google是一个实践Mono Repo(把所有的相关项目都放在一个仓库中)的公司。但更多的公司和组织更多的是用Multi Repo(按模块分为多个仓库)。GOPATH在一定程度上简化了项目构建，但是这种系统级别的环境变量，要求你的全部项目必须位于 GOPATH 下，否则编译器找不到它。如果想要使用你自己的项目组织结构，要么你需要为每个项目设置一个 GOPATH，或者使用软链接来实现。

另外由于GOPATH对其管理下的库不能区分版本，都是指master最新代码，就有了依赖包的管理问题。举个例子，2个项目都依赖了库X，但是A项目要求库的v1版本，B项目要求库的版本是v2版本。那必须设置两个GOPATH来区分，并且在切换工程的时候GOPATH也得切换，徒增开发和实现的复杂度。当然这么设计也是收到了Google 内部广泛使用的基于主干 (trunk/mainline based) 的开发模型影响。所以说GOPATH带有鲜明的Google色彩。

#### Go Vendoring

到了 2014 年，有人提出了`external packages`的概念，在项目的目录下增加一个 vendor 目录来存放外部的包，同时让 go 的 tools 能够感知到这是一个 vendor。在Go 1.5 中vendor作为试验推出，在Go 1.6中作为默认参数被启用。

在这种模式下，会将第三方依赖的源码下载到本地，不同项目下可以有自己不同的vendor。但是这样做又引入了新的问题，因为随着项目的依赖增多，代码库会越来越大，尤其很多带前端资源的项目，几十M到几百M都是很常见的，你的项目可以就依赖它某一点内容，但是却集成了全部资源。

#### 百花齐放

基于前面的问题，Golang 团队成员确定意识到要对包管理提出一个统一的规则。但是问题不是没有规则，而是规则太多了。往往就是一个意见不合，一下子就杀出来一个新的工具。仅官方推荐的包管理工具就有 15 种之多。其中著名的包含如下几个：

1. Godep。它的原理是扫描记录版本控制的信息，并在go命令前加壳来做到依赖管理。
2. Govendor。Govendor是在vendor之后出来的，功能相对Godep多一点，不过就核心问题的解决来说基本是一样的，知不是在vendor目录下通过 vendor.json 文件来记录依赖包的版本。
3. Glide。它算是一个完整的包管理工具，比较像Node里面的npm或者Python里面的pip/pipenv，它通过`glide.yaml`记录依赖信息，通过`glide.lock`文件追踪每个包的具体修改(这样就能够重用依赖树)。

#### dep

对于从零构建项目的新用户来说，Glide功能足够，是个不错的选择。由于那个时间点Golang 依赖管理工具混乱，最终由谁终结这个场面需要官方来决定。后来Golang官方接纳了由社区组织合作开发的[dep](https://github.com/golang/dep)为`official experiment`(Go 1.9)，在相当长的一段时间里面作为标准成为事实上的官方包管理工具。

由于dep已经成为了`official experiment`的过去时，现在我们就不必再去学习了，直接去了解谁才是未来的`official experiment`！

### Go Modules

Modules是Go 1.11中新增的实验性功能，基于vgo演变而来，是Golang官方的包管理工具，本文就来感受一下它。

先铺垫点准备知识。通常我们会在一个Repo(仓库)中创建一组Go Package。举个例子Gorm中共有5个包，一个是gorm，还有4个不同数据库驱动，这么用：

{{< highlight bash >}}
import (
  "github.com/jinzhu/gorm"
  _ "github.com/jinzhu/gorm/dialects/sqlite"
)
{{< /highlight >}}

Go 1.11给这样的一组在同一Repo下面的packages赋予了一个新的抽象概念: module，module是可以版本化的。使用Modules启用一个新的文件go.mod记录module的元信息。

当然一个Repo下也可以有多个module。

### 如何激活Modules?

首先要确保你使用的Go要>=1.11，目前我本地的是最新的1.12所以不需要升级Go：

{{< highlight bash >}}
❯ go version
go version go1.12.6 darwin/amd64
{{< /highlight >}}

再则，目前Modules并不是默认启用的，可以通过环境变量GO111MODULE开启或者关闭，它有三个可选值：off、on、auto，默认值是 auto。

1. off。关闭支持，go 会从GOPATH和vendor文件夹寻找包。
2. on。开启支持，go 会忽略 GOPATH 和 vendor 文件夹，只根据go.mod下载依赖。
3. auto。当项目在$GOPATH/src外，且项目根目录有go.mod文件时，开启模块支持。

注意，使用Modules后就不再依赖GOPATH了，但目前我推荐用auto，因为还有一些库没有支持Go Modules，相信从很快就发布的Go 1.13开始，GOPATH就会逐渐退出Go开发者是视线了。届时就不需要`export GOPATH`，忘掉它吧！

### 使用Go Modules

{{< highlight bash >}}
❯ export GO111MODULE=on  # 对于当前终端会一直让Go Modules在开启状态
❯ go mod init hello  # 初始化，创建hello.mod文件
go: creating new go.mod: module hello
❯ ls
go.mod
❯ cat go.mod
module hello

go 1.12
{{< /highlight >}}

默认的go.mod里面只定义了module名字是hello，并指定了Go的版本。接着创建一个程序hello.go：

{{< highlight go >}}
package main

import (
  log "github.com/sirupsen/logrus"
)

func main() {
  log.WithFields(log.Fields{
    "animal": "walrus",
  }).Info("A walrus appears")
}
{{< /highlight >}}

这里面用了第三方的包`github.com/sirupsen/logrus`，在main函数中打印了一个日志。构建一下：

{{< highlight bash >}}
❯ go build
go: finding github.com/sirupsen/logrus v1.4.2
go: downloading github.com/sirupsen/logrus v1.4.2
go: extracting github.com/sirupsen/logrus v1.4.2
go: finding github.com/stretchr/objx v0.1.1
go: finding github.com/stretchr/testify v1.2.2
go: finding github.com/davecgh/go-spew v1.1.1
go: finding github.com/pmezard/go-difflib v1.0.0
go: finding github.com/konsorten/go-windows-terminal-sequences v1.0.1
go: finding golang.org/x/sys v0.0.0-20190422165155-953cdadca894
go: downloading golang.org/x/sys v0.0.0-20190422165155-953cdadca894
go: extracting golang.org/x/sys v0.0.0-20190422165155-953cdadca894
❯ ./hello
INFO[0000] A walrus appears                              animal=walrus
{{< /highlight >}}

构建过程会更新go.mod文件(在执行go get、go build、 go test、 go list等命令时都会根据需要的依赖自动和更新require语句)：

{{< highlight bash >}}
❯ cat go.mod
module hello

go 1.12

require github.com/sirupsen/logrus v1.4.2
{{< /highlight >}}

显示的用require声明依赖`github.com/sirupsen/logrus`，版本为1.4.2。这个`v1.4.2`是一个semver标签，就是语义化版本号。它的格式应该是`v(major).(minor).(patch)`：

1. v不可省略
2. major: 当做了不兼容的升级时升
3. minor: 当做了向前兼容的升级时升
4. patch: 未改变功能，只是修了bug之类的问题时升

具体的语义化内容可以看延伸阅读链接6。另外在目录下新增了go.sum文件：

{{< highlight bash >}}
❯ cat go.sum
github.com/davecgh/go-spew v1.1.1/go.mod h1:J7Y8YcW2NihsgmVo/mv3lAwl/skON4iLHjSsI+c5H38=
github.com/konsorten/go-windows-terminal-sequences v1.0.1/go.mod h1:T0+1ngSBFLxvqU3pZ+m/2kptfBszLMUkC4ZK/EgS/cQ=
github.com/pmezard/go-difflib v1.0.0/go.mod h1:iKH77koFhYxTK1pcRnkKkqfTogsbg7gZNVY4sRDYZ/4=
github.com/sirupsen/logrus v1.4.2 h1:SPIRibHv4MatM3XXNO2BJeFLZwZ2LvZgfQ5+UNI2im4=
github.com/sirupsen/logrus v1.4.2/go.mod h1:tLMulIdttU9McNUspp0xgXVQah82FyeX6MwdIuYE2rE=
github.com/stretchr/objx v0.1.1/go.mod h1:HFkY916IF+rwdDfMAkV7OtwuqBVzrE8GR6GFx+wExME=
github.com/stretchr/testify v1.2.2/go.mod h1:a8OnRcib4nhh0OaRAV+Yts87kKdq0PP7pXfy6kDkUVs=
golang.org/x/sys v0.0.0-20190422165155-953cdadca894 h1:Cz4ceDQGXuKRnVBDTS23GTn/pU5OE2C0WrNTOYK1Uuc=
golang.org/x/sys v0.0.0-20190422165155-953cdadca894/go.mod h1:h1NjWce9XRLGQEsW7wpKNCjG9DtNlClVuFLEZdDNbEs=
{{< /highlight >}}

这个文件表示了logrus依赖的那些包的具体版本和hash值。它的作用相当于Npm里面的package-lock.json。

这些依赖的包都会被下载到`$GOPATH/pkg/mod`中：

{{< highlight bash >}}
❯ tree $GOPATH/pkg/mod/github.com/ -L 2
/Users/dongweiming/.go/pkg/mod/github.com/
├── davecgh
│   └── go-spew@v1.1.1
├── konsorten
│   └── go-windows-terminal-sequences@v1.0.1
├── pmezard
│   └── go-difflib@v1.0.0
├── sirupsen
│   └── logrus@v1.4.2
└── stretchr
    ├── objx@v0.1.1
    └── testify@v1.2.2

11 directories, 0 files
{{< /highlight >}}

`github.com/sirupsen/logrus`用了这些依赖包对应的版本，其实就是在它的go.mod里面定义的：

{{< highlight bash >}}
❯ cat $GOPATH/pkg/mod/github.com/sirupsen/logrus@v1.4.2/go.mod
module github.com/sirupsen/logrus

require (
	github.com/davecgh/go-spew v1.1.1 // indirect
	github.com/konsorten/go-windows-terminal-sequences v1.0.1
	github.com/pmezard/go-difflib v1.0.0 // indirect
	github.com/stretchr/objx v0.1.1 // indirect
	github.com/stretchr/testify v1.2.2
	golang.org/x/sys v0.0.0-20190422165155-953cdadca894
)
{{< /highlight >}}

其中indirect表示间接引用。GoModules支持很多包管理命令，常用的go mod命令如下：

{{< highlight bash >}}
❯ go list -m all  # 列出全部依赖包
hello
github.com/davecgh/go-spew v1.1.1
github.com/konsorten/go-windows-terminal-sequences v1.0.1
github.com/pmezard/go-difflib v1.0.0
github.com/sirupsen/logrus v1.4.2
github.com/stretchr/objx v0.1.1
github.com/stretchr/testify v1.2.2
golang.org/x/sys v0.0.0-20190422165155-953cdadca894
❯ go get github.com/sirupsen/logrus@v1.4.1  # 下载指定版本的module(非最新)
❯ grep logrus go.mod
	github.com/sirupsen/logrus v1.4.1
❯ go get -u github.com/sirupsen/logrus  # 更新到最新
❯ grep logrus go.mod
	github.com/sirupsen/logrus v1.4.2
❯ tree $GOPATH/pkg/mod/github.com/sirupsen -L 1
/Users/dongweiming/.go/pkg/mod/github.com/sirupsen
├── logrus@v1.4.1
└── logrus@v1.4.2
{{< /highlight >}}

可以看到本地有了2个版本的`github.com/sirupsen/logrus`，按需使用之。另外还可以使用`go get -u=patch`升级到最新的修订版本。

### 发布到Github

如果你希望你的包被别人使用就需要发布到Github或者其他平台上。首先在Github创建一个项目，然后克隆下来，再把前面创建的三个文件拷贝进来，不过go.mod中的module和package要改一下：

{{< highlight bash >}}
❯ git clone https://github.com/golang-dev/helloworld
❯ cd hello
❯ head -n1 go.mod
module github.com/golang-dev/helloworld
{{< /highlight >}}

为了方便大家体验，创建分支v1存储第一个版本，并发布v1.0.0版本(省略git add /commit等操作)：

{{< highlight bash >}}
❯ gco -b v1
❯ git push -u origin v1
❯ git tag v1.0.0  # 创建v1.0.0版本
❯ git push --tags
❯ gco master  # 切回master分支
{{< /highlight >}}

在下一个版本中替换日志库，改用zap:

{{< highlight bash >}}
package hello

import (
	"time"

	"go.uber.org/zap"
)

func Test() {
	sugar := zap.NewExample().Sugar()
	defer sugar.Sync()
	sugar.Infow("failed to fetch URL",
		"url", "http://example.com",
		"attempt", 3,
		"backoff", time.Second,
	)
}

func main() {
    Test()
}
{{< /highlight >}}

注意为了让包可导出，package 后面不是main而是hello。在执行`go build`之后，go.mod和go.sum都改变了，都添加了相关的依赖信息。如go.mod：

{{< highlight bash >}}
diff --git a/go.mod b/go.mod
index 668236a..e6fff1d 100644
--- a/go.mod
+++ b/go.mod
@@ -6,5 +6,9 @@ require (
        github.com/konsorten/go-windows-terminal-sequences v1.0.2 // indirect
        github.com/sirupsen/logrus v1.4.2
        github.com/stretchr/testify v1.3.0 // indirect
+       github.com/uber-go/zap v1.10.0
+       go.uber.org/atomic v1.4.0 // indirect
+       go.uber.org/multierr v1.1.0 // indirect
+       go.uber.org/zap v1.10.0
        golang.org/x/sys v0.0.0-20190726091711-fc99dfbffb4e // indirect
 )
{{< /highlight >}}

这个时候可以使用tidy清理那些用不到的包(发布前都应该清理一下)：

{{< highlight bash >}}
❯ go mod tidy
{{< /highlight >}}

接着发布成v2版本：

{{< highlight bash >}}
❯ gco -b v1.01
❯ git push -f origin v1.01
❯ git tag v1.01
❯ git push --tags
{{< /highlight >}}

现在`github.com/golang-dev/helloworld`就有2个版本了。

然后在另外一个目录下初始化一个新的go.mod就可以依赖并使用它了：

{{< highlight bash >}}
❯ cd ~/strconv.code/modules/github
❯ go mod init hello2
❯ go get github.com/golang-dev/helloworld@v1  # 先安装v1版本的
❯ cat go.mod
module hello2

go 1.12

require github.com/golang-dev/hello v1.0.0 // indirect
{{< /highlight >}}

然后写一个简单的例子运行一下：

{{< highlight go >}}
❯ cat test.go
package main

import (
	hello "github.com/golang-dev/helloworld"
)

func main() {
	hello.Test()
}
❯ go run test.go
INFO[0000] A walrus appears                              animal=walrus
{{< /highlight >}}

这样可以用hello包里面导出的函数Test了。当然，我这里仅仅是为了测试，这个包完全没有用！

接着下载最新的hello包版本再体验：

{{< highlight bash >}}
❯ go get -u github.com/golang-dev/helloworld
go: finding github.com/golang-dev/helloworld v1.0.1
...
❯ go run test.go
{"level":"info","msg":"failed to fetch URL","url":"http://example.com","attempt":3,"backoff":"1s"}
{{< /highlight >}}

这样就更新到最新的版本1.0.1了。

### 后记

Modules将成为Go 1.13默认的依赖包管理方法。目前很多开源项目已经改用Go Modules，如果你在Github上看到项目根目录下包含go.mod 和 go.sum 就说明它支持最新的Go Modules。如果大家之后再开源新的项目，应该支持Go Modules。

### 代码地址

完整代码可以在[这个地址](https://github.com/golang-dev/strconv.code/blob/master/modules)和[这个项目](https://github.com/golang-dev/helloworld)找到。

### 延伸阅读

1. https://xuanwo.io/2019/05/27/go-modules/
2. https://github.com/golang/dep
3. https://github.com/golang/go/wiki/Modules
4. https://roberto.selbach.ca/intro-to-go-modules/
5. https://blog.golang.org/using-go-modules
5. https://semver.org/
