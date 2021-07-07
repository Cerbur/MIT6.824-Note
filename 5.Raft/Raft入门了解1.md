# Raft 入门了解

- 本文内容引用 Raft协议详解 - 丁凯的文章 - 知乎 https://zhuanlan.zhihu.com/p/27207160 版权归作者所有

  用于入门了解 Raft 算法，此文章截取出来我认为重要的部分。

Raft 协议将一致性协议的核心内容分拆成为几个关键阶段，以简化流程，提高协议的可理解性。

## 架构

![](https://pic2.zhimg.com/v2-e747e997d58aee9252b187c8434a1cc1_b.jpg)

系统中每个结点有三个组件：

> 状态机: 当我们说一致性的时候，实际就是在说要保证这个状态机的一致性。状态机会从log里面取出所有的命令，然后执行一遍，得到的结果就是我们对外提供的保证了一致性的数据
> Log: 保存了所有修改记录
> 一致性模块: 一致性模块算法就是用来保证写入的log的命令的一致性，这也是raft算法核心内容

## **Leader election** （领导选举）

Raft协议的每个副本都会处于三种状态之一：Leader(领导)、Follower(跟随者)、Candidate(候选人)。

> **Leader**：所有请求的处理者，Leader副本接受client的更新请求，本地处理后再同步至多个其他副本；
> **Follower**：请求的被动更新者，从Leader接受更新请求，然后写入本地日志文件
> **Candidate**：如果Follower副本在一段时间内没有收到Leader副本的心跳，则判断Leader可能已经故障，此时启动选主过程，此时副本会变成Candidate状态，直到选主结束。

时间被分为很多连续的随机长度的term，term有唯一的id。每个term一开始就进行选主：

1. Follower将自己维护的current_term_id加1。
2. 然后将自己的状态转成Candidate
3. 发送RequestVoteRPC消息(带上current_term_id) 给 其它所有server

这个过程会有三种结果：

- 自己被选成了主。当收到了majority的投票后，状态切成Leader，并且定期给其它的所有server发心跳消息（不带log的AppendEntriesRPC）以告诉对方自己是current_term_id所标识的term的leader。每个term最多只有一个leader，term id作为logical clock，在每个RPC消息中都会带上，用于检测过期的消息。当一个server收到的RPC消息中的rpc_term_id比本地的current_term_id更大时，就更新current_term_id为rpc_term_id，并且如果当前state为leader或者candidate时，将自己的状态切成follower。如果rpc_term_id比本地的current_term_id更小，则拒绝这个RPC消息。
- 别人成为了主。如1所述，当Candidator在等待投票的过程中，收到了大于或者等于本地的current_term_id的声明对方是leader的AppendEntriesRPC时，则将自己的state切成follower，并且更新本地的current_term_id。
- 没有选出主。当投票被瓜分，没有任何一个candidate收到了majority的vote时，没有leader被选出。这种情况下，每个candidate等待的投票的过程就超时了，接着candidates都会将本地的current_term_id再加1，发起RequestVoteRPC进行新一轮的leader election。

### 投票策略

- 每个节点只会给每个term投一票，具体的是否同意和后续的Safety有关。
- 当投票被瓜分后，所有的candidate同时超时，然后有可能进入新一轮的票数被瓜分，为了避免这个问题，Raft采用一种很简单的方法：每个Candidate的election timeout从150ms-300ms之间随机取，那么第一个超时的Candidate就可以发起新一轮的leader election，带着最大的term_id给其它所有server发送RequestVoteRPC消息，从而自己成为leader，然后给他们发送心跳消息以告诉他们自己是主。

## **Log Replication**

当Leader被选出来后，就可以接受客户端发来的请求了，每个请求包含一条需要被replicated state machines执行的命令。leader会把它作为一个log entry append到日志中，然后给其它的server发AppendEntriesRPC请求。当Leader确定一个log entry被safely replicated了（大多数副本已经将该命令写入日志当中），就apply这条log entry到状态机中然后返回结果给客户端。如果某个Follower宕机了或者运行的很慢，或者网络丢包了，则会一直给这个Follower发AppendEntriesRPC直到日志一致。

当一条日志是commited时，Leader才可以将它应用到状态机中。Raft保证一条commited的log entry已经持久化了并且会被所有的节点执行。

当一个新的Leader被选出来时，它的日志和其它的Follower的日志可能不一样，这个时候，就需要一个机制来保证日志的一致性。一个新leader产生时，集群状态可能如下：

![](https://pic4.zhimg.com/v2-8884bbb61f0eed0aba4e8320c2692783_b.jpg)

最上面这个是新Leader，a~f是Follower，每个格子代表一条log entry，格子内的数字代表这个log entry是在哪个term上产生的。

新Leader产生后，就以Leader上的log为准。其它的follower要么少了数据比如b，要么多了数据，比如d，要么既少了又多了数据，比如f。

因此，需要有一种机制来让leader和follower对log达成一致，leader会为每个follower维护一个nextIndex，表示leader给各个follower发送的下一条log entry在log中的index，初始化为leader的最后一条log entry的下一个位置。leader给follower发送AppendEntriesRPC消息，带着(term_id, (nextIndex-1))， term_id即(nextIndex-1)这个槽位的log entry的term_id，follower接收到AppendEntriesRPC后，会从自己的log中找是不是存在这样的log entry，如果不存在，就给leader回复拒绝消息，然后leader则将nextIndex减1，再重复，知道AppendEntriesRPC消息被接收。

以leader和b为例：

初始化，nextIndex为11，leader给b发送AppendEntriesRPC(6,10)，b在自己log的10号槽位中没有找到term_id为6的log entry。则给leader回应一个拒绝消息。接着，leader将nextIndex减一，变成10，然后给b发送AppendEntriesRPC(6, 9)，b在自己log的9号槽位中同样没有找到term_id为6的log entry。循环下去，直到leader发送了AppendEntriesRPC(4,4)，b在自己log的槽位4中找到了term_id为4的log entry。接收了消息。随后，leader就可以从槽位5开始给b推送日志了。

## **Safety**

- 哪些follower有资格成为leader?

> Raft保证被选为新leader的节点拥有所有已提交的log entry，这与ViewStamped Replication不同，后者不需要这个保证，而是通过其他机制从follower拉取自己没有的提交的日志记录

这个保证是在RequestVoteRPC阶段做的，candidate在发送RequestVoteRPC时，会带上自己的最后一条日志记录的term_id和index，其他节点收到消息时，如果发现自己的日志比RPC请求中携带的更新，拒绝投票。日志比较的原则是，如果本地的最后一条log entry的term id更大，则更新，如果term id一样大，则日志更多的更大(index更大)。

- 哪些日志记录被认为是commited?

1. leader正在replicate当前term（即term 2）的日志记录给其它Follower，一旦leader确认了这条log entry被majority写盘了，这条log entry就被认为是committed。如图a，S1作为当前term即term2的leader，log index为2的日志被majority写盘了，这条log entry被认为是commited
2. leader正在replicate更早的term的log entry给其它follower。图b的状态是这么出来的。

**对协议的一点修正**

在实际的协议中，需要进行一些微调，这是因为可能会出现下面这种情况：

![img](https://pic3.zhimg.com/v2-77762ec1707940412736c332376dd0c6_b.jpg)

1. 在阶段a，term为2，S1是Leader，且S1写入日志（term, index）为(2, 2)，并且日志被同步写入了S2；
2. 在阶段b，S1离线，触发一次新的选主，此时S5被选为新的Leader，此时系统term为3，且写入了日志（term, index）为（3， 2）;
3. S5尚未将日志推送到Followers变离线了，进而触发了一次新的选主，而之前离线的S1经过重新上线后被选中变成Leader，此时系统term为4，此时S1会将自己的日志同步到Followers，按照上图就是将日志（2， 2）同步到了S3，而此时由于该日志已经被同步到了多数节点（S1, S2, S3），因此，此时日志（2，2）可以被commit了（即更新到状态机）；
4. 在阶段d，S1又很不幸地下线了，系统触发一次选主，而S5有可能被选为新的Leader（这是因为S5可以满足作为主的一切条件：1. term = 3 > 2, 2. 最新的日志index为2，比大多数节点（如S2/S3/S4的日志都新），然后S5会将自己的日志更新到Followers，于是S2、S3中已经被提交的日志（2，2）被截断了，这是致命性的错误，因为一致性协议中不允许出现已经应用到状态机中的日志被截断。

为了避免这种致命错误，需要对协议进行一个微调：

> 只允许主节点提交包含当前term的日志

针对上述情况就是：即使日志（2，2）已经被大多数节点（S1、S2、S3）确认了，但是它不能被Commit，因为它是来自之前term(2)的日志，直到S1在当前term（4）产生的日志（4， 3）被大多数Follower确认，S1方可Commit（4，3）这条日志，当然，根据Raft定义，（4，3）之前的所有日志也会被Commit。此时即使S1再下线，重新选主时S5不可能成为Leader，因为它没有包含大多数节点已经拥有的日志（4，3）。