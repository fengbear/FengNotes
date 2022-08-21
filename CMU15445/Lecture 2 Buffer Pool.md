### Lecture 2 Buffer Pool



上节中提到，DBMS 的磁盘管理模块主要解决两个问题：

1. 如何使用磁盘文件来表示数据库的数据（元数据、索引、数据表等）
2. （本节）如何管理数据在内存与磁盘之间的移动

本节将讨论第二个问题。管理数据在内存与磁盘之间的移动又分为两个方面：空间控制（Spatial Control）和时间控制（Temporal Control）

**Spatial Control**

空间控制策略通过决定将 pages 写到磁盘的哪个位置，使得常常一起使用的 pages 能离得更近，从而提高 I/O 效率。

**Temporal Control**

时间控制策略通过决定何时将 pages 读入内存，写回磁盘，使得读写的次数最小，从而提高 I/O 效率。

整个 big picture 如下图所示：

![img](C:/Users/xf/Desktop/CMU15445/pictures/assets%252F-LMjQD5UezC9P8miypMG%252F-LZUa28zEosfmp4uFA77%252F-LZUJRryGLyHUdcOnhD0%252FScreen%20Shot%202019-02-24%20at%207.43.51%20PM.jpg)

本节的提纲如下：

+ Buffer Pool Manager
+ Replacement Policies
+ Allocation Policies
+ Other Memory Pools

#### 1 Buffer Pool Manager

##### 1.2 Buffer Pool Organization

和操作系统中的概念一样，我们将硬盘上的 block 称为 page, 将内存 (buffer pool) 中的 block 称为 frame，frame 可以容纳 page。

`pin(int pageid)` 这个函数就代表了之前的 `get page` 这个操作。这个函数之后我们可以获得一个指向这个 page 数据的指针。我们使用完这个 page 之后，需要 `unpin(int pageid)`。

在下面 3 张图中：

- 一开始 buffer pool 是空的
- 然后我们 `pin(pageid 1)`, page1 被加载进入 buffer pool
- 然后我们 `pin(pageid 3)`, page3 被加载进入 buffer pool
- 我们会在后面看到一个 page table，他负责告诉我们 page 和 frame 的对应，例如：page1 在 frame1, page3 在 frame2

![img](C:/Users/xf/Desktop/CMU15445/pictures/10.jpg)

![11.jpg](C:/Users/xf/Desktop/CMU15445/pictures/11.jpg)

![12.jpg](C:/Users/xf/Desktop/CMU15445/pictures/12-16610731140644.jpg)

##### 1.2 Buffer Pool Metadata

们在这里讨论缓存区的 metadata, 它们是管理区的管理信息。

- page table (也和操作系统中概念类似，只是不需要 MMU, TLB) 是一个 in-memory hashtable, 负责 `page id -> frame id` 的映射，例如：page1 在 frame1, page3 在 frame2。
- dirty flage: 告诉我们一个 buffer pool 中的 frame 是否被改写。如果被改写，即 page 上出现新数据，需要被写回至硬盘。
- pin /reference counter: 对一个 page 当前使用的 thread 数进行计数。我们只允许将此数字为 0 的 page 写回硬盘，即当前没有任何 thread 在使用此 page。

下面的例子中很生动地表达了，如何在多线程的情况下，向 buffer pool 中加载一个新的 page:

- 首先我们 `pin(pageid 2)`，但是我们在 page table 中找不到对应，这时是一个 page fault, 说明我们需要从硬盘中加载这个 page2
- 这时我们给 page table 上一个 global latch，确保我们在整个过程中，没有其他线程会改变这个 page table (比如加载 page, 移除 page)
- 确认获得 global latch 之后，我们向 buffer pool 加载这个 page2, 同时也更新 page table 等 metadata
- 整个过程结束后，我们松开 global latch。这样整个过程中，只有我们这个当前线程更改了 page table。
- 其他 metadata 是否也需要线程安全，取决于具体实现。

##### 1.3 Locks vs. Latches 

