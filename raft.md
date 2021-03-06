# Raft协议

[ratis](https://ratis.incubator.apache.org/#gettingstarted)

[ratis的分享ppt](https://github.com/chuanlei/tech-notes/blob/master/paper/ratis-ppt.pdf)

[TiKV 源码解析系列 - 如何使用 Raft](https://pingcap.com/blog-cn/tikv-how-to-use-raft/)

[etcd raft库的使用](https://zhuanlan.zhihu.com/p/27767675)

1. 实现`Storage`接口,etcd自带了`MemoryStorage`：和etcd的wal和snap包做持久化，重启的时候从wal和snap中获取日志恢复MemoryStorage
2. 对`Ready`的处理

这个Ready结构体封装了一批更新，这些更新包括：

1. `pb.HardState`: 包含当前节点见过的最大的term，以及在这个term给谁投过票，已经当前节点知道的commit index
2. `Messages`: 需要广播给所有peers的消息
3. `CommittedEntries`:已经commit了，还没有apply到状态机的日志
4. `Snapshot`:需要持久化的快照

应用需要对Ready的处理包括:

1. 将`HardState`, `Entries`, `Snapshot`持久化到storage。
2. 将`Messages`(上文提到的msgs)非阻塞的广播给其他peers
3. 将`CommittedEntries`(已经commit还没有apply)应用到状态机。
4. 如果发现`CommittedEntries`中有成员变更类型的entry，调用node的`ApplyConfChange()`方法让node知道(这里和raft论文不一样，论文中只要节点收到了成员变更日志就应用)
5. 调用`Node.Advance()`告诉raft node，这批状态更新处理完了，状态已经演进了，可以给我下一批Ready让我处理。

![raft-ready-processing](https://github.com/chuanlei/tech-notes/blob/master/pics/raft-ready.jpg)

## ratis使用小结

![启动raft-server](https://github.com/chuanlei/tech-notes/blob/master/pics/start-raft-server.jpg)


## 视频讲解

https://www.youtube.com/watch?v=P9Ydif5_qvE

## etcd-raft使用示例

WAL、Snapshot、网络传输都需要由该库的使用者来完成。

内存kv存储系统

set/get都是通过http协议

核心数据结构为

![raft-kv-struct](https://github.com/chuanlei/tech-notes/blob/master/pics/raft-kv-struct.jpg)

其中`proposeC`是与底层raft库交互的接口

**raftNode**

![raftNode](https://github.com/chuanlei/tech-notes/blob/master/pics/v2-ef297c4a45c3d20daff24e49b79b5e41_b.jpg)

![raftNode](https://github.com/chuanlei/tech-notes/blob/master/pics/raftNode.jpg)

其核心数据结构为
1. proposeC：这是应用将客户的更新请求传递至底层raft组件的管道；
2. commitC：这是底层raft组件通知应用准备提交的指令的通道；
3. node：是对底层的raft组件的抽象，所有与底层raft组件的交互都通过该结构暴露的API来实现；
4. wal：WAL日志管理，etcd-raft将日志的管理交给应用层来处理；
5. snapshotter：同wal，etcd-raft将快照的管理也交给应用层来处理；
6. transport：应用同其他节点应用的网络传输接口，同wal，etcd-raft将集群节点之间的网络请求发送和接收也交给应用层来处理。

raftNode是应用与raft核心库连接的桥梁。该结构需要处理的内容包括
1. 应用的更新传递给底层raft
2. raft已提交的请求传输给应用以更新应用的状态机
3. 处理raft组件产生的指令，如选举指令、数据复制指令、集群节点变更指令等；
4. 处理WAL日志
5. 将底层raft组件的指令通过网络传输至集群其他节点等

### kvstore的启动

![kvstore启动](https://github.com/chuanlei/tech-notes/blob/master/pics/kvstore-start.jpg)

包含三个部分
1. 初始化raft(在创建raft node的时候，需要指出集群其他节点ip、该节点在集群中的id以及角色等)
2. 应用初始化
3. 应用开启对外服务大门

![初始化raftNode](https://github.com/chuanlei/tech-notes/blob/master/pics/newRaftNode.jpg)

初始化raftNode本质上的关键点为
1. 创建waldir和一个snapdir，为以后存储WAL日志和snapshot数据；
2. 该函数给调用者返回的是三个channel（commitC、errorC和snapshotterReady），这些channel后续作用是什么？不得而知
3. 启动内部协程startRaft

其中startRaft比较关键，会启动底层一些与raft协议处理相关联的组件


**readCommits**逻辑

![readCommits](https://github.com/chuanlei/tech-notes/blob/master/pics/readCommits.jpg)

**http服务函数**

![http api](https://github.com/chuanlei/tech-notes/blob/master/pics/http-api.jpg)

**raftNode**对propose的处理

![proposal_over_raft](https://github.com/chuanlei/tech-notes/blob/master/pics/proposal_over_raft.jpg)

来自`serveChannels`函数

**接收来自raft的返回**

![来自raft的返回](https://github.com/chuanlei/tech-notes/blob/master/pics/raft_return.jpg)

其中`publishEntries`方法将返回推到应用层

## 日志管理

### 日志追加

![日志追加](https://github.com/chuanlei/tech-notes/blob/master/pics/log-append.jpg)

### 日志重放
在应用程序启动时，第一步便是进行日志重放，构建内存状态机。

因为日志又总是和snapshot搅和在一起的，因此，构建内存状态机必须是Snapshot + 日志一起。

![日志重放](https://github.com/chuanlei/tech-notes/blob/master/pics/log_replay.jpg)

### 日志压缩：snapshot

### 创建snapshot

![创建snapshot](https://github.com/chuanlei/tech-notes/blob/master/pics/create-snapshot.jpg)

## rpc

![raft-rpc](https://github.com/chuanlei/tech-notes/blob/master/pics/raft_rpc.jpg)

## 启动流程
![启动流程](https://github.com/chuanlei/tech-notes/blob/master/pics/node_start.jpg)

而主要的运行逻辑为

![运行逻辑](https://github.com/chuanlei/tech-notes/blob/master/pics/main_run.jpg)

### 选举流程

`randomizedElectionTimeout`参数的设置与作用

`tickElection()`为谁调用

raft状态机中维护当前的状态和对应的tick函数
![tick](https://github.com/chuanlei/tech-notes/blob/master/pics/tick.jpg)

状态机启动的配置为

![状态机配置](https://github.com/chuanlei/tech-notes/blob/master/pics/fsm-config.jpg)

那么`step`回调函数是怎么使用的呢？

**如何将消息发送到其他raft节点**

![发送消息到其他raft节点](https://github.com/chuanlei/tech-notes/blob/master/pics/send.jpg)

这个问题的关键是`Ready`内容是如何构成的？

![newReady](https://github.com/chuanlei/tech-notes/blob/master/pics/newReady.jpg)

由`node`组件调用`newReady`方法。

由外围节点（即我们应用代码）消费`Ready`中的数据并发送出去。

![发送mesg](https://github.com/chuanlei/tech-notes/blob/master/pics/send_msgs.jpg)



## 总结

etcd-raft和应用之间是通过channel进行消息的通信，而消息的结构也是由raft库定义好。具体来说，应用通过raft库提供的Ready()接口获取到消息传输管道，并从该管道接收raft库发出的各种指令(Message)，最后再通过Advance()通知raft库命令处理结果。应用处理指令的典型流程是：

1. 将指令写入WAL日志
2. 将指令写入raft组件内存中（为什么要做这个？）
3. 将消息中指定的已经commit的日志进行提交，也即：应用到应用状态机中
4. 调用Advance接口，应该是通知raft当前命令执行完成，可以继续提供下一条指令了


