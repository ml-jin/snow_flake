# Snowflake数据库调研及架构介绍



[TOC]

## 简介
https://docs.snowflake.com/en/user-guide/intro-key-concepts.html

Snowflake是作为软件即服务（SaaS）提供的分析数据仓库。与传统的数据仓库产品相比，Snowflake提供了一个更快，更易于使用且更加灵活的数据仓库。Snowflake的数据仓库不是建立在现有数据库或Hadoop等“大数据”软件平台上的，Snowflake数据仓库使用新的SQL数据库引擎，该引擎具有为云设计的独特架构。对于用户而言，Snowflake与其他企业数据仓库有很多相似之处，但还具有一些其他特有功能。

## 整体类别
计算

计算+接口

计算+存储+接口

## 存储
https://zhuanlan.zhihu.com/p/56745552

https://zhuanlan.zhihu.com/p/126357511

Snowflake是一款面向Amazon Cloud上EC2和S3而构建起来的在线数仓系统，支持极致弹性、多租户、端到端安全、完整CRUD、事务、内置半结构化、无结构化数据等特性。

属于对象存储类型。

这里其实snowflake是做了较多的调研，包括hdfs，s3，最后的结论是发现s3在peformance，usabliity， high avaibability， strong durabllity guarantees hard to beat。所以存储层聚焦到virtual warehouse层的data cache和skew resilience。skew resilience指的是即使ecs的规格一样，但是由于网络、磁盘io等原因，还是会导致不同节点之间性能不均衡的问题，云计算超卖原罪。

底层选择S3解决了很多的存储问题，但在现阶段延时问题还是存在的，因此，针对热数据做本地SSD缓存+多级缓存是很自然的选择。因为VirtualWarehouse单元是用户级别的，所以这些热数据缓存可以被Query级、进程级、EC2级、用户级大量复用，从而极大的降低成本。

Snowflake选择在多个VW间构建Consistent Hashing的缓存层，来管理S3上的表文件和对应node节点（真正的计算节点）之间的关系；同时优化器感知这个缓存层，在做物理执行计划时，将query中的扫表算子按表文件名分派到对应的node上，从而实现缓存的高命中率；同时，因为存储计算分离+share data架构，计算上并不强耦合缓存层，所以node节点的增删并不需要立即做缓存数据的shuffle，而是基于LRU，在多个后续Query中摊还的替换表文件，完成缓存层的Lazy Replacement，平滑过渡。

S3无限容量+数据多副本+分布式强一致等，还给Snowflake带来更多红利。

## S3的问题：

1. latency

2. cpu overhead，使用HTTP连接。http解包，封包

## S3的优势：

1. 操作简单，put、get、delete

2. 文件只能被整个重写，甚至不能在文件末尾append，文件大小必须在put的时候指定。

3. Get可以取部分文件。

## 基于这些特性做了很多适配的设计：

1. 表被水平划分成large，immutable文件，等同于传统数据库的block或者page；

2. 列或者属性使用PAX格式混合列存；

3. 每个文件有header，保存metadata；

4. 不仅仅使用s3作为table的存储，还使用s3保存临时数据（当节点的磁盘满的时候）

5. 大的query结果，因为结果可以全部写入s3，所以不需要传统数据库的curser

Metadata例如catalog对象，table由哪些s3文件组成，统计，锁，事务日志保存在一个kv存储里面，这个kv存储作为Cloud Services layer一部分。

 

## 存储引擎
https://docs.snowflake.com/en/user-guide/intro-key-concepts.html

Snowflake数据仓库使用新的SQL数据库引擎，该引擎具有为云设计的独特架构。

https://www.sohu.com/a/411196821_185201?_trans_=000014_bdss_dkygcbz

（关于引擎的具体设计没有查到，Snowflake既无法建立索引，又不可捕获统计信息，更无法管理分区，能知道的引擎功能只有压缩很好）

https://zhuanlan.zhihu.com/p/126357511

snowflake自己实现了一个执行引擎，engine build is ：

列存， 向量化，和push-based（这里针对是传统的火山模型）。这里并没有提code-gen。

