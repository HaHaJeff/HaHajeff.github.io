---
?layout: post
title: Designing Data-Intensive Applications——index
category: 读书笔记
tags: [读书笔记]
keywords: B-Tree, LSM-Tree
---

---

## Hash index

如图所示，在内存中维护了一个Hash map

![](https://raw.githubusercontent.com/HaHaJeff/HaHaJeff.github.io/master/img/index/hash_index.png)

- hash index适用于：写操作很多，且key重复度比较低的同时能够在内存中维护一个完整的hash map，例如Bitcask采用hash index，注意**只有追加操作**

  ![](https://raw.githubusercontent.com/HaHaJeff/HaHaJeff.github.io/master/img/index/compaction_hash.png)

- 因为只有**追加操作**，为了防止磁盘耗尽，将log文件划分为data file segment，并对其进行合并

### 几个问题

- file format：采用csv格式不是明智之举，应该采用二进制
- deleting record：采用标记删除法，当data segment file被合并时，才进行真正的删除操作
- crash recovery：可以从log文件重建hash map，但是效率比较低下，可以采用snapshot定期对hash map进行持久化
- partially written records：采用checksum进行数据完整性验证
- concurrency control：采用追加的方式更新文件，所以采用单线程写即可

采用追加写的方式乍看之下浪费了空间，但是：

- 顺序写的方式明显优于随即写，在HDD下犹为明显
- 简化了并发控制
- 简化了crash recovery，例如：采用update in place，如果更新到一般，database宕了，可能出现数据不一致
- merge操作避免了空间浪费

但是，hash index还是有如下缺点：

- hash map需要全部存储在内存中，当数据量很大时，可能无法满足这一条件
- 范围查询很低效

## SSTable and LSM-Trees

SSTbale——sort string table：log segment按照key排序，在每一个已经合并过的log segment中要求key唯一，这样做的优点：

1. 简化merge操作的同时提高效率，采用merge sort即可完成一次merge操作，即使文件大小超过可用内存，也能高效运行

![](https://raw.githubusercontent.com/HaHaJeff/HaHaJeff.github.io/master/img/index/merge_sstable.png)

 	2. 为了找到某一个特定的key，也不需要像hash index一样在内存中维护一张包含全部key的hash map，因为sstable有序，所以只需要存储稀疏的几个key即可完成查找(**越密集效率越高**)

![](https://raw.githubusercontent.com/HaHaJeff/HaHaJeff.github.io/master/img/index/sstable_with_index.png)

3. 可以采用压缩的方式减少存储开销，因为一个key包含了一段数据(例如采用key的前缀压缩)

### constructing and maintain

- 在内存中维护一个memtable，向memtable中写入数据
- 当memtable达到阈值时，将其转换为sstable，该sstable成为最新的
- read请求先在memtable中查找key，如果没有找到，则在最新的sstable中再次查找，以此类推
- 定期在后台执行sstable的compaction操作

上述方案存在的问题：如果database宕了，那么memtable中的数据将丢失，所以WAL非常重要

### 性能优化

- bloomfilters：避免无效的读请求
- compaction操作

## B-Trees

与log-structured index将database数据分为segments不同，B-Tree将database分为不同的page(4KB左右)，每次读写都是按照page进行(为了避免磁盘寻道，贴近底层硬件啊)

![](https://raw.githubusercontent.com/HaHaJeff/HaHaJeff.github.io/master/img/index/btree_index.png)

- 更新一个已有key的value需搜索到叶子节点(难道这不是B+树的特性吗？)

- 添加一个key，找到包含该key的page，在该page中添加一个key，如果page空间不够，则需要分裂成两个page

![](https://raw.githubusercontent.com/HaHaJeff/HaHaJeff.github.io/master/img/index/btree_split.png)

### Making B-trees reliable

- 写操作相当于在B-Tree的page中更新数据
  - 在HDD上，先移动磁头，在完成写入
  - 在SSD上，由于SSD**块擦出页写入特性**，无法对page进行overwrite，只能先擦除在写入，也就是说在SSD中，即使更改了某个页面中的部分内容也不能向SSD那样，而是选择一个空闲的page进行写入，将原来的page标记删除(这就是SSD写放大)
- 在insert操作过程中，可能需要spilt page，并且需要修改parent page的index数据

上述操作可能导致database陷入不一致的状态，所以需要采用WAL

- 并发控制需要涉及latches(lightweight locks，与事务涉及的lock不一样，latch在线程并发访问B-Tree数据结构时使用)

### B-Tree optimizations

- crash recovery不采用overwrite pages和WAL，而是采用COW机制
- 在page中对key进行压缩，减少空间占用
- 分配连续page，例如将leaf page分配在连续的空间
- leaf page采用链表进行串联，避免扫描parent page

## Comparing B-Trees and LSM-Trees

- LSM-trees加速了write，将random write转变为sequential write

- B-Trees加速了read，因为B-Trees的深度比较低，以及page大小与disk适配，这减少了磁头的移动

### Advantages of LSM-trees

相较而言，写性能优秀，B-Tree index的一次写入操作涉及到两次磁盘操作：WAL的日志操作+page dirty

- 因为多次compaction的存在，基于log-strutured index写入操作会导致写放大(写放大指：一条数据被多次写入)，这对SSD影响比较大
- 负载多为写请求的场景下，磁盘带宽将成为瓶颈：
  - 写放大导致用于处理写请求的磁盘带宽有限，假设磁盘顺序写的速度为200MB/s，然而真正写入的速度只有10MB/s~20MB/s，其余的带宽都在用于compaction

除去上述两点。LSM-Tree相比于B-Tree表现了更好的写吞吐，与此同时，B-Tree还会面临碎片问题(LSM-Tree由于append以及compact操作不会面临该问题)

### Downsides of LSM-Trees

- compaction可能会干扰到read和write的性能
- 当compaction的速度无法匹配write速度的时候，read操作延时会增加(因为SSTable太多)
- 基于LSM-Trees无法很好的保证事务
