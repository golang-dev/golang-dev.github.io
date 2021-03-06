+++
date = "2019-07-31"
title = "使用Elasticsearch(附Golang代码)"
slug = "use-elastic"
tags = ["elasticsearch"]
categories = ["elasticsearch"]
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

这样说明安装并启动了，接着我们使用Golang操作它。目前有2个可供选择的Elasticsearch Golang客户端，我们分别体验它们。

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

es提供了完整的QueryDSL，支持查询的种类很多。olivere/elastic都支持，有兴趣的可以去项目源代码以`search_queries_`开头的文件。前面我们的搜索用的是Termquery，可以感受下实际的请求的json数据((全部代码见query.go))：

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

#### 批量操作(Bulk)

数据库都要支持批量执行的操作，如批量写入。否则设想有一亿条数据，如果一个一个插入并发满了效率太低，并发高了数据库负载扛不住。作为开发者好的习惯是在需要的时候应该一次性的写入一批数据，减少对数据库写入频率。在es里面也支持批量操作：这个「批量」定义要更泛化，不止是指一次多写，还可以删除更新等！

看个例子(全部代码见bulk.go)：

{{< highlight go >}}
subjects := []Subject{
    Subject{
        ID:     1,
        Title:  "肖恩克的救赎",
        Genres: []string{"犯罪", "剧情"},
    },
    Subject{
        ID:     2,
        Title:  "千与千寻",
        Genres: []string{"剧情", "喜剧", "爱情", "战争"},
    },
}

bulkRequest := client.Bulk()
for _, subject := range subjects {
    doc := elastic.NewBulkIndexRequest().Index(indexName).Id(strconv.Itoa(subject.ID)).Doc(subject)
    bulkRequest = bulkRequest.Add(doc)
}

response, err := bulkRequest.Do(ctx)
if err != nil {
    panic(err)
}
failed := response.Failed()
l := len(failed)
if l > 0 {
    fmt.Printf("Error(%d)", l, response.Errors)
}
{{< /highlight >}}

这样就可以一次性的把2个记录写到es里面。再看一个复杂的例子：

{{< highlight go >}}
subject3 := Subject{
    ID:     3,
    Title:  "这个杀手太冷",
    Genres: []string{"剧情", "动作", "犯罪"},
}
subject4 := Subject{
    ID:     4,
    Title:  "阿甘正传",
    Genres: []string{"剧情", "爱情"},
}

subject5 := subject3
subject5.Title = "这个杀手不太冷"

index1Req := elastic.NewBulkIndexRequest().Index(indexName).Id("3").Doc(subject3)
index2Req := elastic.NewBulkIndexRequest().OpType("create").Index(indexName).Id("4").Doc(subject4)
delete1Req := elastic.NewBulkDeleteRequest().Index(indexName).Id("1")
update2Req := elastic.NewBulkUpdateRequest().Index(indexName).Id("3").
            Doc(subject5)

bulkRequest = client.Bulk()
bulkRequest = bulkRequest.Add(index1Req)
bulkRequest = bulkRequest.Add(index2Req)
bulkRequest = bulkRequest.Add(delete1Req)
bulkRequest = bulkRequest.Add(update2Req)

_, err = bulkRequest.Refresh("wait_for").Do(ctx)
if err != nil {
    panic(err)
}

if bulkRequest.NumberOfActions() == 0 {
    fmt.Println("Actions all clear!")
}

searchResult, err := client.Search().
    Index(indexName).
    Sort("id", false). // 按id升序排序
    Pretty(true).
    Do(ctx) // 执行
if err != nil {
    panic(err)
}
var subject Subject
for _, item := range searchResult.Each(reflect.TypeOf(subject)) {
    if t, ok := item.(Subject); ok {
        fmt.Printf("Found: Subject(id=%d, title=%s)\n", t.ID, t.Title)
    }
}
{{< /highlight >}}

这个批量操作里面做了4件事：添加subject3(ID为3)、添加subject4(ID为4)、删除ID为1的记录、更新ID为三的记录(subject5，在原来的subject3中Title故意写错了)。完成bulk操作之后通过搜索(无term条件，表示全部)验证下当前es里面的全部文档：

{{< highlight go >}}
❯ go run bulk.go
Actions all clear!
Found: Subject(id=4, title=阿甘正传)
Found: Subject(id=3, title=这个杀手不太冷)
Found: Subject(id=2, title=千与千寻)
{{< /highlight >}}

