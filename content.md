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

1. A vector engine. All operations are written for vectors, instead of for separate values. This means you don’t need to call operations very often, and dispatching costs are negligible. Operation code contains an optimized internal cycle.
2. 一个向量引擎。所有的操作都是为向量而写的，而不是单独的值。这意味着你不需要经常调用操作，并且调度成本可以忽略不计。操作代码包含一个优化的内部循环。
3. Code generation. The code generated for the query has all the indirect calls in it.

This is not done in “normal” databases, because it doesn’t make sense when running simple queries. However, there are exceptions. For example, MemSQL uses code generation to reduce latency when processing SQL queries. \(For comparison, analytical DBMSs require optimization of throughput, not latency.\)

Note that for CPU efficiency, the query language must be declarative \(SQL or MDX\), or at least a vector \(J, K\). The query should only contain implicit loops, allowing for optimization.

