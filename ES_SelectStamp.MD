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
          //"_source":[字段列表],
          "query":{}
      }
      ```
      *在`_source`启用的情况下，其可接收一个或多个通配符控制返回值：*
      ```json
      {
          "_source":"obj.*",
        //"_source":["obj1.*","obj2.*"]
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
      对于不同的查询，query中几乎都有对应的API以实现其功能
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