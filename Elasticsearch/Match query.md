> Returns documents that match a provided text, number, date or boolean value. The provided text is analyzed before matching.
>
> The `match` query is the standard query for performing a full-text search, including options for fuzzy matching.

会返回匹配到text、number、date或者Boolean类型的文档，匹配之前会对text类型进行分词。

match是全文检索的标准检索，包括可以设置模糊匹配的选项。

请求示例

```json
GET /_search
{
    "query": {
        "match" : {
            "message" : {
                "query" : "this is a test"
            }
        }
    }
}
```

#### Top-level parameters for match ---顶级参数

<field>  (Required, object) Field you wish to search.

##### field---(必传，对象)检索字段

### Parameters for <field>

|                                         |                                                              |
| --------------------------------------- | ------------------------------------------------------------ |
| **query**                               | (Required) Text, number, boolean value or date you wish to find in the provided `<field>`.<br />The `match` query [analyzes](https://www.elastic.co/guide/en/elasticsearch/reference/7.4/analysis.html) any provided text before performing a search. This means the `match` query can search [`text`](https://www.elastic.co/guide/en/elasticsearch/reference/7.4/text.html) fields for analyzed tokens rather than an exact term. |
|                                         | （必传）希望在提供的<field>字段中搜索text，number，boolean或者日期类型。<br />检索之前match会对text类型字段进行分词。这就意味着match对搜索的text字段进行分词后的token查询，而不是具体的词条查询。 |
| **analyzer**                            | (Optional, string) [Analyzer](https://www.elastic.co/guide/en/elasticsearch/reference/7.4/analysis.html) used to convert the text in the `query` value into tokens. Defaults to the [index-time analyzer](https://www.elastic.co/guide/en/elasticsearch/reference/7.4/analysis.html#specify-index-time-analyzer) mapped for the `<field>`. If no analyzer is mapped, the index’s default analyzer is used. |
|                                         | （可选，string）分词器会将query中text类型的值转换为token。默认情况下，使用索引分词器映射到这个字段。如果没有映射分词器，索引则会使用默认分词器。 |
| **auto_generate_synonyms_phrase_query** | (Optional, boolean) If `true`, [match phrase](https://www.elastic.co/guide/en/elasticsearch/reference/7.4/query-dsl-match-query-phrase.html) queries are automatically created for multi-term synonyms. Defaults to `true`.<br />See [Use synonyms with match query](https://www.elastic.co/guide/en/elasticsearch/reference/7.4/query-dsl-match-query.html#query-dsl-match-query-synonyms) for an example. |
|                                         | （可选，布尔类型）如果为true，会自动为multi-term同义词创建match_phrase。<br />可以参考match查询同义词的例子。 |
| **fuzziness**                           | (Optional, string) Maximum edit distance allowed for matching. See [Fuzziness](https://www.elastic.co/guide/en/elasticsearch/reference/7.4/common-options.html#fuzziness) for valid values and more information. See [Fuzziness in the match query](https://www.elastic.co/guide/en/elasticsearch/reference/7.4/query-dsl-match-query.html#query-dsl-match-query-fuzziness) for an example. |
|                                         | （可选，字符串类型）匹配的时候可以指定最大编辑距离，查看模糊查询可用值或者更多信息，查看match中模糊查询的例子。 |
| **max_expansions**                      | (Optional, integer) Maximum number of terms to which the query will expand. Defaults to `50`. |
|                                         | （可选，数值类型）query中词条扩展的最大长度，默认为50        |
| **prefix_length**                       | (Optional, integer) Number of beginning characters left unchanged for fuzzy matching. Defaults to `0`. |
|                                         | （可选，数值类型）左侧不参与模糊查询的字符数，默认为0.       |
| **fuzzy_transpositions**                | (Optional, boolean) If `true`, edits for fuzzy matching include transpositions of two adjacent characters (ab → ba). Defaults to `true`. |
|                                         | （可选，布尔类型）如果为true，将相邻位置字符互换算作一次编辑距离，默认为true |
| **fuzzy_rewrite**                       | (Optional, string) Method used to rewrite the query. See the [`rewrite` parameter](https://www.elastic.co/guide/en/elasticsearch/reference/7.4/query-dsl-multi-term-rewrite.html) for valid values and more information.<br />If the `fuzziness` parameter is not `0`, the `match` query uses a `rewrite` method of `top_terms_blended_freqs_${max_expansions}` by default. |
|                                         | （可选，字符串类型）这个方法是用来重写query的。查看可用参数的值或者更多详情请跳转到rewrite参数链接。 |
| **lenient**                             | (Optional, boolean) If `true`, format-based errors, such as providing a text `query` value for a [numeric](https://www.elastic.co/guide/en/elasticsearch/reference/7.4/number.html) field, are ignored. Defaults to `false`. |
|                                         | （可选，布尔类型）如果为true，像text query检索一个数值类型的字段，格式错误会忽略。默认为false。 |
| **operator**                            | (Optional, string) Boolean logic used to interpret text in the `query` value. Valid values are:<br />**`OR` (Default)**For example, a `query` value of `capital of Hungary` is interpreted as `capital OR of OR Hungary`. <br />**`AND`**For example, a `query` value of `capital of Hungary` is interpreted as `capital AND of AND Hungary`. |
|                                         | （可选，字符串类型）布尔逻辑是用来在query中的文本上的。<br />OR（默认）例如，检索的值为capital of Hungary解释为capital OR of OR Hungary。<br />AND 例如，检索值capital OR of OR Hungary被解释为capital AND of AND Hungary。 |
| **minimum_should_match**                | (Optional, string) Minimum number of clauses that must match for a document to be returned. See the [`minimum_should_match` parameter](https://www.elastic.co/guide/en/elasticsearch/reference/7.4/query-dsl-minimum-should-match.html) for valid values and more information.<br /> |
|                                         | （可选，字符串类型）匹配应该返回的文档最小数量。（如果匹配到的文档小于这个值，则不返回文档）可以去查看[`minimum_should_match` parameter](https://www.elastic.co/guide/en/elasticsearch/reference/7.4/query-dsl-minimum-should-match.html)链接 查看可用的值。 |
| **zero_terms_query**                    | (Optional, string) Indicates whether no documents are returned if the `analyzer` removes all tokens, such as when using a `stop` filter. Valid values are:<br />**`none` (Default)**<br />No documents are returned if the `analyzer` removes all tokens.<br />**`all`**<br />Returns all documents, similar to a [`match_all`](https://www.elastic.co/guide/en/elasticsearch/reference/7.4/query-dsl-match-all-query.html) query.<br />See [Zero terms query](https://www.elastic.co/guide/en/elasticsearch/reference/7.4/query-dsl-match-query.html#query-dsl-match-query-zero) for an example. |
|                                         | （可选，字符串类型）这个参数表示没有文档返回的时候，分词器是否删除所有的token，比如当使用 停用词 过滤的时候，可用的参数为：<br />none（默认）如果分词器删除所有的token，则不返回任何文档。<br />all  返回所有文档，和match_all 查询很像。查看例子可以访问zero terms query 的链接。 |
|                                         |                                                              |

### Notes

#### Short request example - 简单的请求参数案例

> You can simplify the match query syntax by combining the `<field>` and `query` parameters. For example:

```json
GET /_search
{
    "query": {
        "match" : {
            "message" : "this is a test"
        }
    }
}
```

#### How the match query works -那么match 查询是怎么工作的呢

>The `match` query is of type `boolean`. It means that the text provided is analyzed and the analysis process constructs a boolean query from the provided text. The `operator` parameter can be set to `or` or `and` to control the boolean clauses (defaults to `or`). The minimum number of optional `should` clauses to match can be set using the [`minimum_should_match`](https://www.elastic.co/guide/en/elasticsearch/reference/7.4/query-dsl-minimum-should-match.html) parameter.
>
>Here is an example with the `operator` parameter:

match 是一种boolean查询，这就意味查询的text文本会被分词，并且分词过程会基于提供的文本构造一个boolean查询。operator参数可以设置为or或者and来控制boolean查询语句（默认是or）。[`minimum_should_match`](https://www.elastic.co/guide/en/elasticsearch/reference/7.4/query-dsl-minimum-should-match.html)参数可以设置，should语句的最小匹配数量。

下面是一个operator参数的例子：

```json
GET /_search
{
    "query": {
        "match" : {
            "message" : {
                "query" : "this is a test",
                "operator" : "and"
            }
        }
    }
}
```

> The `analyzer` can be set to control which analyzer will perform the analysis process on the text. It defaults to the field explicit mapping definition, or the default search analyzer.
>
> The `lenient` parameter can be set to `true` to ignore exceptions caused by data-type mismatches, such as trying to query a numeric field with a text query string. Defaults to `false`.

analyzer参数可以对text类型的字段在分词阶段使用哪个分词器。默认是取mapping定义中显式设置的分词器，没有设置则会使用默认分词器（standard分词器）。

lenient参数可以设置为true来忽略类型不匹配导致的异常，比如用text类型去检索去查数值类型，默认是false。

#### Fuzziness in the match query - match中的模糊搜索

> `fuzziness` allows *fuzzy matching* based on the type of field being queried. See [Fuzziness](https://www.elastic.co/guide/en/elasticsearch/reference/7.4/common-options.html#fuzziness) for allowed settings.
>
> The `prefix_length` and `max_expansions` can be set in this case to control the fuzzy process. If the fuzzy option is set the query will use `top_terms_blended_freqs_${max_expansions}` as its [rewrite method](https://www.elastic.co/guide/en/elasticsearch/reference/7.4/query-dsl-multi-term-rewrite.html) the `fuzzy_rewrite` parameter allows to control how the query will get rewritten.
>
> Fuzzy transpositions (`ab` → `ba`) are allowed by default but can be disabled by setting `fuzzy_transpositions` to `false`.

fuzziness参数允许基于被检索的字段进行模糊匹配。可以查看 [Fuzziness](https://www.elastic.co/guide/en/elasticsearch/reference/7.4/common-options.html#fuzziness) 更多的设置。

prefix_length和max_expansions参数可以在这个案例中设置来控制模糊查询流程。如果模糊查询选项被设置了，这个查询会使用top_terms_blended_freqs_${max_expansions}作为它的重写方法。fuzzy_rewrite参数可以控制怎么重写。

模糊翻转（ab->ba）默认是开启的，但是你可以通过将fuzzy_transpositions参数设置为false来关闭。

> NOTE: Fuzzy matching is not applied to terms with synonyms or in cases where the analysis process produces multiple tokens at the same position. Under the hood these terms are expanded to a special synonym query that blends term frequencies, which does not support fuzzy expansion.

备注：模糊匹配不会应用在同义词词条上，分词过程产生的一个位置上的多个token上也不会生效。底层这些词条会扩展成一个特殊的同义词query，它混合了词条频数，不支持模糊扩展。

```json
GET /_search
{
    "query": {
        "match" : {
            "message" : {
                "query" : "this is a test",
                "operator" : "and"
            }
        }
    }
}
```

#### Zero terms query

> If the analyzer used removes all tokens in a query like a `stop` filter does, the default behavior is to match no documents at all. In order to change that the `zero_terms_query` option can be used, which accepts `none` (default) and `all` which corresponds to a `match_all` query.

如果使用的分词器想stop分词过滤一样删除了所有的token，那么就匹配不到任何文档。为了避免这种情况，zero_terms_query选项的作用就体现出来了，这个选项可以传入none(默认)和all（相当于match_all）。

```json
GET /_search
{
    "query": {
        "match" : {
            "message" : {
                "query" : "to be or not to be",
                "operator" : "and",
                "zero_terms_query": "all"
            }
        }
    }
}
```

#### Cutoff frequency

> ### WARNING
>
> ### Deprecated in 7.3.0.
>
> This option can be omitted as the [Match](https://www.elastic.co/guide/en/elasticsearch/reference/7.4/query-dsl-match-query.html) can skip blocks of documents efficiently, without any configuration, provided that the total number of hits is not tracked.

注意：

7.3.0中已经废弃了

由于Match可用有效的跳过文档块，所以这个选项可以被忽略，在不跟踪中命中总数的前提下，无需任何配置。

> The match query supports a `cutoff_frequency` that allows specifying an absolute or relative document frequency where high frequency terms are moved into an optional subquery and are only scored if one of the low frequency (below the cutoff) terms in the case of an `or` operator or all of the low frequency terms in the case of an `and` operator match.

Match query支持cutoff_frequency（截止频率），cutoff_frequency允许你指定绝对或者相对文档频率，其中高频词项被移动到可选的子查询中，并且只有在`or`操作符的情况下低频词（低于截止值）之一 或所有在`and` 运算符匹配的情况下的低频词项。

> This query allows handling `stopwords` dynamically at runtime, is domain independent and doesn’t require a stopword file. It prevents scoring / iterating high frequency terms and only takes the terms into account if a more significant / lower frequency term matches a document. Yet, if all of the query terms are above the given `cutoff_frequency` the query is automatically transformed into a pure conjunction (`and`) query to ensure fast execution.

这个query允许运行时动态处理停用词，独立于域并且不需要依赖停用词文件。它可以防止对高频词打分、迭代，并且仅在更重要/较低频率的词与文档匹配时才考虑这些词。如果所有的查询此项都高于跟定的cuttof_frequency，这个查询会自动转换为纯连接词query（and）来确保更快的执行。

> The `cutoff_frequency` can either be relative to the total number of documents if in the range from 0 (inclusive) to 1 (exclusive) or absolute if greater or equal to `1.0`.

cuttof_frequency 既可以与文档总数有关，如果范围大于等于0小于1，也可以是绝对的如果大于等于1.0。

> Here is an example showing a query composed of stopwords exclusively:

这里有个例子，完全由停用词组合成的

```json
GET /_search
{
    "query": {
        "match" : {
            "message" : {
                "query" : "to be or not to be",
                "cutoff_frequency" : 0.001
            }
        }
    }
}
```

> The `cutoff_frequency` option operates on a per-shard-level. This means that when trying it out on test indexes with low document numbers you should follow the advice in [Relevance is broken](https://www.elastic.co/guide/en/elasticsearch/guide/2.x/relevance-is-broken.html).

cuttof_frequency选项在分片级别上生效。这就意味着当在文档数少的索引上尝试时，你应该遵循Relevance is broken的建议。

#### Synonyms--同义词

> The `match` query supports multi-terms synonym expansion with the [synonym_graph](https://www.elastic.co/guide/en/elasticsearch/reference/7.4/analysis-synonym-graph-tokenfilter.html) token filter. When this filter is used, the parser creates a phrase query for each multi-terms synonyms. For example, the following synonym: `"ny, new york" would produce:`
>
> ```
> (ny OR ("new york"))
> ```
>
> It is also possible to match multi terms synonyms with conjunctions instead:

match检索支持通过[synonym_graph](https://www.elastic.co/guide/en/elasticsearch/reference/7.4/analysis-synonym-graph-tokenfilter.html)词条过滤器扩展多词项同义词。当使用这个过滤器的时候，语法解析器会创建一个检索每个多词条同义词的语法。例如，下面的同义词：“ny,new york”会被解析成（ny OR ("new york")）

也有可能以连词匹配多个词项同义词：

```json
GET /_search
{
   "query": {
       "match" : {
           "message": {
               "query" : "ny city",
               "auto_generate_synonyms_phrase_query" : false
           }
       }
   }
}
```

> The example above creates a boolean query:
>
> ```
> (ny OR (new AND york)) city
> ```
>
> that matches documents with the term `ny` or the conjunction `new AND york`. By default the parameter `auto_generate_synonyms_phrase_query` is set to `true`.

上述例子会创建一个bool query：（ny OR (new AND york)）city ，以cy或者连词 new AND york 匹配文档。默认auto_generate_synonyms_phrase_query参数为true。

