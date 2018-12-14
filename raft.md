# Raft协议

某个java实现为https://github.com/wenweihu86/raft-java

maven依赖为

```xml
<dependency>
    <groupId>com.github.wenweihu86.raft</groupId>
    <artifactId>raft-java-core</artifactId>
    <version>1.8.0</version>
</dependency>
```

视频讲解

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

