---
?layout: post
title: percolator
category: 论文阅读
tags: [论文阅读]
keywords: 2pc, bigtable
---

---


# 分布式事务

- 单机事务

  ```
  BEGIN TRANSACTION
  SELECT something FROM myTable
  UPDATE something IN myTable
  COMMIT
  ```

- 跨界点事务

  ```
  BEGIN TRANSACTION
  UPDATE amount = amount - 100 IN bankAccounts WHERE accountNr = 1
  UPDATE amount = amount + 100 IN someRemoteDatabaseAtSomeOtherBank.bankAccounts WHERE accountNr = 2
  COMMIT
  ```

  **跨界点事务**即为分布式事务，难点如何在跨网络的环境下确认**line 3**正确发生，从而确保跨界点事务的**原子性**

  ## 两阶段提交
  <img src="https://raw.githubusercontent.com/HaHaJeff/HaHaJeff.github.io/master/img/percolator/2pc.jpg" width="100%" height="100%">


- prepare阶段：向所有参与者发送prepare请求，参与者记录redo日志，若参与者执行成功，向协调者返回成功，否者返回失败。
- commit阶段：若参与者的应答都为成功，协调者完成事务的commit，持久化事务状态，并向所有参与者发送commit请求，否则发送abort请求。当参与者收到协调者发送的commit/abort消息后，记录事务commit/abort信息。

## 2pc终止协议

- 考虑参与者2正在等待协调者的commit/abort请求，假定参与者2的ACK信息是YES
- 参与者2向参与者1询问
  - 没有收到参与者1的回复，参与者2需要等继续待协调者（如参与者1宕机或网络分区）
  - 参与者1收到了协调者的commit/abort请求，参与者2同理执行commit/abort
  - 参与者1没有向协调者发送ACK或ACK信息为NO，参与者1与参与者2同时执行abort，因为此时协调者上还没有决定事务状态commit/abort
  - 参与者1的ACK信息为YES，此时必须等待协调者的commit/abort请求
    - 如果协调者收到了参与者1与参与者2的ACK，则执行commit
    - 如果超时，abort

## 2pc局限

- 协调者宕机，重新开始工作后查看持久化设备上是否记录事务状态信息，如有，按照记录的状态信息发送，否则，发送abort请求
- 参与者宕机
  - 如果参与者查看持久化设备上记录的ACK信息是NO或ACK为null，则abort事务
  - 如果参与者查看持久化设备上记录的ACK信息是YES，此时需要走2pc终止协议

- 交互延迟
  - 参与者在收到协调者的prepare请求后，ACK前，持久化ACK信息
  - 协调者在收到所有的参与者的ACK信息后，发送commit/abort信息前，持久化事务状态信息
  - 参与者在收到协调者的commit请求后，执行事务的commit/abort前，持久化事务状态信息

## 故障场景

1. 参与者在发送ACK应道之前发生故障，此时重启后的参与者发现并没有持久化ACK应答信息，需要向协调者进行**重确认**

<img src="https://raw.githubusercontent.com/HaHaJeff/HaHaJeff.github.io/master/img/percolator/1.png" width="65%" height="65%">

2. 参与者在发送ACK应答后发生故障，协调者定时向宕机后的参与者发送commit请求

<img src="https://raw.githubusercontent.com/HaHaJeff/HaHaJeff.github.io/master/img/percolator/2.png" width="65%" height="65%">

3. 协调者丢失了一个参与者的ACK应答，超时后，直接向参与者发送abort请求，重启后的参与者向协调者进行**重确认**

<img src="https://raw.githubusercontent.com/HaHaJeff/HaHaJeff.github.io/master/img/percolator/3.png" width="65%" height="65%">

4. 协调者在发送prepare请求之前宕机，重写后重新开始2pc

<img src="https://raw.githubusercontent.com/HaHaJeff/HaHaJeff.github.io/master/img/percolator/4.png" width="65%" height="65%">

5. 协调者在发送prepare请求后宕机，此时协调者并没有持久化事务状态，所以需要向参与者发送abort请求，然后在重新开始进行2pc

<img src="https://raw.githubusercontent.com/HaHaJeff/HaHaJeff.github.io/master/img/percolator/5.png" width="65%" height="65%">

6. 协调者在收到ACK应答后宕机，此时协调者并没有持久化事务状态，所以需要向参与者发送abort请求，然后在重新开始进行2pc

<img src="https://raw.githubusercontent.com/HaHaJeff/HaHaJeff.github.io/master/img/percolator/6.png" width="65%" height="65%">

7. 协调者在发送部分commit/abort请求后宕机，参与者与协调者进行**重确认**

<img src="https://raw.githubusercontent.com/HaHaJeff/HaHaJeff.github.io/master/img/percolator/7.png" width="65%" height="65%">

