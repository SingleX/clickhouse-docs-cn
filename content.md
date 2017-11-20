## 介绍

### ClickHouse 是什么?

ClickHouse 是一个用于联机分析处理（OLAP）的列式的数据库管理系统（DBMS）。

在一个“正常”的面向行的数据库管理系统中，数据是以如下顺序存储：

```
5123456789123456789     1       Eurobasket - Greece - Bosnia and Herzegovina - example.com      1       2011-09-01 01:03:02     6274717   1294101174      11409   612345678912345678      0       33      6       http://www.example.com/basketball/team/123/match/456789.html http://www.example.com/basketball/team/123/match/987654.html       0       1366    768     32      10      3183      0       0       13      0\0     1       1       0       0                       2011142 -1      0               0       01321     613     660     2011-09-01 08:01:17     0       0       0       0       utf-8   1466    0       0       0       5678901234567890123               277789954       0       0       0       0       0
5234985259563631958     0       Consulting, Tax assessment, Accounting, Law       1       2011-09-01 01:03:02     6320881   2111222333      213     6458937489576391093     0       3       2       http://www.example.ru/         0       800     600       16      10      2       153.1   0       0       10      63      1       1       0       0                       2111678 000       0       588     368     240     2011-09-01 01:03:17     4       0       60310   0       windows-1251    1466    0       000               778899001       0       0       0       0       0
...
```

换句话说，所有与同一行相关的值都被存储在一起。面向行的数据库管理系统举例有：MySQL，PostgreSQL，MS SQL Server 等。

在一个面向列的数据库管理系统中，数据像这样被存储：

```
WatchID:    5385521489354350662     5385521490329509958     5385521489953706054     5385521490476781638     5385521490583269446     5385521490218868806     5385521491437850694   5385521491090174022      5385521490792669254     5385521490420695110     5385521491532181574     5385521491559694406     5385521491459625030     5385521492275175494   5385521492781318214      5385521492710027334     5385521492955615302     5385521493708759110     5385521494506434630     5385521493104611398
JavaEnable: 1       0       1       0       0       0       1       0       1       1       1       1       1       1       0       1       0       0       1       1
Title:      Yandex  Announcements - Investor Relations - Yandex     Yandex — Contact us — Moscow    Yandex — Mission        Ru      Yandex — History — History of Yandex    Yandex Financial Releases - Investor Relations - Yandex Yandex — Locations      Yandex Board of Directors - Corporate Governance - Yandex       Yandex — Technologies
GoodEvent:  1       1       1       1       1       1       1       1       1       1       1       1       1       1       1       1       1       1       1       1
EventTime:  2016-05-18 05:19:20     2016-05-18 08:10:20     2016-05-18 07:38:00     2016-05-18 01:13:08     2016-05-18 00:04:06     2016-05-18 04:21:30     2016-05-18 00:34:16     2016-05-18 07:35:49     2016-05-18 11:41:59     2016-05-18 01:13:32
```

这些例子仅展示了数据的排列顺序。不同列的值被分开存储，同一列的值被存储在一起。面向行的数据库管理系统举例有： Vertica，Paraccel（Actian Matrix）（Amazon Redshift），SybaseIQ，Exasol，Infobright，InfiniDB，MonetDB（VectorWise）（Actian Vector），LucidDB，SAP HANA，Google Dremel，Google PowerDrill，Druid，kdb+ 等。

存储数据的不同顺序更好的适用于不同的场景。数据获取场景涉及到产生了什么样的查询，频率和占比情况；每种类型的查询要读取多少数据——行数，列数和字节数；读取和更新数据的关系；工作数据的大小和本地使用情况；是否使用了事务，以及它们是如何隔离的；对数据复制和逻辑完整性的要求；对每种查询的延迟和吞吐量的要求等等。

系统负载越高，为场景定制系统就越重要，这种定制就越特殊。没有一个系统能同等的适用于显著不同的场景。如果一个系统适用于一组广泛的场景，在高负载下，系统会同样糟糕的处理所有这些场景，或者仅在其中一个场景下工作的很好。

对 OLAP 场景，我们会说以下是正确的：

