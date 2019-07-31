+++
date = "2019-07-31"
title = "使用Elasticsearch(附Golang代码)"
slug = "use-elastic"
tags = ["elasticsearch"]
categories = ["elasticsearch"]
draft = true
+++

Elasticsearch(以下简称es)是当今最流行的(全文)搜索引擎，在国内外使用它的公司和产品非常多。它使用Lucene作为其核心来实现所有索引和搜索的功能，但是它的目的是通过简单的RESTful API来隐藏Lucene的复杂性，从而让全文搜索变得简单。另外它还可以用作分布式的实时文件存储，每个字段都可以被索引并可被搜索。

### 安装Elasticsearch

Homebrew目前最新的版本是6.8，为了让本文更有价值，会选择更新的7.X版本，需要使用Elastic官方仓库：

{{< highlight bash >}}
❯ brew tap elastic/tap
❯ brew install elastic/tap/elasticsearch-full
❯ brew services start elastic/tap/elasticsearch-full

❯ curl http://localhost:9200
{
  "name" : "xiaoxiMacBook-Pro.local",
  "cluster_name" : "elasticsearch_xiaoxi",
  "cluster_uuid" : "L80N4IVOS7S3BVaXC3wAsw",
  "version" : {
    "number" : "7.2.0",
    "build_flavor" : "default",
    "build_type" : "tar",
    "build_hash" : "508c38a",
    "build_date" : "2019-06-20T15:54:18.811730Z",
    "build_snapshot" : false,
    "lucene_version" : "8.0.0",
    "minimum_wire_compatibility_version" : "6.8.0",
    "minimum_index_compatibility_version" : "6.0.0-beta1"
  },
  "tagline" : "You Know, for Search"
}
{{< /highlight >}}

这样说明安装并启动了，接着我们使用Golang操作它。目前有2个Elasticsearch Golang客户端，我们分别体验它们。

### olivere/elastic

