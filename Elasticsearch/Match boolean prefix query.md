### Match boolean prefix query

> A `match_bool_prefix` query analyzes its input and constructs a [`bool` query](https://www.elastic.co/guide/en/elasticsearch/reference/7.4/query-dsl-bool-query.html) from the terms. Each term except the last is used in a `term` query. The last term is used in a `prefix` query. A `match_bool_prefix` query such as

match_bool_prefix查询会对输入的文本进行分词，并且根据分词后的词项构建一个bool query。除了最后一个词项会使用term查询，最后一个词项会做前缀查询，下面是个例子：

```json
GET /_search
{
    "query": {
        "match_bool_prefix" : {
            "message" : "quick brown f"
        }
    }
}
```

>  where analysis produces the terms `quick`, `brown`, and `f` is similar to the following `bool` query

生成的词项quick,brown和f和下面的bool查询很像

```json
GET /_search
{
    "query": {
        "bool" : {
            "should": [
                { "term": { "message": "quick" }},
                { "term": { "message": "brown" }},
                { "prefix": { "message": "f"}}
            ]
        }
    }
}
```

> An important difference between the `match_bool_prefix` query and [`match_phrase_prefix`](https://www.elastic.co/guide/en/elasticsearch/reference/7.4/query-dsl-match-query-phrase-prefix.html) is that the `match_phrase_prefix` query matches its terms as a phrase, but the `match_bool_prefix` query can match its terms in any position. The example `match_bool_prefix` query above could match a field containing containing `quick brown fox`, but it could also match `brown fox quick`. It could also match a field containing the term `quick`, the term `brown` and a term starting with `f`, appearing in any position.

match_bool_prefix和match_phrase_prefix的一个重要区别是match_phrase_prefix匹配词项违一个短语，而match_bool_prefix则是会匹配任意位置的词项。上面这个match_bool_prefix的例子匹配字段包含quick brown fox的文档，但是也会匹配brown fox quick，同时也会匹配词项包含quick，brown和词项以f开头的，对位置没有限制。

### Parameters

> By default, `match_bool_prefix` queries' input text will be analyzed using the analyzer from the queried field’s mapping. A different search analyzer can be configured with the `analyzer` parameter

默认情况下，match_bool_prefix检索的文本内容会使用该字段的mapping中的分词器，当然也可以通过analyzer参数指定其他分词器。

```console
GET /_search
{
    "query": {
        "match_bool_prefix" : {
            "message": {
                "query": "quick brown f",
                "analyzer": "keyword"
            }
        }
    }
}
```

> `match_bool_prefix` queries support the [`minimum_should_match`](https://www.elastic.co/guide/en/elasticsearch/reference/7.4/query-dsl-minimum-should-match.html) and `operator` parameters as described for the [`match` query](https://www.elastic.co/guide/en/elasticsearch/reference/7.4/query-dsl-match-query.html#query-dsl-match-query-boolean), applying the setting to the constructed `bool` query. The number of clauses in the constructed `bool` query will in most cases be the number of terms produced by analysis of the query text.
>
> The [`fuzziness`](https://www.elastic.co/guide/en/elasticsearch/reference/7.4/query-dsl-match-query.html#query-dsl-match-query-fuzziness), `prefix_length`, `max_expansions`, `fuzzy_transpositions`, and `fuzzy_rewrite` parameters can be applied to the `term` subqueries constructed for all terms but the final term. They do not have any effect on the prefix query constructed for the final term.

match_bool_prefix支持matchquery中介绍的minimum_should_match和operator参数，应用这些设置构成bool查询。大多数情况下，在构造的bool中语句的数量就是对检索的文本字段分词后的词项数量。

[`fuzziness`](https://www.elastic.co/guide/en/elasticsearch/reference/7.4/query-dsl-match-query.html#query-dsl-match-query-fuzziness), `prefix_length`, `max_expansions`, `fuzzy_transpositions`和fuzzy_rewrite这些参数都可以应用到为所有的词项构造的子查询中，而不是应用在最终的词项上。他们对为最终词项构造的前缀查询没有影响。

