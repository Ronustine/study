[TOC]

## 集群
> [官方说明](https://www.rabbitmq.com/distributed.html)
> 三种集群：自带、Federation（插件）、Shovel（插件）

#### 自带
RabbitMQ本身是基于Erlang编写，Erlang语言天生具备分布式特性（通过同步Erlang集群各节点的erlang.cookie）。RabbitMQ天然支持集群。
元数据信息在所有节点上是一致的，而Queue（存放消息的队列）的完整数据则只会存在于它所创建的那个节点上。其他节点只知道这个queue的metadata信息和一个指向queue的owner node的指针。

RabbitMQ集群会始终同步四种类型的内部元数据：
- 队列元数据：队列名称和它的属性；
- 交换器元数据：交换器名称、类型和属性；
- 绑定元数据：一张简单的表格展示了如何将消息路由到队列；
- vhost元数据：为vhost内的队列、交换器和绑定提供命名空间和安全属性；

当用户访问其中任何一个RabbitMQ节点时，通过rabbitmqctl查询到的queue／user／exchange/vhost等信息都是相同的。可以减少

###### 工作原理
> [文档的一句话](https://www.rabbitmq.com/clustering.html#clustering-and-clients) Assuming all cluster members are available, a client can connect to any node and perform any operation. Nodes will route operations to the `quorum queue leader` or `queue master replica` transparently to clients.

如果有一个消息生产者或者消息消费者是连接队列所在节点，那么此时的集群中的消息收发只与此节点相关。

如果消息生产者所连接的是非队列数据所在节点，此时队列1的完整数据不在该两个节点上，那么在发送消息过程中这两个节点主要起了一个路由转发作用，根据这两个节点上的元数据转发至节点1上。

普通集群模式，并不保证队列的高可用性。尽管交换机、绑定这些可以复制到集群里的任何一个节点，但是队列内容不会复制。虽然该模式解决一项目组节点压力，但队列节点宕机直接导致该队列无法应用，只能等待重启。所以要想在队列节点宕机或故障也能正常应用，就要复制队列内容到集群里的每个节点，必须要创建镜像队列。

**镜像队列**
是一个特殊的BackingQueue，它内部包裹了一个普通的BackingQueue做本地消息持久化处理，在此基础上增加了将消息和ack复制到所有镜像的功能。所有对mirror_queue_master的操作，会通过可靠组播GM的方式同步到各slave节点。GM负责消息的广播，mirror_queue_slave负责回调处理，而master上的回调处理是由coordinator负责完成。mirror_queue_slave中包含了普通的BackingQueue进行消息的存储，master节点中BackingQueue包含在mirror_queue_master中由AMQQueue进行调用。

消息的发布（除了Basic.Publish之外）与消费都是通过master节点完成。master节点对消息进行处理的同时将消息的处理动作通过GM广播给所有的slave节点，slave节点的GM收到消息后，通过回调交由mirror_queue_slave进行实际的处理。

对于Basic.Publish，消息同时发送到master和所有slave上，如果此时master宕掉了，消息还发送slave上，这样当slave提升为master的时候消息也不会丢失。

**GM(Guarenteed Multicast)**
GM模块实现的一种可靠的组播通讯协议，该协议能够保证组播消息的原子性，即保证组中活着的节点要么都收到消息要么都收不到。

大致实现：
将所有的节点形成一个循环链表，每个节点都会监控位于自己左右两边的节点，当有节点新增时，相邻的节点保证当前广播的消息会复制到新的节点上；当有节点失效时，相邻的节点会接管保证本次广播的消息会复制到所有的节点。在master节点和slave节点上的这些gm形成一个group，group（gm_group）的信息会记录在mnesia中。不同的镜像队列形成不同的group。消息从master节点对于的gm发出后，顺着链表依次传送到所有的节点，由于所有节点组成一个循环链表，master节点对应的gm最终会收到自己发送的消息，这个时候master节点就知道消息已经复制到所有的slave节点了。
新增：
每当一个节点加入或者重新加入（例如从网络分区中恢复过来）镜像队列，之前保存的队列内容会被清空。
踢出：
如果某个slave失效了，系统处理做些记录外几乎啥都不做。master依旧是master，客户端不需要采取任何行动，或者被通知slave失效。
如果master失效了，那么slave中的一个必须被选中为master（不是HA吗？？）。被选中作为新的master的slave通常是最老的那个，因为最老的slave与前任master之间的同步状态应该是最好的。然而，需要注意的是，如果存在没有任何一个slave与master完全同步的情况，**那么前任master中未被同步的消息将会丢失。**