- locks 是一个 high level 很抽象的概念，它直接和 transaction 事务相关，应用于事务的 lock protocol 中。是应用层面的。是逻辑内容的互斥，比如行，表，事务。
- latch 基本上和 low level 实现相关，指 `std::mutex` 这类和代码相关的类 (class)，来保护代码中的 critical section。是应用不可见的，内部数据的互斥。
- (这部分我们以后还会提到)

> - Locks: 更高级的逻辑原语，暴露给开发人员（可以看到此时持有哪些锁）
>
>   保护数据更高级的逻辑内容，例如元组，表格或者数据库
>
>   事务运行时持有锁
>
>   需要考虑回滚（*还是有点疑惑，以后谈并发控制的时候会再做说明*）
>
> - Latches: 底层的保护原语，可以使用自旋锁
>
>   保护DBMS内部实现时的临界区，包括贡献内存等，协调DBMS内部的多线程
>
>   操作执行时持有锁。例如更新页表某一项时，先申请锁，防止被其他线程修改
>
>   不需要考虑回滚（如果没申请到锁，就终止操作）

[![18.jpg](C:/Users/xf/Desktop/CMU15445/pictures/18.jpg)](https://cakebytheoceanluo.github.io/images/CMU1544564/Lec05/18.jpg)

##### 1.4 Page Table vs. Page Directory

- page directory是一个特殊的 page, 记录了 page 对应文件的位置，因此这个 page directory 也会以 page 的方式从硬盘读取到内存，最后从内存写回至硬盘。即它是持久化的 (persistent, durable)。
- page table 只是一个 `page id -> frame id` 的映射，具体是一个 `page id -> frame pointer` 的 hash table，指针的位置只是在本次启动有效，因为关机重启后，我们会 allocate 另一块内存区域给数据库 buffer pool。这类完全依赖内存的数据结构 (in-memory data structure) 没有意义去存到硬盘上。

##### 1.5 Allocation Policies

- global policies: 指所有 query, transaction 使用同一个 buffer pool 的 replacement policy
- local policies: 指每一个 query, transaction 可以有一个自己的 buffer pool， 和自己的 replacement policy。同时可以在一个共有的 buffer pool 去 share pages
- 实际上会混合这两种主意， 比如数据 table 拥有一个 buffer pool, index (索引数据结构) 拥有另一个 buffer pool。这个我们会在 index 的课上提到。

![20.jpg](C:/Users/xf/Desktop/CMU15445/pictures/20.jpg)

#### 2 Buffer Pool Optimizations

![img](C:/Users/xf/Desktop/CMU15445/pictures/21-16610731140658.jpg)

##### 2.1 Multiple Buffer Pools

为了减少并发控制的开销以及利用数据的 locality，DBMS 可能在不同维度上维护多个 Buffer Pools：

- 多个 Buffer Pools 实例，相同的 page hash 到相同的实例上
- 每个 Database 分配一个 Buffer Pool
- 每种 Page 类型一个 Buffer Pool

例如：

- Approach #2: Hashing用一个 hash function: `RID -> buffer pool id`, 把有相同 hash value 的 tuple 分配到同一个 buffer pool，提高 buffer pool 中的 spatial locality (空间局部性)。(类似 hash join)

![Lec5_missing_1.jpg](C:/Users/xf/Desktop/CMU15445/pictures/missing_1.png)

![Lec5_missing_2.jpg](C:/Users/xf/Desktop/CMU15445/pictures/missing_2.png)]

