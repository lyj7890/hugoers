+++
date = '2024-12-12T15:45:48+08:00'
draft = false
title = '分布式文件系统 - Gluster'
+++

# 简介

### **概述**

GlusterFS系统是一个可扩展的网络文件系统，相比其他分布式文件系统，GlusterFS具有高扩展性、高可用性、高性能、可横向扩展等特点，并且其没有元数据服务器的设计，让整个服务没有单点故障的隐患。

当客户端访问GlusterFS存储时，首先程序通过访问挂载点的形式读写数据，对于用户和程序而言，集群文件系统是透明的，用户和程序根本感觉不到文件系统是本地还是在远程服务器上。读写操作将会被交给VFS(Virtual File System)来处理，VFS会将请求交给FUSE内核模块，而FUSE又会通过设备/dev/fuse将数据交给GlusterFS Client。最后经过GlusterFS Client的计算，并最终经过网络将请求或数据发送到GlusterFS Server上。

## **基础术语**

Brick: 最基本的存储单元，表示为trusted storage pool中输出的目录，供客户端挂载用。

Volume: 一个卷。在逻辑上由N个bricks组成。

FUSE: Unix-like OS上的可动态加载的模块，允许用户不用修改内核即可创建自己的文件系统。

Glusterd: Gluster management daemon，要在trusted storage pool中所有的服务器上运行。

POSIX: 一个标准，GlusterFS 兼容。

## **卷类型**

为了满足不同应用对高性能、高可用的需求，GlusterFS 支持 7 种卷，即 distribute卷、stripe卷、replica卷、distribute stripe卷、distribute replica 卷、stripe Replica卷、distribute stripe replica 卷。其实不难看出，GlusterFS 卷类型实际上可以分为 3 种基本卷和 4 种复合卷，每种类型的卷都有其自身的特点和适用场景。

GlusterFS 集群的模式只数据在集群中的存放结构，类似于磁盘阵列中的级别。

### 1、分布式卷（Distributed Volume）

文件没有分片，文件根据hash算法写入各个节点的硬盘上，优点是容量大，读/写性能好，缺点是没冗余。不指定卷类型，默认是分布式卷。

