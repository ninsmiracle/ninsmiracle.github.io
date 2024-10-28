---
layout: post
title:  "Raft与PacificA对比"
categories: 一致性协议
tags:  分布式 Raft 一致性协议
author: nins
---

* content
{:toc}
上传一篇简单的博客，主要做测试用，主要看看博客页面效果。
这里是 helper文档：[]{https://github.com/qiubaiying/qiubaiying.github.io/wiki/%E5%8D%9A%E5%AE%A2%E6%90%AD%E5%BB%BA%E8%AF%A6%E7%BB%86%E6%95%99%E7%A8%8B}


- PacificA

- Raft

- 对比

  


# PacificA
[Image]  

（TODO：找个方便的方式插入图片）
该协议由Manager指定一个Replication Group中的primary，写入和读取请求都发往Primary，以此来保证强一致性。
对于写入，要求必须所有副本都写入成功才能提交。
对于读取，则只需要从Primary本地读取就可以了。
当集群中的节点挂掉的时候，manager会及时将其从集群中踢掉，以免宕机节点影响读写。这样只要至少有一个节点存活，集群就可以正常对外提供服务。所以对于拥有2N+1个节点的PacificA集群可以容忍2N个节点宕机。

# Raft
[Image]
同PacificA一样，读写都是发往Primary。与PacificA不同的是，其不用依赖额外的Manager指定primary，而是通过选举的方式选出primary。
另外，Raft要求所有写入只要有超过一半的副本写入成功就可以了。所以对于拥有2N+1个节点的PacificA集群最多只能容忍N个节点宕机。
对比总结
所以相对于PacificA，Raft有如下优点：

1. PacificA抖动更多。由于PacificA的写入需要所有节点都成功写入，所以只要有一个节点写入较慢，就会导致写入抖动。然而Raft协议要求只要超过一半的节点写入成功就可以了，所以少数的慢节点基本不会影响Raft协议的写入，该节点可以写入完成后再同步追赶日志。
2. Raft可用性不如PacificA。对于拥有2N+1个节点的集群，PacificA可以容忍2N个节点挂掉。而Raft则只能容忍N个节点挂掉。然而在Pegasus的实现中，至少需要两个以上副本存活时才允许写入。因此对于3副本的PacificA，如果有2个副本挂掉，集群同样也不允许写入。所以从这一点来说，PacificA与Raft可用性持平。
3. Raft协议不需要manager。当集群中节点数量很多时，manager与节点间的心跳、配置同步等机制会消耗大量的资源、导致manager成为集群的瓶颈。由于Raft不需要manager，可以规避大集群导致的manager瓶颈问题。

# 对比
（tips: markdown表格 https://markdown.com.cn/extended-syntax/tables.html）
|                      | **PacificA**                                            | **Raft**                                                     |
| -------------------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| **选主模式**         | 由 meta（configuration manager）直接指定 Replica 成为 primary，相比 raft 省去了投票选主的过程。meta 的选主由外部 Zookeeper 保证。 | 在节点频繁故障的情况下，选举过程（包括投票选主失败）可能会导致较长的故障恢复时间。 |
| **多于半数节点宕机** | 通过租约机制和三路数据复制，即使在多于 50%的节点故障的情况下仍能提供服务。 | 多于 50%的节点宕机会导致投票无法成功，系统整体不可用。       |
| **租约机制**         | 租约机制的实现和调优相对复杂，需要在故障检测和性能之间找到平衡。 | 故障探测由心跳实现，没有租约过期时间的故障恢复延迟。选举过程清晰，选举结果迅速。 |
| **外部保障**         | 需要外部 Paxos 系统保障 meta 内部的一致性。                  | 实现简单，应用广泛。                                         |