8. 场景7解决方案2，参与者在与协调者通信超时后，可以向其他参与学习协调者的事务状态

<img src="https://raw.githubusercontent.com/HaHaJeff/HaHaJeff.github.io/master/img/percolator/8.png" width="65%" height="65%">

## percolator

2pc的局限性在于**协调者需要持久化事务状态**，**percolator**则是在bigtable行事务的基础上，为每一个Cell增加了两个属性：write以及lock

<img src="https://raw.githubusercontent.com/HaHaJeff/HaHaJeff.github.io/master/img/percolator/write_lock.png" width="65%" height="65%">

percolator在bigtable上实现分布式事务过程如下：

1. 初始状态，Bob账户中余额为10，Joe账户余额为2

<img src="https://raw.githubusercontent.com/HaHaJeff/HaHaJeff.github.io/master/img/percolator/init.png" width="65%" height="65%">

2. 事务开始，首先向TSO（全局授时服务器）获取start_ts，然后在Bob的lock字段进行写入，完成上锁，最后在data段完成数据更新

<img src="https://raw.githubusercontent.com/HaHaJeff/HaHaJeff.github.io/master/img/percolator/one.png" width="65%" height="65%">

3. 完成Joe的锁定以及数据写入，**Joe的lock字段写入的是Bob的位置信息**

<img src="https://raw.githubusercontent.com/HaHaJeff/HaHaJeff.github.io/master/img/percolator/two.png" width="65%" height="65%">

4. commit point，首先Bob向TSO请求一个commit_ts，擦除lock信息，随后在write字段写入一个指针：指向timestamp=7的数据

<img src="https://raw.githubusercontent.com/HaHaJeff/HaHaJeff.github.io/master/img/percolator/three.png" width="65%" height="65%">

5. 在Joe上执行一样的操作

<img src="https://raw.githubusercontent.com/HaHaJeff/HaHaJeff.github.io/master/img/percolator/four.png" width="65%" height="65%">

### pwrite

<img src="https://raw.githubusercontent.com/HaHaJeff/HaHaJeff.github.io/master/img/percolator/get.png" width="65%" height="65%">

1. 在事务开启时会从 TSO 获取一个 timestamp 作为 start_ts。
2. 选择一个 Write 作为 primary，其它 Write 则是 secondary；primary 作为事务提交的 sync point 来保证故障恢复的安全性。
3. 先 Prewrite primary，成功后再 Prewrite secondaries；先 Prewrite primary 是 failover 的要求。 对于每一个 Write:
   1. Write-write conflict 检查： 以 Write.row 作为 key，检查 Write.col 对应的 write 列在 [start_ts, max) 之间是否存在相同 key 的数据 。如果存在，则说明存在 write-write conflict ，直接 Abort 整个事务。
   2.  检查 lock 列中该 Write 是否被上锁，如果锁存在，Percolator 不会等待锁被删除，而是选择直接Abort 整个事务。这种简单粗暴的冲突处理方案避免了死锁发生的可能。​
   3. 步骤 3.1，3.2 成功后，以 start_ts 作为 Bigtable 的 timestamp，将数据写入 data 列。由于此时 write 列尚未写入，因此数据对其它事务不可见。​
   4. 对 Write 上锁：以 start_ts 作为 timestamp，以 {primary.row, primary.col} 作为 value，写入 lock列 。{Priamry.row, Primary.col} 用于 failover 时定位到 primary Write。

对于一个 Writer，3 中多个步骤都是在同一个 Bigtable 单行事务中进行，保证原子性，避免两个事务对同一行进行并发操作时的 race；任意 Writer Prewrite 失败都会导致整个事务 Abort；Prewrite 阶段写入 data 列的数据对其它事务并不可见。

### commit

<img src="https://raw.githubusercontent.com/HaHaJeff/HaHaJeff.github.io/master/img/percolator/commit.png" width="65%" height="65%">

如果 Prewrite 成功，则进入 Commit 阶段：

1. 从 TimeOracle 获取一个 timestamp 作为 commit_ts。
2. 先 Commit primary , 如果失败则 Abort 事务：
   1. 检测 lock 列 primary 对应的锁是否存在，如果锁已经被其它事务清理（触发 failover 可能导致该情况），则失败。
   2. 步骤 2.1 成功后以 commit_ts 作为 timestamp，以 start_ts 作为 value 写 write 列。读操作会先读 write 列获取 start_ts，然后再以 start_ts 去读取 data 列中的 value。​
   3. 删除 lock列中对应的锁。
3. 步骤 2 成功意味着事务已提交成功，此时 Client 可以返回用户提交成功，再异步的进行 Commit secondary。 Commit secondary 无需检测 lock 列锁是否还存在，一定不会失败，只需要执行 2.2 和 2.3。

