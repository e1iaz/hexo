---
title: 《Elasticsearch权威指南》读书笔记
date: 2019-11-13 17:25:30
categories: ["Elasticsearch", "读书笔记"]
tags: ["Elasticsearch", "读书笔记"]
---

# 《Elasticsearch权威指南》读书笔记

## 基础入门

Elasticsearch是一个实时的分布式搜索分析引擎，被用作全文检索、结构化搜索、分析以及这三个功能的组合。

### 你知道的，为了搜索

#### 和Elasticsearch交互

##### Java API

Elasticsearch内置了两个客户端

###### 节点客户端（Node client）

节点客户端是一个非数据节点加入本地集群，但它知道数据在哪个几点钟， 并把请求转发到正确的节点。

###### 传输客户端（Transport client）

将请求发送到远程集群中的节点上，本身不加入集群

两个客户端都是通过9300端口并使用Elasticsearch的原生传输协议和集群交互，集群中也通过9300端口通信

##### RESTful API with JSON over HTTP

所有语言可以通过RESTful API通过9200端口与Elasticsearch通信，模拟一个HTTP请求就可以进行交互

#### 面向文档

文档是以某种格式或者编码来封装和编码数据，课粗略的等价于对象这个概念，文档存储在一个单一存储中允许不同类型的文档，运行在文档中的字段是可选的。

#### 索引员工文档

一个Elasticsearch集群可以包含多个索引，每个索引可以包含多个类型，每个类型存储多个文档，每个文档有多个属性