* 绝大多数的请求是读取
* 数据被大批量的更新（&gt;1000行），而不是单行，或不更新
* 数据被添加进数据库，但不修改
* 对读取，相当大量的行被从数据库抽取，但仅仅是列的一个很小的子集
* 表是宽的，意味着它们包含大量的列
* 查询相对较少（通常每个 Server 每秒约成百上千的查询甚至更少）
* 对简单查询，50ms 左右的延迟是可以接受的
* 列的值相当小——数字和短字符串（如，每个 URL 平均 60 bytes）
* 当处理一个独立查询时，需要高吞吐量（高达每台 Server 每秒数十亿行）
* 没有事务
* 对数据一致性要求较低
* 每个查询对应一个大的表，所有的表都比较小，除了一个
* 一次查询的结果相对于源数据来说相当小，就是说，数据是被过滤或聚合过的。结果适合于单个 Server 的 RAM。

很容易看出，OLAP 场景与其他流行场景非常不同（如 OLTP 或 K-V 存取）。所以说通过尝试使用 OLTP 或 K-V 数据库处理分析查询获得适当的性能是没有意义的。例如，如果你尝试使用 MongoDB 或 Elliptics 做分析，与 OLAP 型数据库相比你会得到非常差的性能表现。

面向列的数据库能更好的适合 OLAP 场景（大多数查询的处理速度至少提高了100倍），理由如下：

1. 为了 I/O
2. 对一个分析查询，仅有很小数量的表的列需要读取。在一个列式数据库中，你可以仅读取需要的数据。例如，如果你需要 100 列中的 5 列，你可以预期 I/O 能减少20倍
3. 既然数据是按 packets 读取的，因此容易压缩。列式的数据也容易压缩。这进一步减少了 I/O 量
4. 由于减少了 I/O，可以有更多的数据适合在系统缓存中

例如，查询“每个广告平台的记录的条数”需要读取一个“广告平台ID”的列，这会占用 1 个未压缩的字节。如果大多数流量不是来自广告平台，那么你可以期望此列至少有10倍的压缩。当使用快速压缩算法时，数据解压缩速度可以达到每秒至少几千兆字节的未压缩数据。换句话说，这个查询可以在一台服务器上以每秒大约几十亿行的速度处理。这个速度实际上是在实践中实现的。

举例：

```
milovidov@hostname:~$ clickhouse-client
ClickHouse client version 0.0.52053.
Connecting to localhost:9000.
Connected to ClickHouse server version 0.0.52053.

:) SELECT CounterID, count() FROM hits GROUP BY CounterID ORDER BY count() DESC LIMIT 20

SELECT
    CounterID,
    count()
FROM hits
GROUP BY CounterID
ORDER BY count() DESC
LIMIT 20

┌─CounterID─┬──count()─┐
│    114208 │ 56057344 │
│    115080 │ 51619590 │
│      3228 │ 44658301 │
│     38230 │ 42045932 │
│    145263 │ 42042158 │
│     91244 │ 38297270 │
│    154139 │ 26647572 │
│    150748 │ 24112755 │
│    242232 │ 21302571 │
│    338158 │ 13507087 │
│     62180 │ 12229491 │
│     82264 │ 12187441 │
│    232261 │ 12148031 │
│    146272 │ 11438516 │
│    168777 │ 11403636 │
│   4120072 │ 11227824 │
│  10938808 │ 10519739 │
│     74088 │  9047015 │
│    115079 │  8837972 │
│    337234 │  8205961 │
└───────────┴──────────┘

20 rows in set. Elapsed: 0.153 sec. Processed 1.00 billion rows, 4.00 GB (6.53 billion rows/s., 26.10 GB/s.)

:)
```

1. CPU 方面

既然执行一次查询需要处理大量的行，它有助于调度整个向量的所有操作，而不是单独的行，或者实现查询引擎那样就几乎没有调度成本了。如果不这样做，对任何像样的磁盘子系统，查询解释器必将拖慢 CPU。将数据列式存储并在可能的情况下按列处理它是有意义的。

有两种方法可以做到这一点：

1. 一个向量引擎。所有的操作都是为向量而写的，而不是单独的值。这意味着你不需要经常调用操作，并且调度成本可以忽略不计。操作代码包含一个优化的内部循环。
2. 代码生成。为查询生成的代码具有所有的间接调用。

