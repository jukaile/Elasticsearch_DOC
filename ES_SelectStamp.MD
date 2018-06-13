这是整理关于ElasticSearch查询语句相关的文档
===========================================

--------

***作者：jukaile***  
***时间：2018-06-04***

--------

1.概述
-------

ElasticSearch(以下简称`ES`)的查询语法规则为嵌套式查询
为说明方便，假定建立的ES数据库地址为: `http:\\localhost:9200` 索引和数据类型为: `jukaile`、`doc`
下面针对基本的操作做简要说明:

* 如果把整个查询语句全部抽象为嵌套的话，那描述查询的位置的http地址即为最外层
    查询方法为：  
    **<center><VERB> \<PROTOCOL>://\<HOST>:\<PORT>/\<PATH>?\<QUERY_STRING></center>**
    
    被<>标记的部件的解释：
    
    |VERB|PROTOCOL|HOST|PORT|PATH|QUERY_STRING|
    |-|-|-|-|-|-|
    |HTTP方法、谓词|HTTP、HTTPS|ES节点地址|运行ES WEB服务的端口|ES查询API 可以连用 用"/"连接|可选的查询字符串参数|
    
    ES使用http链接来指定查询的索引和查询类型，对应mongodb中的`db`与`collection`
    
  ***注意***：
  * ES支持在HTTP请求中直接编辑查询语句，通过API直接查询，但因为请求的表达能力有限，因此除了简单询，通常不会在HTTP请求中，这里给出一个查询例子：
  
    **<center>```HTTP://localhost:9200/jukaile/doc/_search```</center>**
  * 通常在生产中使用的查询为是在给出的http中加入body，通过传值的方式进行查询

2.查询语句
----------

ES查询的通常做法为在请求的请求体（body）中写入格式化查询语句：
get和post都可以用来执行带有请求体的搜索

    get http://localhost:9200/jukaile/doc/_search
    body{
      //查询体
    }
为了说明方便，下面将省略请求头：

    get jukaile/doc/_search
    {
      //查询体
    }
请求体查询语句的格式为功能嵌套

最外层为功能层，由外到内功能逐渐细分，直到最后的查询条件

1. **功能API**

    |Name|Description|
    |:-:|:-:|
    |`timeout`|超时时间，默认没有|
    |`form`②|结果偏移量，默认为0|
    |`size`②|文档查询总数，既在hits列表中返回的数量。默认为10|
    |`search_type`①|执行搜索时的模式类型，可设置的搜索模式有：`dfs_query_then_fetch`、`dfs_query_and_fetch`、`query_then_fetch`、`query_and_fetch`、`count`、`scan`，默认值为`query_then_fetch`|
    |`query_type`①|设置查询结果是否缓存，需要选项支持（`?search_type=count`)
    |`terminate_after`|The maximum number of documents to collect for each shard, upon reaching which the query execution will terminate early. If set, the response will have a boolean field terminated_early to indicate whether the query execution has actually terminated_early. Defaults to no terminate_after.(翻译不能……)|
    |`_source`③|主要用来返回_source字段内容，由于ES查询机制，也可用于挑选返回字段，用`[]`表示需要返回的字段，中括号内部可接收多个值|
    |还有其他API|TODT：暂时可能用到的API先写这么多，以后有用到新API在添加|

    * ***注释:***
      *①：这两个参数必须通过查询参数字符串传递*
      *②：这两个参数用来对结果进行分页，这里给出一个例子，其意义为：查询结果从第10个开始返回，共返回10个结果*
      ```json
      {
          "form":10,
          "size":10,
          "query":{}
      }
      ```
      *③：默认情况下，将返回_source字段的内容；_source被禁用的话，将不返回。给出一个例子：*
      ```json
      {
          "_source":false,
          "_source":["字段列表"],
          "query":{}
      }
      ```
      *在`_source`启用的情况下，其可接收一个或多个通配符控制返回值：*
      ```json
      {
          "_source":"obj.*",
          "_source":["obj1.*","obj2.*"],
          "query":{}
      }
      ```
      *同时，也可以通过包含`include`和`exclude`完成完整的控制：*
      ```json
      {
          "_source":{
              "include":["obj1.*", "obj2.*"],
              "exclude":["*.description"],
          },
          "query":{}
      }
      ```
