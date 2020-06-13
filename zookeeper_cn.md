# ZooKeeper: wait-free 的大型网络协调系统

## Abstract

在这篇论文中，我们描述了 ZooKeeper，一个协调分布式应用的服务。ZooKeeper 是基础架构的一部分，目标是提供一个简单的、高性能的内核，供客户端构建更复杂的协调原语。它在多副本、中心化的服务中，组合了 group messaging, shared registers 和分布式锁服务。ZooKeeper 提供的接口有 wait-free 的 shared registers, 和一个与文件系统 cache 失效相似的事件驱动机制，用于提供一个简单但功能强大的协调服务。

ZooKeeper 接口使实现高性能服务成为可能。除了 wait-free 的属性之外，ZooKeeper 提供了一个每个客户端请求是 FIFO 执行的语义，和改变 ZooKeeper 状态的请求都是 Linearizable 的语义。这些设计决策可以使 ZooKeeper 的读可以读本地服务器，使实现高性能处理请求流水线成为可能。对于目标工作负载，我们显示了2:1到100:1 的读/写请求比例，ZooKeeper 每秒可以处理成千上万的事务。 这种性能使ZooKeeper可以被客户端应用程序广泛使用。

## 1. Introduction



## 2. The ZooKeeper Service



### 2.1 Service Overview



### 2.2 Client API



### 2.3 ZooKeeper guarantees



### 2.4 Examples of primitives



## 3 ZooKeeper Applications



## 4 ZooKeeper Implementation



### 4.1 Request Processor



