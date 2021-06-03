# Databricks产品学习笔记
Databricks本身是一家公司，旗下有多款开源产品，包括：Spark、Delta Lake、MLFlow、Koalas、Delta Sharing等。
其通过将上述开源产品，组装为一个数据分析平台，提供云服务进行盈利。 其理念是提供一个：统一数据分析平台。 

databricks最新估值为280亿美元。

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
Delta Lake是为Spark等大数据引擎提供了可扩展、ACID事务的存储层。 其基于现有的存储系统而构建，如：S3、ADLS、GCS、HDFS。

![image](Delta-Lake-marketecture-0423c.png)

#### 关键特性
* ACID事务：数据湖通常有多个数据管道并发地读取和写入数据，由于缺乏事务，数据工程师必须通过一个繁琐的过程来确保数据完整性。
  Delta Lake为数据湖带来了ACID事务，并且提供可序列化这种最高级别的事务隔离级别。
* 可扩展的元数据处理：在大数据场景下，元数据本身就可能变成“大数据”。
  Delta Lake把元数据当做数据一样的对待，利用Spark的分布式处理能力来管理所有的元数据。 
  这样Delta Lake就可以轻松处理拥有数十亿个分区和文件的pb级表。
* 时间闪回(数据版本)：Delta Lake提供数据快照，使开发人员能够访问并恢复到早期版本的数据，以进行审计、回滚或重现实验。
* 开发格式：Delta Lake中的所有数据都存储在Apache Parquet格式中，使Delta Lake能够利用Parquet固有的高效压缩和编码方案。
* 统一批处理和流处理：Delta Lake中的表既是一个批处理表，也是一个流处理源和流接受器（source and sink）。
  流数据摄取、批量历史回填和交互式查询都是开箱即用的。
* 强模式(Schema)：Delta Lake提供了制定Schema的能力，并强制校验模式。 以确保数据类型是正确的，并且提供了所需的列，防止脏数据导致数据损坏。
* 模式演变：大数据是不断变化的。 Delta Lake可以更改可以自动应用的表模式，而无需笨重的DDL。（应用程序写入数据时，指定一个mergeSchema=true的参数）
* 审计历史：Delta Lake事务日志记录了对数据所做的每一次更改的详细信息，提供了对更改的完整审计跟踪。
* 更新和删除：Delta Lake支持Scala、Java、Python和SQL api来合并、更新和删除数据集。
  这使您可以轻松地遵守GDPR(General Data Protection Regulation:通用数据保护条例)等条例，并简化了更改数据采集等用例。
* 完全兼容Spark API：由于Delta Lake与常用的大数据处理引擎Spark完全兼容，开发人员可以将Delta Lake与现有的数据管道一起使用，只需要做少量的改动。
* 随处访问：使用你选择的语言，服务，连接器或数据库与Delta Lake建立连接。 包括：Rust，Python，DBT，Hive，Presto等等。


## 参考文档
* [Spark诞生头十年：Hadoop由盛转衰，统一数据分析大行其道](https://cloud.tencent.com/developer/news/491196)
* [MLflow](https://github.com/mlflow/mlflow)
* [Delta Sharing](https://github.com/delta-io/delta-sharing)
* [Delta Lake](https://github.com/delta-io/delta)
