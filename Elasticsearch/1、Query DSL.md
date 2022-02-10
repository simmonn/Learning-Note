### Query DSL

> ES提供了一套基于json的检索语言，考虑到query dsl是一个抽象语法树，包含两类语法。

#### 叶子检索语法

> 叶子检索语法看起来像是一个在特定字段放一个特定的值，比如，match,term或者range查询。

#### 复合查询语法

> 复合查询语法将叶子或者复杂查询包装结合到一个复合查询中（比如bool、dis_max查询），或者改变原有的查询（比如constant_score）

检索语法根据在query上下文还是filter上下文的不同会有一定的差异。

### Query and filter context

#### 相关性分数

默认情况下，es排序会根据相关性分数匹配搜索结果。那就意味着表示了每一篇文档与query语句的匹配程度。

这个相关性分数是一个正浮点数，搜索API会返回这个元数据字段_score。

_score越高，与文档的相关性越高。而且每个query类型会有不同的打分方式，打分也会和query还是filter查询的不同而不同。

#### Query上下文

在检索上下文忠，一个检索语法表示这个文档和这个检索语句的匹配程度。除了判定是否和文档匹配外，query语句也会在_score元数据字段中计算相关得分。

无论什么时候检索语句被传递到query参数中，检索上下文都会生效。例如searh api中的query。

#### Filter 上下文

filter上下文表示这个查询语句是否匹配，结果只能是true或者false，不计算分数。一般用于过滤结构化数据比如 timestamp、status.

filter不参与打分，不过如果要打分，可以包裹在constant_socre中

```json
GET /_search
{
  "query": { 
    "bool": { 
      "must": [
        { "match": { "title":   "Search"        }},
        { "match": { "content": "Elasticsearch" }}
      ],
      "filter": [ 
        { "term":  { "status": "published" }},
        { "range": { "publish_date": { "gte": "2015-01-01" }}}
      ]
    }
  }
}
```

