+++
date = "2019-07-25"
title = "Golang开发必备: 打造MacOS下开发环境"
slug = "setup-env-on-macos"
tags = ["macos", "env"]
categories = ["macos"]
+++

学习一门编程语言除了学习它的语法，也要搭建一个正确好用的开发环境，这篇文章分享一下我的Golang开发环境。

### 安装Golang

在MacOS下我一般都是用Homebrew：

{{< highlight bash >}}
❯ brew update
❯ brew install golang
❯ go version  # 安装完成了
go version go1.12.6 darwin/amd64
{{< /highlight >}}

### Go环境变量配置

安装之后要做一点环境变量配置。

#### GOPATH

GOPATH环境变量表示Go的工作目录，这个目录指定了需要从哪个地方寻找GO的包、可执行程序等。用`go get`下载的包都放在这个目录下。

从Go 1.8开始，GOPATH默认是$HOME/go，如果你希望使用其他目录可以在你用的Shell配置文件(如~/.zshrc、~/.bashrc、.bash_profile)里面指定，**我推荐显示的指定GOPATH，哪怕用了默认的目录**。

#### GOROOT

GOROOT指Go语言编译、工具、标准库等的安装路径。

从Go 1.0开始，也不必显示地设置GOROOT环境变量。通常这个路径是`/usr/local/go`，当使用Homebrew安装Golang时目录会放在"$(brew --prefix golang)/libexec"这个目录下。

#### 小结

我的`.zshrc`里面是这样配置的：

{{< highlight bash >}}
export GOPATH="${HOME}/.go"  # 我的个人习惯
export GOROOT="$(brew --prefix golang)/libexec"  # 默认路径
export PATH="$PATH:${GOPATH}/bin:${GOROOT}/bin"
{{< /highlight >}}

### IDE

现在有非常多成熟的集成开发环境（IDE），如Emacs、GoLand、VS Code、Vim、LiteIDE等。它们都是跨平台的，支持代码语法高亮、自动补全、错误检查、代码引用查询与跳转、格式化和调试等特性，可以作为Golang程序开发工具。

其实选择哪个IDE没有标准答案，看你的习惯和喜好，如果你愿意花时间去深入和琢磨一套属于自己的用法和快捷键，上面提到的任何一个 IDE 都可以帮助你完成开发。

我日常用的编辑器是VS Code，所以开发环境都是基于VS Code搞的。

### 安装VS Code

