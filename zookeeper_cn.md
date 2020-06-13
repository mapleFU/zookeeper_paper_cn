# ZooKeeper: wait-free 的大型网络协调系统

## Abstract

在这篇论文中，我们描述了 ZooKeeper，一个协调分布式应用的服务。ZooKeeper 是基础架构的一部分，目标是提供一个简单的、高性能的内核，供客户端构建更复杂的协调原语。它在多副本、中心化的服务中，组合了 group messaging, shared registers 和分布式锁服务。ZooKeeper 提供的接口有 wait-free 的 shared registers, 和一个与文件系统 cache 失效相似的事件驱动机制，用于提供一个简单但功能强大的协调服务。

ZooKeeper 接口使实现高性能服务成为可能。除了 wait-free 的属性之外，ZooKeeper 提供了一个每个客户端请求是 FIFO 执行的语义，和改变 ZooKeeper 状态的请求都是 Linearizable 的语义。这些设计决策可以使 ZooKeeper 的读可以读本地服务器，使实现高性能处理请求流水线成为可能。对于目标工作负载，我们显示了2:1到100:1 的读/写请求比例，ZooKeeper 每秒可以处理成千上万的事务。 这种性能使ZooKeeper可以被客户端应用程序广泛使用。

## 1. Introduction

大规模的分布式应用需要不同形式的协调。配置是最基础的形式之一。在它最简单的形式，系统进程中，配置是一个系统进程的的可操作的参数列表，更复杂的系统有动态的配置参数。Group membership 和领导选举也是分布式系统的共同有服务：通常，进程需要知道哪些别的进程状态是 alive 的和这些进程负责什么。Locks 建立了一个强大的同步原语，实现对关键资源的互斥访问。

一种协调的方式是，为不同的需求开发不同的服务。例如，Amazon Simple Queue Service \[3\] 专注于队列服务。其他服务被开发，用于特定的用处，比如领导选举和配置。实现了更强原语的服务可以被用于实现没有那么强大原语的服务。例如，Chubby 是一个有很强同步保证的锁服务。锁可以用来实现领导选举，组成员等服务。

当设计我们的协调服务的时候，我们抛弃了在服务端测实现特定的原语，作为替代，我们选择暴露一个能够使应用开发者实现他们自己同步原语的 API。这样的决定让我们实现一个使不需要改变服务而实现不同同步服务成为可能的*协调内核*。这种方法允许用户实现多种形式的协调，以适应应用程序的需求，而不是将开发人员限制在一组固定的原语上。

当设计 ZooKeeper 的 API 的时候，我们抛弃了阻塞原语，例如 Locks。对于一个协调服务，阻塞原语可以导致速度缓慢或故障的客户端等其他问题，从而对速度较快的客户端的性能产生负面影响。如果处理请求取决于响应和其他客户端的失败检测，则服务本身的实现将变得更加复杂。因此，我们的系统 ZooKeeper 实现了一个API，该API可以按文件系统的层次结构来组织简单的 wait-free 数据对象。 实际上，ZooKeeper API类似于任何其他文件系统，并且仅从 API 签名来看，ZooKeeper 似乎很像没有锁定方法，打开和关闭方法的 Chubby。 但是，实现 wait-free 数据对象使 ZooKeeper 与基于锁之类的阻塞原语的系统明显不同。

尽管无等待属性对于性能和容错性很重要，但不足以进行协调。我们还必须停工操作的顺序保证。特殊的，我们发现，对客户端所有操作提供 FIFO 语义与提供 *linearizable writes* 可以高效的实现服务，并且足以实现应用程序感兴趣的协调原语。实际上，对于使用API的任意数量的进程，都可以实现一致性，根据Herlihy给出的层次结构，ZooKeeper实现了全局对象。

