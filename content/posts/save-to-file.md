+++
date = "2019-07-13T10:10:10"
title = "用Golang写爬虫(七) - 如何保存数据到文件"
slug = "save-to-file"
tags = ["crawler", "csv"]
categories = ["crawler"]
+++

在之前的练习中获得的条目ID和标题直接用`fmt.Println`或者`log.Println`在终端打印出来了，但是在实际工作需要把它保存在文件或者数据库中。这篇文章学习一下保存到纯文本、CSV和JSON三种文件里。

### 保存在纯文本文件

纯文本文件是只保存文本了，不保存其格式设置的文件，最常见的如txt后缀文件、配置文件、源代码等等。。代码修改思路是

#### 1. 修改parseUrls方法中打印的部分，改成写入文件：

{{< highlight golang >}}
func checkError(err error) {
    if err != nil {
        panic(err)
    }
}

_, err := f.WriteString(strings.Split(htmlquery.InnerText(url), "/")[4] + "\t" +
        htmlquery.InnerText(title) + "\n")
checkError(err)
{{< /highlight >}}

这样就把id和标题用制表符隔开，当然要注意最后还写入了换行符。每次调用WriteString写入文本后要判断是不是有错误err，这次抽象成了一个函数checkError，如果有错误说明有严重问题，这不是只打印信息说明文件就可以了，所以用panic直接抛错结束程序。

PS: 说一个细节，在Python中字符串拼接用+最好应该用join方法，但是在Golang中strings.Join是最慢的，而最快的是bytes.Buffer，具体的可以看延伸阅读链接2里面的实验。但是对于我们这种少量小文本拼接的需求就直接使用+，没有引入bytes.Buffer。

#### 2. 修改main方法，初始化文件，并用defer在函数执行结束后关闭文件

{{< highlight golang >}}
func main() {
    start := time.Now()
    ch := make(chan bool)
    f, err := os.Create("movie.txt")
    checkError(err)
    defer f.Close()
    _, err = f.WriteString("ID\tTitle\n")
    checkError(err)

    for i := 0; i < 10; i++ {
        go parseUrls("https://movie.douban.com/top250?start="+strconv.Itoa(25*i), ch, f)
    }

    for i := 0; i < 10; i++ {
        <-ch
    }
    f.Sync()

    elapsed := time.Since(start)
    log.Printf("Took %s", elapsed)
}
{{< /highlight >}}

这段代码中，用`os.Create`创建一个文件，然后写了一行`ID\tTitle\n`描述字段。而在parseUrls函数中增加了参数f（类型是*os.File），这样就可以在parseUrls把抓到的数据写入文件了。最后还用了Sync方法，它的作用是将文件系统的最近写入的数据在内存中的拷贝刷新到硬盘中，这是一个安全保存数据的习惯。

运行一下：

{{< highlight bash >}}
❯ go run doubanCrawler1.go
...
❯ wc -l movie.txt
251 movie.txt  # 一行字段描述，250行代表Top250个条目
❯ head -3 movie.txt
ID	Title
1292052	肖申克的救赎
1291546	霸王别姬
{{< /highlight >}}

### 保存在CSV文件

数据处理工作中经常用到csv（逗号分隔值Comma-Separated Values）文件，其文件以纯文本形式存储表格数据（数字和文本），字段间用某种分隔符（常见逗号或制表符）分隔。Golang标准库中有用于编码csv文件的`encoding/csv`包，可以直接使用。还是按之前的2步：

先修改 parseUrls 方法中打印的部分，改成写入csv文件

{{< highlight golang >}}
import (
    ...
    "encoding/csv"
)

err := w.Write([]string{
    strings.Split(htmlquery.InnerText(url), "/")[4],
    htmlquery.InnerText(title)})
checkError(err)
{{< /highlight >}}

Write方法传入的是一个切片，元素分别是ID和标题。然后改main函数：

{{< highlight golang >}}
func main() {
    start := time.Now()
    ch := make(chan bool)
    f, err := os.Create("movie.csv")
    checkError(err)
    defer f.Close()

    writer := csv.NewWriter(f)
    defer writer.Flush()

    err = writer.Write([]string{"ID", "Title"})
    checkError(err)

    for i := 0; i < 10; i++ {
        go parseUrls("https://movie.douban.com/top250?start="+strconv.Itoa(25*i), ch, writer)
    }

    for i := 0; i < 10; i++ {
        <-ch
    }

    elapsed := time.Since(start)
    log.Printf("Took %s", elapsed)
}
{{< /highlight >}}

首先还是用os.Create创建一个文件，按习惯使用csv后缀。用`csv.NewWriter`创建一个写csv的writer。parseUrls的最后一个参数改成了writer（类型是*csv.Writer）。这样就可以了：

{{< highlight bash >}}
❯ go run doubanCrawler2.go
...
❯ head -3 movie.csv
ID,Title
1295865,燃情岁月
1395091,未麻的部屋
{{< /highlight >}}

### 后记

虽然本节只是学习文件的写，没有读的内容。但是也算大体了解了文件操作。

### 代码地址

完整代码可以在[这个地址](https://github.com/golang-dev/strconv.code/blob/master/file)找到。

### 延伸阅读

1. https://tutorialedge.net/golang/reading-writing-files-in-go/
2. https://gocn.vip/question/265
3. https://golang.org/pkg/encoding/csv/
