+++
date = "2019-07-30"
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

这样是说明安装并启动了。接着我们使用Golang操作它。

### olivere/elastic

目前有2个Elasticsearch Golang客户端，其中被广泛应用的是第三方开发者做的[olivere/elastic](https://github.com/olivere/elastic/tree/release-branch.v7)，由于我们用的Elasticsearch的版本是7.X，所以大家要注意使用`release-branch.v7`版本。首先安装它，现在7.X版本要求使用GoModules，我们这样初始化：

{{< highlight bash >}}
❯ mkdir olivere-elasticsearch-app && cd olivere-elasticsearch-app
cat > go.mod <<-END
  module my-elasticsearch-app

  require github.com/elastic/go-elasticsearch/v7 7.x
END
{{< /highlight >}}

### go-elasticsearch

目前有2个Elasticsearch Golang客户端。先说官方的Golang客户端[go-elasticsearch](https://github.com/elastic/go-elasticsearch/tree/7.x)。

go-elasticsearch起步比较晚，客户端写的还比较简单，所以大部分开发者都

### 代码地址

完整代码可以在[这个地址](https://github.com/golang-dev/strconv.code/blob/master/file)找到。

### 延伸阅读

1. https://tutorialedge.net/golang/reading-writing-files-in-go/
2. https://gocn.vip/question/265
3. https://golang.org/pkg/encoding/csv/