列存：对cpu cache更加友好，避免cache miss，能使用SIMD指令；
向量化：避免雾化中间结果，相反数据是以pipeline的方式处理，数据以千行为单位一个batch一个batch处理，这种方式提高了IO效率和cache效率
Push-based：相比经典的火山模型。基于push的模型可以提高cache效率，因为他减少了loop的控制流，另外他还是snowflate可以有效的处理DAG类型的plan，相比于tree型执行计划，为sharing和pilelining中间结果创造了机会。
## 平台管理
架构
https://docs.snowflake.com/en/user-guide/intro-key-concepts.html

Snowflake的架构是传统shared-disk数据库架构和shared-nothing数据库架构的混合体。

与shared-disk数据库架构相似，Snowflake数据仓库中所有计算节点访问的持久化数据使用中央数据存储库存储。

也与shared-nothing数据库架构相似，Snowflake使用MPP（大规模并行处理）计算集群处理查询，集群中的每个节点都在本地存储整个数据集的一部分。

这种方法简化了shared-disk数据库架构的数据管理，还具备shared-nothing数据库的性能和横向扩展优势。



## Snowflake的独特架构包括三个关键层：

### 数据库存储

将数据加载到Snowflake后，Snowflake会将数据重组为内部优化的压缩列式格式。Snowflake将此优化的数据存储在云存储中。

Snowflake管理着存储此数据的所有方面——组织（organization），文件大小，结构，压缩，元数据，统计信息，并且数据存储的其他方面由Snowflake处理。Snowflake存储的数据对象不直接可见，客户也无法访问；只能通过使用Snowflake运行SQL查询操作才能进行访问。

### 查询处理层

查询功能在处理层中执行，Snowflake使用“虚拟仓库”（virtual warehouses）处理查询。每个虚拟仓库都是一个MPP计算集群，集群使用多个云提供商提供的计算节点，由Snowflake分配组成。

每个虚拟仓库是一个独立的计算集群，不与其他虚拟仓库共享计算资源。因此，每个虚拟仓库都不会影响其他虚拟仓库的性能。

### 云服务层

云服务层是协调整个Snowflake服务的集合。这些服务将Snowflake的所有不同组件结合在一起，以便处理从登录到查询等不同阶段分发的用户请求。云服务层也运行在由Snowflake提供且来自于云提供商的计算实例上。

这一层中包含如下服务：

认证方式
基础设施管理
元数据管理
查询解析与优化
访问控制


https://zhuanlan.zhihu.com/p/126357511

snowflake的架构带来的优势：

1. 在执行过程中不需要事务管理。当前只专注于处理immutable的文件。

2. 不需要buffer pool。（本地cache其实也是一种buffer pool，只是相比事务型数据库，少了很多一致性和加锁保护，本质还是由于处理的是immutable的文件）

3. 允许主要的操作（join,group by,sort）当主存满的时候，使用磁盘空间，纯粹的使用主存在分析型场景下太受限，大部分分析都会有大join和aggeration。

## 部署
自动化
snowflake的部署不需要自己布置，用户只需要选择想要购买的资源。

## 文档
https://docs.snowflake.com/en/other-resources.html

## 官方有挺全的教学文档

监控
资源

## 监控仓库负荷

https://docs.snowflake.com/en/user-guide/warehouses-load-monitoring.html

Web界面提供了一个查询负载图，该图描述了仓库在两周时间内处理的并发查询。仓库查询负载衡量在特定时间间隔内正在运行或排队的查询的平均数量。

tps/qps，慢查询
使用历史记录页面监视查询

https://docs.snowflake.com/en/user-guide/ui-history.html

通过“历史记录” 页面，可以查看和深入查看最近14天执行的所有查询的详细信息。该页面显示查询的历史列表，包括从SnowSQL或其他SQL客户端执行的查询。为每个查询显示的默认信息包括：

查询的当前状态：正在队列中等待，正在运行，成功，失败。
查询的SQL文本。
查询ID。
有关用于执行查询的仓库的信息。
查询开始和结束时间以及持续时间。
有关查询的信息，包括扫描的字节数和返回的行数。
管控
高可用，升级，提升配置
https://www.zhihu.com/question/421034559/answer/1482996654

https://docs.snowflake.com/en/user-guide/data-time-travel.html

