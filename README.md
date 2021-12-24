## InfluxDB

#### 1. [https://gitee.com/l-iberty/influxdb](https://gitee.com/l-iberty/influxdb)

3个分支：

- `b1.8.3` (默认分支)
	最早开始做的存储引擎改造：乱序表、层间拷贝、消除compaction带来的写停顿。后面又将 L2 TSM 发送到 storage pool 的存储节点上。
- `b1.8.3_p1_no_writestall`
	这是从默认分支上剥离出来的，只包括存储引擎改造部分。
- `b1.8.3_p2_btreeindex`
	在`b1.8.3_p1_no_writestall`基础上的改造。在与华为初期的讨论中我了解到高基数问题，进行了相关调研后我把 series index 的实现由 hashmap 改为 btree，实验证明这并不能带来性能提升。

#### 2. [https://gitee.com/l-iberty/distributed-influxdb](https://gitee.com/l-iberty/distributed-influxdb) `v1.8.2_distributed`分支(默认)

参考[https://github.com/jasonjoo2010/chronus](https://github.com/jasonjoo2010/chronus)和[https://github.com/angopher/chronus](https://github.com/angopher/chronus)在 v1.8.2 基础上进行的分布式改造，meta nodes 使用 raft 实现强一致性，data nodes 使用 influxdb 官方集群方案。

#### 3. [https://gitee.com/l-iberty/distributed-influxdb2](https://gitee.com/l-iberty/distributed-influxdb2) `v1.8.2_distributed`分支(默认)

在[distributed-influxdb](https://gitee.com/l-iberty/distributed-influxdb)的基础上，把之前对存储引擎的改造，以及发送 L2 TSM 到 storage pool 的代码迁移过来。

#### 4. [https://gitee.com/l-iberty/distributed-influxdb3](https://gitee.com/l-iberty/distributed-influxdb3) `master`分支(默认)

在[distribuetd-influxdb2](https://gitee.com/l-iberty/distributed-influxdb2)基础上，采用 raft 实现 data nodes 的强一致性（只实现了写操作的强一致性保证，读操作不走 raft 流程）。读操作分为本地读取和远端读取(“远端”指的是 storage pool 里的镜像)，实现时参考了[distributed-influxdb](https://gitee.com/l-iberty/distributed-influxdb)的方案。

此外我还在[distributed-influxdb2](https://gitee.com/l-iberty/distributed-influxdb2)基础上进行了裁剪，删除了很多我不关心的功能模块。

#### 5. [https://gitee.com/l-iberty/influxdb-storage-mirror](https://gitee.com/l-iberty/influxdb-storage-mirror) `v1.8.2_storage_mirror`分支(默认)

位于 storage pool 的镜像节点，用于存储 L2 TSM。镜像节点上开启远程查询服务供上层 data nodes 调用。这部分代码同样参考了[distributed-influxdb](https://gitee.com/l-iberty/distributed-influxdb)。

## Influx-Proxy

[https://github.com/l-iberty/influx-proxy](https://github.com/l-iberty/influx-proxy)

参考：[InfluxDB集群化方案之influx-proxy的说明](https://sun-iot.gitee.io/posts/2755494b)

每个 circle 存储一份全量数据，所以为了实现“分库分表”，也就是根据 db 和 measurement 把数据打到不同的 influxdb raft group，需要把多个 group 里的 influxdb 实例全部配置在一个 circle 里面。

存在的问题：influx-proxy 在把接收到的有序数据转发给 influxdb 实例时，会产生少量的紊乱。如果 influx-proxy 连接到我们改造后的 influxdb 上就会发生少量乱序数据被丢弃的现象。

配置选项中的`flush_size`(默认10000)和`conn_pool_size`(默认20)直接关系到 InfluxDB 表现给客户端的吞吐量。

## Storage Pool

1. 用 etcd 搭建的 master 集群：[https://github.com/l-iberty/etcd/tree/v3.4.9/contrib/raftexample](https://github.com/l-iberty/etcd/tree/v3.4.9/contrib/raftexample)

2. slaves 节点：[https://github.com/l-iberty/slave](https://github.com/l-iberty/slave)

3. InfluxDB 镜像：[https://gitee.com/l-iberty/influxdb-storage-mirror](https://gitee.com/l-iberty/influxdb-storage-mirror)

## 部署
在未引入 influx-proxy 时使用的本地的3台虚拟机，仅支持一个3副本的 influxdb raft 集群。引入 influx-proxy 后机器性能会变得非常不稳定，时常因为 out of memory 而崩溃。

为了在较稳定的环境下测试，我把上面的全部组件打包到了实验室的主机上运行。配置如下：

![](config.svg)

其他详细信息见配置文件。

## 测试
测试程序和数据：[https://github.com/taosdata/TDengine/tree/develop/tests/comparisonTest](https://github.com/taosdata/TDengine/tree/develop/tests/comparisonTest)

我把这部分代码单独放在这里[https://github.com/l-iberty/influxdb-test](https://github.com/l-iberty/influxdb-test)。这里面还有我的一些修改，例如流式查询，以及与分布式有关的参数设置：

在对[distributed-influxdb](https://gitee.com/l-iberty/distributed-influxdb)和 [distributed-influxdb2](https://gitee.com/l-iberty/distributed-influxdb2)进行测试时，如果要保证3个 data nodes 每个都存储全量数据（默认每个节点只存储一部分数据），需要将`consistency`设置成`all`、将`replicaN`设置成3。

完整的测试包：[https://gitee.com/l-iberty/distributed_influxdb_test_pkg](https://gitee.com/l-iberty/distributed_influxdb_test_pkg)

**一个小问题**：data nodes 有时会报一个异常，来自`(*ClusterMetaClient).syncLoop`同步元数据的操作：`index X < local index Y`。这个异常无关紧要，但我想出现这个异常的时候是否需要设置`needSync = true`？异常的本质是本地数据可能比 meta nodes 更新，所以是否需要把本地数据退回和 meta nodes 一致的版本？