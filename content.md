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

3. 支持为有限数量的随机 key（而不是所有 key）运行聚合。在数据 key 分发的一定条件下，这将提供一个相当准确的结果，同时使用较少的资源。

#### 数据复制和支持副本上的数据完整性

使用异步多主复制。写入任何可用的副本后，数据将分发到所有剩余的副本。系统在不同的副本上保持相同的数据。数据在失败后自动恢复，或对复杂情况使用“开关”。有关更多信息，请参阅“数据复制”（Data replication）一节。

### ClickHouse 中可能被认为是缺点的特性

1. 没有事务
2. 对于聚合，查询结果的大小必须适合于单个服务器上的RAM。但是，查询的源数据量可能无限大
3. 缺乏完整的 `UPDATE` / `DELETE` 实现

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

YT 允许您存储单独的一组列。但 YT 不是一个真正的列式存储系统，因为系统没有固定长度的数据类型（所以你可以有效地存储一个数字而不是“垃圾”），而且没有矢量引擎。 YT 中的任务由流模式下的任意代码执行，因此无法进行充分优化（每台服务器每秒高达数亿行）。在 2014-2016 年，YT 将开发使用 Merge Tree，强类型值和类 SQL 语言支持的“动态表排序”功能。动态排序的表不适用于OLAP 任务，因为数据存储在行中。 YT 中的查询语言开发仍处于孵化阶段，导致它不能专注于这个功能。 YT 开发者正在考虑在OLTP 和 Key-Value 场景中使用动态排序的表。

### 性能

根据内部测试结果，ClickHouse 在可用于测试的同类系统中显示出最佳性能。这包括长查询的最高吞吐量和短查询的最低延迟。测试结果显示在此页面上。

#### 单个大型查询的吞吐量

吞吐量可以以每秒行数或兆字节每秒来衡量。如果数据放置在页面缓存（page cache）中，则在现代硬件上运行的一个不太复杂的查询，在单个服务器上以大约 2-10 GB/s 的未压缩数据的速度进行处理（对于最简单的情况，速度可能会达到 30 GB/s）。如果数据不在页面缓存中，则速度取决于磁盘子系统和数据压缩率。例如，如果磁盘子系统允许以 400 MB/s 读取数据，并且数据压缩率为3，则速度将在 1.2 GB/s 左右。要获得每秒行数的速度，请将查询中使用的列的总大小除以每秒字节数。例如，如果提取 10 个字节的列，速度将在每秒大约1到2亿行。

处理速度对于分布式处理几乎线性地增加，但是只有在聚合或排序产生的行数不太大的情况下。

#### 处理短查询时的延迟

如果一个查询使用一个主键，并且没有选择太多的行来处理（成千上万），并且不使用太多的列，如果数据被放置在页面缓存中，我们可以预计不到50毫秒的延迟（最好的情况下，单位为毫秒）。否则，延迟是根据搜索的数量来计算的。如果使用旋转驱动器，对于没有过载的系统，延迟时间通过以下公式计算：查找时间（10 ms）\* 查询的列数 \* 数据部分的数量（number of data parts）。

#### 处理大量短查询时的吞吐量

在相同的条件下，ClickHouse 可以在单个服务器上每秒处理几百个查询（在最好的情况下可以达到几千个）。由于这种情况对于分析DBMS 并不典型，因此我们建议每秒最多可以查询100个查询。

#### 数据插入性能

我们建议插入数据时，至少1000行/数据包，或者每秒不超过一个请求。从 tab 分隔转储插入到 MergeTree 表时，插入速度将从50到200 MB/s。如果插入的行大小约为1Kb，速度将从每秒 50,000 到 200,000 行。如果行很小，则每秒行数的性能表现会更高（Yandex Banner System数据 - &gt; 500,000行/秒，Graphite数据 -&gt; 1,000,000行/秒）。为了提高性能，可以并行执行多个 `INSERT` 查询，并且性能会线性增加。

## 入门

### 系统要求

这不是一个跨平台的系统。它需要 Linux Ubuntu Precise（12.04）或更新的x86\_64体系结构和 SSE 4.2指令集。要测试SSE 4.2支持，请执行以下操作

```
grep -q sse4_2 /proc/cpuinfo && echo "SSE 4.2 supported" || echo "SSE 4.2 not supported"
```

我们推荐使用 Ubuntu Trusty 或 Ubuntu Xenial 或 Ubuntu Precise。终端必须使用 UTF-8 编码（在Ubuntu中是默认的）。

### 安装

对于测试和开发，系统可以安装在单台服务器或台式计算机上。

#### 软件包安装

在 `/etc/apt/sources.list`\(或者在一个独立的 `/etc/apt/sources.list.d/clickhouse.list` 文件中\), 添加仓库:

```
deb http://repo.yandex.ru/clickhouse/trusty stable main
```

对于其他 Ubuntu 版本，请替换 `trusty` 为 `xenial` 或 `precise`。如果想使用最新的测试版 ClickHouse，请更换 `stable` 为 `testing`。

然后运行：

```
sudo apt-key adv --keyserver keyserver.ubuntu.com --recv E0C56BD4    # optional
sudo apt-get update
sudo apt-get install clickhouse-client clickhouse-server-common
```

您也可以从这里手动下载和安装软件包：

