# Databricks产品学习笔记
Databricks本身是一家公司，旗下有多款开源产品，包括：Spark、Delta Lake、MLFlow、Koalas、Delta Sharing等。
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
这是一个能够覆盖机器学习全流程（从数据准备到模型训练到最终部署）的新平台，旨在为数据科学家构建、测试和部署机器学习模型的复杂过程做一些简化工作。
MLflow提供了一组轻量级api，可以与任何现有的机器学习应用程序或库(TensorFlow, PyTorch, XGBoost等)一起使用，
无论你当前在哪里运行ML代码(例如在笔记本、独立应用程序或云)。

### Koalas
Koalas是对标Python pandas数据分析库而创建的库，接口API几乎相同，但是在处理大数量的场景下，具有更好的性能。  

### Delta Sharing
Delta Sharing是一个数据共享的开源协议（基于REST HTTP协议）。
自称是业界第一个用于安全数据共享的开放协议；使得与其他组织共享数据变得简单，而不管他们使用的是哪种计算平台。

#### 关键特性
* 直接分享实时数据：轻松地共享Delta Lake中现有的实时数据，而无需将其复制到另一个系统。
* 支持多种客户端：数据接收者可以直接从Pandas、Apache Spark、Rust和其他系统连接到Delta Shares，而无需首先部署特定的计算平台。
* 安全和治理：Delta Sharing可以轻松地管理、跟踪和审计对共享数据集的访问。
* 可扩展：通过利用S3、ADLS和GCS等云存储系统，可靠而有效地共享tb级的数据集。

从协议上看，是数据提供者直接提供接口，给数据使用者调用（可以传sql谓词进行数据过滤），不需要先copy数据到对方的系统中。


### Delta Lake
Delta Lake是数据的存储层，相对重点的说明。





## 参考文档
* [Spark诞生头十年：Hadoop由盛转衰，统一数据分析大行其道](https://cloud.tencent.com/developer/news/491196)
* [MLflow](https://github.com/mlflow/mlflow)
* [Delta Sharing](https://github.com/delta-io/delta-sharing)
