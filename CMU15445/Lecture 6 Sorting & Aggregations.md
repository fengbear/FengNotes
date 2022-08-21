### Lecture 6 Sorting & Aggregations



我们开始学习 Operator Execution 执行引擎部分。这次我们关注排序和聚合是如何在数据库中完成的。同时也思考，如果内存容量不够，如何进行排序和聚合。

#### 1 Sort

> We need sorting because in the relation model, **tuples in a table have no specific order Sorting** is (potentially) used in *ORDER BY*, *GROUP BY*, *JOIN*, and *DISTINCT* operators.
>
> We can accelerate sorting using a **clustered B+tree** by scanning the leaf nodes from left to right. This is a bad idea, however, if we use an unclustered B+tree to sort because it causes a lot of I/O reads (random access through pointer chasing).

![7.jpg](C:/Users/xf/Desktop/CMU15445/pictures/7-16610732510571.jpg)

![8.jpg](C:/Users/xf/Desktop/CMU15445/pictures/8-16610732510583.jpg)

**quick-sort 只适用在纯内存环境下**. 因为如果在磁盘环境下，每次选取 pivot, 不管我们是随机选还是选中位数 (median) 都非常可能会引起 random I/O。这会导致性能非常低效。

##### 1.1 External Merge Sort

>  Divide-and-conquer sorting algorithm that splits the data set into separate *runs* and then sorts them individually. It can spill runs to disk as needed then read them back in one at a time.

![9.jpg](C:/Users/xf/Desktop/CMU15445/pictures/9-16610732510585.jpg)

##### 1.2 2-Way External Merge Sort

![10.jpg](C:/Users/xf/Desktop/CMU15445/pictures/10-16610732510587.jpg)

- *N*, *B* 都是已知的。

###### Example

**Pass #0: Sort Runs (in-place sort)**

下面几张图代表 Pass #0：

+ 每次从硬盘上读入 *B* 个 page 进入我们固定大小的内存缓存区。在内存中对它们排序，再将这些数量的 page 写回硬盘。(例子中写回在硬盘的另外一个位置，但是实际上可以写回同一个位置，去覆盖未排序的数据。)

 ![11.jpg](C:/Users/xf/Desktop/CMU15445/pictures/11-16610732510589.jpg)

![12.jpg](C:/Users/xf/Desktop/CMU15445/pictures/12-166107325105811.jpg)

![13.jpg](C:/Users/xf/Desktop/CMU15445/pictures/13-166107325105813.jpg)

![14.jpg](C:/Users/xf/Desktop/CMU15445/pictures/14-166107325105815.jpg)

![15.jpg](C:/Users/xf/Desktop/CMU15445/pictures/15-166107325105817.jpg)



**Pass #1,2,3…: Merge Runs (non in-place sort)**

然后开始 merge， 我们需要**至少 3 个 page** 在内存中：

- 从硬盘上读入 2 个 page，然后剩一个 page 是缓存最终排序结果并准备写入硬盘。

![16.jpg](C:/Users/xf/Desktop/CMU15445/pictures/16-166107325105819.jpg)

![17.jpg](C:/Users/xf/Desktop/CMU15445/pictures/17-166107325105821.jpg)

![18.jpg](C:/Users/xf/Desktop/CMU15445/pictures/18-166107325105823.jpg)

###### 总结

![19.jpg](C:/Users/xf/Desktop/CMU15445/pictures/19-166107325105825.jpg)

- 上图中 Number of passes 中的 `1`: *Pass #0*
- 上图中 Total I/O cost 中的 `2`: 每次循环都需要从硬盘中读出和写入进硬盘。

![20.jpg](C:/Users/xf/Desktop/CMU15445/pictures/20-166107325105927.jpg)

- 上图中，即使有更多的缓存区，也不能帮助 2-Way External Merge Sort， 最后造成的 I/O 次数依然是一样的。