>索引（名词）:相当于关系型数据库中的database  
>索引（动词）：索引一个文档是指存储一个文档到一个索引中，相当于`INSERT`  
>倒排索引：MySQL通过添加一个索引，比如B+树索引到指定的列上，以提高检索速度，Elasticsearch使用`倒排索引`来提高检索速度  
>>倒排索引是存储在全文搜索下某个单词在一个文档或者一组文档中的存储位置的映射。详情可看[倒排索引](https://zh.wikipedia.org/wiki/%E5%80%92%E6%8E%92%E7%B4%A2%E5%BC%95)

添加一个文档的命令很简单

```bash
curl -X PUT 'http://localhost:9200/megacorp/employee/1' -d '
{
    "first_name" : "John",
    "last_name" : "Smith",
    "age" : 25,
    "about" : "I love to go rock climbing",
    "interests" : ["sports", "music"]
}
'
```

`megacorp`是索引名称，`employee`是类型名称，`1`是id，`-d`的请求体是JSON文档内容

#### 轻量搜索

搜索可以通过`GET`请求获取文档，比如  

```bash
curl -X GET 'http://localhost:9200/megacorp/employee/1'
```

来获取`megacorp`索引下的`employee`类型的id为`1`的文档。  
如果想检索指定索引类型下的所有文档，可以使用`_search`代替`1`。如果想返回`last_name`是`Smith`的文档，则在url最后追加`?q=last_name:Smith`
>在url后面添加`?pretty`可以使返回的JSON结构化展示

#### 使用查询表达式搜索

通过命令可以方便查询，但有局限性，所以可以使用查询表达式，例如上一条查询可以修改为

```bash
curl -X GET 'http://localhost:9200/megacorp/employee/_search' -d '
{
    "query" : {
        "match" : {
            "last_name" : "Smith"
        }
    }
}
'
```

`match`的搜索会将词分开，比如`{match:{"about":"rock climbing"}}`，它会匹配`about`中包含`rock`和`climbing`，而不是会匹配`rock climbing`，要想匹配短语，则需要使用`match_phrase`关键字了

#### 分析

有了数据后可以做分析，elasticsearch有个功能叫做聚合，例子如下

```bash
curl -X GET 'http://localhost:9200/megacorp/employee/_search' -d '
{
    "aggs" : {
        "all_interests" : {
            "terms" : { "field" : "interests" }
        }
    }
}
'
```

这条语句是将数据中所有的`interests`分类，并统计数量，同时这个聚合功能可以与其他查询构成一个组合查询，比如`last_name`是`Smith`的爱好。而且还构成分级查询，比如各个爱好的平均年龄

```bash
curl -X GET 'http://localhost:9200/megacorp/employee/_search' -d '
{
    "aggs" : {
        "all_interests" : {
            "terms" : { "field" : "interests" },
            "aggs" : {
                "avg_age" : {
                    "avg" : {"field" : "age"}
                }
            }
        }
    }
}
'
```

### 集群内的原理

#### 空集群

空集群是指一个单独的节点，里面不包含任何数据和索引。一个运行的实例是一个节点，而集群是由一个或多个拥有相同的`cluster.name`配置的节点组成，共同承担数据和负载压力，当有节点加入或移除，集群会重新平均分布所有数据。  
当一个节点的配置`node.master: true`，则有资格被选举为主节点，主节点负责管理集群范围内的所有变更，而不需要涉及到文档数据的操作，流量的增加也不会成为瓶颈。  
每个节点都知道任何文档的位置，且可以将请求直接转发到处理文档的节点并收集回数据，将结果返回给客户端。  

#### 添加索引

索引是保存相关数据的地方，实际上指向一个或者多个物理分片的逻辑命名空间。  
而分片是一个底层的工作单元，保存全部数据的一部分。一个分片是一个Lucene实例，且本身就是一个完成的搜索引擎。文档被存储和索引到分片内，但应用程序是世界与索引交互而不是和分片交互。  
Elasticsearch利用分片将数据分发到集群内各处，分片是数据的容器，文档保存在分片内，分片又被分配到集群的各个节点，当集群规模扩大或者缩小，Elasticsearch会自动在各节点迁移分片，使得数据君越分布在集群内。  
分片分为主分片和副本分片，任何一个分片都归属于一个主分片。副本分片是主分片的拷贝，是作为硬件故障下保护数据不丢失的冗余备份，并为文档操作提供服务。  
在索引建立的时候就确定了主分片的数量，而副本分片数量可以随时修改。命令如下

```bash
curl -X PUT 'http://localhost:9200/blogs' -d '
{
    "settings" : {
        "number_of_shards" : 3,
        "number_of_replicas" : 1
    }
}
'
```

可以通过`_cluster/health`来查询集群健康，在单节点执行上面的命令后，集群健康为`yellow`，表示主分片正常运行（集群可以正常服务所有请求），但是副本分片没有全部处于正常状态。这里是因为副本分片还没有被分配到任何节点，在同一节点上即保存主分片又保存副本分片没有意义，副本分片的意义是为了保护数据不丢失。  
>green：所有的主分片和副本分片都正常运行  
>yellow：所有的主分片都正常运行，但不是所有的副本分片都正常运行  
>red：主分片没能正常运行

#### 添加故障转移

启动第二个节点就可以避免单点故障问题——没有冗余，需要配置与第一个节点相同的`cluster.name`，这样就会自动加入到集群当中。这时，第一个节点就会将副本分片分配到第二个节点了，就无须担心一个节点宕机。

#### 水平扩容

分片是一个功能完整的搜索引擎，拥有使用一个节点的所有资源的能力，如果有6个分片（3主 + 3副本）的索引可以最大扩容到6个节点，每个节点一个分片，每个分片拥有所有节点的全部资源。随着节点的增加，可以增加副本分片的数量，这样可以提高吞吐量。  
从性能上看，肯定是节点越多、副本分片数量越多越好，可实际上，要找到一个平衡点，避免资源浪费。

#### 应对故障

假设一个集群中有3个几点，我们关闭一个主节点，此时集群中必须拥有一个主节点来保证正常工作，所以发生的第一件事情就是选举一个新的主节点。  
因为关闭了一个主节点，则上面的主分片也不会存在，此时检查集群健康则状态是`red`。好在其他节点有主分片的完整副本，所以新的主节点会将这些在其他节点上的对应副本节点提升为主分片。

### 数据输入和输出

#### 文档元数据

一个文档不仅有数据，还有元数据，既有关文档的信息，有三个必须的元数据元素：`_index` `_type` `_id`。  

##### _index

文档在哪存放，即索引，是因有共同的特性被分组到一起的文档集合。实际上，Elasticsearch中数据被存储和索引在分片中，而索引是逻辑上的命名空间，由一个或者多个分片组合在一起，对于程序而言根本不关心分片，只需要知道文档位于哪个索引内。  
>索引名必须小写，不能以下划线开头，不能包含逗号

##### _type

对数据进行逻辑分区
>命名可以是大写或者小写，但不能以下划线或者句号开头，不应该包含逗号，且长度限制为256个字符

##### _id

ID是一个字符串，与`_index`和`_type`组成可以唯一确定文档。ID可有Elasticsearch自动生成

#### 取回一个文档

模板是`curl -X GET 'http://localhost:9200/{_index}/{_type}/{_id}'`，可以在后面加一些参数（`pretty`等）。传递curl的`-i`参数可以显示相应的头，传递`?_source`可以返回指定的字段，`/_source`只返回`_source`字段，不需要任何元数据。

#### 更新整个文档

更新现有文档可以使用`index`API进行实现，即PUT，在内部，elasticsearch把旧文档标记为已删除，并增加新的文档，虽然不再对旧文档进行访问，但不会立即消失，而是在之后有elasticsearch在后台自动清理。  
还有使用`update`API，实现过程与PUT相同  

1. 从旧文档构建JSON
2. 更新该JSON
3. 删除旧文档
4. 索引新的文档

#### 创建新的文档

`_index` `_type` `_id`组合可以唯一标识一个文档，所以可以使用`POST /index/type`然后让elasticsearch自动生成id。如果存在自己定义id，则要让elasticsearch在`_index` `_type` `_id`不存在的情况下再创建，使用`PUT /index/type/id?op_type=create`或者`PUT /index/type/id/_create`。船舰成功返回201，不成功即原先已存在返回409

#### 乐观并发控制

乐观锁，并不是去上锁，而是选择一个版本号，然后通过版本号去判断读写之间数据有没有被改变。elasticsearch的版本号分为内部和外部，都是在`PUT /index/type/id?version=`来判断的，只是内部版本号判断是相等，只有PUT的版本号等于现在的版本号才会去更新。外部版本号通常是其他数据库做主数据库，elasticsearch做搜索，如果主数据库有了版本号或者可以作为版本号的字段就可以在elasticsearch种通过增加`version_type=external`到查询字符串的方式重用，且检查不是判断是否相等，而是是否大于版本号。在索引、删除和创建新文档的时候可以指定外部版本号。

#### 取回多个文档

将多个请求合并成一个，避免单独处理花费的网络延时和开销，可以使用`mget`API将检索请求放到一个请求中。API要求有一个`docs`数组作为参数，每个元素需要包含`_index` `_type` `_id`，如果想检索一个或者多个特定的字段，可以通过`_source`参数指定这些的名字。

``` bash
curl -X GET 'http://localhost:9200/_mget?pretty' -d '
{
  "docs" : [{
    "_index" : "megacorp",
    "_type" : "employee",
    "_id" : 1
  }]
}
'
```

响应体也包含docs数据，对于每个请求中指定的文档，每个数组都包含对应的相应，且与请求中的顺序相同。每个文档都是单独检索和报告的。  
即使文档没有找到，HTTP的状态码仍然是200，因为mget已经成功执行了，为了确保文档是否成功还是失败，需要检查found标记。

#### 代价较小的批量操作

`bulk`API允许在单个步骤中进行多次`create` `index` `update` `delete`请求，但是规定了请求格式

```bash
{ action: { metadata }}\n
{ request body }\n
```

但是并不是批量越大越好，有个均衡的最佳值

### 分布式文档存储

#### 路由一个文档到一个分片中

当索引一个文档的时候，文档会被存储到一个主分片中，但是主分片有很多个，具体应该是放到哪个分片中，有一个公式决定的`shard = hash(routing) % number_of_primary_shards`。`routing`是一个可变值，默认是文档的`_id`，也可以是自定义值。通过hash函数生成要给数字再除以`number_of_primary_shards`（主分片的数量）得到的余数就是哪个分片的位置。这也是主分片数量不允许修改的原因。

#### 新建、索引和删除文档

新建、索引和删除都是写操作，必须在主分片完成后才能被复制到相关的副本分片。还有一些可选的参数，但很少使用

- consistency，即一致性，在默认设置下，即使仅仅是在试图执行一个写操作之前，主分片会要求必须要有规定数量的分片副本处于活动可用状态，才会执行写操作。这是为了避免网络故障出现数据不一致。数量为`int((primary + number_of_replicas) / 2) + 1`。值可以是`one`（主分片没问题就执行）或者`all`（所有主副分片没有问题）默认是`quorum`（大部分分片副本没问题），如果此时只启动两个节点就永远达不到
- timeout，如果没有足够的副本分片elasticsearch会等待，默认时间为1分钟，可以通过timeout设置。

>timeout并不是停止执行查询，而是告知正在协调的节点返回目前为止收集的结果并关闭链接，在后台，其他分片可能还在执行查询。  
>基于文档的复制
>当主分片把更改转发到副本分片时，他不会转发更新请求，而是转发完整文档的新版本，且这些更改会异步转发到副本分片，不能保证它们以发送的顺序到达。

#### 分布式取回一个文档

在处理读请求时，协调结点在每次请求时，都会通过轮询所有的副本分片来达到负载均衡。  
如果文档被检索时，已经被索引的文档可能在已经存在于主分片上，但是还没有复制到副本分片，此时副本分片报告不存在，主分片返回成功文档。一旦索引请求成功返回给用户，文档在主分片和副本分片都可用。

### 搜索 —— 最基本的工具

#### 分页

在分布式系统中深度分页是有问题的，假设有5个主分片的索引中的索引，当请求结果的的第一页，每个分片产生前十的结果，然后返回给协调节点，协调节点对50个结果排序得到全部结果。再假设请求的是1000页结果，从100001到10010条，所有节点都要产生前10010个结果，然后协调节点要对50050个结果排序，然后抛弃50040

### 映射和分析

当索引一个文档的时候，elasticsearch取出所有字段的值拼接成一个大的字符串，作为`_all`字段进行索引。除非设置特定字段，否则查询字符串就是用`_all`字段进行搜索。特殊类型比如date  

```bash
GET /_search?q=2019                 # 12 results
GET /_search?q=2019-11-29           # 12 results
GET /_search?q=date:2019-11-29      # 1 result
GET /_search?q=date:2014            # 0 results
```

造成结果不同的就是类型的问题，可以使用`GET /index/type/tweet`查看，date字段的映射是date类型，而_all是text类型

#### 倒排索引

倒排索引之前已经写过了，是由文档中所有不重复词的列表构成，对于每个词都有一个包含它的文档列表。  
假设有两个文档`The quick brown fox jumped over the lazy dog`和`Quick brown foxes leap over lazy dogs in summer`。如果我们搜索`quick brown`则两个文档都有，只是第一个文档的包含两个，第二个文档只包含了`brown`，所以第一个文档匹配度更高。但是这样的倒排索引有些问题，比如`Quick`和`quick`可以是相同的词，`fox`和`foxes`是相似的，`jumped`和`leap`意思相近，因此如果搜索`Quick fox`就不会出现任何文档。所以我们要将倒排索引重构，根据一定的规则，让它更符合全文搜索而不是精准查询。

#### 分析和分析器

如果只有倒排归类好了，没有分析，查询`leap`可能就会没有返回。所以存在分析，首先将一块文本分成适合于倒排索引的词条，之后将这些词条统一化为标准格式。  
分析器实际上就是将三个功能封装到一个包里：字符过滤器、分词器、Token过滤器。字符过滤器，在分词前整理字符串，比如去掉HTML，把&转为and；分词器是将字符串分为单个词条，像是jieba分词；Tonken过滤器，会改变词条，比如转为小写，去掉无关词条(a, and, or)，增加词条，比如leap转为jump。  
elasticsearch内置了一些分析器，拿个字符串举例`"Set the shape to semi-transparent by calling set_trans(5)"`

- 标准分析器，默认使用的分析器，根据[Unicode 联盟](http://www.unicode.org/reports/tr29/)定义单词划分文本，删除大部分标点，然后小写`set, the, shape, to, semi, transparent, by, calling, set_trans, 5`
- 简单分析器，在任何不是字母的地方分割文本，小写`set, the, shape, to, semi, transparent, by, calling, set, trans`
- 空格分析器，在空格的地方划分文本`Set, the, shape, to, semi-transparent, by, calling, set_trans(5)`
- 语言分析器，针对指定语言的特点设计的分析器，比如英语分析器附带一组无用词，它们会删除，是词条更符合英语语法`set, shape, semi, transpar, call, set_tran, 5`

不是所有情况都需要分析器的，要是查询精确值就不会分析字符串。  
可以使用`analyze`API查看文本是如何分析的

```bash
curl -X GET 'http://localhost:9200/_analyze' -d '
{
    "analyzer" : "standard",
    "text" : "Text to analyze"
}
```

#### 映射

映射类似于MySQL的schema，它包含了字段、类型等信息。当索引一个包含新域的文档，elasticsearch会使用动态映射，即通过JSON中基本数据类型尝试猜测域类型。  
在首次创建一个索引的时候，可以指定类型的映射，也可以为新类型增加映射，或者为已存在的类型更新映射。但是不能修改存在的域映射，如果一个域的映射已经存在，那么该域的数据可能已经被索引，如果修改这个域的映射，索引的数据可能出错，不能被正常搜索。  
这段看的有些蒙，附上原版  
>You can specify the mapping for a type when you first create an index. Alternatively, you can add the mapping for a new type (or update the mapping for an existing type) later, using the /_mapping endpoint.  
>Although you can add to an existing mapping, you can’t change existing field mappings. If a mapping already exists for a field, data from that field has probably been indexed. If you were to change the field mapping, the indexed data would be wrong and would not be properly searchable.

结合返回的参数来看

```json
"first_name" : {
    "type" : "text",
    "fields" : {
        "keyword" : {
            "type" : "keyword",
            "ignore_above" : 256
        }
    }
}
```

type指的text，而域是指的text下的fields。elasticsearch的专有名词，一直，很迷。

#### 复杂核心域类型

键值对有的时候并不是简单类型key-value，它的value可能是空可能是数组也有可能是个对象。  
如果是多值域的话是以数组的形式存储的`{ "tag" : ["search" , "nosql"]}`。数组内所有值必须是相同数据类型的，如果通过索引来创建新的域，elasticsearch会用数组中第一个值的数据类型作为这个域的类型。数组是可以被搜索的，但不是像Java中的数组那样是有序的，在搜索的时候不能指定第几个值，更像是袋子里的值。  
空域，Lucene中是不能存null的，所以认为存在null值的域为空域，不会被索引。  
多层级对象，比如

```json
"user": {
    "id":           "@johnsmith",
    "gender":       "male",
    "age":          26,
    "name": {
        "full":     "John Smith",
        "first":    "John",
        "last":     "Smith"
    }
}
```

name写的就是一个对象，其`properties`属性`type`显示为`object`。但是在lucene中是不理解对象的概念，elasticsearch为了有效的所以内部，会把文档转换为

```json
"user.id":          [@johnsmith],
"user.gender":      [male],
"user.age":         [26],
"user.name.full":   [john, smith],
"user.name.first":  [john],
"user.name.last":   [smith]
```

如果是对象是数组的话

```json
{
    "followers": [
        { "age": 35, "name": "Mary White"},
        { "age": 26, "name": "Alex Jones"},
        { "age": 19, "name": "Lisa Smith"}
    ]
}
{
    "followers.age":    [19, 26, 35],
    "followers.name":   [alex, jones, lisa, smith, mary, white]
}
```

显然age和name的相关性已经丢失了，所以我们只能搜索到`有一个26岁的追随者`，而不能得到`是否有一个26岁名字叫Alen Jones的追随者`。

## 书中疑问自己找的

### POST和PUT

PUT是幂等方法，用于替换；POST是非幂等方法，用于创建。  
但创建的时候既可以使用POST，也可以使用PUT，其区别在于POST作用于一个集合资源上（/index/type），而PUT作用域具体资源上（/index/type/id）。但是分档部分更新是用POST的。  
我觉得应该区分一下替换和更新。在RESTful中规定POST是Create，而PUT是Update/Replace

> 幂等：数学概念，某一元运算为幂等时，其作用在任一元素两次后会与其作用一次的结果相同 $f(x) = f(f(x))$

### 节点和分片

任何一个节点都知道文档存在哪个节点上，会选择一个主节点，用来管理集群中的一些变更， 比如新建或删除索引、增加和移除节点等  
新建、索引和清楚都是写操作，它们必须在主分片上成功完成才能复制到相关的副本分片上。  
副本分片负责容错，以及承担读请求的负载均衡。  
待补充

### 一个概念

- 索引，相当于库
- 文档，相当于一行数据
- 类型，相当于表
- 域，相当于字段

## 疑问

- 读为提交？
- 读操作如果副本分片没有找到会查询主分片？？
- [取回一个文档](https://www.elastic.co/guide/cn/elasticsearch/guide/current/distrib-read.html)说在处理读取请求时，协调结点在每次请求的时候都会通过轮询所有的副本分片来达到负载均衡；[多索引多类型](https://www.elastic.co/guide/cn/elasticsearch/guide/current/multi-index-multi-type.html)说当在单一的索引下进行搜索的时候，Elasticsearch 转发请求到索引的每个分片中，可以是主分片也可以是副本分片，然后从每个分片中收集结果。到底问不问主节点？？