从[官网](https://code.visualstudio.com/)下载对应平台安装文件安装就可以了。

接着安装官方扩展[Go for Visual Studio Code](https://marketplace.visualstudio.com/items?itemName=ms-vscode.Go)，浏览器访问后直接点击`install`按钮即可自动打开VS Code安装之。

### 安装必要的包

接着需要安装一些必要的包。

如果你在VS Code(下文简称Code)里面打开(新建)一个go程序文件，右下角会有类似这样的提示：

{{< highlight bash >}}
The "go-outline" command is not available. Use "go get -v github.com/ramya-rao-a/go-outline" to install.`
{{< /highlight >}}

然后是2个按钮`Install`和`Install All`，因为这样的相关包还很多，通常可以直接选择`Install All`，但是由于某些包被不明原因限制访问，最后会超时失败，错误信息差不多是这样的：

{{< highlight bash >}}
Installing 1 tool at /Users/xiaoxi/bin
  go-outline

Installing github.com/mdempsky/gocode FAILED
Installing github.com/ramya-rao-a/go-outline FAILED

1 tools failed to install.

go-outline:
Error: Command failed: /Users/xiaoxi/opt/go/libexec/bin/go get -u -v github.com/ramya-rao-a/go-outline
github.com/ramya-rao-a/go-outline (download)
Fetching https://golang.org/x/tools/go/buildutil?go-get=1
https fetch failed: Get https://golang.org/x/tools/go/buildutil?go-get=1: dial tcp 216.239.37.1:443: i/o timeout
golang.org/x/tools (download)
# cd /Users/xiaoxi/src/golang.org/x/tools; git pull --ff-only
fatal: unable to access 'https://go.googlesource.com/tools/': Failed to connect to go.googlesource.com port 443: Operation timed out
package golang.org/x/tools/go/buildutil: exit status 1
...
{{< /highlight >}}

这时候需要在Code里面设置代理(Proxy)，为了让大家知道都安装了那些包，另外我一般喜欢在命令行下执行，所以在终端可以这样安装：

{{< highlight bash >}}
export http_proxy=http://10.8.0.1:8118;export https_proxy=http://10.8.0.1:8118;  # 设置代理，这里你可以替换成你自己的代理

go get -u -v github.com/mdempsky/gocode
go get -u -v github.com/rogpeppe/godef
go get -u -v golang.org/x/tools/cmd/guru
go get -u -v golang.org/x/tools/cmd/gorename
go get -u -v golang.org/x/tools/cmd/goimports
go get -u -v github.com/uudashr/gopkgs/cmd/gopkgs
go get -u -v github.com/golangci/golangci-lint/cmd/golangci-lint
go get -u -v github.com/ramya-rao-a/go-outline
go get -u -v github.com/acroca/go-symbols
go get -u -v github.com/zmb3/gogetdoc
go get -u -v github.com/fatih/gomodifytags
go get -u -v github.com/cweill/gotests/...
go get -u -v github.com/josharian/impl
go get -u -v github.com/uudashr/gopkgs/cmd/gopkgs
go get -u -v github.com/davidrjenni/reftools/cmd/fillstruct
{{< /highlight >}}

一眼看去好多包啊。我来分别介绍一下它们是用来做什么的：

1. gocode。代码补全，注意，网上有很多文章用的是https://github.com/nsf/gocode，这是错误的，因为nsf/gocode现在已经不维护了。
2. godef。跳转到代码对应定义的，这个可以说是各种编程语言IDE必备的功能。
3. guru。guru原来叫做oracle，能获得全部代码对应引用
4. gorename。重命名Go源码文件
5. goimports。自动帮你格式化import语句的，它来帮你决定import顺序
6. gopkgs。列出Go包。
7. golangci-lint。旧的用法是gometalinter，现在应该选择快5倍的golangci-lint，这个工具用来做静态检查工具，它会反馈代码风格并给出建议，这样就可以在源码保存是运行它能告诉你现有代码中有什么问题
8. go-outline。当前文件做符号(Symbol)搜索，相当于文件大纲
9. go-symbols。工作区符号搜索
10. gogetdoc。鼠标悬停时可以显示文档(方法和类的签名等帮助信息)
11. gomodifytags。提供tags管理，可以对struct的tag增删改
12. gotests。用例测试
13. impl。自动生成接口实现代码(stubs)
14. gopkgs。导入包时自动补全(列出可导入的包列表，主要用于包导入时的提示功能)
15. fillstruct。根据结构体定义在使用时自动填充默认值

更多信息可以看延伸阅读链接1。注意有些包我没有安装，如golint、gometalinter、goreturns等，因为在对应场景下使用了其他包，就没必要装了(大家不要直接粘贴我或者其他开发者安装的包列表，有自己的思考选择性的安装)

另外这些包其实不止针对于Code，对于其他编辑器也是适用的，你可以根据包名搜索有没有对应编辑器的扩展或者插件，大部分都可以集成进去。

### 配置VS Code

安装之后需要对Code做一些设置。首先使用快捷键`Command+,`打开设置页面(用IDE一定要学好快捷键)，搜索`Go configuration`就可以看到Golang语言相关设置项了。不同的设置项类型不同：有下拉菜单、单选框、Input框，设置需要直接编辑配置文件settings.json。

Code给每一项都设置了默认值，settings.json文件里面的设置会覆盖默认的，你可以直接编辑settings.json文件，也可以在设置页面对设置项做选择（如打勾、从下拉菜单选择你要的选项、在输入框输入对应值等）。我一般直接修改settings.json文件，这样换一台电脑这些配置项可以直接拷贝过去就可用了。下面是我的设置项，都带了说明：

{{< highlight json >}}
{
    "go.docsTool": "gogetdoc", // 使用gogetdoc获取方法和类的签名帮助信息
    "workbench.colorTheme": "Dracula", // 换成我喜欢的主题
    "go.formatTool": "goimports", // 指定代码格式化工具，其他的如gofmt goreturns goformat
    "go.autocompleteUnimportedPackages": true, // 未完成的包会自动补全
    "go.inferGopath": true, // 让编辑器自动推断 GOPATH(包含全局的GOPATH，并沿着当前文件向上找到src目录)
    "go.useCodeSnippetsOnFunctionSuggest": true, // 使用代码片段提示，一会会继续说Snippets
    "go.useCodeSnippetsOnFunctionSuggestWithoutType": true,
    "go.lintTool": "golangci-lint", // 代码静态检查工具，其他的如gometalinter golint revive staticcheck
    "go.gotoSymbol.includeGoroot": true, // 在工作空间进行符号搜索文件时(Shift+cOMMAND+O, #开头)包含位于GOROOT里面的标准库
}
{{< /highlight >}}

PS: 在MacOS下配置文件的路径是`$HOME/Library/Application Support/Code/User/settings.json`。

有一点要说明，我之前也在网上看过别人的配置，发现很多项的键值其实现在都已经不再可用了。怎么知道可用选项、设置说明、可选项等信息呢？

答案是: 看官方vscode-go项目下的package.json文件(延伸阅读链接3)，都在那里面。举个`go.formatTool`的例子：

{{< highlight json >}}
    ...
    "go.formatTool": {
      "type": "string",
      "default": "goreturns",
      "description": "Pick 'gofmt', 'goimports', 'goreturns' or 'goformat' to run on format. Not applicable when using the language server. Choosing 'goimport' or 'goreturns' will add missing imports and remove unused imports.",
      "scope": "resource",
      "enum": [
        "gofmt",
        "goimports",
        "goreturns",
        "goformat"
      ]
    },
    ...
{{< /highlight >}}

直接在文件里面搜很方便，特别适合我这种喜欢在终端打开文件的开发者。

### 使用Snippets(代码片段)

Code中添加Snippet可以提高写代码的效率，就是用别人写好的代码片段实现一些功能。在前面装的官方扩展「Go for Visual Studio Code」里面就带了很多代码片段。举个例子，如果想写一个`for range`语句，你只需要在编辑文件是打入`forr`，然后按Tab，就会出现这样的一段代码：

{{< highlight go >}}
for _, var := range var {

}
{{< /highlight >}}

并且把游标定位到循环里面，非常方便。全部的代码片段可以看延伸阅读链接5

另外Code市场有专门的[Go Snippets](https://marketplace.visualstudio.com/items?itemName=banxi.go-snippets)插件，里面也有很多片段。

当然可以自定义代码片段，可以把你日常常用的一些代码做成代码片段，首先使用`Shift + Command + P`调出命令窗口，输入「snippets」，选择 「Preferences: Open User Snippets」，然后选择需要添加Snippet的语言(对我们来说就是选 go.json)，go.json默认是一个包含详细注释的例子，按它的格式写就可以。我这里举个例子：

{{< highlight go >}}
{
    "println":{
        "prefix": "pln",
        "body":"fmt.Println($0)",
        "description": "Snippet for Println"
    },
    "printf":{
        "prefix": "plf",
        "body": "fmt.Printf(\"$0\")",
        "description": "Snippet for Printf"
    }
}
{{< /highlight >}}

这样写代码时用`pln`再按Tab就能打出对应的`fmt.Println`。当然这只是举个栗子，这种常用的代码在vscode-go项目中都有(分别是`fp`和`ff`)

PS: 使用Snippets最重要的是要记住各种缩写前缀(如前面的forr、pln)，写起来代码来飞快~

### 后记

好啦，就先说到这里，以后有其他的设置和用法我再更新~

### 延伸阅读

1. https://github.com/Microsoft/vscode-go/wiki/Go-tools-that-the-Go-extension-depends-on
2. https://code.visualstudio.com/docs/getstarted/settings
3. https://github.com/microsoft/vscode-go/blob/master/package.json
4. https://code.visualstudio.com/docs/languages/go
5. https://github.com/microsoft/vscode-go/blob/master/snippets/go.json