2. **数据处理API**

    |Name|Description|
    |:-:|:-:|
    |`query`|查询|
    |`sort`|排序|
    |`aggregations`|聚合|
    
    数据处理最重要和最常用的三个API就是上表中的API，下面将会对每个API进行详细说明
    * `query`：查询API
    
      先给出一个例子：
      ```json
      {
          "query":{
              "match_all":{}
          }
      }
      ```
      上面的例子将返回所有数据
      
      对于ES而言，查询有全文搜索和精确查询之分
      
      全文搜索：
      * ES是基于倒排索引实现的。为了提高查询速度，在数据进入ES时，必须要对其进行索引化（既为切词、归档过程），因此对于全文搜索，其隐含条件就是模糊查询
      * 由于全文搜索是模糊搜索，为了衡量文档的匹配程度，ES给每个词对应的文档赋予一个评分，放在查询结果的`_score`中
      
      对于不同的查询，`query`中几乎都有对应的API以实现其功能
      * 全文搜索API
      
        |Name|Description|
        |:-:|:-:|
        |`match_all`|查询所有|
        |`match_none`|查询无|
        |`match`|全文搜索的标准查询，同样包含模糊、短语和（或）接近匹配|
        |`match_phrase`|用于匹配准确的短语或接近的单词匹配|
        |`match_phrase_prefix`|可支持通配符查询，功能与`match`接近|
        |`multi_match`|`match`查询的多字段版本|
        |`common_terms`|查询低频率单词|
        |`query_string`|支持多单词合并查询，使用紧凑的Lucene语法编辑查询语句|
        |`simple_query_string`|`query_string`的高级版本，适合向用户直接公开|
      * 筛选器API
        * ***`bool`*** 此筛选器通过`_score`评分判断文档匹配程度，并返回子句规定的文档
        常用的可选项：
          1. ***`must`*** 在must子句中的查询项必须在文档中完全满足，文档才会被筛选
          2. ***`must_not`*** 在must_not子句中的查询项必须在文档中完全排除，文档才会被筛选
          3. ***`should`***  在should子句中的查询项不必完全满足，既每个项目之间的关系是`或`时，文档会被筛选
        * ***`filter`*** 此筛选器根据过滤器条件，过滤掉不符合的文档。该筛选器对评分没有影响
        常用可选项：
          1. ***`term`*** 该选项指定的字段值必须存在
          2. ***`terms`*** 功能跟`term`相同，但支持多个字段
          3. ***`range`*** 该字段用于对内部字段范围进行指定
      * 要防止因为倒排索机制导致搜索不精确的问题，可以在索引化时配置让需要的字段不切词，整体索引化
      * 筛选器可以相互嵌套，也可以同层连用：
        * 在相互嵌套的情况下，筛选器之间的逻辑是递进逻辑；
        * 在同层连用情况下，筛选器之间逻辑类似管道逻辑，数据先由前面的筛选器筛选，而后被后面的筛选器筛选。
      * 例子：
        ```json
        {
            "query": {
                "bool": {
                    "must": [
                        {"match": { "title": "Search"}},
                        {"match": { "content": "Elasticsearch"}}
                    ],
                    "filter": [
                        {"term":  { "status": "published"}},
                        {"range": { "publish_date": { "gte": "2015-01-01"}}}
                    ]
                }
            }
        }
        ```
    * `sort` 排序API
      `sort`关键字是ES的排序API，其针对的是由`query`搜索或`aggs`聚合执行后拿到的数据。在没有给出`query`或`aggs`的情况下，默认对全文根据条件排序
      排序可支持多个字段排序，
      * 例子：
        ```json
        {
            "sort" : [
                {"post_date" : { "order" : "asc"}},
                "user",
                { "name" : "desc" },
                { "age" : "desc" },
                "_score"
            ],
            "query" : {
                "term" : { "user" : "kimchy" }
            }
        }
        ```
      因此，可以给出标准`sort`模型：
        ```json
        {
            "sort":[
                {"排序字段":{"排序属性":"属性值"}}
            ]
        }
        ```
      * 排序属性：
        |属性名|说明|
        |:-:|:-:|:-:|
        |`order`|排序顺序|
        |`mod`|排序模式（使用的排序值）|
        |`nested`|嵌套排序指定|
        |`missing`|缺少的值，该API用于指定如何处理缺少该字段的文档，可将值设定为默认或自定义值（制定的值将用于排序值）|
        |`_script`|用于接收自定义脚本作为排序方法|
        属性的具体说明：
        1. ***`order`***
          用于指定字段的排序顺序的选项，有两个属性值：
            * `asc`： 升序排序
            * `desc`：降序排序

            若不指定该属性，则默认为asc
            给出一个例子：
            ```json
            {
                "query":{
                    "term":{"key_1":"value"}
                },
                "sort":{
                    "key_sort":{
                        "order":"asc",
                        "mode":"avg"
                    }
                }
            }
            ```
            *注：例子中连用了`mod`属性，将在下面说明*
        2. ***`mod`***
          用于指定排序模式的选项，可选的属性值：
            |属性值|说明|
            |:-:|:-:|
            |`min`|取最小值|
            |`max`|取最大值|
            |`sum`|使用所有值的总和作为排序值。只适用于数字数组类型字段|
            |`avg`|使用平均值作为排序值。只适用于数字数组类型字段|
            |`median`|使用终点值作为排序值(集合取查询集)。只适用于数字数组类型字段|
        3. ***`nested`***
          用于给多嵌套指定位置的值
            * *注意*：`nested`不仅仅是`sort`专用的属性。实际上，在文档中包含多嵌套关系的情况下，不管是`query`查询、`sort`排序还是`aggs`，该属性都是不可或缺的。原因是在ES中，若不指定嵌套索引`nested`，只依靠动态映射，则多嵌套会被自动建立成为一个`object`，其中的内容会被扁平化成为key-value键值对形式。例如对于文档：
              ```json
              {
                "title": "Nest eggs",
                "body":  "Making your money work...",
                "tags":  [ "cash", "shares" ],
                "comments": [ <1>
                  {
                    "name":    "John Smith",
                    "comment": "Great article",
                    "age":     28,
                    "stars":   4,
                    "date":    "2014-09-01"
                  },
                  {
                    "name":    "Alice White",
                    "comment": "More like this please",
                    "age":     31,
                    "stars":   5,
                    "date":    "2014-10-22"
                  }
                ]
              }
              ```
              在扁平化后，该文档被转换成：
              ```json
              {
                "title":            [ eggs, nest ],
                "body":             [ making, money, work, your ],
                "tags":             [ cash, shares ],
                "comments.name":    [ alice, john, smith, white ],
                "comments.comment": [ article, great, like, more, please, this ],
                "comments.age":     [ 28, 31 ],
                "comments.stars":   [ 4, 5 ],
                "comments.date":    [ 2014-09-01, 2014-10-22 ]
              }
              ```
              在这种扁平化处理后，嵌套文档中的关联顺序已经完全消失，因此文档<1>会匹配这种搜索：
              ```json
              {
                "query": {
                  "bool": {
                    "must": [
                      { "match": { "name": "Alice" }},
                      { "match": { "age":  28      }} <1>
                    ]
                  }
                }
              }
              ```
              这显然是不可容忍的。
              对于这种有嵌套关系的文档，我们要么重新构建索引规则并用新规则索引化，要么指定嵌套路径。
              对于上面的例子重新索引化：
              ```json
              {
                "mappings": {
                  "blogpost": {
                    "properties": {
                      "comments": {
                        "type": "nested", <1>
                        "properties": {
                          "name":    { "type": "string"  },
                          "comment": { "type": "string"  },
                          "age":     { "type": "short"   },
                          "stars":   { "type": "short"   },
                          "date":    { "type": "date"    }
                        }
                      }
                    }
                  }
                }
              }
              ```
              在重新索引化后，每个嵌套对象被索引化为一个隐藏的分割文档：
              ```json
              { <1>
                "comments.name":    [ john, smith ],
                "comments.comment": [ article, great ],
                "comments.age":     [ 28 ],
                "comments.stars":   [ 4 ],
                "comments.date":    [ 2014-09-01 ]
              }
              { <2>
                "comments.name":    [ alice, white ],
                "comments.comment": [ like, more, please, this ],
                "comments.age":     [ 31 ],
                "comments.stars":   [ 5 ],
                "comments.date":    [ 2014-10-22 ]
              }
              { <3>
                "title":            [ eggs, nest ],
                "body":             [ making, money, work, your ],
                "tags":             [ cash, shares ]
              }
              ```
              *注*：
              * <1>第一个嵌套对象
              * <2>第二个嵌套对象
              * <3>根或父文档

              但是，由于是隐藏的文档，是无法直接访问的。想要访问这种对象，需要使用`nested`查询或`nested`过滤器。这里给出3个查询API对应的例子：
              1.查询`query`:
                ```json
                {
                  "query": {
                    "bool": {
                      "must": [
                        { "match": { "title": "eggs" }},
                        {
                          "nested": {
                            "path": "comments",
                            "score_mode": "max",
                            "query": {
                              "bool": {
                                "must": [
                                  { "match": { "comments.name": "john" }},
                                  { "match": { "comments.age":  28     }}
                                ]
                        }}}}
                      ]
                }}}
                ```
              2.排序`sort`：
              ```json
              {
                "query": {
                  "nested": { <1>
                    "path": "comments",
                    "filter": {
                      "range": {
                        "comments.date": {
                          "gte": "2014-10-01",
                          "lt":  "2014-11-01"
                        }
                      }
                    }
                  }
                },
                "sort": {
                  "comments.stars": { <2>
                    "order": "asc",   <2>
                    "mode":  "min",   <2>
                    "nested_filter": { <3>
                      "range": {
                        "comments.date": {
                          "gte": "2014-10-01",
                          "lt":  "2014-11-01"
                        }
                      }
                    }
                  }
                }
              }
              ```
              解释：
              * <1>限定10月
              * <2>结果在所有匹配的回应中依照comment.stars栏位的最小值(min)作递增(asc)的排序。
              * <3>排序条件中的nested_filter与主查询query条件中的nested查询相同。
              为什么我们要在nested_filter重复写上查询条件？ 原因是排序在於执行查询后才发生。 此查询匹配了在十月中有收到回应的blog文章，回传blog文章文档作为结果。 如果我们不加上nested_filter条件，我们最後会依照任何blog文章曾经收到过的回应作排序，而不是在十月份收到的。

              3.聚合`aggs`:
              ```json
              {
                "aggs": {
                  "comments": { <1>
                    "nested": {
                      "path": "comments"
                    },
                    "aggs": {
                      "by_month": {
                        "date_histogram": { <2>
                          "field":    "comments.date",
                          "interval": "month",
                          "format":   "yyyy-MM"
                        },
                        "aggs": {
                          "avg_stars": {
                            "avg": { <3>
                              "field": "comments.stars"
                            }
                          }
                        }
                      }
                    }
                  }
                }
              }
              ```
              解释：
              * <1> nested集合深入嵌套对象的comments栏位
              * <2> 评论基於comments.date栏位被分至各个月份分段
              * <3> 每个月份分段单独计算星号的平均数
        4. ***`missing`***
            用于处理在排序条件不存在的文档，有3个可选的属性：
            * `_last`:最小值
            * `_first`:最大值
            * 自选值
            给出例子：
            ```json
            {
                "sort" : [
                    { "price" : {"missing" : "_last"} }
                ],
                "query" : {
                    "term" : { "product" : "chocolate" }
                }
            }
            ```
        5. ***`_script`***
            脚本排序，给出例子：
            ```json
            {
                "query" : {
                    "term" : { "user" : "kimchy" }
                },
                "sort" : {
                    "_script" : {
                        "type" : "number",
                        "script" : {
                            "lang": "painless",
                            "source": "doc['field_name'].value * params.factor",
                            "params" : {
                                "factor" : 1.1
                            }
                        },
                        "order" : "asc"
                    }
                }
            }
            ```
    * `aggs`聚合API
        用于生成聚合的api
        在ES中，有很多不同类型的聚合。通常可以分成4类：
        * `Bucketing`：桶聚合，最基本的聚合。每个桶都与一个key和一个文档条件相关联，可以形象的理解为：当文档满足聚合条件时，该文档就会落入这个聚合条件所规定的桶内。
        * `Mertic`：度量聚合，在文档中通过对某一度量的跟踪和计算来达到聚合目的
        * `Martix`：矩阵聚合，度量聚合的高维形式，通过在文档的多个字段上的计算和跟踪，生成矩阵达到聚合目的
        * `Pipeline`：管道聚合，通过接收其他聚合的输出及其关联的度量值的聚合

      说明：聚合本质意义实际上是在全文或索引化后的文档集中，单独取出一部分满足条件的文档集的操作，所以其取出的实体是一个集合。因此可以对其进行二次或者嵌套聚合。`ES的嵌套聚合的级别和深度没有限制`

      --------
      构造聚合的基本结构：
      ```json
      "aggregations" : {
          "<aggregation_name>" : {
              "<aggregation_type>" : {
                  <aggregation_body>
              }
              [,"meta" : {  [<meta_data_body>] } ]?
              [,"aggregations" : { [<sub_aggregation>]+ } ]?
          }
          [,"<aggregation_name_2>" : { ... } ]*
      }
      ```
<<<<<<< HEAD
      由于聚合过于复杂，且过于特化应用场景，这里不在具体分类介绍，只列举一些例子
      

=======
>>>>>>> 1d109ad674de887c6cca6a89df8bd6e8c0ca1231
