# snowflake学习笔记
记录调研snowflake过程中的所思所感。 

snowflake是一款云上的在线数据仓库产品，其称为：数据平台(data platform)。 目前市值679亿美元（最高时1200亿美元）。
与传统产品相比，snowflake可以实现更快、更易于使用、更灵活的数据库存储、处理和分析解决方案。

snowflake数据平台，不是基于现有数据库技术或者大数据平台，如Hadoop等。 
它通过将一个全新的SQL查询引擎和一个创新的云原生架构结合起来。
对于用户来说，snowflake提供了企业分析数据库所需的所有功能，并且附加了许多特殊功能及独特能力（如数据共享）。

## 数据平台作为云服务(Data Platform as a Cloud Service)
Snowflake是一款真正的SaaS产品。 更具体的说：
* 不需要选择、安全、配置、管理硬件资源
* 几乎没有软件需要安装、配置、管理
* 持续的维护、管理、升级和调优都是由Snowflake处理

Snowflake完全跑在云基础设施上面。所有的组件服务（除了可选的命令行工具、驱动、连接器）都运行在公有云的基础设施之上。

Snowflake使用公有云的虚拟计算实例用于计算需要，存储服务用于数据的持久化。 Snowflake不能运行在私有的云环境中（包括自有的或托管的）。

Snowflake不是一个打包软件需要用户去安装的。 Snowflake在云上管理软件的安装更新等所有方面。

## 整体架构
![image](architecture-overview.png)

Snowflake的架构是混合了传统的共享磁盘(shared-disk)和无共享(shared-nothing)的架构。
和shared-disk架构类似的，Snowflake使用一个中心化的数据仓库用来持久化数据，然后其能够被所有在平台中的计算节点访问到。
同时和shared-nothing架构类似的，Snowflake使用MPP(大规模并行处理)计算集群处理查询，集群中的每个节点本地存储整个数据集的一部分。
这种方式提供了shared-disk架构数据管理的简单性，同时又具有shared-nothing架构的性能和横向扩展优势。 

Snowflake独特的架构由3个关键层组成：
* Database Storage
* Query Processing
* Cloud Services

### Database Storage
当数据加载进Snowflake，其组织数据进入其内部优化，压缩，列存格式。 Snowflake存储这些优化后的数据到云存储服务中。

Snowflake管理数据存储的各个方面，包括：组织，文件大小，结构，压缩，元数据，统计信息以及其他存储方面的事情都由Snowflake处理。
数据对象不能直接被用户可视化或可见，想让其可见只能通过在Snowflake上执行SQL操作。

### Query Processing
SQL的执行在此层进行处理，Snowflake处理查询通过“virtual warehouses(是其内部的抽象名称，对应这一层的处理节点)”，
每个“virtual warehouses“是由Snowflake从云提供商处分配的多个计算节点组成的MPP计算集群。

每个“virtual warehouses“是一个独立的计算集群，不与其他“virtual warehouses“共享计算资源。
因此，每个“virtual warehouses“不会对其他虚拟仓库的性能产生影响。

### Cloud Services
云服务层是一个服务集合，用于协调Snowflake上的操作。
这些服务将Snowflake的所有不同组件结合到一起，以处理从登录到查询分发的用户请求。
云服务层也运行在Snowflake从云服务商获取的计算实例上。

此层管理的服务包括：
* 认证
* 基础设施管理
* 元数据管理
* 查询解析和优化
* 访问控制

## 连接到Snowflake(Connecting to Snowflake)
Snowflake支持多种方式连接到其服务：
* 一个基于web的用户界面，可以从所有方面管理和使用Snowflake
* 命令行客户端（eg. SnowSQL），也能够访问和管理Snowflake的所有功能
* ODBC和JDBC驱动程序可以被其他应用程序用来连接Snowflake   
* 原生连接器(Native connectors)，其可被用于开发应用程序，用来连接到Snowflake
* 第三方连接可被用于连接应用到Snowflake，如ETL工具(eg. Informatica)和BI工具(eg. ThoughtSpot)

## 目录
* [初步使用体验及思考](初步使用体验及思考.md)