[![Lec5_missing_3.jpg](C:/Users/xf/Desktop/CMU15445/pictures/missing_3.png)](https://cakebytheoceanluo.github.io/images/CMU1544564/Lec05/missing_3.png)

##### 2.2 Pre-Fetching

我们可以预加载一些 page, 一个 thread 在对 page A 进行扫描计算的同时，另外一个 thread 可以将下一个 page B 加载到 buffer pool 中。这个方法，是用猜测或者预先的只是，去加载未来可能用得到的 page, 最终目的减少从硬盘加载的时间。具体有两种情况，根据这个**下一个 page** 是哪一个:

- Sequential Scans: 即物理上连续存储的下一个 page
- Index Scans

###### 2.2.1 Sequential Scans

![img](C:/Users/xf/Desktop/CMU15445/pictures/23-166107311406513.jpg)

![24.jpg](C:/Users/xf/Desktop/CMU15445/pictures/24.jpg)

Q1 在扫描 page1 的时候，buffer pool manager 发现这是一个 sequenial scan，决定 pre fetch 后面的 page，所以在同时将 page2,3 加载进 buffer pool。理想的话，等 Q1 结束对 page1 的扫描，page2,3 就已经在 buffer pool，相当于没有收到 page fault 的影响，没有浪费时间等待。

![25.jpg](C:/Users/xf/Desktop/CMU15445/pictures/25.jpg)

![26.jpg](C:/Users/xf/Desktop/CMU15445/pictures/26.jpg)

![27.jpg](C:/Users/xf/Desktop/CMU15445/pictures/27.jpg)

[![28.jpg](C:/Users/xf/Desktop/CMU15445/pictures/28.jpg)](https://cakebytheoceanluo.github.io/images/CMU1544564/Lec05/28.jpg) 

###### 2.2.2 Index Scans

这里的 index 是一个 B+ Tree， 具体作法是从根节点查找至叶子节点，再水平遍历有关的叶子节点。这时候我们需要的 page 不一定是物理上连续存储的：

![29.jpg](C:/Users/xf/Desktop/CMU15445/pictures/29.jpg)

![30.jpg](C:/Users/xf/Desktop/CMU15445/pictures/30-166107311406521.jpg)

![31.jpg](C:/Users/xf/Desktop/CMU15445/pictures/31.jpg)

[![32.jpg](C:/Users/xf/Desktop/CMU15445/pictures/32.jpg)](https://cakebytheoceanluo.github.io/images/CMU1544564/Lec05/32.jpg)

下图中很明显表达，我们接下来需要的是 page3 和 page5, 它们不是连续的，也不紧跟在 page1 (当前 page) 之后。 但是 index 这个数据结构告诉我们，下一步是 page3 和 page5 被需要，因此去 pre fetch 它们俩。这里也是一个很明显的例子，操作系统在这一点帮不了我们，操作系统最多能帮我们 pre fetch 连续的 page，但是它不知道 index 的存在，没有办法帮助我们准确去得到 page3 和 page5。

![33.jpg](C:/Users/xf/Desktop/CMU15445/pictures/33-166107311406525.jpg)

##### 2.3 Scan Sharing

利用某个查询从磁盘中读取的数据，并将该数据用于其他查询中。这跟结果缓存（result caching）不一样，结果缓存只是对相同查询的缓存起来，下次执行同样查询的时候就直接把以前的答案给你罢了。扫描共享允许多个不同的查询共用一个table扫描的游标（cursor）。

例子：

Q1 需要扫描所有的 page, 将 val 不断累加。

![36.jpg](C:/Users/xf/Desktop/CMU15445/pictures/36-166107311406527.jpg)

![37.jpg](C:/Users/xf/Desktop/CMU15445/pictures/37-166107311406529.jpg)

![38.jpg](C:/Users/xf/Desktop/CMU15445/pictures/38.jpg)

![39.jpg](C:/Users/xf/Desktop/CMU15445/pictures/39-166107311406532.jpg)

在 Q1 扫描至 page3 的时候，Q2 加入。很巧，Q2 也需要扫描所有的 page, 将 val 不断累加。这说明 Q2 已加入就可以使用 Q1 的中间结果，并且可以和 Q1 结合在一起扫描剩下的 page：
![40.jpg](C:/Users/xf/Desktop/CMU15445/pictures/40.jpg)

![41.jpg](C:/Users/xf/Desktop/CMU15445/pictures/41.jpg)

![42.jpg](C:/Users/xf/Desktop/CMU15445/pictures/42.jpg)

Q1 结束后， 这个例子中，Q2 还需要扫描它没有扫描过的 page， 因为 Q2 计算平均值， 需要知道每一个 page 上 tuple 的个数。

![43.jpg](C:/Users/xf/Desktop/CMU15445/pictures/43.jpg)

![44.jpg](C:/Users/xf/Desktop/CMU15445/pictures/44.jpg)

![45.jpg](C:/Users/xf/Desktop/CMU15445/pictures/45.jpg)

我们也想象一下，如果没有 scan sharing 这个 feature， Q1 和 Q2 相互竞争 buffer pool 会导致这两个 Query 性能都很差。

##### 2.4 Buffer Pool Bypass

sequantial scan 会将大部分 page 加载进 buffer pool, 而这些 page 又并不一定在未来会被重复利用。对这种 sequantial scan 的 Query，我们可以单独给 allocate 一块内存区域，而独立于且不影响 buffer pool。（**而且这样的操作可以避免去page table查询的开销，因为page table都是带latch的，有一定的代价，我直接去这块内存找page，毕竟是顺序扫描**）这一块专属的内存区域依然能保证对当前 Query 的性能，而且因为不会打乱污染 buffer pool 而影响其他 Query 的性能。这块内存区域会在该 sequantial scan Query 结束后被释放。

#### 3 OS Page Cache

然后要介绍的是DBMS中的buffer pool与OS的page cache之间的关联。大多数的磁盘操作都是通过调用OS提供的API来完成的，而一般OS的文件系统缓存中也会对page进行缓存，这样就会产生2份的page缓存。是否要对OS提供的缓存进行利用就是一种取舍。

> 大多数 DBMS 都会使用 (O_DIRECT) 来告诉 OS 不要缓存这些数据，除了 Postgres

![img](C:/Users/xf/Desktop/CMU15445/pictures/47-166107311406540.jpg)

+ 大部分 DBMS **使用 direct I/O**，省去文件被加载入操作系统文件缓存区。direct I/O 可以直接将文件读取到数据库缓存区的 address space。省去了从操作系统缓存区复制进数据库缓存区的 address space，这样性能也更好。
+ 也有其他的 DBMS **不使用 direct I/O**， 比如 PostgreSQL。它们从工程的角度觉得不使用 direct I/O 更好：数据库 buffer pool 出现 page fault 的时候，如果对应的 page 在操作系统的文件缓存区中，那这时候只需要一次内存中的复制 (从操作系统的文件缓存区复制到数据库 buffer pool)，而不是去硬盘做漫长的 I/O。比如 PostgreSQL 重启后 buffer pool 是空的，但是操作系统的 page cache 以及还是有对应文件，这时候可以从 page cache 中复制，避免冷启动。

#### 4 Buffer Replacement Policies

当 Buffer Pool 空间不足时，读入新的 pages 必然需要 DBMS 从已经在 Buffer Pool 中的 pages 选择一些移除，这个选择就由 Buffer Replacement Policies 负责完成。它的主要目标是：

- Correctness：操作过程中要保证脏数据同步到 disk
- Accuracy：尽量选择不常用的 pages 移除
- Speed：决策要迅速，每次移除 pages 都需要申请 latch，使用太久将使得并发度下降
- Meta-data overhead：决策所使用的元信息占用的量不能太大

##### 4.1 LRU

维护每个 page 上一次被访问的时间戳，每次移除时间戳最早的 page。

##### 4.2 Clock

Clock 是 LRU 的近似策略，它不需要每个 page 上次被访问的时间戳，而是为每个 page 保存一个 reference bit

- 每当 page 被访问时，reference bit 设置为 1
- 每当需要移除 page 时，从上次访问的位置开始，按顺序轮询每个 page 的 reference bit，若该 bit 为 1，则重置为 0；若该 bit 为 0，则移除该 page

##### 4.3 LRU和Clock的问题-Sequential Flooding 顺序泛洪

LRU 和 Clock 容易收到 Sequential Flooding 的影响。Sequential Flooding 我们之前也间接的提到过，一个 Query 如果需要扫描所有的 page, buffer pool 会因为它的变得很乱，而最后留在 buffer pool 中的 page 也不一定对未来的 Query 有帮助。这里我们更严谨定义这个行为叫 Sequential Flooding，最后留在 buffer pool 中的 page 是最近被使用过的 (least recently used)，　但是它们实际上是最不需要的 (most unneeded)。

例子：

![59.jpg](C:/Users/xf/Desktop/CMU15445/pictures/59.jpg)

![60.jpg](C:/Users/xf/Desktop/CMU15445/pictures/60.jpg)

![61.jpg](C:/Users/xf/Desktop/CMU15445/pictures/61.jpg)

![62.jpg](C:/Users/xf/Desktop/CMU15445/pictures/62.jpg)

[![63.jpg](C:/Users/xf/Desktop/CMU15445/pictures/63.jpg)](https://cakebytheoceanluo.github.io/images/CMU1544564/Lec05/63.jpg)

##### 4.4 LRU-K

LRU-K 中的 `K` 代表**最近使用的次数**，因此 LRU 可以认为是 LRU-1。LRU-K 的主要目的是为了解决 LRU 算法” 缓存污染” 的问题，其核心思想是将 “最近使用过 1 次” 的判断标准**扩展为 “最近使用过 K 次”**。

相比 LRU，LRU-K 需要多维护**一个队列：缓存队列。它用于记录所有缓存数据被访问的历史。只有当数据的访问次数达到 K 次的时候，才将数据放入缓存队列**。当需要淘汰数据时，LRU-K 会淘汰第 K 次访问时间距当前时间最大的数据。详细实现如下：

1. 数据第一次被访问，加入到访问**历史队列**
2. 如果数据在访问**历史队列**里后没有达到 K 次访问，则按照一定规则 (FIFO，LRU) 淘汰
3. 当访问**历史队列**中的数据访问次数达到 K 次后，将数据索引从**历史队列**删除，将数据移到**缓存队列**中，并缓存此数据，**缓存队列**重新按照时间排序
4. **缓存队列**中被再次访问后，重新排序
5. 需要淘汰数据时，淘汰**缓存队列**中排在末尾的数据，即淘汰” 倒数第 K 次访问离现在最久” 的数据

##### 4.6 Localization

PostgreSQL 给每一个 Query 一个独立的 buffer pool， 同时所有 Query 公用一个 shared buffer pool。这最大限度地减少了每个 query 对 buffer pool 的污染。

> 对不同的访问采用不同的内存池

##### 4.7 Priority Hints

这也是需要一些先验知识。如果例如我们访问的是index，而index-page按照B+树或者有组织的形式存储，我们就可以按照这个信息来帮助页面置换信息的判断。

#### 5 Dirty Pages

只有被更改过的 page 才有必要写回硬盘，持久化。

DBMS 可以定期遍历 page table 并将脏页写回硬盘。因为如果每次等 eviction 的时候再去 flush 脏页，会让 eviction 的过程非常的慢，所以一般会有个后台进程定期批量的去写回脏页。当安全地写会脏页之后，DBMS 可以 evict 页面或者只是取消设置 dirty bit, 因为这个页面已经**不再脏了**。需要注意的是，在写日志记录之前，我们不会去写脏页。

![img](C:/Users/xf/Desktop/CMU15445/pictures/69.jpg)

> 如果缓存池中的page被修改了，他就成了脏页。脏页的evict需要将脏页的内容先写到磁盘中（为了持久化）才能加载入新的page，这就带来了2次的磁盘IO。为了减少每次置换的开销，我们可以用一个后台的的线程来向磁盘写入脏页。一旦脏页被写入磁盘，我们可以将其dirty flag置为false，这样他就不是脏页了。不过有一点需要注意的就是，在脏页写入磁盘前，需要先写日志（log record），这对于数据的一致性和rollback非常的重要（这也就是所谓的WAL，Write Ahead Logging）。