这不是在“普通”数据库中完成的，因为运行简单查询时没有意义。但是，也有例外。例如，MemSQL 在处理 SQL 查询时使用代码生成来减少延迟。 （为了比较，分析式的 DBMS 需要优化吞吐量，而不是延迟。）

请注意，为了提高 CPU 效率，查询语言必须是声明式的（SQL 或 MDX），或者至少是一个向量（J，K）。查询应该只包含隐式循环，以便于优化。

### ClickHouse 的独特功能

#### 真正的面向列的 DBMS

在一个真正的面向列的DBMS中，没有任何“垃圾”存储在值中。例如，必须支持恒定长度值，以避免在值旁边存储长度“数字”。例如，十亿个UInt8类型的值实际上应该消耗未压缩状态下大约 1 GB的空间，否则这将强烈影响 CPU 的使用。因为解压缩的速度（CPU使用率）主要取决于未压缩的数据量，所以即使在未压缩的情况下，紧凑地存储数据（没有任何“垃圾”）也是非常重要的。

这值得注意，因为有些系统可以单独存储单独列的值，但由于其他场景的优化，无法有效处理分析查询。例如 HBase，BigTable，Cassandra 和 HyperTable。在这些系统中，每秒钟可以获得大约十万行的吞吐量，但是每秒不会达到数亿行。

另外请注意，ClickHouse 是一个 DBMS，而不是一个单一的数据库。 ClickHouse 允许在运行时创建表和数据库，加载数据和运行查询，而无需重新配置和重新启动服务器。

#### 数据压缩

一些面向列的DBMS（InfiniDB CE 和 MonetDB）不使用数据压缩。但是，数据压缩确实提高了性能。

#### 数据磁盘存储

Many column-oriented DBMSs \(SAP HANA, and Google PowerDrill\) can only work in RAM. But even on thousands of servers, the RAM is too small for storing all the pageviews and sessions in Yandex.Metrica.

许多面向列的DBMS（SAP HANA 和 Google PowerDrill）只能在 RAM 中工作。但即使在数千台服务器上，RAM也太小，无法存储Yandex.Metrica 中的所有浏览量和会话。

#### 多核并行处理

大型查询以自然的方式并行化。

#### 多机分布式处理

上面列出的列式 DBMS 几乎都不支持分布式处理。在ClickHouse中，数据可以驻留在不同的分片（shards）。每个分片可以是用于容错的一组副本（replicas）。查询在所有分片上并行处理。这对用户来说是透明的。

#### SQL 支持

如果你熟悉标准的 SQL，我们不能真正谈论 SQL 的支持。 `NULL` 不被支持。所有的功能都有不同的名字。但是，这是一个基于 SQL 的声明性查询语言，在许多情况下不能与 SQL 区分开来。 `JOIN` 是支持的。子查询在 `FROM`，`IN`，`JOIN` 子句中受支持；也支持标量子查询。关联子查询不受支持。

#### 向量引擎

数据不仅按列存储，而且由列的部分——向量来进行处理。这使我们能够实现比较高的 CPU 性能表现。

#### 实时数据更新

ClickHouse 支持主键（primary key）表。为了快速执行对主键范围的查询，数据使用 merge tree 进行递增排序。由于这个原因，数据可以不断地添加到表中。添加数据时没有锁。

#### 索引

例如，拥有主键可以在特定时间范围内为特定客户端（Metrica 计数器）提取数据，并且延迟时间小于几十毫秒。

#### 适合在线查询

这使我们可以使用该系统作为Web界面的后端。低延迟意味着 Yandex.Metrica 界面网页正在加载时（在线模式）可以毫不拖延地处理查询。

#### 支持近似计算

1. 该系统包含用于近似计算各种值，中位数和分位数的集合函数。

2. 支持基于数据的一部分（样本）运行查询并获得近似结果。在这种情况下，从磁盘检索的数据比例较少。

3.  支持为有限数量的随机 key（而不是所有 key）运行聚合。在数据 key 分发的一定条件下，这将提供一个相当准确的结果，同时使用较少的资源。

#### 数据复制和支持副本上的数据完整性

使用异步多主复制。写入任何可用的副本后，数据将分发到所有剩余的副本。系统在不同的副本上保持相同的数据。数据在失败后自动恢复，或对复杂情况使用“开关”。有关更多信息，请参阅“数据复制”（Data replication）一节。

