# databricks产品学习笔记
databricks本身是一家公司，旗下有多款开源产品，包括：Spark、Delta Lake、MLFlow、Koalas、Delta Sharing等。
其通过将上述开源产品，组装为一个数据分析平台，提供云服务进行盈利。 其理念是提供一个：统一数据分析平台。 

databricks最新估值为280亿美元。

此学习笔记将重点关注，databricks整体对外提供的产品形式，以及Delta Lake这个产品的架构。

## 核心开源组件说明
### Spark
Spark是一款大数据分析引擎，是“统一数据分析平台”的最好践行者。其扮演者承上启下的角色。

**承上：** 
支持交互式查询、批处理、流处理、机器学习、图计算等使用场景。

**启下：** 
支持基本上你能见到的所有数据格式类型，包括Parquet、CSV、Json、ORC等。
同时所有数据源类型，包括流式的kinesis、Kafka等，数据库如MySQL等，Spark都能有机地统一起来。
不会像原来Hadoop那样，把一堆散落的零件丢给用户，让用户自己去拿MapReduce拼。

### MLFlow




## 参考文档
* [Spark诞生头十年：Hadoop由盛转衰，统一数据分析大行其道](https://cloud.tencent.com/developer/news/491196)