目前被广泛应用的是第三方开发者做的[olivere/elastic](https://github.com/olivere/elastic/tree/release-branch.v7)，由于我们用的Elasticsearch的版本是7.X，所以大家要注意使用`release-branch.v7`版本。首先安装它，现在7.X版本要求使用GoModules，我们这样初始化：

{{< highlight bash >}}
❯ mkdir olivere-elasticsearch-app && cd olivere-elasticsearch-app
❯ cat > go.mod <<-END
  module olivere-elasticsearch-app

  require github.com/olivere/elastic/v7 7.x
END
{{< /highlight >}}

接着了解一下es的各种操作，首先导入相关的包，定义结构体和变量：

{{< highlight go >}}
package main

import (
    "context"
    "encoding/json"
    "fmt"
    "reflect"
    "strconv"

    "github.com/olivere/elastic/v7"
)


var (
    subject   Subject
    indexName = "subject"
    servers   = []string{"http://localhost:9200/"}
)

type Subject struct {
    ID     int      `json:"id"`
    Title  string   `json:"title"`
    Genres []string `json:"genres"`
}
{{< /highlight >}}

由于我们用的是es7，所以`import "github.com/olivere/elastic/v7"`。结构体Subject是基于之前的豆瓣电影Top250的数据定义的，还扩充了genres字段，之后会把抓取部分逻辑集成进来。

{{< highlight go >}}

const mapping = `
{
    "mappings": {
        "properties": {
            "id": {
                "type": "long"
            },
            "title": {
                "type": "text"
            },
            "genres": {
                "type": "keyword"
            }
        }
    }
}`

ctx := context.Background()
client, err := elastic.NewClient(elastic.SetURL(servers...))
if err != nil {
    panic(err)
}

exists, err := client.IndexExists(indexName).Do(ctx)
if err != nil {
    panic(err)
}
if !exists {
    _, err := client.CreateIndex(indexName).BodyString(mapping).Do(ctx)
    if err != nil {
        panic(err)
    }
}
{{< /highlight >}}

上面这部分创建了client，用IndexExists检查索引是否存在。如果索引不存在用CreateIndex创建索引，mapping内容用BodyString传入。

接着是写入数据：

{{< highlight go >}}
subject = Subject{
    ID:     1,
    Title:  "肖恩克的救赎",
    Genres: []string{"犯罪", "剧情"},
}

// 写入
doc, err := client.Index().
    Index(indexName).
    Id(strconv.Itoa(subject.ID)).
    BodyJson(subject).
    Refresh("wait_for").
    Do(ctx)

if err != nil {
    panic(err)
}
fmt.Printf("Indexed with id=%v, type=%s\n", doc.Id, doc.Type)
subject = Subject{
    ID:     2,
    Title:  "千与千寻",
    Genres: []string{"剧情", "喜剧", "爱情", "战争"},
}
fmt.Println(string(subject.ID))
doc, err = client.Index().
    Index(indexName).
    Id(strconv.Itoa(subject.ID)).
    BodyJson(subject).
    Refresh("wait_for").
    Do(ctx)

if err != nil {
    panic(err)
}
{{< /highlight >}}

上面写入2个文档(记录)，Index内传入索引名字，Id内传入对应字符串类型ID，BodyJson传入结构体变量，尤其要注意用`Refresh("wait_for")`，相当于sync方式的写入，最后的 Do就是执行操作。

接着看获取：

{{< highlight go >}}
result, err := client.Get().
    Index(indexName).
    Id(strconv.Itoa(subject.ID)).
    Do(ctx)
if err != nil {
    panic(err)
}
if result.Found {
    fmt.Printf("Got document %v (version=%d, index=%s, type=%s)\n",
        result.Id, result.Version, result.Index, result.Type)
    err := json.Unmarshal(result.Source, &subject)
    if err != nil {
        panic(err)
    }
    fmt.Println(subject.ID, subject.Title, subject.Genres)
}
{{< /highlight >}}

获取用`client.Get()`，传入索引名字和 想要找的记录ID。`result.Found`为真表示找到这个记录了，其中的数据在`result.Source`里面，可以手动用`json.Unmarshal`解析到结构体变量上。

感受一下搜索：

{{< highlight go >}}
func Search(client *elastic.Client, ctx context.Context, genre string) {
    fmt.Printf("Search: %s", genre)
    // Term搜索
    termQuery := elastic.NewTermQuery("genres", genre)
    searchResult, err := client.Search().
        Index(indexName).
        Query(termQuery).
        Sort("id", true). // 按id升序排序
        From(0).Size(10). // 拿前10个结果
        Pretty(true).
        Do(ctx) // 执行
    if err != nil {
        panic(err)
    }
    total := searchResult.TotalHits()
    fmt.Printf("Found %d subjects\n", total)
    if total > 0 {
        for _, item := range searchResult.Each(reflect.TypeOf(subject)) {
            if t, ok := item.(Subject); ok {
                fmt.Printf("Found: Subject(id=%d, title=%s)\n", t.ID, t.Title)
            }
        }

    } else {
        fmt.Println("Not found!")
    }
}

Search(client, ctx, "剧情")
fmt.Println("****")
Search(client, ctx, "犯罪")
{{< /highlight >}}

这个例子中把搜索抽象到函数，分别搜索类型中包含「剧情」和「犯罪」的记录，显示搜索结果总数，并且打印每个记录。这次没有用前面提到的`json.Unmarshal`，而是用`reflect.TypeOf`，同样可以拿到对应的结构体对象。

最后是删除：

{{< highlight go >}}
res, err := client.Delete().
    Index(indexName).
    Id("1").
    Refresh("wait_for").
    Do(ctx)
if err != nil {
    panic(err)
}
if res.Result == "deleted" {
    fmt.Println("Document 1: deleted")
}
fmt.Println("****")
Search(client, ctx, "犯罪")
{{< /highlight >}}

删除使用`client.Delete()`，传入索引名字和ID，另外也要注意使用`Refresh("wait_for")`，要不就是异步的删除，虽然速度快了，但是由于在没有确认删除就执行下一句`Search(client, ctx, "犯罪")`，所以仍然可以搜到那条记录。

上面这些代码片段都在basic.go里面，运行一下:

{{< highlight go >}}
❯ go run basic.go
Indexed with id=1, type=_doc

Got document 2 (version=824633821488, index=subject, type=_doc)
2 千与千寻 [剧情 喜剧 爱情 战争]
Search: 剧情Found 2 subjects
Found: Subject(id=1, title=肖恩克的救赎)
Found: Subject(id=2, title=千与千寻)
****
Search: 犯罪Found 1 subjects
Found: Subject(id=1, title=肖恩克的救赎)
Document 1: deleted
****
Search: 犯罪Found 0 subjects
Not found!
{{< /highlight >}}

#### QueryDSL

es提供了完整的QueryDSL，支持查询的种类很多。olivere/elastic都支持，有兴趣的可以去项目源代码以`search_queries_`开头的文件。前面我们的搜索用的是Termquery，可以感受下实际的请求的json数据：

{{< highlight go >}}
package main

import (
    "encoding/json"
    "fmt"

    "github.com/olivere/elastic/v7"
)

func PrintQuery(src interface{}) {
    fmt.Println("*****")
    data, err := json.MarshalIndent(src, "", "  ")
    if err != nil {
        panic(err)
    }
    fmt.Println(string(data))
}

func main() {
    query := elastic.NewTermQuery("genres", "动画")
    src, err := query.Source()
    if err != nil {
        panic(err)
    }
    PrintQuery(src)
}
{{< /highlight >}}

它的JSON数据格式化后是这样的：

{{< highlight json >}}
{
  "term": {
    "genres": "动画"
  }
}
{{< /highlight >}}

再举例2个Query：

{{< highlight go >}}
boolQuery := elastic.NewBoolQuery()
boolQuery = boolQuery.Must(elastic.NewTermQuery("genres", "剧情"))
boolQuery = boolQuery.Filter(elastic.NewTermQuery("id", 1))
src, err = boolQuery.Source()
if err != nil {
    panic(err)
}
PrintQuery(src)

rangeQuery := elastic.NewRangeQuery("born").
    Gte("2012/01/01").
    Lte("now").
    Format("yyyy/MM/dd")
src, err = rangeQuery.Source()
PrintQuery(src)
{{< /highlight >}}

一个是布尔值Query，一个是区间Query，看一下实际的JSON数据：

{{< highlight json >}}
*****
{
  "bool": {
    "filter": {
      "term": {
        "id": 1
      }
    },
    "must": {
      "term": {
        "genres": "剧情"
      }
    }
  }
}
*****
{
  "range": {
    "born": {
      "format": "yyyy/MM/dd",
      "from": "2012/01/01",
      "include_lower": true,
      "include_upper": true,
      "to": "now"
    }
  }
}
{{< /highlight >}}

查询请求就是这么拼出来的，这部分很重要，具体的可以看延伸阅读链接1和链接2。

#### 批量操作

### go-elasticsearch

目前有2个Elasticsearch Golang客户端。先说官方的Golang客户端[go-elasticsearch](https://github.com/elastic/go-elasticsearch/tree/7.x)。

go-elasticsearch起步比较晚，客户端写的还比较简单，所以大部分开发者都

### 豆瓣电影Top250数据存入Elasticsearch

### 代码地址

完整代码可以在[这个地址](https://github.com/golang-dev/strconv.code/blob/master/file)找到。

### 延伸阅读

1. https://www.elastic.co/guide/en/elasticsearch/reference/7.0/query-dsl.html
2. https://github.com/olivere/elastic/wiki/QueryDSL
