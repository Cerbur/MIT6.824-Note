# 课前准备

## 论文

中文版： https://zhuanlan.zhihu.com/p/87314382

英文原版： http://nil.lcs.mit.edu/6.824/2020/papers/gfs.pdf

### 论文摘要

#### GFS 设计原则

##### **节点失效是常态**

##### **存储内容以大文件为主**

##### 系统主要文件操作为**大容量连续读**、**小容量随机读**以及**追加连续写**

##### 支持**原子的文件追加**

##### **高吞吐量比低延时更重要**

#### GFS架构设计

![](https://raw.githubusercontent.com/Cerbur/pic/main/20210626162758.png)

>一个GFS集群由一个**Master**节点和若干个**Chunk Server**节点组成，可以被若干个**客户端**访问。每个节点作为用户级进程运行在Linux机器上。

这里也说明需要一个 Master 对多个 Server 进行调度。Chunk由一个Master分配的不可改变且全局唯一的64位Chunk句柄进行标识。Chunk Server在本地磁盘上存储Chunk，通过指定Chunk句柄和字节范围读取或写入Chunk。Master负责维护整个集群的元数据。

>客户端与Master交互获得集群的元数据。当客户端需要进行数据读写时，不会通过Master直接进行，而是询问Master应该与哪些Chunk Server通信，然后直接与Chunk Server通信进行数据读写，以此避免Master成为整个集群数据传输的瓶颈，同时，客户端会在一定时间内缓存Chunk Server信息，后续操作可以直接与Chunk Server进行通信。

所以说 Master 是负责将 Chunk Server 进行搭桥。

#### Chunk大小的选取

>通常情况下，Chunk大小为64MB，这比典型的文件系统block大小大得多，可以通过惰性空间分配策略，来避免因内部碎片造成的空间浪费。

##### 优点

1. **减少客户端与Master的通信次数**
2. 增大客户端操作落到同一个Chunk上的概率。**减少网络负载**。
3. **减少Master保存的元数据大小**

#### GFS元数据管理

GFS元数据全部保存在Master的内存中，主要包括以下三类信息：

- 文件和Chunk的命名空间
- 文件与Chunk的映射
- Chunk Replica的位置信息

#### Namespace管理

> 在逻辑上，Master并不会根据文件与目录的关系以分层的结构来管理这部分数据，而是单纯地将其表示为从完整路径名到对应文件元数据的映射表，并在路径名上应用前缀压缩以减少内存占用。

锁

> 为了管理来自不同客户端的并发请求对Namespace的修改，Master 、会为Namespace中的每个文件和目录都分配一个读写锁（Read-Write Lock）。由此，对不同Namespace区域的并发请求便可以同时进行。
>
> 所有Master操作在执行前都会需要先获取一系列的锁：通常，当操作涉及某个路径 /d1/d2/.../dn/leaf时，Master会需要先获取从/d1、/d1/d2到/d1/d2/.../dn的读锁，然后再根据操作的类型获取 /d1/d2/.../dn/lead的读锁或写锁。获取父目录的读锁是为了避免父目录在此次操作执行的过程中被重命名或删除。
>
> 由于大量的读写锁可能会造成较高的内存占用，这些锁会在实际需要时才进行创建，并在不再需要时被销毁。

#### Chunk租约（lease）和变更顺序

> GFS使用租约（lease）机制来保持多个副本间变更顺序的一致性。在客户端对某个Chunk做出变更时，会把该Chunk的Lease交给某个Replica，使其成为Primary：Primary 会负责为这些变更安排一个执行顺序，然后其他Replica便按照相同的顺序执行这些修改

#### 文件写入

![](https://raw.githubusercontent.com/Cerbur/pic/main/20210626170335.png)

1. 客户端向 Master 询问目前哪个 Chunk Server 持有该 Chunk 的 Lease。如果没有一个 Chunk 持有 Lease，Master 就选择其中一个 Replica 建立一个 Lease。
2. Master 向客户端返回 Primary 和其他 Replica 的位置。客户端缓存这些数据以便后续的操作。只有在 Primary Chunk 不可用，或者 Primary Chunk 回复信息表明它已不再持有租约的时候，客户端才需要重新跟 Master 联系。
3. 客户端将数据以任意顺序推送到所有的 Replica 上。Chunk Server 会把这些数据保存在缓冲区中，等待使用。
4. 待所有 Replica 都接收到数据后，客户端发送写请求给 Primary。Primary 为来自各个客户端的修改操作安排连续的执行序列号，并按顺序应用于其本地存储的数据。
5. Primary 将写请求转发给其他 Secondary Replica，Replica 按照相同的顺序应用这些修改。
6. Secondary Replica 响应 Primary，示意自己已经完成操作。
7. Primary 响应客户端，并返回该过程中发生的错误（若有）。

#### **文件追加**

文件追加操作的过程和写入的过程有几分相似：

1. 客户端将数据推送到每个Replica，然后将请求发往Primary。
2. Primary首先判断将数据追加到Chunk后是否会令Chunk的大小超过上限：如果是，那么 Primary会将当前Chunk填充至其大小达到上限，并通知其他Replica执行相同的操作，再响应客户端，通知其应在下一个Chunk上重试该操作。
3. 如果数据能够被放入到当前Chunk中，那么Primary会把数据追加到自己的Replica中，拿到追加成功返回的偏移量，然后通知其他Replica将数据写入到相同位置中。
4. 最后Primary把偏移量返回给客户端。



#### **文件快照**

GFS 提供了文件快照操作，可为指定的文件或目录创建一个副本。

快照操作的实现采用了写时复制（Copy on Write）的思想：

1. 在Master接收到快照请求后，它首先会撤回这些Chunk的Lease，以让接下来其他客户端对这些Chunk进行写入时都需要请求Master获知Primary的位置，Master便可利用这个机会创建新的Chunk拷贝。
2. 当Chunk Lease撤回或失效后，Master把操作以日志的方式记录到硬盘上。然后，Master 节点通过复制源文件或者目录的元数据的方式，把这条日志记录的变化反映到保存在内存的状态中。新创建的快照文件和源文件指向完全相同的Chunk。
3. 当有客户端尝试对这些Chunk进行写入时，Master会注意到这个Chunk的引用计数大于1。此时，Master不会马上响应客户端，而是生成一个Handle，然后通知所有持有这些Chunk的Chunk Server在本地复制出一个新Chunk，Master给新Chunk的一个Replica分配租约，之后响应客户端，客户端得到响应后就可以正常的写这个 Chunk。

#### **文件读取**

![](https://raw.githubusercontent.com/Cerbur/pic/main/20210626162758.png)

1. 对于指定的文件名和读取位置偏移量，客户端可以根据固定大小的Chunk计算出需要读取文件的Chunk索引。
2. 客户端将文件名和Chunk索引发送给Master，Master返回Chunk句柄以及其所有Replica的位置。客户端会以文件名和Chunk索引为key缓存该数据，当向同一个Chunk读取文件时，如果缓存信息没有过期，则直接与Chunk Server通信。
3. 客户端向最近的Replica所在的Chunk Server发送包含Chunk句柄和读取范围的请求。
4. Chunk Server将数据发送给客户端。

#### **文件删除**

当用户对某个文件进行删除时，GFS 不会立刻删除数据，而是采用惰性策略，在文件和Chunk两个层面上对数据进行移除。

> 首先，当客户端删除某个文件时，Master像对待其它修改操作一样，立刻把删除操作以日志的方式记录下来。但Master并不会立刻从Namespace中将其删除，而是将该文件重命名为另一个包含删除时时间戳的隐藏的名称。在Master周期扫描Namespace时，它会删除所有三天前的隐藏文件。在文件被彻底从Namespace删除前，客户端仍然可以利用这个重命名后的隐藏名称读取该文件，甚至再次将其重命名以撤销删除操作。

采用这种删除机制主要有如下三点好处：

1. 对于大规模的分布式系统来说，这样的机制更为简单**可靠**：在Chunk创建时，可能某些Chunk Server创建成功，某些Chunk Server创建失败，这导致某些Chunk Server上可能存在Master不知道的Replica。此外，删除请求可能会发送失败，Master必须重发。相比之下，由Chunk Server主动删除Replica能够以一种更为统一的方式解决以上的问题
2. 这样的删除机制将存储回收过程与Master日常的周期扫描合并在一起，使得这些操作可以以批的形式进行处理，减少资源损耗；此外，这样也得以让Master选择在相对空闲的时候进行这些操作
3. 用户发送删除请求和数据被实际删除之间的延迟也有效避免了用户误操作的问题

#### Replica管理

Chunk的Replica位置选取主要有两个目标：最大化数据可靠性和可用性，以及最大化网络带宽利用率。

Chunk Replica可以由Chunk的创建、重新备份和重新负载均衡三种事件创建。

##### **Chunk的创建**

在Master创建一个新的Chunk时，首先它会需要考虑在哪放置新的Replica。Master会考虑如下几个因素：

1. Master会倾向于把新的Replica放在磁盘利用率较低的Chunk Server上，以平衡Chunk Server的磁盘利用率
2. Master会倾向于确保每个Chunk Server上“较新”的Replica不会太多，因为新Chunk的创建意味着接下来会有大量的写入，如果某些Chunk Server上有太多新的Replica，那么写操作压力就会集中在这些Chunk Server上
3. Master会倾向于把Replica放在不同的机架上

##### **Chunk的重备份**

当某个Chunk的Replica数量低于用户指定的阈值时，Master就会对该Chunk进行重备份。这可能是由Chunk Server失效、Chunk Server存储的副本损坏、Chunk Server磁盘损坏或是用户提高了Replica数量阈值所触发。

首先，Master会按照以下因素为每个需要重备份的Chunk安排优先级：

1. 该Chunk的Replica数量与用户指定的Replica数量阈值的差距有多大
2. 优先为未删除的文件的Chunk进行重备份
3. 提高会阻塞用户操作的Chunk的优先级

##### **Chunk的负载均衡**

Master 会周期地检查每个Chunk当前在集群内的分布情况，并在必要时迁移部分Replica以更好地均衡各节点的磁盘利用率和负载。新Replica的位置选取策略和上面提到的大体相同。此外，Master还需要移除磁盘占用较高的Chunk Server上的Replica，以均衡磁盘使用率。

#### 高可用机制

##### **快速恢复**

GFS 的组件被设计为可以在数秒钟内恢复它们的状态并重新启动。GFS的组件实际上并不区分正常退出和异常退出：要关闭某个组件时直接kill进程即可。

##### **Chunk Server**

作为集群中的Slave角色，Chunk Server失效的几率比Master要大得多。当Chunk Server失效时，其所持有的Replica对应的Chunk的Replica数量便会降低，当Master发现Replica数量低于用户指定阈值时，会进行重备份。

##### **Master**

Master在对元数据做任何操作前对会用**先写日志**的形式将操作进行记录，只有当日志写入完成后才会响应客户端的请求，而这些日志也会备份到多个机器上。日志只有在写入到本地以及远端备份的持久化存储中才被视为完成写入。

##### **数据完整性**

为了保证数据完整，每个Chunk Server都会使用校验和来检测自己保存的数据是否有损坏；在侦测到损坏数据后，Chunk Server可以利用其它Replica来恢复数据。

#### 数据一致性

首先，命名空间完全由单节点Master管理在其内存中，这部分数据的修改可以通过让Master 为其添加互斥锁来解决并发修改的问题，因此命名空间的数据修改是可以确保完全原子的。

文件的数据修改则相对复杂。在讲述接下来的内容前，首先我们先明确，在文件的某一部分被修改后，它可能进入以下三种状态的其中之一：

- 客户端读取不同的Replica时可能会读取到不同的内容，那这部分文件是**不一致**的（Inconsistent）
- 所有客户端无论读取哪个 Replica 都会读取到相同的内容，那这部分文件就是**一致**的（Consistent）
- 所有客户端都能看到上一次修改的完整内容，且这部分文件是一致的，那么我们说这部分文件是**确定**的（Defined）

在修改后，一个文件的当前状态将取决于此次修改的类型以及修改是否成功。具体来说：

- 如果一次写入操作成功且没有与其他并发的写入操作发生重叠，那这部分的文件是**确定**的（同时也是一致的）
- 如果有若干个写入操作并发地执行成功，那么这部分文件会是**一致**的但会是**不确定**的：在这种情况下，客户端所能看到的数据通常不能直接体现出其中的任何一次修改
- 失败的写入操作会让文件进入**不一致**的状态

#### **对应用的影响**

GFS 的一致性模型是相对松散的，这就要求上层应用在使用GFS时能够适应GFS所提供的一致性语义。简单来讲，上层应用可以通过两种方式来做到这一点：更多使用追加操作而不是覆写操作；写入包含校验信息的数据。

## FQA

http://nil.lcs.mit.edu/6.824/2020/papers/gfs-faq.txt

## Questions

http://nil.lcs.mit.edu/6.824/2020/questions.html?q=q-gfs&lec=3