## Fail over

对于分布式事务一个完备的 failover 机制要求满足两点：

1. Safety：针对 Percolator场景即是同一个事务的两个 Write，不存在一个 Write 的状态为 Committed，另一个则是 Aborted。
2. Liveness: 所有 Write 最终都会处于 Committed 或者 Aborted。

Liveness 要求 failover 最终能够被触发，对此 Percolator 采用 lazzy 的方式，一个事务的 failover 由另一个事务的读操作触发：如果一个事务读某个 key 时发现该 key 被锁住，Snapshot-read 要求等待锁删除后才能发起读取；在等待超过一段时间后，会认为锁住该 key 的事务可能由于 Client crash 或者网络分区等原因而 hang 住，尝试通过回滚或者继续提交锁对应的事务的方式来清理锁。

对于一个分布式事务，保证 Safety 并不容易；针对 Percolator 的场景，由于异步网络的存在，永远都无法确定 Client 是否真正 crash。Failover 机制需要处理 failover 和 Client 的正常提交流程并发执行的问题，需要避免发生 failover 触发导致事务 T1 回滚，但是实际上存活的 Client 则继续提交 T1。在异步网络下，这样的 race 需要处理的状态空间通常是比较庞大，在任意情况下保证 Safety 并不容易。

保证 Safety 的一个核心点是如何判定一个分布式事务是否已经被提交：即什么情况下，可以判定事务已 Committed，什么情况下判定事务未 Committed，可以发起 rollback。Percolator 的解法是选择一个 Write 作为 primary，以 primary 作为整个事务的 sync point。具体的：

1. 在 Prewrite 阶段先 Prewrite primary
2. 在 Commit 阶段先 Commit primary。
3. Primary committed 意味着事务 Committed；在 failover 触发时，尝试清理事务 T1 的某个 Write T1.w1 的锁之前会先确认 T1 primary Write 的状态；如果判定 T1 primary Write 未 Committed，则判定 T1 未Committed，会执行 rollback：先清理 primary lock，然后在清理 T1.w1 的锁。如果 判定 T1 primary Write 已 Committed，则判定 T1 已 Committed，执行 roll-forward：从 primary 获取 commit_ts，提交 T1.w1，也达到清理锁的目的。

针对 3 展开描述，具体的：

1. 通过 lock 列存储的 T1.primary 的信息 {primary.row, primary.col} ，并获取 T1.start_ts。

2. 通过 {primary.row, primary.col, T1.start_ts} 查询 lock 列上 T1.primary 的锁是否存在，如果锁存在那么 T1 一定未 Committed，可以执行 rollback，先清理 T1.primary 的锁，再清理 T1.w1 的锁。检查 lock 列上 primary 锁是否存在和清理 T1.primary 的锁在一个 Bigtable 单行事务中执行，保证原子性。 由于先清理 primary 锁，即使此时 Client 此时已经 Prewrite 成功，进入 Commit 阶段， Commit primary 也会由于锁已被清理而失败（见 Commit 流程步骤2）。

3. 如果步骤 2 判定 primary 的 lock 列不存在，需要考虑如下三种可能：

   a. primary 未 Prewrite

   b. primary 已 Committed

   c. primary 已 Aborted

   在 Prewrite 阶段先写 primary 确保 a 不可能发生，T1.w1 存在意味着 primary 必定 Prewrite 成功。笔者并未发现 Percolator 论文关于区分 b, c 的细节，也未发现 Abort 时所执行的动作的细节。故下面是笔者按自己的理解补充的可行方案：此时需要去检查 primary 的 write 列是否存在，由于此时并不知道 commit_ts，因此需要：检查 write 列 [T1.start_ts, max] 范围内是否存在 row 相同且 value 等于 T1.start_ts 的数据。由于 start_ts 的唯一性，存在即可判定 primary 的 write 列存在，即 T1 已 Committed，不存在则认为 T1 已经 Aborted。这也意味着 Percolator 在 Abort 时可以只清理 lock 列，无需持久化一条额外的 Aborted 记录。

4. 如果步骤 3 判定 T1 已 Committed，那么需要执行 roll-forward，从 primary 的 write 列获取 T1 的 commit_ts，针对 T1.w1 写入 write 列后清理锁；如果步骤 3 判定 T1 未 Committed，进行 rollback ，过程同步骤2。

# 参考

[1]. [what is a distributed transaction](https://stackoverflow.com/questions/4217270/what-is-a-distributed-transaction)

[2]. [two phase commit](https://www.cs.princeton.edu/courses/archive/fall16/cos418/docs/L6-2pc.pdf)

[3]. [Database · 原理介绍 · Google Percolator 分布式事务实现原理解读](<http://mysql.taobao.org/monthly/2018/11/02/>)