##### 1.3 Double Buffering Optimization

> **Prefetch the next run in the background** and store it in a second buffer while the system is processing the current run. **This reduces the wait time for I/O requests** at each step by continuously utilizing the disk.

Double Buffering Optimization 主要让 I/O 一直在操作，一个线程在排序的时候，**另外一个线程**依然在做异步I/O - prefetching。这样让硬盘一直在进行 I/O 操作，隐藏 **hide the I/O latency**（时延）。

下图中的例子中:

- Memory 中上面的是当前 sort 的 run (Page#1)
- Memory 中下面那个是 prefetching 的用于下一次的 run (Page#2)

![21.jpg](C:/Users/xf/Desktop/CMU15445/pictures/21-166107325105929.jpg)

##### 1.4 General (K-way) Merge Sort

![24.jpg](C:/Users/xf/Desktop/CMU15445/pictures/24-166107325105931.jpg)

- 上图中的 `B-1`: Merge 时 B-1 个 input page, 1 个 output page

![25.jpg](C:/Users/xf/Desktop/CMU15445/pictures/25-166107325105933.jpg)

+ Pass #0：22轮

##### 1.5 Using B+ Tree For Sorting - Clustered B+ Tree

如果 SQL 要求一个字段 key，而这个字段正好是 B+ Tree 索引的 key。那么这个顺序我们可以直接从索引中获得。

![26.jpg](C:/Users/xf/Desktop/CMU15445/pictures/26-166107325105935.jpg)

###### 1.5.1 Case1：Clustered B+ Tree

**Physical location of tuple on page matches sort order in the index.**

![27.jpg](C:/Users/xf/Desktop/CMU15445/pictures/27-166107325105937.jpg)

###### 1.5.2 Case2：Clustered B+ Tree

这个情况非常不好，下一个 tuple 很可能被存储在另外一个无关联的 page, 寻找它变成了一次 random I/O。如果每一次寻找一个 tuple,　都会是一次 random I/O 的话，性能会非常垃圾。

![28.jpg](C:/Users/xf/Desktop/CMU15445/pictures/28-166107325105939.jpg)

#### 2 Aggregations

Aggregations 对应 SQL 中的关键字:

- `MIN, MAX, COUNT ...`
- `GROUP BY`
- `DISTINCT`
- …

![29.jpg](C:/Users/xf/Desktop/CMU15445/pictures/29-166107325105941.jpg)

##### 2.1 Sorting Aggregate

DBMS首先按GROUP BY键对元组进行排序。如果所有数据都能在缓冲池，它可以使用内存内排序算法(例如，快速排序)，如果数据超出内存，它可以使用外部归并排序算法。

然后DBMS对排序后的数据执行顺序扫描以计算聚合。运算符的输出将按键排序。

![30.jpg](C:/Users/xf/Desktop/CMU15445/pictures/30-166107325105943.jpg)

![31.jpg](C:/Users/xf/Desktop/CMU15445/pictures/31-166107325105945.jpg)

![32.jpg](C:/Users/xf/Desktop/CMU15445/pictures/32-166107325105947.jpg)

![33.jpg](C:/Users/xf/Desktop/CMU15445/pictures/33-166107325105949.jpg)

Sorting is expensive

但很多时候我们并不需要排好序的数据，如：

- Forming groups in GROUP BY
- Removing duplicates in DISTINCT

在这样的场景下 hashing 是更好的选择，它能有效减少排序所需的额外工作。

##### 2.2 Hashing Aggregate

利用一个临时 (ephemeral) 的 hash table 来记录必要的信息，即检查 hash table 中是否存在已经记录过的元素并作出相应操作：

- DISTINCT: Discard duplicate
- GROUP BY: Perform aggregate computation

如果所有信息都能一次性读入内存，那事情就很简单了，但如若不然，我们就得变得更聪明。

hashing aggregation 同样分成两步：

- Partition Phase: 将 tuples 根据 hash key 放入不同的 buckets
  - use a hash function h1 to split tuples into partitions on disk
    - all matches live in the same partition
    - partitions are "spilled" to disk via output buffers
  - 这里有个额外的假设，即每个 partition 能够被放到 memory 中
- ReHash Phase: 在内存中针对每个 partition 利用 hash table 计算 aggregation 的结果

###### 2.2.1 External Hashing Aggregate

![35.jpg](C:/Users/xf/Desktop/CMU15445/pictures/35-166107325105951.jpg)

**Phase #1 – Partition**

> Use a hash function h1 to split tuples into partitions on disk based on target hash key. This will put all tuples that match into the same partition. The DBMS spills partitions to disk via output buffers.

![36.jpg](C:/Users/xf/Desktop/CMU15445/pictures/36-166107325105955.jpg)

![37.jpg](C:/Users/xf/Desktop/CMU15445/pictures/37-166107325105953.jpg)

Partition 结束之后，各个 B-1 个 buffer 里面的 tuple 都有相同的 hash value。通过 Partition 我们获得这种 **locality**，我们获得了不同的 buffer, 而**每一个 buffer page 里面的值有非常大的可能是相同，或者相似，或者有相似特征的**。

**Phase #2 – ReHash**

前后两个 hash function: h1, h2 不需要相同。

![38.jpg](C:/Users/xf/Desktop/CMU15445/pictures/38-166107325105957.jpg)

![39.jpg](C:/Users/xf/Desktop/CMU15445/pictures/39-166107325105959.jpg)

![40.jpg](C:/Users/xf/Desktop/CMU15445/pictures/40-166107325105961.jpg)

![41.jpg](C:/Users/xf/Desktop/CMU15445/pictures/41-166107325105963.jpg)

![42.jpg](C:/Users/xf/Desktop/CMU15445/pictures/42-166107325105965.jpg)

![43.jpg](C:/Users/xf/Desktop/CMU15445/pictures/43-166107325105967.jpg)

![45.jpg](C:/Users/xf/Desktop/CMU15445/pictures/45-166107325106069.jpg)

结束之后 (以及在 ReHash 的过程中)，中间的 hash table 是暂时的，如果用完了它，就可以直接销毁 / 释放。

**Hashing Summarization**

在 ReHash phase 中，存着 (GroupKey→RunningVal)(GroupKey  \rightarrow RunningVal)(GroupKey→RunningVal) 的键值对，当我们需要向 hash table 中插入新的 tuple 时：

- 如果我们发现相应的 GroupKey 已经在内存中，只需要更新 RunningVal 就可以
- 反之，则插入新的 GroupKey 到 RunningVal 的键值对

![46.jpg](C:/Users/xf/Desktop/CMU15445/pictures/46-166107325106071.jpg)

Hashing Summarization 可以帮助我们计算最后的 Aggregate Function: 比如 `sum, count, avg, min, max`。下面例子中的 `value` 是 `(running count, running sum)`：

![47.jpg](C:/Users/xf/Desktop/CMU15445/pictures/47-166107325106073.jpg)

![48.jpg](C:/Users/xf/Desktop/CMU15445/pictures/48-166107325106075.jpg)

![49.jpg](C:/Users/xf/Desktop/CMU15445/pictures/49-166107325106077.jpg)

**Cost Analysis**

使用 hashing aggregation 可以聚合多大的 table ？假设有 B 个 buffer pages

- Phase #1：使用 1 个 page 读数据，B-1 个 page 写出 B-1 个 partition 的数据
- 每个 partition 的数据应当小于 B 个 pages （第二步读取的时候希望一次性能把一个分区的读到内存中来，所以每个分区的数据应该小于B个pages）

因此能够聚合的 table 最大为$B \times (B-1)$ 

- 通常一个大小为 N pages 的 table 需要大约 $\sqrt{N}$ 个 buffer pages