可以看到ID3和ID4这2个文档插入了，而ID3的条目标题被更新成正确的，ID1的条目被删除了：这就是批量操作的效果。

### go-elasticsearch

[go-elasticsearch](https://github.com/elastic/go-elasticsearch/tree/7.x)是官方的Golang客户端。相比`olivere/elastic`它起步比较晚，客户端写的也比较简单，直到最近才更新活跃。所以在开发者使用的角度看大部分选择了`olivere/elastic`。

之前我是想写一些例子体验它的，稍微试了下就放弃了。原因有2个：

#### 封装度不够

我评价一个基础库的标准之一就是看其封装的是不是易用。在go-elasticsearch创建文件这样做：

{{< highlight go >}}
var b strings.Builder
b.WriteString(`{"title" : "`)
b.WriteString(title)
b.WriteString(`"}`)

req := esapi.IndexRequest{
Index:      "test",
DocumentID: strconv.Itoa(i + 1),
Body:       strings.NewReader(b.String()),
Refresh:    "true",
}
{{< /highlight >}}

简单说的，就是用`b.WriteString`拼一个字符串类型的JSON数据，第一感觉是「丑」，第二是易用性很差：举个例子，如果要拼的JSON数据包含多个字段，可以设想需要很多次WriteString，非常容易写出BUG，而且维护性和可读性很差。

再看反序列化：

{{< highlight go >}}
var r map[string]interface{}
if err := json.NewDecoder(res.Body).Decode(&r); err != nil {
    log.Printf("Error parsing the response body: %s", err)
} else {
    log.Printf("[%s] %s; version=%d", res.Status(), r["result"], int(r["_version"].(float64)))
}
{{< /highlight >}}

它完全没有封装成结构体，直接`json.NewDecoder(res.Body).Decode(&r)`，非常原始。

#### 不支持QueryDSL

如果使用关系型数据库我是非常推荐使用ORM的，因为拼SQL非常容易有安全问题也容易出错，还不利于代码复用。在文档型数据库中，DSL就是非常重要的。`olivere/elastic`中用比较清晰的QueryDSL写法就能拼出请求的JSON数据，而在这里还得用`b.WriteString`拼一个字符串类型的JSON数据，我只能说太难用了。

所以我选择放弃go-elasticsearch。

### 豆瓣电影Top250数据存入Elasticsearch

现在终于在多个角度理解开发者为什么更喜欢`olivere/elastic`了。`olivere/elastic`功能齐全，项目一直在不断迭代，所以我选择使用它把豆瓣电影Top250数据存Elasticsearch。

在这里修改[用Golang写爬虫(七) - 如何保存数据到文件](https://strconv.com/posts/save-to-file/)中的抓取部分代码，再把对应es逻辑加进来。这里只展示parseUrls和main的部分(在对应位置加了一些注释，全部代码详见doubanCrawler.go)：

{{< highlight go >}}
var (
    indexName = "subject"
    servers   = []string{"http://localhost:9200/"}
    client, _ = elastic.NewClient(elastic.SetURL(servers...))
)

func parseUrls(url string, ch chan bool) {
    doc := fetch(url)
    ctx := context.Background()
    nodes := htmlquery.Find(doc, `//ol[@class="grid_view"]/li`)

    subjects := []Subject{}

    for _, node := range nodes {
        url := htmlquery.FindOne(node, `.//div[@class="hd"]/a/@href`)
        title := htmlquery.FindOne(node, `.//span[@class="title"]/text()`)
        genre := htmlquery.Find(node, `.//div[@class="bd"]/p/text()`) // 包含了演员、导演、类型等内容的文本Node

        id, _ := strconv.Atoi(strings.Split(htmlquery.InnerText(url), "/")[4]) // 把ID转成数字
        genreStr := strings.Split(htmlquery.InnerText(genre[1]), "/")[2] // 选择第二个Node，用`/`分割后选择第三部分就是类型了
        subject := Subject{id, htmlquery.InnerText(title),
            strings.Split(strings.TrimSpace(genreStr), " ")}  // 拼一个结构体对象
        subjects = append(subjects, subject)  // 使用append放到subjects slice里面
    }

    bulkRequest := client.Bulk() // 借用之前的批量操作的代码
    for _, subject := range subjects {
        doc := elastic.NewBulkIndexRequest().Index(indexName).Id(strconv.Itoa(subject.ID)).Doc(subject)
        bulkRequest = bulkRequest.Add(doc)
    }

    response, err := bulkRequest.Do(ctx) // 每页的25个文档一次性写入到es
    if err != nil {
        panic(err)
    }
    failed := response.Failed()
    l := len(failed)
    if l > 0 {
        fmt.Printf("Error(%d)", l, response.Errors)
    }

    log.Println("Finished Url", url)

    ch <- true
}

func main() {
    start := time.Now()
    ch := make(chan bool)

    for i := 0; i < 10; i++ {
        go parseUrls("https://movie.douban.com/top250?start="+strconv.Itoa(25*i), ch)
    }

    for i := 0; i < 10; i++ {
        <-ch
    }

    elapsed := time.Since(start)
    log.Printf("Took %s", elapsed)
}
{{< /highlight >}}

运行一下：

{{< highlight bash >}}
❯ go run doubanCrawler.go
2019/07/31 16:48:49 Fetch Url https://movie.douban.com/top250?start=0
2019/07/31 16:48:49 Fetch Url https://movie.douban.com/top250?start=225
2019/07/31 16:48:49 Fetch Url https://movie.douban.com/top250?start=100
2019/07/31 16:48:49 Fetch Url https://movie.douban.com/top250?start=125
2019/07/31 16:48:49 Fetch Url https://movie.douban.com/top250?start=25
2019/07/31 16:48:49 Fetch Url https://movie.douban.com/top250?start=150
2019/07/31 16:48:49 Fetch Url https://movie.douban.com/top250?start=50
2019/07/31 16:48:49 Fetch Url https://movie.douban.com/top250?start=175
2019/07/31 16:48:49 Fetch Url https://movie.douban.com/top250?start=200
2019/07/31 16:48:49 Fetch Url https://movie.douban.com/top250?start=75
2019/07/31 16:48:50 Finished Url https://movie.douban.com/top250?start=0
2019/07/31 16:48:50 Finished Url https://movie.douban.com/top250?start=100
2019/07/31 16:48:50 Finished Url https://movie.douban.com/top250?start=25
2019/07/31 16:48:50 Finished Url https://movie.douban.com/top250?start=150
2019/07/31 16:48:50 Finished Url https://movie.douban.com/top250?start=225
2019/07/31 16:48:50 Finished Url https://movie.douban.com/top250?start=200
2019/07/31 16:48:50 Finished Url https://movie.douban.com/top250?start=75
2019/07/31 16:48:50 Finished Url https://movie.douban.com/top250?start=175
2019/07/31 16:48:50 Finished Url https://movie.douban.com/top250?start=125
2019/07/31 16:48:50 Finished Url https://movie.douban.com/top250?start=50
2019/07/31 16:48:50 Took 840.198947ms
{{< /highlight >}}

现在数据就已经写入到es了，这次不用Golang客户端Get了，直接在命令行：

{{< highlight bash >}}
❯ http http://localhost:9200/subject/_doc/1291546
HTTP/1.1 200 OK
content-encoding: gzip
content-length: 192
content-type: application/json; charset=UTF-8

{
    "_id": "1291546",
    "_index": "subject",
    "_primary_term": 1,
    "_seq_no": 1043,
    "_source": {
        "genres": [
            "剧情",
            "爱情",
            "同性"
        ],
        "id": 1291546,
        "title": "霸王别姬"
    },
    "_type": "_doc",
    "_version": 3,
    "found": true
}
{{< /highlight >}}

请求的HTTP地址格式是`http://localhost:9200/{indeName}/{typeName}/{id}`，typeName之前没有用过，指es的docType，从Elasticsearch7.0开始已经不再用Type了(详情请看延伸阅读链接3)，7.x之前用的时候可以这样：

{{< highlight go >}}
doc, err := client.Index().
    Index(indexName).
    Type("movie"). // 多加这句
    Id(strconv.Itoa(subject.ID)).
    BodyJson(subject).
    Refresh("wait_for").
    Do(ctx)
{{< /highlight >}}

如果不指定默认是`_doc`。

### 代码地址

完整代码可以在[这个地址](https://github.com/golang-dev/strconv.code/blob/master/elasticsearch)找到。

### 延伸阅读

1. https://www.elastic.co/guide/en/elasticsearch/reference/7.0/query-dsl.html
2. https://github.com/olivere/elastic/wiki/QueryDSL
3. https://www.elastic.co/guide/en/elasticsearch/reference/master/removal-of-types.html