### ClickHouse 中可能被认为是缺点的特性

1. 没有事务
2. 对于聚合，查询结果的大小必须适合于单个服务器上的RAM。但是，查询的源数据量可能无限大。

3. 缺乏完整的 `UPDATE` / `DELETE` 实现。

### Yandex.Metrica 的任务

ClickHouse 目前驱动着世界上第二大网络分析平台 Yandex.Metrica，每天有超过13万亿的数据库记录和超过200亿个事件，直接从非聚合数据中生成定制报告。

我们需要根据点击和会话获取自定义报告，并使用用户的设置自定义细分。报告数据实时更新。查询必须立即运行（在线模式）。我们必须能够在任何时间段建立报告。必须计算复杂的聚合，例如独立访客的数量。目前（2014年4月），Yandex.Metrica 每天接收约120亿次事件（PV 和鼠标点击）。所有这些事件都必须存储以建立自定义报告。单个查询可能需要几秒钟扫描数亿行，或者数百万行不超过几百毫秒。

#### 在Yandex.Metrica 和其他 Yandex 服务的使用

ClickHouse 在 Yandex.Metrica 中用于多种用途。其主要任务是使用非聚合数据以在线模式构建报告。它使用一个 374 台服务器的集群，在数据库中存储了 20.3 万亿行。压缩数据量不计重复和复制，大约是 2PB。未压缩的数据量（以 TSV 格式）大约为 17PB。

ClickHouse 也用于：

* 存储 WebVisor 数据
* 处理中间数据
* 使用 Analytics 构建全球报告
* 运行查询以调试 Metrica 引擎
* 分析来自 API 和用户界面的日志

ClickHouse 在其他 Yandex 服务中至少有十几次的安装：搜索行业，市场，导向，业务分析，移动开发，AdFox，个人服务等。

#### 聚合与非聚合数据

有一种流行的观点认为，为了有效地计算统计数据，您必须聚合数据，因为这会减少数据量。 

但是数据聚合是一个非常有限的解决方案，原因如下：

* 您必须具有用户需求的预定义的报告列表。用户不能制作自定义报告
* 聚合大量 key 时，数据量不会减少，聚合无用
* 对于大量的报告，聚合变化太多（组合爆炸）
* 聚合高基数 key（如URL）时，数据量不会减少太多（小于两倍）。出于这个原因，聚合的数据量可能会增长，而不是缩小
* 用户不会查看我们为他们计算的所有报告。大部分的计算是无用的
* 数据的逻辑完整性可能与各种聚合相违背

如果我们不汇总任何内容并使用非汇总的数据，这实际上可能会减少计算量。

但是，通过汇总，相当一部分工作可以脱机，相对平静地完成。相反，在线计算需要尽可能的快，因为用户正在等待结果。

Yandex.Metrica 有一个称为 Metrage 的专门的系统来聚合数据，它用于生成大多数报告。从2009年开始，Yandex.Metrica 还使用专门的OLAP 数据库来处理非聚合数据，称为 OLAPServer，以前用于报告生成器。 OLAPServer对非聚合数据运行良好，但是它有许多限制，不能根据需要用于所有报告。其中包括缺乏对数据类型的支持（只支持数字），以及无法实时增量更新数据（只能通过每天重写数据来完成）。OLAPServer不是一个DBMS，而是一个专门的 DB。

为了消除 OLAPServer 的限制，并解决所有报告使用非汇总数据的问题，我们开发了ClickHouse DBMS。

### 可能愚蠢的问题

#### 为什么不使用像 MapReduce 这样的系统？

像 map-reduce 这样的系统是分布式计算系统，其中使用分布式排序来执行 reduce 阶段。关于这个方面，map-reduce 与 YAMR，Hadoop，YT 等其他系统类似。

这些系统由于延迟而不适合在线查询，所以不能用于后端级别的网页界面。这样的系统也不适合实时更新。分布式排序并不是减少操作的最佳解决方案，如果操作的结果和所有中间结果都存在，就像通常在联机分析查询的情况下发生的那样，它们适合单个服务器的运行内存。在这种情况下，执行 reduce 操作的最佳方法是使用 hash-table。map-reduce 任务的常用优化方法是在内存中使用 hash-table 的组合操作（partial reduce）。这个优化是由用户手动完成的。分布式排序是简单的 map-reduce 作业长时间延迟的主要原因。

