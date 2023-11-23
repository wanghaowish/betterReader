### 数据库发展历程

![image-20231123102605991](https://cdn.jsdelivr.net/gh/wanghaowish/picGo@main/img/202311231034966.png)

### 单机数据库发展

​      在数据库发展早期阶段，这两个需求其实不难满足，比如有很多优秀的商业数据库产品，如Oracle/DB2。在1990年之后，出现了开源数据库MySQL和PostgreSQL。这些数据库不断地提升单机实例性能，再加上遵循摩尔定律的硬件提升速度，往往能够很好地支撑业务发展。

​      随着互联网的不断普及特别是移动互联网的兴起，数据规模爆炸式增长，而硬件这些年的进步速度却在逐渐减慢，人们也在担心摩尔定律会失效。在此消彼长的情况下，单机数据库越来越难以满足用户需求，即使是将数据保存下来这个最基本的需求。

### 分布式NOSQL数据库发展

​      HBase本身并不存储数据，这里的Region仅是逻辑上的概念，数据还是以文件的形式存储在HDFS上，HBase并不关心副本个数、位置以及水平扩展问题，这些都依赖于HDFS实现。和BigTable一样，HBase提供行级的一致性，从CAP理论的角度来看，它是一个CP的系统，并且没有更进一步提供 ACID 的跨行事务，也是很遗憾。

​      HBase的优势在于通过扩展Region Server可以几乎线性提升系统的吞吐，及HDFS本身就具有的水平扩展能力，且整个系统成熟稳定。

​      但HBase依然有一些不足

- 首先，Hadoop使用Java开发，GC延迟是一个无法避免问题，这对系统的延迟造成一些影响。
- 另外，由于HBase本身并不存储数据，和HDFS之间的交互会多一层性能损耗。
- 第三，HBase和BigTable一样，并不支持跨行事务，所以在Google内部有团队开发了MegaStore、Percolator这些基于BigTable的事务层。Jeff Dean承认很后悔没有在BigTable中加入跨行事务，这也是Spanner出现的一个原因

### 分布式NewSQL数据库发展

​       2012~2013年Google 相继发表了Spanner和F1两套系统的论文，让业界第一次看到了关系模型和NoSQL的扩展性在一个大规模生产系统上融合的可能性。

​       Spanner 通过使用硬件设备（GPS时钟+原子钟）巧妙地解决时钟同步的问题，而在分布式系统里，时钟正是最让人头痛的问题。Spanner的强大之处在于即使两个数据中心隔得非常远，也能保证通过TrueTime API获取的时间误差在一个很小的范围内（10ms），并且不需要通讯。Spanner的底层仍然基于分布式文件系统，不过论文里也说是可以未来优化的点。

### OLAP、OLTP、HTAP

 数据处理大致可以分成两大类：

 **联机事务处理 OLTP（On-Line Transaction Processing）**

 OLTP 的场景如电商场景中加购物车、下单、支付等需要在原地进行大量insert、update、delete操作

 **联机分析处理 OLAP（On-Line Analytical Processing）**

 OLAP 场景通常是将数据批量导入后，进行任意维度的灵活探索、BI工具洞察、报表制作等

 **混合事务和分析处理 HTAP（Hybrid Transaction and Analytical Process）**

 同时支持在线事务处理与在线分析处理的融合型数据库。

### TiDB基本简介

​    目标是为用户提供一站式 OLTP (Online Transactional Processing)、OLAP (Online Analytical Processing)、HTAP 解决方案。TiDB 适合高可用、强一致要求较高、数据规模较大等各种应用场景。

​     **与传统的单机数据库相比，TiDB具有以下优势**：

- 纯分布式架构，拥有良好的扩展性，支持弹性的扩缩容
- 支持SQL，对外暴露MySQL的网络协议，并兼容大多数MySQL的语法，在大多数场景下可以直接替换MySQL
- 默认支持高可用，在少数副本失效的情况下，数据库本身能够自动进行数据修复和故障转移，对业务透明
- 支持ACID事务，对于一些有强一致需求的场景友好，例如：银行转账
- 具有丰富的工具链生态，覆盖数据迁移、同步、备份等多种场景

### TiDB五大核心特性

​     1.一键水平扩容或者缩容

 得益于 TiDB 存储计算分离的架构的设计，可按需对计算、存储分别进行在线扩容或者缩容，扩容或者缩容 过程中对应用运维人员透明。

 2.金融级高可用

 数据采用多副本存储，数据副本通过 Multi-Raft 协议同步事务日志，多数派写入成功事务才能提交，确保 数据强一致性且少数副本发生故障时不影响数据的可用性。可按需配置副本地理位置、副本数量等策略满足不 同容灾级别的要求。

 3.实时 HTAP

 提供行存储引擎 TiKV、列存储引擎 TiFlash 两款存储引擎，TiFlash 通过 Multi-Raft Learner 协议实时从 TiKV 复制数据，确保行存储引擎 TiKV 和列存储引擎 TiFlash 之间的数据强一致。TiKV、TiFlash 可按需部署 在不同的机器，解决 HTAP 资源隔离的问题。

 4.云原生的分布式数据库

 专为云而设计的分布式数据库，通过 TiDB Operator 可在公有云、私有云、混合云中实现部署工具化、自 动化。

 5.兼容 MySQL 5.7 协议和 MySQL 生态

 兼容 MySQL 5.7 协议、MySQL 常用的功能、MySQL 生态，应用无需或者修改少量代码即可从 MySQL 迁移到 TiDB。提供丰富的数据迁移工具帮助应用便捷完成数据迁移

### TiDB四大核心场景

 **1.对数据一致性及高可靠、系统高可用、可扩展性、容灾要求较高的金融行业属性的场景**

 众所周知，金融行业对数据一致性及高可靠、系统高可用、可扩展性、容灾要求较高。传统的解决方案是同城两个机房提供服务、异地一个机房提供数据容灾能力但不提供服务，此解决方案存在以下缺点：资源利用率低、维护成本高、RTO (Recovery Time Objective) 及 RPO (Recovery Point Objective) 无法真实达到企业所期望的值。TiDB 采用多副本 + Multi-Raft 协议的方式将数据调度到不同的机房、机架、机器，当部分机器出现故障时系统可自动进行切换，确保系统的 RTO 不大于 30s 及 RPO = 0。

 **2.对存储容量、可扩展性、并发要求较高的大量数据及高并发的OLTP场景**

 随着业务的高速发展，数据呈现爆炸性的增长，传统的单机数据库无法满足因数据爆炸性的增长对数据库的容量要求，可行方案是采用分库分表的中间件产品或者 NewSQL 数据库替代、采用高端的存储设备等，其中性价比最大的是 NewSQL 数据库，例如：TiDB。TiDB 采用计算、存储分离的架构，可对计算、存储分别进行扩容和缩容，计算最大支持 512 节点，每个节点最大支持 1000 并发，集群容量最大支持 PB 级别。

 **3.Real-timeHTAP场景**

 随着 5G、物联网、人工智能的高速发展，企业所生产的数据会越来越多，其规模可能达到数百 TB 甚至 PB 级别，传统的解决方案是通过 OLTP 型数据库处理在线联机交易业务，通过 ETL 工具将数据同步到 OLAP 型数据库进行数据分析，这种处理方案存在存储成本高、实时性差等多方面的问题。TiDB 在 4.0 版本中引入列存储引擎 TiFlash 结合行存储引擎 TiKV 构建真正的 HTAP 数据库，在增加少量存储成本的情况下，可以同一个系统中做联机交易处理、实时数据分析，极大地节省企业的成本。

 **4.数据汇聚、二次加工处理的场景**

 当前绝大部分企业的业务数据都分散在不同的系统中，没有一个统一的汇总，随着业务的发展，企业的决策层需要了解整个公司的业务状况以便及时作出决策，故需要将分散在各个系统的数据汇聚在同一个系统并进行二次加工处理生成 T+0 或 T+1 的报表。传统常见的解决方案是采用 ETL + Hadoop 来完成，但 Hadoop 体系太复杂，运维、存储成本太高无法满足用户的需求。与 Hadoop 相比，TiDB 就简单得多，业务通过 ETL 工具或者 TiDB 的同步工具将数据同步到 TiDB，在 TiDB 中可通过 SQL 直接生成报表

### TiDB架构图

在内核设计上，TiDB分布式数据库将整体架构拆分成了多个模块，各模块之间互相通信，组成完整的TiDB系统。对应的架构图如下：

![img](https://cdn.jsdelivr.net/gh/wanghaowish/picGo@main/img/202311231034520.png)

TiDB Server：SQL 层，负责接受客户端的连接，执行 SQL 解析和优化，最终生成分布式执行计划。可以启动多个 TiDB 实例，通过负载均衡组件（如 LVS、HAProxy 或 F5）对外提供统一的接入地址，客户端的连接可以均匀地分摊在多个 TiDB 实例上以达到负载均衡的效果。TiDB Server 本身并不存储数据，只是解析 SQL，将实际的数据读取请求转发给底层的存储节点 TiKV（或 TiFlash）

PD Server：整个 TiDB 集群的元信息管理模块，负责存储每个 TiKV 节点实时的数据分布情况和集群的整体拓扑结构，提供 TiDB Dashboard 管控界面，并为分布式事务分配事务 ID。PD 不仅存储元信息，同时还会根据 TiKV 节点实时上报的数据分布状态，下发数据调度命令给具体的 TiKV 节点，可以说是整个集群的“大脑”。

Storage Server：数据存储。

- TiKV Server：负责存储数据，从外部看 TiKV 是一个 Key-Value 存储引擎。存储数据的基本单位是 Region。TiKV 的 API 在 KV 键值对层面提供对分布式事务的原生支持，默认提供了 SI (Snapshot Isolation) 的隔离级别，这也是 TiDB 在 SQL 层面支持分布式事务的核心。TiDB 的 SQL 层做完 SQL 解析后，会将 SQL 的执行计划转换为对 TiKV API 的实际调用。所以，数据都存储在 TiKV 中。
- TiFlash：TiFlash 是一类特殊的存储节点。和普通 TiKV 节点不一样的是，在 TiFlash 内部，数据是以列式的形式进行存储，主要的功能是为分析型的场景加速。

简化为 PD -> TiDB -> TiKV 的架构：

### TiDB 存储

① 存储数据结构： Key-Value Pairs（键值对）：TiKV 选择的是 Key-Value 模型，并且提供有序遍历方法。

② 数据落盘方式： 本地存储 (RocksDB)：TiKV 没有选择直接向磁盘上写数据，而是把数据保存在 RocksDB 中，具体的数据落地由 RocksDB 负责（RocksDB 是由 Facebook 开源的一个非常优秀的单机 KV 存储引擎）。

③ 数据同步协议：Raft 协议：TiKV 利用 Raft 来做数据复制，每个数据变更都会落地为一条 Raft 日志，通过 Raft 的日志复制功能，将数据安全可靠地同步到复制组的每一个节点中。

④ 分布式存储解决方案：Region：以 Region 为单位，将数据分散在集群中所有的节点上，并且尽量保证每个节点上 Region 的数量差不多；以 Region 为单位做 Raft 的复制和成员管理。

### 文档参考

三篇文章了解 TiDB 技术内幕 – 说存储：https://cn.pingcap.com/blog/tidb-internal-1/

三篇文章了解 TiDB 技术内幕 – 说计算：https://cn.pingcap.com/blog/tidb-internal-2/

三篇文章了解 TiDB 技术内幕 – 谈调度：https://cn.pingcap.com/blog/tidb-internal-3/

博客 - TiDB 源码阅读：https://cn.pingcap.com/blog/tag/tidb-source-code-reading/

分布式 Schema 变更在 Google F1 的实践： https://zhuanlan.zhihu.com/p/367041904