ZooKeeper 服务用使用了复制来保证高可用和性能的服务器组成。它的高性能使包含大量进程的应用程序可以使用这种协调内核来管理协调的各个方面。我们能够使用一个简单的流水线架构，让我们能够处理成百上千的请求，同时仍然保持低延迟。这样的流水线很自然地可以保证对于单个客户端所有操作执行的顺序性。FIFO客户端顺序使得客户端可以异步提交操作请求。使用异步操作，客户端可以同时处理多个未完成操作。使用异步操作，客户端一次可以执行多个未完成的操作。这个功能是很棒的，例如，当新客户端成为领导者并且必须操纵元数据并相应地对其进行更新时。 如果不可能进行多个未完成的操作，则初始化时间可以是几秒左右，而不是亚秒级。

为了保证更新操作满足 linearizability，我们实现了一个基于 leader 的原子广播协议 Zab。然而一个典型的 ZooKeeper 应用，在支配地位的负载是读操作，所以需要保证读吞吐量的扩展性。在ZooKeeper中，服务器在本地处理读操作，并不需要使用Zab。

在客户端测缓存数据是提升读性能的重要技术。例如，对于一个进程，缓存现有 Leader 的 id 而不是每次需要 leader 都探测 leader 是很有效的。ZooKeeper 使用一种 watch 机制而不是直接操作缓存，来保证客户端缓存数据。有了 watch 机制，一个客户端可以 watch 一个给给定对象的更新请求，并接收到 update 的请求。（作为对比）Chubby 字节管理客户端的 cache，它会阻塞更新，以使更新部分的客户端的缓存全部失效。在这样的设计下，如果任何客户端响应慢或者出现错误，更新会被延迟。Chubby 使用 lease 机制防治一个客户端用用阻塞系统。但 leases 只能约束慢客户端或者错误客户端的影响，然后 ZooKeeper 的 watches 可以完全避免这个问题。

本论文讨论ZooKeeper的设计和实现，使用ZooKeeper，即使只有写入是可线性化的，我们也可以实现应用程序所需的所有协调原语。 为了验证我们的方法，我们展示了如何使用 ZooKeeper 实现一些协调原语。

作为总结，这篇文章中，我们主要的贡献是：

* **Coordination kernel**: 为了在分布式系统中使用，我们添了一个提供 relaxed 的一致性保证的 wait-free 的协作服务。特别是，我们描述了*协调内核*的设计和实现，我们已经在许多关键应用程序中使用了协调内核来实现各种协调技术。
* **Coordination recipes**: 我们展示了 ZooKeeper 如何可用于构建通常在分布式应用程序中使用的高级协调原语，甚至包含阻塞和强一致性原语。
* **Experience with Coordination**: 我们分享了一些使用 ZooKeeper 并评估其性能的方式。

## 2. The ZooKeeper Service

客户端通过 ZooKeeper client API library 向 ZooKeeper 提交请求。除了暴露 ZooKeeper 服务的 client API 接口, ZooKeeper Client library 也管理 client 和 ZooKeeper 服务器间的网络连接。

在这一届中，我们写提供一个 ZooKeeper 服务的 high-level view。然后我们讨论 client 与 ZooKeeper 操作的 API。

> **术语** 在这篇论文章，我们使用 *`client`* 来表示一个 ZooKeeper 服务的使用者，*`znode`* 表示一个内存中的 ZooKeeper 数据的节点。`znode` 会被组织成一颗被称为 `data tree` 的带层次的名称空间。我们还使用术语“更新和写入”来指代任何修改数据树状态的操作。 客户端在连接到 ZooKeeper 并获得 `session` 句柄以发出请求时建立 `session`。

### 2.1 Service Overview

ZooKeeper 给它的客户端提供了“data nodes\(`znodes`\)的集合”的抽象，用层次的名称空间组织他们。client 通过 API 操纵在这个层次中的数据对象。层次的名称空间在文件系统里被广泛使用。这是一种组织层次空间的可靠方法，因为用户习惯这种抽象，同时它使更好的应用元数据组织成为可能。为了引用一个给定的 `znode` ，我们使用标准的 UNIX 文件系统路径。例如，我们使用 `/A/B/C` 来表示到 `znode` C 的路径，B 是 C 的父节点，同时 A 是 B 的父节点。所有的 `znode` 都存储数据，同时，除了 `ephemeral znodes` 外的所有的 `znode` ，都能拥有子节点。

