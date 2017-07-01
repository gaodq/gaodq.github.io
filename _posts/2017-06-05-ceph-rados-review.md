---
layout: post
title: Ceph RADOS Review
---

* contents
{:toc}


##  1. 总体介绍


RADOS(Reliable Autonomic Distributed Object Store)是Ceph的基石，旨在提供一个可靠、高性能以及可扩展的PB级的对象存储集群，可以向上提供文件系统、对象存储和块存储功能。RADOS采用中心节点monitor集群管理集群元信息，同时又将部分元信息管理任务分散到存储节点及客户端中，减轻了中心节点的压力。

本文主要是对论文[RADOS: A Scalable, Reliable Storage Service for Petabyte-scale Storage Clusters](https://ceph.com/wp-content/uploads/2016/08/weil-rados-pdsw07.pdf)的总结介绍。



## 2. 可扩展集群的管理

![529A9DE4-3754-4100-B8A5-FC1791F401FF](https://ws2.sinaimg.cn/large/006tNc79ly1fgahapwspjj30ch076gnc.jpg)

RADOS由一大规模OSD（存储节点）集群以及负责管理OSD之间关系的小规模monitor集群构成。



### 2.1 Cluster Map

![FABFA192-F96A-490C-B08F-E54F057FA218](https://ws3.sinaimg.cn/large/006tNc79ly1fgahj9xn7uj30bd03d0ts.jpg)

cluster map包括OSD Map、Monitor Map、PG Map以及CRUSH Map。

monitor集群通过cluster map管理所有存储节点，每个存储节点以及客户端都有cluster map副本。OSD状态变化及存储节点布局变化都会引起cluster map的epoch增加，epoch是用来保证所有通信节点持有信息的正确性。RADOS用incremental maps传输cluster map更新增量信息，用简短的增量信息描述两个相邻map epoch之间的变化。



### 2.2 数据存储分布



RADOS采用一种伪随机算法来分布存放数据，当新增设备时，会随机迁移一部分数据到新设备，以此保持存储平衡。数据存放位置通过两步计算得到，因此不需要中心化的比较重的数据分布表。

每个object通过object name的hash值对应到一个placement group(PG)，作为副本的基本单位。随着集群的扩展，需要周期性得调整PG数量，而且应该渐进式得增加，限制不同节点间PG迁移的量。

PG通过cluster map中的CRUSH规则映射到相应OSD上。每个PG对应的一组*r*个OSD（*r*为副本数）记录在cluster map中。CRUSH类似于一致性hash，具有稳定性，当集群节点变化，CRUSH仅仅迁移一部分数据来保持集群平衡。

PG提供了控制declustering replication的方法。我理解的clustering就是如果一个OSD把自己本地所有object数据全部都与另一OSD副本化，也就是做节点镜像（mirroring），而如果OSD中以一个object为单位副本化那么就是complete declustering。RADOS中副本对与PG数*u*相关，一个PG包含多个object。通常每个OSD负责100个PG左右。因为object分布比较随机，因此不同的u通常对应不同的存储设备利用率：OSD负责的PG越多（也就是*u*/osd<sub>n</sub>越大）存储分布越均衡。declustering有利于系统数据分散，而且恢复数据时可以并行化。新上的OSD负责的PG同时会有其它多个OSD负责，因此可以多个OSD并行同步数据。同时，通过减少OSD间重叠共享的数据量，可以避免OSD同时故障时的损失，例如三副本的三个OSD坏了，如果没有PG，那所有数据都不可用，否则只是与这些OSD相关的PG不可用。



### 2.3 节点状态



RADOS维护两个维度的状态信息，up、down和in、out。中间状态in & down，表示节点发生故障，但是还没有新OSD代替，同样类似的还有out & up。这种二维状态方便了多种情景的处理：1. 发生间断性的由OSD重启或网络问题引起的服务不可用时，因为在能容忍的期限内很快就会恢复，就不会有新的OSD加入负责PG，也就不用进行数据迁移（down & in）；2. 新存储节点上线时不用立即使用，可以先检查OSD自身的网络状态（out & up）；3. 可以从要将要下线的存储节点安全得迁移数据（out & up）。



### 2.4 Cluster Map传播更新



向上千个节点广播cluster map效率太低，而map epoch仅对于两个通信节点来说有意义，由于这个特点monitor可以不必负责主动地向所有OSD广播发布最新cluster map，而可以让OSD之间交换最新的map。

每个OSD维护map操作的一份历史增量，用当前的epoch作为所有通信信息的tag，同时注意自己对端的OSD epoch，如果收到了对端过时的map，则给对方发送对应的增量来保持同步。同样的，如果与一个对端联系发现对方的epoch过时，会主动得发送更新增量。故障检测心跳信息的周期性交换保证了map更新信息在O(log<sub>n</sub>)时间复杂度内快速传播（n为OSD数量）。

举个栗子，一个OSD第一次启动，向monitor发消息告诉他我启动了，同时附带自己的最新map epoch，monitor标记该OSD up，并回复需要的map增量将OSD更新到最新状态。然后这个新OSD开始与自己的副本OSD通信时，会把与自己通信的所有OSD的cluster map都更新到最新状态。由于刚启动的OSD并不知道他对端确切的epoch，所以会发送一份至少30秒左右的增量信息。

这种主动发送式的更新map方式比较保守：OSD在刚与对端通信时会一直发送自己的map增量，直到对端确认，这会使OSD收到多芬重复更新，但这个数字受对端数目限制，也就是受OSD负责的PG数限制。



## 3. 智能存储节点



cluster map中包含了节点分布信息，RADOS可以把将存储节点的副本冗余、故障检测及故障恢复任务分散到组成存储集群的OSD中去，有效利用了OSD节点的处理能力。

目前在RADOS中每个PG实现了n路主从副本以及Objects version，short-term PG log机制。主从副本由OSD自己进行：客户端只向主OSD提交写操作请求，主OSD负责所有副本的一致性以及更新。这样就将副本冗余的任务转移到了OSD之间，简化了客户端设计。Object versions以及short-term log可以使因间歇性节点不可用（例如：网络断了、节点挂了或重启）引起的数据恢复快速完成。



### 3.1 主从副本

![136AB015-C955-48B6-AA72-87309EFF278D](https://ws3.sinaimg.cn/large/006tNc79ly1fgahisopinj30d3092wgg.jpg)

RADOS实现了三种副本策略。

*primary-copy* 并行主从复制、读写都在主节点

*chain* 主节点处理写请求，最后一个从节点处理读请求

*hybrid(splay replication)* 结合上述两种方法，更省时，因为主并发写从



### 3.2 强一致性



所有的RADOS请求信息都会含有map epoch，用来保证强一致性。如果客户端由于参考过期的cluster map将请求发向了错误的OSD，该OSD会将更新增量回复给客户端使其转发到正确的节点。这样客户端可以从data node直接获取正确的map信息，保证以后的请求都正确。

有一种情况是如果由于PG的成员信息改变引起cluster map信息更新，但是只要一些旧PG成员没有收到心跳通知，他们就可能还会继续执行写操作。但是如果是PG的从节点收到了更新心跳更新了自己的cluster map，然后主节点向该从节点发写副本请求，从节点会将最新的cluster map更新增量回复给该节点。所以要求任何新加入的负责PG的OSD向所有之前负责该PG的现存活节点联系，判断该PG的正确内容，确保了先前负责该PG的OSD们收到变化通知，并在新OSD启动前停止I/O。

读一致性的实现与写一致性略微不同。如果网络故障导致了某个OSD部分不可达（client可达、monitor不可达），该OSD上的PG成为了不可读状态，但是一些客户端持有旧的cluster map，仍认为该OSD可读。与此同时，新cluster map添加了新的OSD。为了阻止新OSD更新了数据后，旧OSD处理读请求（这时有可能读出旧数据）。我们要求每个PG对应的OSD之间要有周期性心跳检测，检测PG的可读性。就是如果处理读请求的OSD H秒（会造成H秒的不一致？由于新主接管时要得到旧主确认，所以不会有新写入）没有收到其它副本的心跳就阻塞读请求。在其它OSD接管该PG的主时，同时也要获取旧主OSD的确认（上面所说），确保他们已经知道，或者也延迟同样时间，这样别的OSD就会检测到（故障检测？）。主OSD负责写操作，才可能会引起读的不一致，所以上面都是说的新主OSD加入的情况。



### 3.3 故障检测



RADOS检测到TCP故障时会重试几次后再向monitor汇报。负责同一PG的OSD之间会有阶段性的心跳检测。OSD如果发现自己被下线就会同步数据到磁盘，然后退出进程。



### 3.4 数据迁移及故障恢复



RADOS中cluster map改变、引起PG映射到的OSD改变，从而触发数据迁移。引起的原因可能是设备故障、故障恢复、集群扩容、缩容或者由新CRUSH副本策略带来的全量数据重排。

RADOS采用一种peering算法来确保PG内容一致性或恢复数据及副本。这种算法要求OSD记录PG当前object version的PG log（无论本地object是否丢失）。这样确保了PG的metadata安全，简化了数据恢复算法，也可以此判断数据是否丢失。



#### 3.4.1 Peering



当OSD收到新的cluster map后，会检查所有增量，然后调整PG状态。所有有变化的PG都需要re-peer。同时不仅仅是当前最新的map epoch需要考虑，两个epoch中间出现过的状态也需考虑在内，例如一个OSD从某一PG中被移除然后又被添加，期间有可能发生了写操作。每一个PG的peering操作是独立进行的。

peering是由PG中当前的主OSD发起的。对于含有该PG信息的所有**非当前主**的OSD，他会向当前的主发送一条包含本地所储存的关于PG的消息，其中包含最近的写操作，PG log的范围和最近的epoch。当主收到后，会生成一个包含所有负责该PG的OSD的prior set，包含了所有自上次成功peering负责过该PG的OSD。由于这个集合是主OSD主动收集notify，所以避免了对不负责该PG的OSD的无限期等待（例如一个中间状态的PG mapping OSD，它不存储该PG任何数据）。

拥有了整个prior set的PG metadata，当前主OSD就可以弄清楚所有PG节点中最新的写操作，可以从prior set中获取任一块PG log并据此更新到现在所有有效副本中，如果某个OSD没有足够的PG log（例如一个全新的OSD没有PG data），则会生成全量PG content信息。

最后主OSD和从OSD根据PG log补齐所有的PG数据。



#### 3.4.2 Recovery



declustered replication的一个重要优势就是能够并行recovery，任何一个故障节点会有很多其它OSD的副本，然后每个PG选择单独的一块，从而多OSD同时恢复副本，使得recovery非常迅速。

RADOS的recovery的性能瓶颈经常在读操作上，虽然每个OSD都有所有PG的metadata可以独立寻找缺失的数据，但这个策略有两个限制：1. 多OSD独立对同一PG做recovery的时候，它们可能不是同时从同一OSD上获取相同的object数据，这就导致了多次重复seek和read；2. 写副本的策略（3.1）会变得很复杂，例如从OSD没有从主收到修改的的object，与其它副本的object不一致。

由于这些限制，RADOS中的PG恢复由主OSD主导，像往常一样对缺失的object的操作推迟到主OSD有了本地副本。由于主OSD在peering阶段知道了哪些object是所有副本都缺失的，他可以主动的将object推送到这些副本中去，简化了副本逻辑，同时保证了缺失的object只读一次。如果主正在推送一个object（例如相应pull操作），或者它pull了一个自己缺失的object，那应该把这个object想所有需要的副本推送一次，因为现在object在内存中。因此从总体上看每个副本只被读了一次。



## 4. Monitors



RADOS用小规模monitor集群来管理存储集群的cluster map，monitor集群采用Paxos一致性协议，保证cluster map的一致性和持久性。



### 4.1 Paxos服务



monitor略微简化了标准Paxos实现，一次只允许写单map，同时使用租约来确保任何monitor的读操作的一致性。

monitor集群初始化后会选举leader，然后leader决定epoch并保存到每个monitor，在规定时间内响应leader的monitor会作为接下来的quorum。如果大多数monitor是active，通过Paxos确保每个monitor持有最新的map epoch，随后开始向active monitor发放租约。

每份租约允许active monitor向来请求cluster map的客户端响应读cluster map操作。如果租约T时间过了而且没有被更新，则认为leader挂了，会发起一个新的选举请求。leader发放租约会收到回复，如果没有收到，则认为该monitor挂了，也会发起新的选举。当一个monitor第一次启动或者发现之前的选举在合理时间内没有结束，则也会发起新的选举。

如果一个有效monitor收到了更新请求（例如故障心跳报告），首先要检查请求是否是最新。例如请求更新的OSD已经被标记为了down，那monitor向该OSD回复最新的更新增量。但是如果是最新的故障检测信息（写操作）会转发给leader，leader会聚合写操作。leader会按时更新map epoch然后向其它monitor发送更新提议，同时废除租约。如果大多数monitor回应了更新提议，leader就发放新租约。

两阶段提交和T间隔检测结合起来确保即使有效monitor集合改变了，原来的租约也会失效。



### 4.2 负载及可扩展性



通常情况下，monitor的工作并不多，租约机制使得每个monitor都可以提供读操作。由于OSD之间的cluster map主动推送更新，OSD很少向monitor发送读cluster map请求，客户端也只在OSD超时或故障时才会请求读新map。

Cluster map的写操作会转发给leader，leader会聚合写操作。如果出现大规模故障时则可能对leader造成一些压力，例如每个OSD负责*u*个PG然后*f*个OSD故障，就会有*uf*个故障报告。为了防止这种情况，OSD的心跳信息是半随机的间隔时间，并且限制了故障报告上限，非leader monitor只会向leader转发一次故障报告。这些机制实现保证了monitor的可扩展性。



## 问题

1. monitor按时更新map epoch是为啥，为什么只更新pg map的epoch
2. peering时如果不是通过notify，对不含PG数据的OSD无限等待是什么意思（3.4.1）
3. PG间同步log格式？
4. 如果主OSD故障后怎么切主？crush怎么变？