与 map-reduce 类似的系统可以在集群上运行任何代码。但是对于 OLAP 用例，声明式查询语言更适合，因为它们允许更快地进行搜索。例如，Hadoop 有 Hive 和 Pig。还有其他的：Cloudera Impala，Shark（已弃用）和 Spark SQL for Spark，Presto，Apache Drill。然而，与专门系统的性能相比，这些任务的性能是非常不理想的，相对较高的延迟不允许将这些系统用作网络接口的后端。

YT allows you to store separate groups of columns. But YT is not a truly columnar storage system, as the system has no fixed length data types \(so you can efficiently store a number without “garbage”\), and there is no vector engine. Tasks in YT are performed by arbitrary code in streaming mode, so can not be sufficiently optimized \(up to hundreds of millions of lines per second per server\). In 2014-2016 YT is to develop “dynamic table sorting” functionality using Merge Tree, strongly typed values ​​and SQL-like language support. Dynamically sorted tables are not suited for OLAP tasks, since the data is stored in rows. Query language development in YT is still in incubating phase, which does not allow it to focus on this functionality. YT developers are considering dynamically sorted tables for use in OLTP and Key-Value scenarios.

YT允许您存储单独的一组列。但YT不是一个真正的柱状存储系统，因为系统没有固定长度的数据类型（所以你可以有效地存储一个数字而不是“垃圾”），而且没有矢量引擎。 YT中的任务由流模式下的任意代码执行，因此无法进行充分优化（每台服务器每秒高达数亿行）。在2014-2016年，YT将开发使用合并树，强类型值和类SQL语言支持的“动态表格排序”功能。动态排序的表不适用于OLAP任务，因为数据存储在行中。 YT中的查询语言开发仍处于孵化阶段，不能专注于这个功能。 YT开发者正在考虑在OLTP和Key-Value场景中使用动态排序的表。

### Performance

According to internal testing results, ClickHouse shows the best performance for comparable operating scenarios among systems of its class that were available for testing. This includes the highest throughput for long queries, and the lowest latency on short queries. Testing results are shown on this page.

#### Throughput for a single large query

Throughput can be measured in rows per second or in megabytes per second. If the data is placed in the page cache, a query that is not too complex is processed on modern hardware at a speed of approximately 2-10 GB/s of uncompressed data on a single server \(for the simplest cases, the speed may reach 30 GB/s\). If data is not placed in the page cache, the speed depends on the disk subsystem and the data compression rate. For example, if the disk subsystem allows reading data at 400 MB/s, and the data compression rate is 3, the speed will be around 1.2 GB/s. To get the speed in rows per second, divide the speed in bytes per second by the total size of the columns used in the query. For example, if 10 bytes of columns are extracted, the speed will be around 100-200 million rows per second.

The processing speed increases almost linearly for distributed processing, but only if the number of rows resulting from aggregation or sorting is not too large.

#### Latency when processing short queries

If a query uses a primary key and does not select too many rows to process \(hundreds of thousands\), and does not use too many columns, we can expect less than 50 milliseconds of latency \(single digits of milliseconds in the best case\) if data is placed in the page cache. Otherwise, latency is calculated from the number of seeks. If you use rotating drives, for a system that is not overloaded, the latency is calculated by this formula: seek time \(10 ms\) \* number of columns queried \* number of data parts.

#### Throughput when processing a large quantity of short queries

Under the same conditions, ClickHouse can handle several hundred queries per second on a single server \(up to several thousand in the best case\). Since this scenario is not typical for analytical DBMSs, we recommend expecting a maximum of 100 queries per second.

#### Performance on data insertion

We recommend inserting data in packets of at least 1000 rows, or no more than a single request per second. When inserting to a MergeTree table from a tab-separated dump, the insertion speed will be from 50 to 200 MB/s. If the inserted rows are around 1 Kb in size, the speed will be from 50,000 to 200,000 rows per second. If the rows are small, the performance will be higher in rows per second \(on Yandex Banner System data -&gt; 500,000 rows per second, on Graphite data -&gt; 1,000,000 rows per second\). To improve performance, you can make multiple INSERT queries in parallel, and performance will increase linearly.  


  




