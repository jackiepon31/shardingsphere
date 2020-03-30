+++
pre = "<b>3.7.3. </b>"
toc = true
title = "实现原理"
weight = 3
+++

## 实现原理

考虑到ShardingSphere的弹性伸缩模块的几个挑战，目前的弹性伸缩解决方案为：临时地使用两个数据库集群，伸缩完成后切换的方式实现。

![伸缩总揽](https://shardingsphere.apache.org/document/current/img/scaling/scaling-principle-overview.cn.png)

这种实现方式有以下优点：

1. 伸缩过程中，原始数据没有任何影响。
2. 伸缩失败无风险。
3. 不受分片策略限制。

同时也存在一定的缺点：

1. 在一定时间内存在冗余服务器
2. 所有数据都需要移动

弹性伸缩模块会通过解析旧分片规则，提取配置中的数据源、数据节点等信息，之后创建伸缩作业工作流，将一次弹性伸缩拆解为4个主要阶段

1. 准备阶段
2. 存量数据迁移阶段
3. 增量数据同步阶段
4. 规则切换阶段

![伸缩工作流](https://shardingsphere.apache.org/document/current/img/scaling/workflow.cn.png)

### 准备阶段

在准备阶段，弹性伸缩模块会进行数据源连通性及权限的校验，同时进行存量数据的统计、日志位点的记录，最后根据数据量和用户设置的并行度，对任务进行分片。

### 存量数据迁移阶段

执行在准备阶段拆分好的存量数据迁移作业，存量迁移阶段采用JDBC查询的方式，直接从数据节点中读取数据，并使用新规则写入到新集群中。

### 增量数据同步阶段

由于存量数据迁移耗费的时间受到数据量和并行度等因素影响，此时需要对这段时间内业务新增的数据进行同步。不同的数据库使用的技术细节不同，但总体上均为基于复制协议或WAL日志实现的变更数据捕获功能。

- MySQL：订阅并解析binlog
- PostgreSQL：采用官方逻辑复制 [test_decoding](https://www.postgresql.org/docs/9.4/test-decoding.html)

这些捕获的增量数据，同样会由弹性伸缩模块根据新规则写入到新数据节点中。当增量数据基本同步完成时（由于业务系统未停止，增量数据是不断的），则进入规则切换阶段。

### 规则切换阶段

在此阶段，可能存在一定时间的业务只读窗口期，通过设置数据库只读或ShardingSphere的熔断机制，让旧数据节点中的数据短暂静态，确保增量同步已完全完成。

这个窗口期时间短则数秒，长则数分钟，取决于数据量和用户是否需要对数据进行强校验。确认完成后，ShardingSphere可通过配置中心修改配置，将业务导向新规则的集群，弹性伸缩完成。