* [http://repo.yandex.ru/clickhouse/trusty/pool/main/c/clickhouse/](http://repo.yandex.ru/clickhouse/trusty/pool/main/c/clickhouse/)，
* [http://repo.yandex.ru/clickhouse/xenial/pool/main/c/clickhouse/](http://repo.yandex.ru/clickhouse/xenial/pool/main/c/clickhouse/)，
* [http://repo.yandex.ru/clickhouse/precise/pool/main/c/clickhouse/](http://repo.yandex.ru/clickhouse/precise/pool/main/c/clickhouse/)

ClickHouse 包含访问限制设置。它们位于 `users.xml` 文件（`config.xml` 旁边）。默认情况下，默认用户无需密码即可访问。请参阅“用户/默认/网络”。有关更多信息，请参阅“配置文件”一节。

#### 源码安装

构建时，请按照 build.md（适用于Linux）或 build\_osx.md（适用于Mac OS X）中的说明进行操作。

你可以编译软件包并安装它们。您也可以使用程序而不安装软件包。

```
Client: dbms/src/Client/
Server: dbms/src/Server/
```

对于服务端，创建一个数据目录，比如：

```
/opt/clickhouse/data/default/
/opt/clickhouse/metadata/default/
```

（在服务端配置中配置）为所需用户运行“chown”。

注意 Server 配置文件中的 log 文件路径 \(`src/dbms/src/Server/config.xml`\).

#### 其他安装方法

Docker image：[https://hub.docker.com/r/yandex/clickhouse-server/](https://hub.docker.com/r/yandex/clickhouse-server/)

RPM 包，for CentOS, RHEL：[https://github.com/Altinity/clickhouse-rpm-install](https://github.com/Altinity/clickhouse-rpm-install)

Gentoo overlay：[https://github.com/kmeaw/clickhouse-overlay](https://github.com/kmeaw/clickhouse-overlay)

### 启动

启动服务器 \(as a daemon\)，运行：

```
sudo service clickhouse-server start
```

可以在目录 `/var/log/clickhouse-server/` 查看日志文件

如果服务器没有启动，请检查 `/etc/clickhouse-server/config.xml` 文件中的配置

您也可以从控制台启动服务器：

```
clickhouse-server --config-file=/etc/clickhouse-server/config.xml
```

在这种情况下，日志将被打印到控制台，这在开发过程中很方便。如果配置文件位于当前目录中，则不需要指定 `--config-file` 参数。默认情况下，它使用 `./config.xml` 。

您可以使用命令行客户端连接到服务器：

```
clickhouse-client
```

默认参数表示使用用户“default”，无需密码即可连接 localhost:9000。客户端可以用于连接到远程服务器。例如：

```
clickhouse-client --host=example.com
```

有关更多信息，请参阅“命令行客户端”一节。

检查系统：

```
milovidov@hostname:~/work/metrica/src/dbms/src/Client$ ./clickhouse-client
ClickHouse client version 0.0.18749.
Connecting to localhost:9000.
Connected to ClickHouse server version 0.0.18749.

:) SELECT 1

SELECT 1

┌─1─┐
│ 1 │
└───┘

1 rows in set. Elapsed: 0.003 sec.

:)
```

恭喜，它工作了！

对于进一步的实验，您可以尝试使用以下示例数据集之一进行：

#### AMPLab Big Data Benchmark

参见：[https://amplab.cs.berkeley.edu/benchmark/](https://amplab.cs.berkeley.edu/benchmark/)

在 [https://aws.amazon.com](https://aws.amazon.com/) 注册免费账号 —— 你将需要信用卡，email，电话号码来创建新的 access key ：[https://console.aws.amazon.com/iam/home?nc2=h\_m\_sc\#security\_credential](https://console.aws.amazon.com/iam/home?nc2=h_m_sc#security_credential)

在 shell 中执行：

```
sudo apt-get install s3cmd
mkdir tiny; cd tiny;
s3cmd sync s3://big-data-benchmark/pavlo/text-deflate/tiny/ .
cd ..
mkdir 1node; cd 1node;
s3cmd sync s3://big-data-benchmark/pavlo/text-deflate/1node/ .
cd ..
mkdir 5nodes; cd 5nodes;
s3cmd sync s3://big-data-benchmark/pavlo/text-deflate/5nodes/ .
cd ..
```

执行这些 ClickHouse 查询：

```
CREATE TABLE rankings_tiny
(
    pageURL String,
    pageRank UInt32,
    avgDuration UInt32
) ENGINE = Log;

CREATE TABLE uservisits_tiny
(
    sourceIP String,
    destinationURL String,
    visitDate Date,
    adRevenue Float32,
    UserAgent String,
    cCode FixedString(3),
    lCode FixedString(6),
    searchWord String,
    duration UInt32
) ENGINE = MergeTree(visitDate, visitDate, 8192);

CREATE TABLE rankings_1node
(
    pageURL String,
    pageRank UInt32,
    avgDuration UInt32
) ENGINE = Log;

CREATE TABLE uservisits_1node
(
    sourceIP String,
    destinationURL String,
    visitDate Date,
    adRevenue Float32,
    UserAgent String,
    cCode FixedString(3),
    lCode FixedString(6),
    searchWord String,
    duration UInt32
) ENGINE = MergeTree(visitDate, visitDate, 8192);

CREATE TABLE rankings_5nodes_on_single
(
    pageURL String,
    pageRank UInt32,
    avgDuration UInt32
) ENGINE = Log;

CREATE TABLE uservisits_5nodes_on_single
(
    sourceIP String,
    destinationURL String,
    visitDate Date,
    adRevenue Float32,
    UserAgent String,
    cCode FixedString(3),
    lCode FixedString(6),
    searchWord String,
    duration UInt32
) ENGINE = MergeTree(visitDate, visitDate, 8192);
```

返回到 shell：

```
for i in tiny/rankings/*.deflate; do echo $i; zlib-flate -uncompress < $i | clickhouse-client --host=example-perftest01j --query="INSERT INTO rankings_tiny FORMAT CSV"; done
for i in tiny/uservisits/*.deflate; do echo $i; zlib-flate -uncompress < $i | clickhouse-client --host=example-perftest01j --query="INSERT INTO uservisits_tiny FORMAT CSV"; done
for i in 1node/rankings/*.deflate; do echo $i; zlib-flate -uncompress < $i | clickhouse-client --host=example-perftest01j --query="INSERT INTO rankings_1node FORMAT CSV"; done
for i in 1node/uservisits/*.deflate; do echo $i; zlib-flate -uncompress < $i | clickhouse-client --host=example-perftest01j --query="INSERT INTO uservisits_1node FORMAT CSV"; done
for i in 5nodes/rankings/*.deflate; do echo $i; zlib-flate -uncompress < $i | clickhouse-client --host=example-perftest01j --query="INSERT INTO rankings_5nodes_on_single FORMAT CSV"; done
for i in 5nodes/uservisits/*.deflate; do echo $i; zlib-flate -uncompress < $i | clickhouse-client --host=example-perftest01j --query="INSERT INTO uservisits_5nodes_on_single FORMAT CSV"; done
```

数据检索查询：

```
SELECT pageURL, pageRank FROM rankings_1node WHERE pageRank > 1000

SELECT substring(sourceIP, 1, 8), sum(adRevenue) FROM uservisits_1node GROUP BY substring(sourceIP, 1, 8)

SELECT
    sourceIP,
    sum(adRevenue) AS totalRevenue,
    avg(pageRank) AS pageRank
FROM rankings_1node ALL INNER JOIN
(
    SELECT
        sourceIP,
        destinationURL AS pageURL,
        adRevenue
    FROM uservisits_1node
    WHERE (visitDate > '1980-01-01') AND (visitDate < '1980-04-01')
) USING pageURL
GROUP BY sourceIP
ORDER BY totalRevenue DESC
LIMIT 1
```

#### Criteo Terabyte Click Logs

可从以下网址下载全部数据：[http://labs.criteo.com/downloads/download-terabyte-click-logs/](http://labs.criteo.com/downloads/download-terabyte-click-logs/)

创建 log 表：

```
CREATE TABLE criteo_log (date Date, clicked UInt8, int1 Int32, int2 Int32, int3 Int32, int4 Int32, int5 Int32, int6 Int32, int7 Int32, int8 Int32, int9 Int32, int10 Int32, int11 Int32, int12 Int32, int13 Int32, cat1 String, cat2 String, cat3 String, cat4 String, cat5 String, cat6 String, cat7 String, cat8 String, cat9 String, cat10 String, cat11 String, cat12 String, cat13 String, cat14 String, cat15 String, cat16 String, cat17 String, cat18 String, cat19 String, cat20 String, cat21 String, cat22 String, cat23 String, cat24 String, cat25 String, cat26 String) ENGINE = Log
```

加载数据：

```
for i in {00..23}; do echo $i; zcat datasets/criteo/day_${i#0}.gz | sed -r 's/^/2000-01-'${i/00/24}'\t/' | clickhouse-client --host=example-perftest01j --query="INSERT INTO criteo_log FORMAT TabSeparated"; done
```

为转换的数据创建表：

```
CREATE TABLE criteo
(
    date Date,
    clicked UInt8,
    int1 Int32,
    int2 Int32,
    int3 Int32,
    int4 Int32,
    int5 Int32,
    int6 Int32,
    int7 Int32,
    int8 Int32,
    int9 Int32,
    int10 Int32,
    int11 Int32,
    int12 Int32,
    int13 Int32,
    icat1 UInt32,
    icat2 UInt32,
    icat3 UInt32,
    icat4 UInt32,
    icat5 UInt32,
    icat6 UInt32,
    icat7 UInt32,
    icat8 UInt32,
    icat9 UInt32,
    icat10 UInt32,
    icat11 UInt32,
    icat12 UInt32,
    icat13 UInt32,
    icat14 UInt32,
    icat15 UInt32,
    icat16 UInt32,
    icat17 UInt32,
    icat18 UInt32,
    icat19 UInt32,
    icat20 UInt32,
    icat21 UInt32,
    icat22 UInt32,
    icat23 UInt32,
    icat24 UInt32,
    icat25 UInt32,
    icat26 UInt32
) ENGINE = MergeTree(date, intHash32(icat1), (date, intHash32(icat1)), 8192)
```

解析来自原始日志的数据并将其放入第二个表中：

```
INSERT INTO criteo SELECT date, clicked, int1, int2, int3, int4, int5, int6, int7, int8, int9, int10, int11, int12, int13, reinterpretAsUInt32(unhex(cat1)) AS icat1, reinterpretAsUInt32(unhex(cat2)) AS icat2, reinterpretAsUInt32(unhex(cat3)) AS icat3, reinterpretAsUInt32(unhex(cat4)) AS icat4, reinterpretAsUInt32(unhex(cat5)) AS icat5, reinterpretAsUInt32(unhex(cat6)) AS icat6, reinterpretAsUInt32(unhex(cat7)) AS icat7, reinterpretAsUInt32(unhex(cat8)) AS icat8, reinterpretAsUInt32(unhex(cat9)) AS icat9, reinterpretAsUInt32(unhex(cat10)) AS icat10, reinterpretAsUInt32(unhex(cat11)) AS icat11, reinterpretAsUInt32(unhex(cat12)) AS icat12, reinterpretAsUInt32(unhex(cat13)) AS icat13, reinterpretAsUInt32(unhex(cat14)) AS icat14, reinterpretAsUInt32(unhex(cat15)) AS icat15, reinterpretAsUInt32(unhex(cat16)) AS icat16, reinterpretAsUInt32(unhex(cat17)) AS icat17, reinterpretAsUInt32(unhex(cat18)) AS icat18, reinterpretAsUInt32(unhex(cat19)) AS icat19, reinterpretAsUInt32(unhex(cat20)) AS icat20, reinterpretAsUInt32(unhex(cat21)) AS icat21, reinterpretAsUInt32(unhex(cat22)) AS icat22, reinterpretAsUInt32(unhex(cat23)) AS icat23, reinterpretAsUInt32(unhex(cat24)) AS icat24, reinterpretAsUInt32(unhex(cat25)) AS icat25, reinterpretAsUInt32(unhex(cat26)) AS icat26 FROM criteo_log;

DROP TABLE criteo_log;
```

#### NYC Taxi Dataset

如何从原始数据创建数据集

从 [https://github.com/toddwschneider/nyc-taxi-data](https://github.com/toddwschneider/nyc-taxi-data) 和 [http://tech.marksblogg.com/billion-nyc-taxi-rides-redshift.html](http://tech.marksblogg.com/billion-nyc-taxi-rides-redshift.html) 查看数据集和加载指令的描述。

数据将下载到约 227 GB的未压缩的CSV文件。 1GB 连接大约需要一个小时。（从 s3.amazonaws.com 并行下载至少一个千兆的一半。）有些文件可能不完整地下载。看看可疑的文件大小，并重复下载不完整的文件。

某些文件包含损坏的行。要更正它们，请运行：

```
sed -E '/(.*,){18,}/d' data/yellow_tripdata_2010-02.csv > data/yellow_tripdata_2010-02.csv_
sed -E '/(.*,){18,}/d' data/yellow_tripdata_2010-03.csv > data/yellow_tripdata_2010-03.csv_
mv data/yellow_tripdata_2010-02.csv_ data/yellow_tripdata_2010-02.csv
mv data/yellow_tripdata_2010-03.csv_ data/yellow_tripdata_2010-03.csv
```

那么数据必须在 PostgreSQL 里预处理。它将做多边形点查找（地图指向纽约地区），最后将所有数据 JOIN 到单个非标准化的水平的表。您必须安装支持 PostGIS 的 PostgreSQL。

运行 `initialize_database.sh`时，请小心并手动检查，以确保所有表成功创建。

在 PostgreSQL 里处理每个月的黄色出租车数据大概需要 20-30 分钟，总共大约需要48小时。

检查加载行的确切数量：

```
time psql nyc-taxi-data -c "SELECT count(*) FROM trips;"
   count
------------
 1298979494
(1 row)

real    7m9.164s
```

（这是 Mark Litwintschik 在一系列博客文章中报告的略多于11亿行） 

PostgreSQL 中的数据需要370 GB（346 GiB）。 

从PostgreSQL 导出数据：

```
COPY
(
    SELECT trips.id,
           trips.vendor_id,
           trips.pickup_datetime,
           trips.dropoff_datetime,
           trips.store_and_fwd_flag,
           trips.rate_code_id,
           trips.pickup_longitude,
           trips.pickup_latitude,
           trips.dropoff_longitude,
           trips.dropoff_latitude,
           trips.passenger_count,
           trips.trip_distance,
           trips.fare_amount,
           trips.extra,
           trips.mta_tax,
           trips.tip_amount,
           trips.tolls_amount,
           trips.ehail_fee,
           trips.improvement_surcharge,
           trips.total_amount,
           trips.payment_type,
           trips.trip_type,
           trips.pickup,
           trips.dropoff,

           cab_types.type cab_type,

           weather.precipitation_tenths_of_mm rain,
           weather.snow_depth_mm,
           weather.snowfall_mm,
           weather.max_temperature_tenths_degrees_celsius max_temp,
           weather.min_temperature_tenths_degrees_celsius min_temp,
           weather.average_wind_speed_tenths_of_meters_per_second wind,

           pick_up.gid pickup_nyct2010_gid,
           pick_up.ctlabel pickup_ctlabel,
           pick_up.borocode pickup_borocode,
           pick_up.boroname pickup_boroname,
           pick_up.ct2010 pickup_ct2010,
           pick_up.boroct2010 pickup_boroct2010,
           pick_up.cdeligibil pickup_cdeligibil,
           pick_up.ntacode pickup_ntacode,
           pick_up.ntaname pickup_ntaname,
           pick_up.puma pickup_puma,

           drop_off.gid dropoff_nyct2010_gid,
           drop_off.ctlabel dropoff_ctlabel,
           drop_off.borocode dropoff_borocode,
           drop_off.boroname dropoff_boroname,
           drop_off.ct2010 dropoff_ct2010,
           drop_off.boroct2010 dropoff_boroct2010,
           drop_off.cdeligibil dropoff_cdeligibil,
           drop_off.ntacode dropoff_ntacode,
           drop_off.ntaname dropoff_ntaname,
           drop_off.puma dropoff_puma
    FROM trips
    LEFT JOIN cab_types
        ON trips.cab_type_id = cab_types.id
    LEFT JOIN central_park_weather_observations_raw weather
        ON weather.date = trips.pickup_datetime::date
    LEFT JOIN nyct2010 pick_up
        ON pick_up.gid = trips.pickup_nyct2010_gid
    LEFT JOIN nyct2010 drop_off
        ON drop_off.gid = trips.dropoff_nyct2010_gid
) TO '/opt/milovidov/nyc-taxi-data/trips.tsv';
```

转储速度约为50 MB /秒。在创建转储时，PostgreSQL 以大约 28 MB/s 的速度从磁盘读取数据。大约需要5个小时。生成的 tsv 文件是 590612904969字节。 

在 ClickHouse 中创建临时表：

```
CREATE TABLE trips
(
trip_id                 UInt32,
vendor_id               String,
pickup_datetime         DateTime,
dropoff_datetime        Nullable(DateTime),
store_and_fwd_flag      Nullable(FixedString(1)),
rate_code_id            Nullable(UInt8),
pickup_longitude        Nullable(Float64),
pickup_latitude         Nullable(Float64),
dropoff_longitude       Nullable(Float64),
dropoff_latitude        Nullable(Float64),
passenger_count         Nullable(UInt8),
trip_distance           Nullable(Float64),
fare_amount             Nullable(Float32),
extra                   Nullable(Float32),
mta_tax                 Nullable(Float32),
tip_amount              Nullable(Float32),
tolls_amount            Nullable(Float32),
ehail_fee               Nullable(Float32),
improvement_surcharge   Nullable(Float32),
total_amount            Nullable(Float32),
payment_type            Nullable(String),
trip_type               Nullable(UInt8),
pickup                  Nullable(String),
dropoff                 Nullable(String),
cab_type                Nullable(String),
precipitation           Nullable(UInt8),
snow_depth              Nullable(UInt8),
snowfall                Nullable(UInt8),
max_temperature         Nullable(UInt8),
min_temperature         Nullable(UInt8),
average_wind_speed      Nullable(UInt8),
pickup_nyct2010_gid     Nullable(UInt8),
pickup_ctlabel          Nullable(String),
pickup_borocode         Nullable(UInt8),
pickup_boroname         Nullable(String),
pickup_ct2010           Nullable(String),
pickup_boroct2010       Nullable(String),
pickup_cdeligibil       Nullable(FixedString(1)),
pickup_ntacode          Nullable(String),
pickup_ntaname          Nullable(String),
pickup_puma             Nullable(String),
dropoff_nyct2010_gid    Nullable(UInt8),
dropoff_ctlabel         Nullable(String),
dropoff_borocode        Nullable(UInt8),
dropoff_boroname        Nullable(String),
dropoff_ct2010          Nullable(String),
dropoff_boroct2010      Nullable(String),
dropoff_cdeligibil      Nullable(String),
dropoff_ntacode         Nullable(String),
dropoff_ntaname         Nullable(String),
dropoff_puma            Nullable(String)
) ENGINE = Log;
```

这个表需要为字段选择更好的数据类型，如果可能的话，去除 NULL 类型。

```
time clickhouse-client --query="INSERT INTO trips FORMAT TabSeparated" < trips.tsv

real    75m56.214s
```

数据以 112-140 MB/s 读取。在单线程中将数据加载到日志表中花费了76分钟。日志表中的数据需要142 GB。

（你也可以直接从 Postgres 导入数据，使用`COPY...TO PROGRAM`）

可悲的是，所有天气相关的字段（precipitation... average\_wind\_speed）都是 NULL。我们将从最终数据集中省略它们。 

第一次，我们将在单个服务器上创建一个表。稍后我们将分发这个表。创建并填充最终的表：

```
CREATE TABLE trips_mergetree
ENGINE = MergeTree(pickup_date, pickup_datetime, 8192)
AS SELECT

trip_id,
CAST(vendor_id AS Enum8('1' = 1, '2' = 2, 'CMT' = 3, 'VTS' = 4, 'DDS' = 5, 'B02512' = 10, 'B02598' = 11, 'B02617' = 12, 'B02682' = 13, 'B02764' = 14)) AS vendor_id,
toDate(pickup_datetime) AS pickup_date,
ifNull(pickup_datetime, toDateTime(0)) AS pickup_datetime,
toDate(dropoff_datetime) AS dropoff_date,
ifNull(dropoff_datetime, toDateTime(0)) AS dropoff_datetime,
assumeNotNull(store_and_fwd_flag) IN ('Y', '1', '2') AS store_and_fwd_flag,
assumeNotNull(rate_code_id) AS rate_code_id,
assumeNotNull(pickup_longitude) AS pickup_longitude,
assumeNotNull(pickup_latitude) AS pickup_latitude,
assumeNotNull(dropoff_longitude) AS dropoff_longitude,
assumeNotNull(dropoff_latitude) AS dropoff_latitude,
assumeNotNull(passenger_count) AS passenger_count,
assumeNotNull(trip_distance) AS trip_distance,
assumeNotNull(fare_amount) AS fare_amount,
assumeNotNull(extra) AS extra,
assumeNotNull(mta_tax) AS mta_tax,
assumeNotNull(tip_amount) AS tip_amount,
assumeNotNull(tolls_amount) AS tolls_amount,
assumeNotNull(ehail_fee) AS ehail_fee,
assumeNotNull(improvement_surcharge) AS improvement_surcharge,
assumeNotNull(total_amount) AS total_amount,
CAST((assumeNotNull(payment_type) AS pt) IN ('CSH', 'CASH', 'Cash', 'CAS', 'Cas', '1') ? 'CSH' : (pt IN ('CRD', 'Credit', 'Cre', 'CRE', 'CREDIT', '2') ? 'CRE' : (pt IN ('NOC', 'No Charge', 'No', '3') ? 'NOC' : (pt IN ('DIS', 'Dispute', 'Dis', '4') ? 'DIS' : 'UNK'))) AS Enum8('CSH' = 1, 'CRE' = 2, 'UNK' = 0, 'NOC' = 3, 'DIS' = 4)) AS payment_type_,
assumeNotNull(trip_type) AS trip_type,
ifNull(toFixedString(unhex(pickup), 25), toFixedString('', 25)) AS pickup,
ifNull(toFixedString(unhex(dropoff), 25), toFixedString('', 25)) AS dropoff,
CAST(assumeNotNull(cab_type) AS Enum8('yellow' = 1, 'green' = 2, 'uber' = 3)) AS cab_type,

assumeNotNull(pickup_nyct2010_gid) AS pickup_nyct2010_gid,
toFloat32(ifNull(pickup_ctlabel, '0')) AS pickup_ctlabel,
assumeNotNull(pickup_borocode) AS pickup_borocode,
CAST(assumeNotNull(pickup_boroname) AS Enum8('Manhattan' = 1, 'Queens' = 4, 'Brooklyn' = 3, '' = 0, 'Bronx' = 2, 'Staten Island' = 5)) AS pickup_boroname,
toFixedString(ifNull(pickup_ct2010, '000000'), 6) AS pickup_ct2010,
toFixedString(ifNull(pickup_boroct2010, '0000000'), 7) AS pickup_boroct2010,
CAST(assumeNotNull(ifNull(pickup_cdeligibil, ' ')) AS Enum8(' ' = 0, 'E' = 1, 'I' = 2)) AS pickup_cdeligibil,
toFixedString(ifNull(pickup_ntacode, '0000'), 4) AS pickup_ntacode,

CAST(assumeNotNull(pickup_ntaname) AS Enum16('' = 0, 'Airport' = 1, 'Allerton-Pelham Gardens' = 2, 'Annadale-Huguenot-Prince\'s Bay-Eltingville' = 3, 'Arden Heights' = 4, 'Astoria' = 5, 'Auburndale' = 6, 'Baisley Park' = 7, 'Bath Beach' = 8, 'Battery Park City-Lower Manhattan' = 9, 'Bay Ridge' = 10, 'Bayside-Bayside Hills' = 11, 'Bedford' = 12, 'Bedford Park-Fordham North' = 13, 'Bellerose' = 14, 'Belmont' = 15, 'Bensonhurst East' = 16, 'Bensonhurst West' = 17, 'Borough Park' = 18, 'Breezy Point-Belle Harbor-Rockaway Park-Broad Channel' = 19, 'Briarwood-Jamaica Hills' = 20, 'Brighton Beach' = 21, 'Bronxdale' = 22, 'Brooklyn Heights-Cobble Hill' = 23, 'Brownsville' = 24, 'Bushwick North' = 25, 'Bushwick South' = 26, 'Cambria Heights' = 27, 'Canarsie' = 28, 'Carroll Gardens-Columbia Street-Red Hook' = 29, 'Central Harlem North-Polo Grounds' = 30, 'Central Harlem South' = 31, 'Charleston-Richmond Valley-Tottenville' = 32, 'Chinatown' = 33, 'Claremont-Bathgate' = 34, 'Clinton' = 35, 'Clinton Hill' = 36, 'Co-op City' = 37, 'College Point' = 38, 'Corona' = 39, 'Crotona Park East' = 40, 'Crown Heights North' = 41, 'Crown Heights South' = 42, 'Cypress Hills-City Line' = 43, 'DUMBO-Vinegar Hill-Downtown Brooklyn-Boerum Hill' = 44, 'Douglas Manor-Douglaston-Little Neck' = 45, 'Dyker Heights' = 46, 'East Concourse-Concourse Village' = 47, 'East Elmhurst' = 48, 'East Flatbush-Farragut' = 49, 'East Flushing' = 50, 'East Harlem North' = 51, 'East Harlem South' = 52, 'East New York' = 53, 'East New York (Pennsylvania Ave)' = 54, 'East Tremont' = 55, 'East Village' = 56, 'East Williamsburg' = 57, 'Eastchester-Edenwald-Baychester' = 58, 'Elmhurst' = 59, 'Elmhurst-Maspeth' = 60, 'Erasmus' = 61, 'Far Rockaway-Bayswater' = 62, 'Flatbush' = 63, 'Flatlands' = 64, 'Flushing' = 65, 'Fordham South' = 66, 'Forest Hills' = 67, 'Fort Greene' = 68, 'Fresh Meadows-Utopia' = 69, 'Ft. Totten-Bay Terrace-Clearview' = 70, 'Georgetown-Marine Park-Bergen Beach-Mill Basin' = 71, 'Glen Oaks-Floral Park-New Hyde Park' = 72, 'Glendale' = 73, 'Gramercy' = 74, 'Grasmere-Arrochar-Ft. Wadsworth' = 75, 'Gravesend' = 76, 'Great Kills' = 77, 'Greenpoint' = 78, 'Grymes Hill-Clifton-Fox Hills' = 79, 'Hamilton Heights' = 80, 'Hammels-Arverne-Edgemere' = 81, 'Highbridge' = 82, 'Hollis' = 83, 'Homecrest' = 84, 'Hudson Yards-Chelsea-Flatiron-Union Square' = 85, 'Hunters Point-Sunnyside-West Maspeth' = 86, 'Hunts Point' = 87, 'Jackson Heights' = 88, 'Jamaica' = 89, 'Jamaica Estates-Holliswood' = 90, 'Kensington-Ocean Parkway' = 91, 'Kew Gardens' = 92, 'Kew Gardens Hills' = 93, 'Kingsbridge Heights' = 94, 'Laurelton' = 95, 'Lenox Hill-Roosevelt Island' = 96, 'Lincoln Square' = 97, 'Lindenwood-Howard Beach' = 98, 'Longwood' = 99, 'Lower East Side' = 100, 'Madison' = 101, 'Manhattanville' = 102, 'Marble Hill-Inwood' = 103, 'Mariner\'s Harbor-Arlington-Port Ivory-Graniteville' = 104, 'Maspeth' = 105, 'Melrose South-Mott Haven North' = 106, 'Middle Village' = 107, 'Midtown-Midtown South' = 108, 'Midwood' = 109, 'Morningside Heights' = 110, 'Morrisania-Melrose' = 111, 'Mott Haven-Port Morris' = 112, 'Mount Hope' = 113, 'Murray Hill' = 114, 'Murray Hill-Kips Bay' = 115, 'New Brighton-Silver Lake' = 116, 'New Dorp-Midland Beach' = 117, 'New Springville-Bloomfield-Travis' = 118, 'North Corona' = 119, 'North Riverdale-Fieldston-Riverdale' = 120, 'North Side-South Side' = 121, 'Norwood' = 122, 'Oakland Gardens' = 123, 'Oakwood-Oakwood Beach' = 124, 'Ocean Hill' = 125, 'Ocean Parkway South' = 126, 'Old Astoria' = 127, 'Old Town-Dongan Hills-South Beach' = 128, 'Ozone Park' = 129, 'Park Slope-Gowanus' = 130, 'Parkchester' = 131, 'Pelham Bay-Country Club-City Island' = 132, 'Pelham Parkway' = 133, 'Pomonok-Flushing Heights-Hillcrest' = 134, 'Port Richmond' = 135, 'Prospect Heights' = 136, 'Prospect Lefferts Gardens-Wingate' = 137, 'Queens Village' = 138, 'Queensboro Hill' = 139, 'Queensbridge-Ravenswood-Long Island City' = 140, 'Rego Park' = 141, 'Richmond Hill' = 142, 'Ridgewood' = 143, 'Rikers Island' = 144, 'Rosedale' = 145, 'Rossville-Woodrow' = 146, 'Rugby-Remsen Village' = 147, 'Schuylerville-Throgs Neck-Edgewater Park' = 148, 'Seagate-Coney Island' = 149, 'Sheepshead Bay-Gerritsen Beach-Manhattan Beach' = 150, 'SoHo-TriBeCa-Civic Center-Little Italy' = 151, 'Soundview-Bruckner' = 152, 'Soundview-Castle Hill-Clason Point-Harding Park' = 153, 'South Jamaica' = 154, 'South Ozone Park' = 155, 'Springfield Gardens North' = 156, 'Springfield Gardens South-Brookville' = 157, 'Spuyten Duyvil-Kingsbridge' = 158, 'St. Albans' = 159, 'Stapleton-Rosebank' = 160, 'Starrett City' = 161, 'Steinway' = 162, 'Stuyvesant Heights' = 163, 'Stuyvesant Town-Cooper Village' = 164, 'Sunset Park East' = 165, 'Sunset Park West' = 166, 'Todt Hill-Emerson Hill-Heartland Village-Lighthouse Hill' = 167, 'Turtle Bay-East Midtown' = 168, 'University Heights-Morris Heights' = 169, 'Upper East Side-Carnegie Hill' = 170, 'Upper West Side' = 171, 'Van Cortlandt Village' = 172, 'Van Nest-Morris Park-Westchester Square' = 173, 'Washington Heights North' = 174, 'Washington Heights South' = 175, 'West Brighton' = 176, 'West Concourse' = 177, 'West Farms-Bronx River' = 178, 'West New Brighton-New Brighton-St. George' = 179, 'West Village' = 180, 'Westchester-Unionport' = 181, 'Westerleigh' = 182, 'Whitestone' = 183, 'Williamsbridge-Olinville' = 184, 'Williamsburg' = 185, 'Windsor Terrace' = 186, 'Woodhaven' = 187, 'Woodlawn-Wakefield' = 188, 'Woodside' = 189, 'Yorkville' = 190, 'park-cemetery-etc-Bronx' = 191, 'park-cemetery-etc-Brooklyn' = 192, 'park-cemetery-etc-Manhattan' = 193, 'park-cemetery-etc-Queens' = 194, 'park-cemetery-etc-Staten Island' = 195)) AS pickup_ntaname,

toUInt16(ifNull(pickup_puma, '0')) AS pickup_puma,

assumeNotNull(dropoff_nyct2010_gid) AS dropoff_nyct2010_gid,
toFloat32(ifNull(dropoff_ctlabel, '0')) AS dropoff_ctlabel,
assumeNotNull(dropoff_borocode) AS dropoff_borocode,
CAST(assumeNotNull(dropoff_boroname) AS Enum8('Manhattan' = 1, 'Queens' = 4, 'Brooklyn' = 3, '' = 0, 'Bronx' = 2, 'Staten Island' = 5)) AS dropoff_boroname,
toFixedString(ifNull(dropoff_ct2010, '000000'), 6) AS dropoff_ct2010,
toFixedString(ifNull(dropoff_boroct2010, '0000000'), 7) AS dropoff_boroct2010,
CAST(assumeNotNull(ifNull(dropoff_cdeligibil, ' ')) AS Enum8(' ' = 0, 'E' = 1, 'I' = 2)) AS dropoff_cdeligibil,
toFixedString(ifNull(dropoff_ntacode, '0000'), 4) AS dropoff_ntacode,

CAST(assumeNotNull(dropoff_ntaname) AS Enum16('' = 0, 'Airport' = 1, 'Allerton-Pelham Gardens' = 2, 'Annadale-Huguenot-Prince\'s Bay-Eltingville' = 3, 'Arden Heights' = 4, 'Astoria' = 5, 'Auburndale' = 6, 'Baisley Park' = 7, 'Bath Beach' = 8, 'Battery Park City-Lower Manhattan' = 9, 'Bay Ridge' = 10, 'Bayside-Bayside Hills' = 11, 'Bedford' = 12, 'Bedford Park-Fordham North' = 13, 'Bellerose' = 14, 'Belmont' = 15, 'Bensonhurst East' = 16, 'Bensonhurst West' = 17, 'Borough Park' = 18, 'Breezy Point-Belle Harbor-Rockaway Park-Broad Channel' = 19, 'Briarwood-Jamaica Hills' = 20, 'Brighton Beach' = 21, 'Bronxdale' = 22, 'Brooklyn Heights-Cobble Hill' = 23, 'Brownsville' = 24, 'Bushwick North' = 25, 'Bushwick South' = 26, 'Cambria Heights' = 27, 'Canarsie' = 28, 'Carroll Gardens-Columbia Street-Red Hook' = 29, 'Central Harlem North-Polo Grounds' = 30, 'Central Harlem South' = 31, 'Charleston-Richmond Valley-Tottenville' = 32, 'Chinatown' = 33, 'Claremont-Bathgate' = 34, 'Clinton' = 35, 'Clinton Hill' = 36, 'Co-op City' = 37, 'College Point' = 38, 'Corona' = 39, 'Crotona Park East' = 40, 'Crown Heights North' = 41, 'Crown Heights South' = 42, 'Cypress Hills-City Line' = 43, 'DUMBO-Vinegar Hill-Downtown Brooklyn-Boerum Hill' = 44, 'Douglas Manor-Douglaston-Little Neck' = 45, 'Dyker Heights' = 46, 'East Concourse-Concourse Village' = 47, 'East Elmhurst' = 48, 'East Flatbush-Farragut' = 49, 'East Flushing' = 50, 'East Harlem North' = 51, 'East Harlem South' = 52, 'East New York' = 53, 'East New York (Pennsylvania Ave)' = 54, 'East Tremont' = 55, 'East Village' = 56, 'East Williamsburg' = 57, 'Eastchester-Edenwald-Baychester' = 58, 'Elmhurst' = 59, 'Elmhurst-Maspeth' = 60, 'Erasmus' = 61, 'Far Rockaway-Bayswater' = 62, 'Flatbush' = 63, 'Flatlands' = 64, 'Flushing' = 65, 'Fordham South' = 66, 'Forest Hills' = 67, 'Fort Greene' = 68, 'Fresh Meadows-Utopia' = 69, 'Ft. Totten-Bay Terrace-Clearview' = 70, 'Georgetown-Marine Park-Bergen Beach-Mill Basin' = 71, 'Glen Oaks-Floral Park-New Hyde Park' = 72, 'Glendale' = 73, 'Gramercy' = 74, 'Grasmere-Arrochar-Ft. Wadsworth' = 75, 'Gravesend' = 76, 'Great Kills' = 77, 'Greenpoint' = 78, 'Grymes Hill-Clifton-Fox Hills' = 79, 'Hamilton Heights' = 80, 'Hammels-Arverne-Edgemere' = 81, 'Highbridge' = 82, 'Hollis' = 83, 'Homecrest' = 84, 'Hudson Yards-Chelsea-Flatiron-Union Square' = 85, 'Hunters Point-Sunnyside-West Maspeth' = 86, 'Hunts Point' = 87, 'Jackson Heights' = 88, 'Jamaica' = 89, 'Jamaica Estates-Holliswood' = 90, 'Kensington-Ocean Parkway' = 91, 'Kew Gardens' = 92, 'Kew Gardens Hills' = 93, 'Kingsbridge Heights' = 94, 'Laurelton' = 95, 'Lenox Hill-Roosevelt Island' = 96, 'Lincoln Square' = 97, 'Lindenwood-Howard Beach' = 98, 'Longwood' = 99, 'Lower East Side' = 100, 'Madison' = 101, 'Manhattanville' = 102, 'Marble Hill-Inwood' = 103, 'Mariner\'s Harbor-Arlington-Port Ivory-Graniteville' = 104, 'Maspeth' = 105, 'Melrose South-Mott Haven North' = 106, 'Middle Village' = 107, 'Midtown-Midtown South' = 108, 'Midwood' = 109, 'Morningside Heights' = 110, 'Morrisania-Melrose' = 111, 'Mott Haven-Port Morris' = 112, 'Mount Hope' = 113, 'Murray Hill' = 114, 'Murray Hill-Kips Bay' = 115, 'New Brighton-Silver Lake' = 116, 'New Dorp-Midland Beach' = 117, 'New Springville-Bloomfield-Travis' = 118, 'North Corona' = 119, 'North Riverdale-Fieldston-Riverdale' = 120, 'North Side-South Side' = 121, 'Norwood' = 122, 'Oakland Gardens' = 123, 'Oakwood-Oakwood Beach' = 124, 'Ocean Hill' = 125, 'Ocean Parkway South' = 126, 'Old Astoria' = 127, 'Old Town-Dongan Hills-South Beach' = 128, 'Ozone Park' = 129, 'Park Slope-Gowanus' = 130, 'Parkchester' = 131, 'Pelham Bay-Country Club-City Island' = 132, 'Pelham Parkway' = 133, 'Pomonok-Flushing Heights-Hillcrest' = 134, 'Port Richmond' = 135, 'Prospect Heights' = 136, 'Prospect Lefferts Gardens-Wingate' = 137, 'Queens Village' = 138, 'Queensboro Hill' = 139, 'Queensbridge-Ravenswood-Long Island City' = 140, 'Rego Park' = 141, 'Richmond Hill' = 142, 'Ridgewood' = 143, 'Rikers Island' = 144, 'Rosedale' = 145, 'Rossville-Woodrow' = 146, 'Rugby-Remsen Village' = 147, 'Schuylerville-Throgs Neck-Edgewater Park' = 148, 'Seagate-Coney Island' = 149, 'Sheepshead Bay-Gerritsen Beach-Manhattan Beach' = 150, 'SoHo-TriBeCa-Civic Center-Little Italy' = 151, 'Soundview-Bruckner' = 152, 'Soundview-Castle Hill-Clason Point-Harding Park' = 153, 'South Jamaica' = 154, 'South Ozone Park' = 155, 'Springfield Gardens North' = 156, 'Springfield Gardens South-Brookville' = 157, 'Spuyten Duyvil-Kingsbridge' = 158, 'St. Albans' = 159, 'Stapleton-Rosebank' = 160, 'Starrett City' = 161, 'Steinway' = 162, 'Stuyvesant Heights' = 163, 'Stuyvesant Town-Cooper Village' = 164, 'Sunset Park East' = 165, 'Sunset Park West' = 166, 'Todt Hill-Emerson Hill-Heartland Village-Lighthouse Hill' = 167, 'Turtle Bay-East Midtown' = 168, 'University Heights-Morris Heights' = 169, 'Upper East Side-Carnegie Hill' = 170, 'Upper West Side' = 171, 'Van Cortlandt Village' = 172, 'Van Nest-Morris Park-Westchester Square' = 173, 'Washington Heights North' = 174, 'Washington Heights South' = 175, 'West Brighton' = 176, 'West Concourse' = 177, 'West Farms-Bronx River' = 178, 'West New Brighton-New Brighton-St. George' = 179, 'West Village' = 180, 'Westchester-Unionport' = 181, 'Westerleigh' = 182, 'Whitestone' = 183, 'Williamsbridge-Olinville' = 184, 'Williamsburg' = 185, 'Windsor Terrace' = 186, 'Woodhaven' = 187, 'Woodlawn-Wakefield' = 188, 'Woodside' = 189, 'Yorkville' = 190, 'park-cemetery-etc-Bronx' = 191, 'park-cemetery-etc-Brooklyn' = 192, 'park-cemetery-etc-Manhattan' = 193, 'park-cemetery-etc-Queens' = 194, 'park-cemetery-etc-Staten Island' = 195)) AS dropoff_ntaname,

toUInt16(ifNull(dropoff_puma, '0')) AS dropoff_puma

FROM trips
```

这在 3030s 内以约 428000行/s 完成。如果你想要更快的加载时间，你可以使用 `Log` 引擎而不是 `MergeTree` 来创建表。在这种情况下，加载低于 200s。 

表使用了126GB 的磁盘空间。

```
:) SELECT formatReadableSize(sum(bytes)) FROM system.parts WHERE table = 'trips_mergetree' AND active

SELECT formatReadableSize(sum(bytes))
FROM system.parts
WHERE (table = 'trips_mergetree') AND active

┌─formatReadableSize(sum(bytes))─┐
│ 126.18 GiB                     │
└────────────────────────────────┘
```

顺便说一句，你可以为 MergeTree 表运行 OPTIMIZE。但这不是必要的，一切都会好起来的。 

在单个服务器上的结果 

Q1：

```
SELECT cab_type, count(*) FROM trips_mergetree GROUP BY cab_type
```

0.490 sec.

Q2：

```
SELECT passenger_count, avg(total_amount) FROM trips_mergetree GROUP BY passenger_count
```

1.224 sec.

Q3：

```
SELECT passenger_count, toYear(pickup_date) AS year, count(*) FROM trips_mergetree GROUP BY passenger_count, year
```

2.104 sec.

Q4：

```
SELECT passenger_count, toYear(pickup_date) AS year, round(trip_distance) AS distance, count(*)
FROM trips_mergetree
GROUP BY passenger_count, year, distance
ORDER BY year, count(*) DESC
```

3.593 sec.

服务器是： 

Two-socket Intel\(R\) Xeon\(R\) CPU E5-2650 v2 @ 2.60GHz，总共16个物理内核，128 GB RAM，RAID-5级别的8x6 TB硬盘 

查询时间是三次运行中最好的。实际上，从第二次运行开始，查询将从 OS 页面缓存中读取数据。不会发生进一步的缓存：在每次运行中读取和处理数据。

在有 3个服务器的群集上创建表 

在每台服务器上：

```
CREATE TABLE default.trips_mergetree_third ( trip_id UInt32,  vendor_id Enum8('1' = 1, '2' = 2, 'CMT' = 3, 'VTS' = 4, 'DDS' = 5, 'B02512' = 10, 'B02598' = 11, 'B02617' = 12, 'B02682' = 13, 'B02764' = 14),  pickup_date Date,  pickup_datetime DateTime,  dropoff_date Date,  dropoff_datetime DateTime,  store_and_fwd_flag UInt8,  rate_code_id UInt8,  pickup_longitude Float64,  pickup_latitude Float64,  dropoff_longitude Float64,  dropoff_latitude Float64,  passenger_count UInt8,  trip_distance Float64,  fare_amount Float32,  extra Float32,  mta_tax Float32,  tip_amount Float32,  tolls_amount Float32,  ehail_fee Float32,  improvement_surcharge Float32,  total_amount Float32,  payment_type_ Enum8('UNK' = 0, 'CSH' = 1, 'CRE' = 2, 'NOC' = 3, 'DIS' = 4),  trip_type UInt8,  pickup FixedString(25),  dropoff FixedString(25),  cab_type Enum8('yellow' = 1, 'green' = 2, 'uber' = 3),  pickup_nyct2010_gid UInt8,  pickup_ctlabel Float32,  pickup_borocode UInt8,  pickup_boroname Enum8('' = 0, 'Manhattan' = 1, 'Bronx' = 2, 'Brooklyn' = 3, 'Queens' = 4, 'Staten Island' = 5),  pickup_ct2010 FixedString(6),  pickup_boroct2010 FixedString(7),  pickup_cdeligibil Enum8(' ' = 0, 'E' = 1, 'I' = 2),  pickup_ntacode FixedString(4),  pickup_ntaname Enum16('' = 0, 'Airport' = 1, 'Allerton-Pelham Gardens' = 2, 'Annadale-Huguenot-Prince\'s Bay-Eltingville' = 3, 'Arden Heights' = 4, 'Astoria' = 5, 'Auburndale' = 6, 'Baisley Park' = 7, 'Bath Beach' = 8, 'Battery Park City-Lower Manhattan' = 9, 'Bay Ridge' = 10, 'Bayside-Bayside Hills' = 11, 'Bedford' = 12, 'Bedford Park-Fordham North' = 13, 'Bellerose' = 14, 'Belmont' = 15, 'Bensonhurst East' = 16, 'Bensonhurst West' = 17, 'Borough Park' = 18, 'Breezy Point-Belle Harbor-Rockaway Park-Broad Channel' = 19, 'Briarwood-Jamaica Hills' = 20, 'Brighton Beach' = 21, 'Bronxdale' = 22, 'Brooklyn Heights-Cobble Hill' = 23, 'Brownsville' = 24, 'Bushwick North' = 25, 'Bushwick South' = 26, 'Cambria Heights' = 27, 'Canarsie' = 28, 'Carroll Gardens-Columbia Street-Red Hook' = 29, 'Central Harlem North-Polo Grounds' = 30, 'Central Harlem South' = 31, 'Charleston-Richmond Valley-Tottenville' = 32, 'Chinatown' = 33, 'Claremont-Bathgate' = 34, 'Clinton' = 35, 'Clinton Hill' = 36, 'Co-op City' = 37, 'College Point' = 38, 'Corona' = 39, 'Crotona Park East' = 40, 'Crown Heights North' = 41, 'Crown Heights South' = 42, 'Cypress Hills-City Line' = 43, 'DUMBO-Vinegar Hill-Downtown Brooklyn-Boerum Hill' = 44, 'Douglas Manor-Douglaston-Little Neck' = 45, 'Dyker Heights' = 46, 'East Concourse-Concourse Village' = 47, 'East Elmhurst' = 48, 'East Flatbush-Farragut' = 49, 'East Flushing' = 50, 'East Harlem North' = 51, 'East Harlem South' = 52, 'East New York' = 53, 'East New York (Pennsylvania Ave)' = 54, 'East Tremont' = 55, 'East Village' = 56, 'East Williamsburg' = 57, 'Eastchester-Edenwald-Baychester' = 58, 'Elmhurst' = 59, 'Elmhurst-Maspeth' = 60, 'Erasmus' = 61, 'Far Rockaway-Bayswater' = 62, 'Flatbush' = 63, 'Flatlands' = 64, 'Flushing' = 65, 'Fordham South' = 66, 'Forest Hills' = 67, 'Fort Greene' = 68, 'Fresh Meadows-Utopia' = 69, 'Ft. Totten-Bay Terrace-Clearview' = 70, 'Georgetown-Marine Park-Bergen Beach-Mill Basin' = 71, 'Glen Oaks-Floral Park-New Hyde Park' = 72, 'Glendale' = 73, 'Gramercy' = 74, 'Grasmere-Arrochar-Ft. Wadsworth' = 75, 'Gravesend' = 76, 'Great Kills' = 77, 'Greenpoint' = 78, 'Grymes Hill-Clifton-Fox Hills' = 79, 'Hamilton Heights' = 80, 'Hammels-Arverne-Edgemere' = 81, 'Highbridge' = 82, 'Hollis' = 83, 'Homecrest' = 84, 'Hudson Yards-Chelsea-Flatiron-Union Square' = 85, 'Hunters Point-Sunnyside-West Maspeth' = 86, 'Hunts Point' = 87, 'Jackson Heights' = 88, 'Jamaica' = 89, 'Jamaica Estates-Holliswood' = 90, 'Kensington-Ocean Parkway' = 91, 'Kew Gardens' = 92, 'Kew Gardens Hills' = 93, 'Kingsbridge Heights' = 94, 'Laurelton' = 95, 'Lenox Hill-Roosevelt Island' = 96, 'Lincoln Square' = 97, 'Lindenwood-Howard Beach' = 98, 'Longwood' = 99, 'Lower East Side' = 100, 'Madison' = 101, 'Manhattanville' = 102, 'Marble Hill-Inwood' = 103, 'Mariner\'s Harbor-Arlington-Port Ivory-Graniteville' = 104, 'Maspeth' = 105, 'Melrose South-Mott Haven North' = 106, 'Middle Village' = 107, 'Midtown-Midtown South' = 108, 'Midwood' = 109, 'Morningside Heights' = 110, 'Morrisania-Melrose' = 111, 'Mott Haven-Port Morris' = 112, 'Mount Hope' = 113, 'Murray Hill' = 114, 'Murray Hill-Kips Bay' = 115, 'New Brighton-Silver Lake' = 116, 'New Dorp-Midland Beach' = 117, 'New Springville-Bloomfield-Travis' = 118, 'North Corona' = 119, 'North Riverdale-Fieldston-Riverdale' = 120, 'North Side-South Side' = 121, 'Norwood' = 122, 'Oakland Gardens' = 123, 'Oakwood-Oakwood Beach' = 124, 'Ocean Hill' = 125, 'Ocean Parkway South' = 126, 'Old Astoria' = 127, 'Old Town-Dongan Hills-South Beach' = 128, 'Ozone Park' = 129, 'Park Slope-Gowanus' = 130, 'Parkchester' = 131, 'Pelham Bay-Country Club-City Island' = 132, 'Pelham Parkway' = 133, 'Pomonok-Flushing Heights-Hillcrest' = 134, 'Port Richmond' = 135, 'Prospect Heights' = 136, 'Prospect Lefferts Gardens-Wingate' = 137, 'Queens Village' = 138, 'Queensboro Hill' = 139, 'Queensbridge-Ravenswood-Long Island City' = 140, 'Rego Park' = 141, 'Richmond Hill' = 142, 'Ridgewood' = 143, 'Rikers Island' = 144, 'Rosedale' = 145, 'Rossville-Woodrow' = 146, 'Rugby-Remsen Village' = 147, 'Schuylerville-Throgs Neck-Edgewater Park' = 148, 'Seagate-Coney Island' = 149, 'Sheepshead Bay-Gerritsen Beach-Manhattan Beach' = 150, 'SoHo-TriBeCa-Civic Center-Little Italy' = 151, 'Soundview-Bruckner' = 152, 'Soundview-Castle Hill-Clason Point-Harding Park' = 153, 'South Jamaica' = 154, 'South Ozone Park' = 155, 'Springfield Gardens North' = 156, 'Springfield Gardens South-Brookville' = 157, 'Spuyten Duyvil-Kingsbridge' = 158, 'St. Albans' = 159, 'Stapleton-Rosebank' = 160, 'Starrett City' = 161, 'Steinway' = 162, 'Stuyvesant Heights' = 163, 'Stuyvesant Town-Cooper Village' = 164, 'Sunset Park East' = 165, 'Sunset Park West' = 166, 'Todt Hill-Emerson Hill-Heartland Village-Lighthouse Hill' = 167, 'Turtle Bay-East Midtown' = 168, 'University Heights-Morris Heights' = 169, 'Upper East Side-Carnegie Hill' = 170, 'Upper West Side' = 171, 'Van Cortlandt Village' = 172, 'Van Nest-Morris Park-Westchester Square' = 173, 'Washington Heights North' = 174, 'Washington Heights South' = 175, 'West Brighton' = 176, 'West Concourse' = 177, 'West Farms-Bronx River' = 178, 'West New Brighton-New Brighton-St. George' = 179, 'West Village' = 180, 'Westchester-Unionport' = 181, 'Westerleigh' = 182, 'Whitestone' = 183, 'Williamsbridge-Olinville' = 184, 'Williamsburg' = 185, 'Windsor Terrace' = 186, 'Woodhaven' = 187, 'Woodlawn-Wakefield' = 188, 'Woodside' = 189, 'Yorkville' = 190, 'park-cemetery-etc-Bronx' = 191, 'park-cemetery-etc-Brooklyn' = 192, 'park-cemetery-etc-Manhattan' = 193, 'park-cemetery-etc-Queens' = 194, 'park-cemetery-etc-Staten Island' = 195),  pickup_puma UInt16,  dropoff_nyct2010_gid UInt8,  dropoff_ctlabel Float32,  dropoff_borocode UInt8,  dropoff_boroname Enum8('' = 0, 'Manhattan' = 1, 'Bronx' = 2, 'Brooklyn' = 3, 'Queens' = 4, 'Staten Island' = 5),  dropoff_ct2010 FixedString(6),  dropoff_boroct2010 FixedString(7),  dropoff_cdeligibil Enum8(' ' = 0, 'E' = 1, 'I' = 2),  dropoff_ntacode FixedString(4),  dropoff_ntaname Enum16('' = 0, 'Airport' = 1, 'Allerton-Pelham Gardens' = 2, 'Annadale-Huguenot-Prince\'s Bay-Eltingville' = 3, 'Arden Heights' = 4, 'Astoria' = 5, 'Auburndale' = 6, 'Baisley Park' = 7, 'Bath Beach' = 8, 'Battery Park City-Lower Manhattan' = 9, 'Bay Ridge' = 10, 'Bayside-Bayside Hills' = 11, 'Bedford' = 12, 'Bedford Park-Fordham North' = 13, 'Bellerose' = 14, 'Belmont' = 15, 'Bensonhurst East' = 16, 'Bensonhurst West' = 17, 'Borough Park' = 18, 'Breezy Point-Belle Harbor-Rockaway Park-Broad Channel' = 19, 'Briarwood-Jamaica Hills' = 20, 'Brighton Beach' = 21, 'Bronxdale' = 22, 'Brooklyn Heights-Cobble Hill' = 23, 'Brownsville' = 24, 'Bushwick North' = 25, 'Bushwick South' = 26, 'Cambria Heights' = 27, 'Canarsie' = 28, 'Carroll Gardens-Columbia Street-Red Hook' = 29, 'Central Harlem North-Polo Grounds' = 30, 'Central Harlem South' = 31, 'Charleston-Richmond Valley-Tottenville' = 32, 'Chinatown' = 33, 'Claremont-Bathgate' = 34, 'Clinton' = 35, 'Clinton Hill' = 36, 'Co-op City' = 37, 'College Point' = 38, 'Corona' = 39, 'Crotona Park East' = 40, 'Crown Heights North' = 41, 'Crown Heights South' = 42, 'Cypress Hills-City Line' = 43, 'DUMBO-Vinegar Hill-Downtown Brooklyn-Boerum Hill' = 44, 'Douglas Manor-Douglaston-Little Neck' = 45, 'Dyker Heights' = 46, 'East Concourse-Concourse Village' = 47, 'East Elmhurst' = 48, 'East Flatbush-Farragut' = 49, 'East Flushing' = 50, 'East Harlem North' = 51, 'East Harlem South' = 52, 'East New York' = 53, 'East New York (Pennsylvania Ave)' = 54, 'East Tremont' = 55, 'East Village' = 56, 'East Williamsburg' = 57, 'Eastchester-Edenwald-Baychester' = 58, 'Elmhurst' = 59, 'Elmhurst-Maspeth' = 60, 'Erasmus' = 61, 'Far Rockaway-Bayswater' = 62, 'Flatbush' = 63, 'Flatlands' = 64, 'Flushing' = 65, 'Fordham South' = 66, 'Forest Hills' = 67, 'Fort Greene' = 68, 'Fresh Meadows-Utopia' = 69, 'Ft. Totten-Bay Terrace-Clearview' = 70, 'Georgetown-Marine Park-Bergen Beach-Mill Basin' = 71, 'Glen Oaks-Floral Park-New Hyde Park' = 72, 'Glendale' = 73, 'Gramercy' = 74, 'Grasmere-Arrochar-Ft. Wadsworth' = 75, 'Gravesend' = 76, 'Great Kills' = 77, 'Greenpoint' = 78, 'Grymes Hill-Clifton-Fox Hills' = 79, 'Hamilton Heights' = 80, 'Hammels-Arverne-Edgemere' = 81, 'Highbridge' = 82, 'Hollis' = 83, 'Homecrest' = 84, 'Hudson Yards-Chelsea-Flatiron-Union Square' = 85, 'Hunters Point-Sunnyside-West Maspeth' = 86, 'Hunts Point' = 87, 'Jackson Heights' = 88, 'Jamaica' = 89, 'Jamaica Estates-Holliswood' = 90, 'Kensington-Ocean Parkway' = 91, 'Kew Gardens' = 92, 'Kew Gardens Hills' = 93, 'Kingsbridge Heights' = 94, 'Laurelton' = 95, 'Lenox Hill-Roosevelt Island' = 96, 'Lincoln Square' = 97, 'Lindenwood-Howard Beach' = 98, 'Longwood' = 99, 'Lower East Side' = 100, 'Madison' = 101, 'Manhattanville' = 102, 'Marble Hill-Inwood' = 103, 'Mariner\'s Harbor-Arlington-Port Ivory-Graniteville' = 104, 'Maspeth' = 105, 'Melrose South-Mott Haven North' = 106, 'Middle Village' = 107, 'Midtown-Midtown South' = 108, 'Midwood' = 109, 'Morningside Heights' = 110, 'Morrisania-Melrose' = 111, 'Mott Haven-Port Morris' = 112, 'Mount Hope' = 113, 'Murray Hill' = 114, 'Murray Hill-Kips Bay' = 115, 'New Brighton-Silver Lake' = 116, 'New Dorp-Midland Beach' = 117, 'New Springville-Bloomfield-Travis' = 118, 'North Corona' = 119, 'North Riverdale-Fieldston-Riverdale' = 120, 'North Side-South Side' = 121, 'Norwood' = 122, 'Oakland Gardens' = 123, 'Oakwood-Oakwood Beach' = 124, 'Ocean Hill' = 125, 'Ocean Parkway South' = 126, 'Old Astoria' = 127, 'Old Town-Dongan Hills-South Beach' = 128, 'Ozone Park' = 129, 'Park Slope-Gowanus' = 130, 'Parkchester' = 131, 'Pelham Bay-Country Club-City Island' = 132, 'Pelham Parkway' = 133, 'Pomonok-Flushing Heights-Hillcrest' = 134, 'Port Richmond' = 135, 'Prospect Heights' = 136, 'Prospect Lefferts Gardens-Wingate' = 137, 'Queens Village' = 138, 'Queensboro Hill' = 139, 'Queensbridge-Ravenswood-Long Island City' = 140, 'Rego Park' = 141, 'Richmond Hill' = 142, 'Ridgewood' = 143, 'Rikers Island' = 144, 'Rosedale' = 145, 'Rossville-Woodrow' = 146, 'Rugby-Remsen Village' = 147, 'Schuylerville-Throgs Neck-Edgewater Park' = 148, 'Seagate-Coney Island' = 149, 'Sheepshead Bay-Gerritsen Beach-Manhattan Beach' = 150, 'SoHo-TriBeCa-Civic Center-Little Italy' = 151, 'Soundview-Bruckner' = 152, 'Soundview-Castle Hill-Clason Point-Harding Park' = 153, 'South Jamaica' = 154, 'South Ozone Park' = 155, 'Springfield Gardens North' = 156, 'Springfield Gardens South-Brookville' = 157, 'Spuyten Duyvil-Kingsbridge' = 158, 'St. Albans' = 159, 'Stapleton-Rosebank' = 160, 'Starrett City' = 161, 'Steinway' = 162, 'Stuyvesant Heights' = 163, 'Stuyvesant Town-Cooper Village' = 164, 'Sunset Park East' = 165, 'Sunset Park West' = 166, 'Todt Hill-Emerson Hill-Heartland Village-Lighthouse Hill' = 167, 'Turtle Bay-East Midtown' = 168, 'University Heights-Morris Heights' = 169, 'Upper East Side-Carnegie Hill' = 170, 'Upper West Side' = 171, 'Van Cortlandt Village' = 172, 'Van Nest-Morris Park-Westchester Square' = 173, 'Washington Heights North' = 174, 'Washington Heights South' = 175, 'West Brighton' = 176, 'West Concourse' = 177, 'West Farms-Bronx River' = 178, 'West New Brighton-New Brighton-St. George' = 179, 'West Village' = 180, 'Westchester-Unionport' = 181, 'Westerleigh' = 182, 'Whitestone' = 183, 'Williamsbridge-Olinville' = 184, 'Williamsburg' = 185, 'Windsor Terrace' = 186, 'Woodhaven' = 187, 'Woodlawn-Wakefield' = 188, 'Woodside' = 189, 'Yorkville' = 190, 'park-cemetery-etc-Bronx' = 191, 'park-cemetery-etc-Brooklyn' = 192, 'park-cemetery-etc-Manhattan' = 193, 'park-cemetery-etc-Queens' = 194, 'park-cemetery-etc-Staten Island' = 195),  dropoff_puma UInt16) ENGINE = MergeTree(pickup_date, pickup_datetime, 8192)
```

在源服务器上：

```
CREATE TABLE trips_mergetree_x3 AS trips_mergetree_third ENGINE = Distributed(perftest, default, trips_mergetree_third, rand())
```

这将分发数据：

```
INSERT INTO trips_mergetree_x3 SELECT * FROM trips_mergetree
```

这将花费 2454s.

在 3 台服务器上：

Q1: 0.212 sec. Q2: 0.438 sec. Q3: 0.733 sec. Q4: 1.241 sec.

毫不奇怪，因为查询是可线性扩展的。

还有140个节点集群的结果：

Q1: 0.028 sec. Q2: 0.043 sec. Q3: 0.051 sec. Q4: 0.072 sec.

In that case, query execution speed is dominated by latency. We do queries from client located in Yandex datacenter in Mäntsälä \(Finland\) to cluster somewhere in Russia, that adds at least 20 ms of latency.

##### Summary





































