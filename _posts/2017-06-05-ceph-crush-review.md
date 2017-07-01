---
layout: post
title: Ceph CRUSH Review
---

* contents
{:toc}

## 1. 介绍


Clinet或OSD从monitor拉取集群拓扑信息（cluster map）以及相应的分片规则（placement rule）。根据这个直接计算出对象x所在的OSD位置。

从（cluster map，placement rule，对象所在分片标识）-> (设备1，设备2，设备3)的转换算法叫做CRUSH（Controlled Replication Under Scalable Hashing）。

有了CRUSH算法，各个节点无需与中心节点通信即可计算出任何需要的object所在的存储节点位置。

本文主要是对论文[CRUSH: Controlled, Scalable, Decentralized Placement of Replicated Data](https://ceph.com/wp-content/uploads/2016/08/weil-crush-sc06.pdf)的总结介绍。



## 2. CRUSH算法介绍



### 2.1 层级化的Cluster Map

CRUSH算法通过层级化的cluster map，根据每个设备的权重将PG分配到各个OSD上，实现了一个近似均匀分布的算法。例如，cluster map可以有机房-机柜-机器-osd四层级，可以根据不同的placement rule将副本分配到不同的故障域。

Cluster map由buckets和devices构成，每一项都可以配置权重。可以配置buckets可以包含其它bucket，也可以直接包含devices。



### 2.2 副本映射过程

![CBF294C7-A8B5-4A58-8375-C4921B273423](https://ws2.sinaimg.cn/large/006tNc79ly1fgaghyjhn1j30b704ft9o.jpg)图 2.1

![1E237332-EAE7-4883-B73B-0F32181C564E](https://ws4.sinaimg.cn/large/006tNc79ly1fgagkhd1dfj30rj0cln0t.jpg)图 2.2

我们可以配置多个placement rule来适应多种场景下的副本映射，例如两副本、三副本策略，或者是EC编码、奇偶校验等等。每种rule规定了一系列操作最后从层级化的cluster map中取出相应的设备号。

举个映射过程的例子，如*图 2.1*所示的rule，此rule的执行过程如*图 2.2*所示。首先由root开始，select(1, row)选出一个row(row2)，接下来的select(3, cabinet)选出三个属于row2的不同的cabinet(cab21, cab23, cab24)，最后的select(1, disk)会在之前选出的三个cabinet里分别选择一个disk然后输出。



#### 3.2.1 映射冲突，设备故障、过载



在select(n, t)过程中会向下遍历多层之道找到n个t类型的节点。在此过程中如果遇到以下三种节点时会reject并重新继续选择：1. 已经存在当前结果集里了；2. 待选的节点是故障节点；3. 待选节点过载。为了要避免不必要的数据迁移，故障和过载节点仍会留在cluster map里，只是被特殊标记。

在节点冲突的情况下，会在本bucket内重新选择。



#### 3.2.2 Replica Ranks



![](https://ws3.sinaimg.cn/large/006tKfTcly1fgg83lm8yfj308z073wew.jpg)图 3.1



![](https://ws3.sinaimg.cn/large/006tKfTcly1fgg84nx93aj306z07wq3f.jpg)图 3.2



CRUSH在奇偶校验和EC编码容灾方案下与主从副本不太一样，主从副本方案里，如果主挂了，剩下的从由于拥有完整的数据副本，可以被选为新主，所以CRUSH可以用“first n方法” *r' = r + f* （*f*为当前失败次数，*r*为当前候选的副本号）将节点选择简单的”前移“，直接将后续的新节点选为该PG的新成员，如*图 3.1*所示。

但是在奇偶校验和EC编码方案中，由于每个节点会存放object的不同部分，所以副本层级序列非常重要。因此如果遇到故障节点，需要用*r' = r + f<sub>r</sub>n* （*f<sub>r</sub>*为当前失败次数，*r同上*）来将后面非本层级的候选节点”前移“作为该PG的新成员，如*图 3.2*所示。



### 2.3 Map变化及数据迁移

如果一个存储节点故障了，CRUSH会标记该节点但仍将节点留在层级结构中，这样的map变化仅仅引起了w<sub>failed</sub>/W<sub>total</sub>的数据迁移。

但如果层级结构改变了情况会比较复杂。这种情况下迁移量的最优是delta(w)/ W，即更改设备占全设备的权重比例。真正迁移的数据比例除以这个权重比例，大于等于1，用来衡量算法对设备更改的容忍性。map改变后权重下降的子树必须将数据迁移到权重增加的子树中来保持数据分布平衡。

### 2.4 Bucket类型

四种实现c(r, x)的策略，即从一个bucket中选择一个item过程使用的策略。

![954C5D77-BB2A-46B7-9569-6F5779355750](https://ws2.sinaimg.cn/large/006tNc79ly1fgaglm2fw4j30e303h0tn.jpg)

#### 2.4.1 Uniform Bucket

适用于所有bucket权重一致的情形。o(1)的计算复杂度，但在设备改变时会带来大量的数据迁移。实现：*c(r, x) = (hash(x) + rp) mod m*，r为replica number，m指子bucket总数，p是一个比m大的素数。这相当于简单的取模哈希uniform策略。

#### 2.4.2 List Bucket

适用于bucket只增不减的情况，o(n)的查找复杂度，设备增加时数据迁移可以是最优情况，但设备减少时会带来大量没必要的数据迁移。实现：将子bucket按照加入顺序来组织成一个list，最后加入的子bucket放在最前。对每个子bucket，比较其权重占全部的比例与hash结果决定是否选择该子bucket。

#### 2.4.3 Tree Bucket

使用二叉树来组织buckets，使查找时间缩减到O(log<sub>n</sub>)，适合管理超大集合。

#### 2.4.4 Straw Bucket

这种策略赋予每个item一固定范围内基于x, r和bucket i的初始值，然后乘上基于自身权重的因数，根据这个来选择item，所以权重较大的更容易被选中，也就是公式所示：*c(r, x) = max<sub>i</sub>(f(w<sub>i</sub>)hash(x, r, i))*，这种策略比list bucket慢两倍，但是当节点增删的时候能做到数据迁移最少。



我们可以根据不同情况选择不同的bucket类型，如果bucket是固定的不会改变的，例如机架，那选择uniform bucket是最快的，如果是只增不减的bucket那最好选择list bucket。



## 问题

1. 机器故障或者过载时就不是active了？不写副本了？一致性怎么保证？
2. 有故障机器时发向从OSD的写请求是怎么处理的
3. 数据迁移部分不是很清楚
   OSD down了，此时是（down & in）状态，crush不变，超过一段时间后，会变为out，重新map，但是没有从crush map中删掉，此时就是文中所说的故障节点的处理