![Untitled](https://mos-bj-exter.intra.mlamp.cn/imgs/Untitled.png)

分布式卷创建指令如下：

```bash
$ gluster volume create NEW-VOLNAME [transport [tcp | rdma | tcp,rdma]] NEW-BRICK...
volume create: test-volume: success: please start the volume to access data
```

显示卷信息：

```
gluster volume info
```

```
Volume Name: test-volume
Type: Distribute
Status: Created
Number of Bricks: 4
Transport-type: tcp
Bricks:
Brick1: server1:/exp1
Brick2: server2:/exp2
Brick3: server3:/exp3
Brick4: server4:/exp4
```

### 2、复制卷（Replicated Volume）

复制卷在创建时可指定复本的数量，通常为2或者3，复本会完整的备份在不同卷（brick）上，当其中一台服务器失效后，可以从另一台服务器读取数据，因此复制GlusterFS卷提高了数据可靠性的同事，还提供了数据冗余的功能。

![Untitled](https://mos-bj-exter.intra.mlamp.cn/imgs/Untitled%201.png)

复制卷创建指令如下：

```bash
gluster volume create NEW-VOLNAME [replica COUNT] [transport [tcp |rdma | tcp,rdma]] NEW-BRICK...
```

```bash
gluster volume create test-volume replica 3 transport tcp server1:/exp1 server2:/exp2 server3:/exp3
```

```bash
volume create: test-volume: success: please start the volume to access data
```

### 3、分布式复制卷（Distributed Replicated Volume）

分布式复制卷是将文件分布到多个副本集中，相邻的brick组成副本集。高可靠性，可扩展。

例如，8个brick，2副本，则每两个brick组成一个副本集，共4个副本集。叫做4 * 2

4副本，每4个brick组成一个副本集，共2个副本集，叫做2 * 4

应用场景：大量文件读和可靠性要求高的场景

优点：高可靠，读性能高

缺点：牺牲存储空间，写性能差

brick数量是replica的倍数

![Untitled](https://mos-bj-exter.intra.mlamp.cn/imgs/Untitled%202.png)

分布式复制卷创建卷指令如下：

```bash
gluster volume create NEW-VOLNAME [replica COUNT] [transport [tcp | rdma | tcp,rdma]] NEW-BRICK...
```

```bash
gluster volume create test-volume replica 3 transport tcp server1:/exp1 server2:/exp2 server3:/exp3 server4:/exp4 server5:/exp5 server6:/exp6
```

### 4、条带卷（Striped Volume）

文件分片均匀写在各个节点的硬盘上的，冗余卷的数量决定了最多可以故障的磁盘数，纠删码模式。分片随机读写可能会导致硬盘IOPS饱和。

![imgs](https://mos-bj-exter.intra.mlamp.cn/imgs/Untitled%203.png)

条带卷创建卷指令如下：

```bash
gluster volume create test-volume [disperse [<COUNT>]] [disperse-data <COUNT>] [redundancy <COUNT>] [transport tcp | rdma | tcp,rdma] <NEW-BRICK>
gluster volume create test-volume disperse 3 redundancy 1 server1:/exp1 server2:/exp2 server3:/exp3
```

### 5、分布式条带卷（Distributed Striped Volume）

当单个文件的体型十分巨大，客户端数量更多时，条带卷已经无法满足需求，此时将分布式与条带化结合起来是一个比较好的选择。其性能与服务器数量有关。

![Untitled](https://mos-bj-exter.intra.mlamp.cn/imgs/Untitled%204.png)

分布式条带卷创建卷指令如下：

```bash
gluster volume create [disperse [<COUNT>]] [disperse-data <COUNT>] [redundancy <COUNT>] [transport tcp | rdma | tcp,rdma] <NEW-BRICK>
gluster volume create test-volume disperse 3 redundancy 1 server1:/exp1 server2:/exp2 server3:/exp3 server4:/exp4 server5:/exp5 server6:/exp6
```

## **工作流程**

### **数据访问流程**

![https://pic2.zhimg.com/80/v2-2e42752b4c6688d954e6bd09db9f6769_1440w.webp](https://pic2.zhimg.com/80/v2-2e42752b4c6688d954e6bd09db9f6769_1440w.webp)

gfs-stack

a）首先是在客户端， 用户通过glusterfs的mount point 来读写数据， 对于用户来说，集群系统的存在对用户是完全透明的，用户感觉不到是操作本地系统还是远端的集群系统。b）用户的这个操作被递交给 本地linux系统的VFS来处理。 c）VFS 将数据递交给FUSE 内核文件系统:在启动 glusterfs 客户端以前，需要想系统注册一个实际的文件系统FUSE,如上图所示，该文件系统与ext3在同一个层次上面， ext3 是对实际的磁盘进行处理， 而fuse 文件系统则是将数据通过/dev/fuse 这个设备文件递交给了glusterfs client端。所以， 我们可以将 fuse文件系统理解为一个代理。

d）数据被fuse 递交给Glusterfs client 后， client 对数据进行一些指定的处理（所谓的指定，是按照client 配置文件据来进行的一系列处理， 我们在启动glusterfs client 时需要指定这个文件。

e）在glusterfs client的处理末端，通过网络将数据递交给 Glusterfs Server，并且将数据写入到服务器所控制的存储设备上。 这样， 整个数据流的处理就完成了;

### **客户端访问流程**

![https://pic1.zhimg.com/80/v2-c818ac9a7abbf8bf3eaf9b10544f7db4_1440w.webp](https://pic1.zhimg.com/80/v2-c818ac9a7abbf8bf3eaf9b10544f7db4_1440w.webp)

gfs-client-stack

## **缺点**

GlusterFS（GNU ClusterFile System）是一个开源的分布式文件系统，它的历史可以追溯到2006年，最初的目标是代替Lustre和GPFS分布式文件系统。经过八年左右的蓬勃发展，GlusterFS目前在开源社区活跃度非常之高，这个后起之秀已经俨然与Lustre、MooseFS、CEPH并列成为四大开源分布式文件系统。由于GlusterFS新颖和KISS（KeepIt as Stupid and Simple）的系统架构，使其在扩展性、可靠性、性能、维护性等方面具有独特的优势，目前开源社区风头有压倒之势，国内外有大量用户在研究、测试和部署应用。

当然，GlusterFS不是一个完美的分布式文件系统，这个系统自身也有许多不足之处，包括众所周知的元数据性能和小文件问题。没有普遍适用各种应用场景的分布式文件系统，通用的意思就是通通不能用，四大开源系统不例外，所有商业产品也不例外。每个分布式文件系统都有它适用的应用场景，适合的才是最好的。这一次我们反其道而行之，不再谈GlusterFS的各种优点，而是深入谈谈GlusterFS当下的问题和不足，从而更加深入地理解GlusterFS系统，期望帮助大家进行正确的系统选型决策和规避应用中的问题。同时，这些问题也是GlusterFS研究和研发的很好切入点;

1. 元数据性能 GlusterFS使用弹性哈希算法代替传统分布式文件系统中的集中或分布式元数据服务，这个是GlusterFS最核心的思想，从而获得了接近线性的高扩展性，同时也提高了系统性能和可靠性。GlusterFS使用算法进行数据定位，集群中的任何服务器和客户端只需根据路径和文件名就可以对数据进行定位和读写访问，文件定位可独立并行化进行。

这种算法的特点是，给定确定的文件名，查找和定位会非常快。但是，如果事先不知道文件名，要列出文件目录（ls或ls -l），性能就会大幅下降。对于Distributed哈希卷，文件通过HASH算法分散到集群节点上，每个节点上的命名空间均不重叠，所有集群共同构成完整的命名空间，访问时使用HASH算法进行查找定位。列文件目录时，需要查询所有节点，并对文件目录信息及属性进行聚合。这时，哈希算法根本发挥不上作用，相对于有中心的元数据服务，查询效率要差很多。

从接触的一些用户和实践来看，当集群规模变大以及文件数量达到百万级别时，ls文件目录和rm删除文件目录这两个典型元数据操作就会变得非常慢，创建和删除100万个空文件可能会花上15分钟。如何解决这个问题呢？ 我们建议合理组织文件目录，目录层次不要太深，单个目录下文件数量不要过多；增大服务器内存配置，并且增大GlusterFS目录缓存参数；网络配置方面，建议采用万兆或者InfiniBand。从研发角度看，可以考虑优化方法提升元数据性能。比如，可以构建全局统一的分布式元数据缓存系统；也可以将元数据与数据重新分离，每个节点上的元数据采用全内存或数据库设计，并采用SSD进行元数据持久化。

1. 小文件问题 理论和实践上分析，GlusterFS目前主要适用大文件存储场景，对于小文件尤其是海量小文件，存储效率和访问性能都表现不佳。海量小文件LOSF问题是工业界和学术界公认的难题，GlusterFS作为通用的分布式文件系统，并没有对小文件作额外的优化措施，性能不好也是可以理解的。

对于LOSF而言，IOPS/OPS是关键性能衡量指标，造成性能和存储效率低下的主要原因包括元数据管理、数据布局和I/O管理、Cache管理、网络开销等方面。从理论分析以及LOSF优化实践来看，优化应该从元数据管理、缓存机制、合并小文件等方面展开，而且优化是一个系统工程，结合硬件、软件，从多个层面同时着手，优化效果会更显著。GlusterFS小文件优化可以考虑这些方法，这里不再赘述，关于小文件问题请参考“海量小文件问题综述”一文。

1. 集群管理模式 GlusterFS集群采用全对等式架构，每个节点在集群中的地位是完全对等的，集群配置信息和卷配置信息在所有节点之间实时同步。这种架构的优点是，每个节点都拥有整个集群的配置信息，具有高度的独立自治性，信息可以本地查询。但同时带来的问题的，一旦配置信息发生变化，信息需要实时同步到其他所有节点，保证配置信息一致性，否则GlusterFS就无法正常工作。在集群规模较大时，不同节点并发修改配置时，这个问题表现尤为突出。因为这个配置信息同步模型是网状的，大规模集群不仅信息同步效率差，而且出现数据不一致的概率会增加。

实际上，大规模集群管理应该是采用集中式管理更好，不仅管理简单，效率也高。可能有人会认为集中式集群管理与GlusterFS的无中心架构不协调，其实不然。GlusterFS 2.0以前，主要通过静态配置文件来对集群进行配置管理，没有Glusterd集群管理服务，这说明glusterd并不是GlusterFS不可或缺的组成部分，它们之间是松耦合关系，可以用其他的方式来替换。从其他分布式系统管理实践来看，也都是采用集群式管理居多，这也算一个佐证，GlusterFS 4.0开发计划也表现有向集中式管理转变的趋势。

1. 容量负载均衡 GlusterFS的哈希分布是以目录为基本单位的，文件的父目录利用扩展属性记录了子卷映射信息，子文件在父目录所属存储服务器中进行分布。由于文件目录事先保存了分布信息，因此新增节点不会影响现有文件存储分布，它将从此后的新创建目录开始参与存储分布调度。这种设计，新增节点不需要移动任何文件，但是负载均衡没有平滑处理，老节点负载较重。GlusterFS实现了容量负载均衡功能，可以对已经存在的目录文件进行Rebalance，使得早先创建的老目录可以在新增存储节点上分布，并可对现有文件数据进行迁移实现容量负载均衡。

GlusterFS目前的容量负载均衡存在一些问题。由于采用Hash算法进行数据分布，容量负载均衡需要对所有数据重新进行计算并分配存储节点，对于那些不需要迁移的数据来说，这个计算是多余的。Hash分布具有随机性和均匀性的特点，数据重新分布之后，老节点会有大量数据迁入和迁出，这个多出了很多数据迁移量。相对于有中心的架构，可谓节点一变而动全身，增加和删除节点增加了大量数据迁移工作。GlusterFS应该优化数据分布，最小化容量负载均衡数据迁移。此外，GlusterFS容量负载均衡也没有很好考虑执行的自动化、智能化和并行化。目前，GlusterFS在增加和删除节点上，需要手工执行负载均衡，也没有考虑当前系统的负载情况，可能影响正常的业务访问。GlusterFS的容量负载均衡是通过在当前执行节点上挂载卷，然后进行文件复制、删除和改名操作实现的，没有在所有集群节点上并发进行，负载均衡性能差。

1. 数据分布问题 Glusterfs主要有三种基本的集群模式，即分布式集群(Distributed cluster)、条带集群(Stripe cluster)、复制集群(Replica cluster)。这三种基本集群还可以采用类似堆积木的方式，构成更加复杂的复合集群。三种基本集群各由一个translator来实现，分别由自己独立的命名空间。对于分布式集群，文件通过HASH算法分散到集群节点上，访问时使用HASH算法进行查找定位。复制集群类似RAID1，所有节点数据完全相同，访问时可以选择任意个节点。条带集群与RAID0相似，文件被分成数据块以Round Robin方式分布到所有节点上，访问时根据位置信息确定节点。

哈希分布可以保证数据分布式的均衡性，但前提是文件数量要足够多，当文件数量较少时，难以保证分布的均衡性，导致节点之间负载不均衡。这个对有中心的分布式系统是很容易做到的，但GlusteFS缺乏集中式的调度，实现起来比较复杂。复制卷包含多个副本，对于读请求可以实现负载均衡，但实际上负载大多集中在第一个副本上，其他副本负载很轻，这个是实现上问题，与理论不太相符。条带卷原本是实现更高性能和超大文件，但在性能方面的表现太差强人意，远远不如哈希卷和复制卷，没有被好好实现，连官方都不推荐应用。

1. 数据可用性问题 副本（Replication）就是对原始数据的完全拷贝。通过为系统中的文件增加各种不同形式的副本，保存冗余的文件数据，可以十分有效地提高文件的可用性，避免在地理上广泛分布的系统节点由网络断开或机器故障等动态不可测因素而引起的数据丢失或不可获取。GlusterFS主要使用复制来提供数据的高可用性，通过的集群模式有复制卷和哈希复制卷两种模式。复制卷是文件级RAID1，具有容错能力，数据同步写到多个brick上，每个副本都可以响应读请求。当有副本节点发生故障，其他副本节点仍然正常提供读写服务，故障节点恢复后通过自修复服务或同步访问时自动进行数据同步。

一般而言，副本数量越多，文件的可靠性就越高，但是如果为所有文件都保存较多的副本数量，存储利用率低（为副本数量分之一），并增加文件管理的复杂度。目前GlusterFS社区正在研发纠删码功能，通过冗余编码提高存储可用性，并且具备较低的空间复杂度和数据冗余度，存储利用率高。

GlusterFS的复制卷以brick为单位进行镜像，这个模式不太灵活，文件的复制关系不能动态调整，在已经有副本发生故障的情况下会一定程度上降低系统的可用性。对于有元数据服务的分布式系统，复制关系可以是以文件为单位的，文件的不同副本动态分布在多个存储节点上；当有副本发生故障，可以重新选择一个存储节点生成一个新副本，从而保证副本数量，保证可用性。另外，还可以实现不同文件目录配置不同的副本数量，热点文件的动态迁移。对于无中心的GlusterFS系统来说，这些看起来理所当然的功能，实现起来都是要大费周折的。不过值得一提的是，4.0开发计划已经在考虑这方面的副本特性。

1. 数据安全问题 GlusterFS以原始数据格式（如EXT4、XFS、ZFS）存储数据，并实现多种数据自动修复机制。因此，系统极具弹性，即使离线情形下文件也可以通过其他标准工具进行访问。如果用户需要从GlusterFS中迁移数据，不需要作任何修改仍然可以完全使用这些数据。

然而，数据安全成了问题，因为数据是以平凡的方式保存的，接触数据的人可以直接复制和查看。这对很多应用显然是不能接受的，比如云存储系统，用户特别关心数据安全。私有存储格式可以保证数据的安全性，即使泄露也是不可知的。GlusterFS要实现自己的私有格式，在设计实现和数据管理上相对复杂一些，也会对性能产生一定影响。

GlusterFS在访问文件目录时根据扩展属性判断副本是否一致，这个进行数据自动修复的前提条件。节点发生正常的故障，以及从挂载点进行正常的操作，这些情况下发生的数据不一致，都是可以判断和自动修复的。但是，如果直接从节点系统底层对原始数据进行修改或者破坏，GlusterFS大多情况下是无法判断的，因为数据本身也没有校验，数据一致性无法保证。

1. Cache一致性 为了简化Cache一致性，GlusterFS没有引入客户端写Cache，而采用了客户端只读Cache。GlusterFS采用简单的弱一致性，数据缓存的更新规则是根据设置的失效时间进行重置的。对于缓存的数据，客户端周期性询问服务器，查询文件最后被修改的时间，如果本地缓存的数据早于该时间，则让缓存数据失效，下次读取数据时就去服务器获取最新的数据。

GlusterFS客户端读Cache刷新的时间缺省是1秒，可以通过重新设置卷参数Performance.cache-refresh-timeout进行调整。这意味着，如果同时有多个用户在读写一个文件，一个用户更新了数据，另一个用户在Cache刷新周期到来前可能读到非最新的数据，即无法保证数据的强一致性。因此实际应用时需要在性能和数据一致性之间进行折中，如果需要更高的数据一致性，就得调小缓存刷新周期，甚至禁用读缓存；反之，是可以把缓存周期调大一点，以提升读性能。