snowflake使用Fail-safe而不是备份的方式实现高可用，用户完全不用操心。

升级提升配置也是购买的方式，剩下的不用管

资源以tshirt的x/xx/xxl方式来定义。这种规格定义初略看起来很有新意，但是归根结底还是以计算节点个数来定义规格，关键的创新还是在于：

用户按使用付费，如果没有跑query就不需要付费。
用户如果不满意，可以方便改规格，改规格还不能满足，还可以增加cluster数量。并且改规格和数据的体验对用户做的非常好，用户改规格和扩容可以做到业务不中断，不需要做数据重分布（这2点应该是开发和使用传统数仓的老疮疤）。
扩容
https://zhuanlan.zhihu.com/p/56745552

同时，Snowflake这套纯SaaS化体验的架构又与现在云平台上主推的“容器化”、“ServerLess“、“Pay-as-you-go”等主流趋势不谋而合。用户完全不需要关心集群有多大，有多少机器，只需要根据自己的“性能需求”和“价格意愿”而选择VirtualWarehouse的尺寸即可，就像买T恤一样非常宽松、简单的规格；面向用户的CloudService和面向数据的Storage，完全实现弹性伸缩、按量计费，用户开通完服务，通过系统分配的Endpoint就可以“拎包入住”了

## 错误查询
Time Travel

https://docs.snowflake.com/en/user-guide/data-time-travel.html

利用Snowflake Time Travel，您可以在定义的时间段内随时访问历史数据（即已更改或删除的数据）。它是执行以下任务的强大工具：

恢复可能意外或有意删除的与数据相关的对象（表，模式和数据库）。
过去从关键点复制和备份数据。
分析指定时间段内的数据使用/操作。
您可以在定义的时间段内执行以下操作：

查询过去已更新或删除的数据。
在过去的特定时间点或之前，创建整个表，模式和数据库的克隆。
还原已删除的表，架构和数据库。
在定义的时间段过后，数据将移入Snowflake Fail-safe，这些操作将不再执行。

Fail-safe

与Time Travel不同，Fail-safe 确保在发生系统故障或其他灾难性事件时保护历史数据，例如，硬件故障或安全漏洞。

为什么使用故障安全而不是备份？

任何数据库管理系统都可能发生数据损坏或丢失。为了降低风险，DBA传统上执行完整和增量备份，但这会使整体数据存储增加一倍甚至三倍。此外，由于多种因素，数据恢复可能会很麻烦且成本很高，其中包括：

重新加载丢失的数据所需的时间。
恢复期间的业务停机时间。
自上次备份以来的数据丢失。
Snowflake的多数据中心冗余架构大大减少了传统备份的需求。但是，由于数据损坏/丢失可能会无意中发生，因此仍然存在风险。

Fail-safe为备份提供了一种高效且具有成本效益的替代方案，可消除残留风险并随数据扩展。

## 性能
https://docs.snowflake.com/en/user-guide/sample-data-tpcds.html

## TPC-DS测试

对于10 TB版本，使用Snowflake 2X-Large仓库，完整的99个TPC-DS查询应在不到2小时的时间内完成。如果使用100 TB版本，则使用4X大型仓库的查询将在大约3个小时内完成。

## 接口
Snowflake支持多种连接服务的方式：

基于Web的用户界面，可从该界面访问管理和使用Snowflake的所有方面。

命令行客户端（例如SnowSQL）也可以访问管理和使用Snowflake的所有方面。

其他应用程序（例如Tableau）可以使用ODBC和JDBC驱动程序来连接到Snowflake。

本机连接器（例如Python）可用于开发用于连接到Snowflake的应用程序。

可用于将ETL工具（例如Informatica）和BI工具等应用程序连接到Snowflake的第三方连接器。

sql增删改查
https://docs.snowflake.com/en/sql-reference-commands.html

增删改查都是支持的

sql兼容性
https://docs.snowflake.com/en/user-guide/querying.html

Snowflake支持标准SQL，包括ANSI SQL：1999和SQL：2003分析扩展的子集。Snowflake还支持许多命令的通用变体，这些变体不会相互冲突。

## 应用
优势
https://www.zhihu.com/question/421034559/answer/1482996654