// TODO: 插入 Figure1

client 可以创建两种 `znode`:

* `Regular`: client 显式操纵和删除 `regular znodes`
* `Ephemeral`: client 会创建这种节点，它可以被显式唱出，也可以在创建这个节点的 `session` 断开的时候自动删除（故意或由于故障）。

此外，当我们创建一个新的 `znode` 的时候，一个 client 可以设置 `sequential` flag. 通过 `sequential` flag 创建的节点会有在名称添加一个单调递增的计数器。如果 `n` 是新的 `znode`, `p` 是他的父节点，那么 `n` 的名称中添加的值不会小与 `p` 的子节点中创建过的任何一个添加的值。

ZooKeeper 通过现实 watch 来允许 client 不需要轮询的定时接收到值的变化的通知。当一个客户端带有 `watch` flag 并发起读请求的时候，该操作将正常完成，除了服务器承诺在返回的信息已更改时会通知客户端。 监视是与会话相关的一次性触发器； 一旦触发或会话关闭，它们将不被注册。 监视表明发生了更改，但未提供更改。 例如，如果客户端在两次更改`/foo`之前发出了请求 `getData('/ foo'，true)`，则客户端将只获得一个 watch事件，告知客户端`/foo`的数据已更改。 `session` 事件（例如连接丢失事件）也将发送到 watch 回调，以便客户端知道 watch 事件可能会延迟.

#### 数据模型

ZooKeeper 的数据模型是一个只有把全部数据整个读/写的文件系统的简化 api, 或者有 key 层次的 key/value 表。层次名称空间对于为不同应用程序的名称空间分配子树以及设置对这些子树的访问权限很有用。 我们还将在客户端利用目录的概念来构建更高级别的原语，如我们在2.4节中将看到的。

与文件系统中的文件不同，znode不适用于常规数据存储。 相反，`znodes` 存储 client 引用的数据，通常是用于协调的元数据。为了图解，在 Figure 1 中我们有两个子树，一个用于应用程序1（`/app1`），另一个用于应用程序2（`/app2`）。应用程序1的子树实现了一个简单的组成员身份协议：每个客户端进程`pi`在`/app1`下创建一个`znode`  `pi`，只要该进程正在运行，该节点便会持续存在。

尽管 `znode` 并非设计用于常规数据存储，但是 ZooKeeper 允许客户端存储一些可用于分布式计算中的元数据或配置的信息。例如，在基于 leader 的应用程序中，这对于刚刚开始了解哪个其他服务器当前是领导者的应用程序服务器很有用。为了实现此目标，我们可以让当前的领导者在znode空间中的已知位置写入此信息。 `znode` 还将 元数据与 timestamp 和 version counter 关联，这使客户端可以跟踪对 `znode` 的更改并根据 `znode` 的版本执行有条件的更新。

#### Sessions

client 连接到 ZooKeeper 并初始化 `session`。`session `具有关联的 timeout。如果ZooKeeper 在 timeout 内没有收到来自 `session` 的 client 的任何消息，则认为该 client 有故障。 当 client 显式关闭会话句柄或 ZooKeeper 检测到 client 故障时，会话结束。 在会话中，客户端观察到一系列状态变化，这些状态变化反映了其操作的执行。 会话使客户端可以在ZooKeeper集成中从一台服务器透明地移动到另一台服务器，因此可以在ZooKeeper服务器之间持久存在。在 `session` 中，client 观察到一系列状态变化，这些状态变化反映了 ZooKeeper 操作的执行。 `session` 使客户端可以在 ZooKeeper 集群中从一台服务器透明地移动到另一台服务器，因此`session`可以在 ZooKeeper 服务器之间持久存在。

### 2.2 Client API



### 2.3 ZooKeeper guarantees



### 2.4 Examples of primitives



## 3 ZooKeeper Applications



## 4 ZooKeeper Implementation



### 4.1 Request Processor



