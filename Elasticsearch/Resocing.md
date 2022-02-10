Rescoring

​		重新打分可以对query和post_filter阶段返回的topN（例如100-500）的文档进行重新打分，以提高精度，这个机制使用辅助算法，而不是使用成本更高的算法对索引中的所有索引进行打分。

​		rescore请求是每个分片的返回结果在节点排序之前执行的。

​		目前来说，rescore API只有一个实现：检索打分（使用query来调整打分）。未来可能会有可替代的重新打分机制。

> search_type设置为scan或者count之后，rescore不会执行