它的优点可以一言以蔽之，那就是一个商业化的“Spark“集群，让你不用再操心高可用，运维，安全等等问题 （当然，SF用的是自己的技术，不是Spark)。

https://zhuanlan.zhihu.com/p/54439354

snowflake的优势用一句概况就是cloud-native的数据仓库服务。

 

具体有点没找到说的，我自己根据资料写几点吧

1.计算层独立于存储层存在，可以随时提高或降低计算资源以应对需求，可以在搬运数据的同时进行查询，可以给各个LOB提供合适的资源并独立出ETL和DevOps的处理需求。

2.数据库规格和cluster数量方便更改，且可以做到业务不中断，不需要做数据重分布，用户体验极佳

3.用户按使用付费，如果没有跑query就不需要付费

4.支持半结构化和非结构化数据，方便用户导入和处理数据，不必线下处理

5.安全性，Snowflake是把安全作为基础能力而设计的。

（https://zhuanlan.zhihu.com/p/56745552，国外的市场和法律对云平台的用户数据安全有着异常严苛的要求，Snowflake从计算链路的端到端，以及数据存储上两方面来构建其安全体系。设计出一套多层次的key加密体系，为这些加密key设计生命周期，通过key rotation来持续更新key，通过rekey来持续更新数据，最顶层key则利用硬件加密（比如阿里云上的KMS服务）来维护。）

6.提出了With Secure Data Sharing（安全数据共享）的新概念。不会在帐户之间复制或传输任何实际数据。所有共享都是通过Snowflake的独特服务层和元数据存储来完成的。这是一个重要的概念，因为这意味着共享数据不会占用消费者帐户中的任何存储，因此不会对消费者的月度数据存储费用有所贡献。

## 约束
网上基本都是各种夸，不好找约束。

1.由于架构和引擎的不同，snowflake不支持索引，迁移数据库时可能需要对sql做特定的改动。

2.数据对用户不透明，不可见，只能通过使用Snowflake运行SQL查询操作才能进行访问。

3.节点故障可以通过retry其他node来解决，当前并不处理部分失败的问题，大事务失败话重试时间会比较久。

4.选择S3这种“Write-once”或“Append-only”的存储作为其底层存储，相比于传统存储有些劣势，在现阶段有延时问题

## 企业实例
https://zhuanlan.zhihu.com/p/248545758?utm_source=wechat_timeline

该公司服务于多个行业的3117家不同规模的机构。在这些客户中，有7家是财富10强企业，146家是财富500强公司，分别占收入的4%和26%。以下是按行业垂直分类的代表性客户名单：



该公司根据使用时间以及使用中的计算能力和存储容量来对其产品进行定价。这与亚马逊Redshift解决方案的定价方式类似，不过在Snowflake中，客户可以将存储和计算分开，并按照使用量付费。

BigQuery更适合偶尔大量使用数据的用户，而Snowflake则更适合稳定使用数据的用户。

## 附录
来自db-engines.com的相关信息

Editorial information provided by DB-Engines
Name	Snowflake
Description	Cloud-based data warehousing service for structured and semi-structured data
Primary database model	Relational DBMS
DB-Engines Ranking 	
Trend Chart
Score	10.10
Rank	#42	  Overall
 	#26	  Relational DBMS
Website	www.snowflake.com
Technical documentation	docs.snowflake.net/­manuals/­index.html
Developer	Snowflake Computing Inc.
Initial release	2014
License 	commercial
Cloud-based only 	yes
DBaaS offerings (sponsored links) 	 
Server operating systems	hosted
Data scheme	yes 
Typing 	yes
XML support 	yes
SQL 	yes
APIs and other access methods	CLI Client
JDBC
ODBC
Supported programming languages	JavaScript (Node.js)
Python
Server-side scripts 	user defined functions
Triggers	no 
Partitioning methods 	yes
Replication methods 	yes
MapReduce 	no
Consistency concepts 	Immediate Consistency
Foreign keys 	yes
Transaction concepts 	ACID
Concurrency 	yes
Durability 	yes
In-memory capabilities 	no
User concepts 	Users with fine-grained authorization concept, user roles and pluggable authentication





