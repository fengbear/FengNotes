### Lecture 5 Index Concurrency Control

![4.jpg](C:/Users/xf/Desktop/CMU15445/pictures/4-16610731990171.jpg)

- Logical Correctness: 指这个数据结构在并发的读出与写入都是正确的。即数据结构逻辑角度。

  > This means that the thread is able to read values that it should be allowed to read.

  （17 节）我是否能看到我应该要看到的数据？

- Physical Correctness: 指并发过程中，不能造成 Segment Fault 等内存不安全的操作。即代码正确性角度 (当然也可以称为硬件角度)。

  > This means that there are not pointers in our data structure that will cause a thread to read invalid memory locations.

  （本节）数据的内部表示是否安好？

前面讨论的数据结构都是在单线程中的情况。但是实际场景中，并发和多线程操作是一定存在的，所以必须要确保DBMS在多线程操作的情况下仍然可以安全的访问数据，尽可能的利用现在越来越多的cpu核心，减少磁盘IO。

并发控制，确保在并发操作中对共享资源的正确访问。

- 逻辑正确：线程能够读取到它应该被允许读到的数据
- 物理正确：确保数据结构中没有乱七八糟的指针导致线程在找数据的时候访问到无效的内存地址

#### 1 Latch

##### 1.1 Lock vs Latch

我们在数据库的语境下比较 Lock 和 Latch。

latch一般被称为闩锁（轻量级的锁），因为其要求锁定的时间必须非常短。若持续的时间长，则应用性能会非常差。目的是用来保证并发线程操作临界资源的正确性，并且通常没有死锁检测机制。（原子性操作）

lock的对象是事务，用来锁定的是数据库中的对象，如表、页、行。并且一般lock的对象仅在事务commit或rollback进行释放。lock是有死锁机制的。

![img](C:/Users/xf/Desktop/CMU15445/pictures/1530782-20190523185855638-180122626.png)

##### 1.2 Latch Modes

![8.jpg](C:/Users/xf/Desktop/CMU15445/pictures/8-16610731990174.jpg)

Read Mode

- 多个线程可以同时读取相同的数据
- 针对相同的数据，当别的线程已经获得处于 read mode 的 latch，新的线程也可以继续获取 read mode 的 latch

Write Mode

- 同一时间只有单个线程可以访问
- 针对相同的数据，如果获取前已经有别的线程获得任何 mode 的 latch，新的线程就无法获取 write mode  的 latch

##### 1.3 Latch Implementations

我们可以通过现代cpu提供的原子比较和交换(CAS，compare-and-swap)指令来实现latch。这样，线程就可以检查内存位置的内容，看看它是否有某个值。如果有，那么CPU将用一个新的值交换旧的值。否则，内存位置将保持不变。

在DBMS中实现latch有几种方法。每种方法在工程复杂性和运行时性能方面都有不同的权衡。

###### 1.3.1 Blocking OS Mutex

使用操作系统内置互斥锁作为latch。std::mutex内部实现就是futex

> Futex是一种用户态和内核态混合的同步机制。首先，同步的进程间通过mmap共享一段内存，futex变量就位于这段共享的内存中且操作是原子的，当进程尝试进入互斥区或者退出互斥区的时候，先去查看共享内存中的futex变量，如果没有竞争发生，则只修改futex,而不用再执行系统调用了。当通过访问futex变量告诉进程有竞争发生，则还是得执行系统调用去完成相应的处理(wait 或者 wake up)。
>
> 所有的futex同步操作都应该从用户空间开始，首先创建一个futex同步变量，也就是位于共享内存的一个整型计数器。
> 当 进程尝试持有锁或者要进入互斥区的时候，对futex执行"down"操作，即原子性的给futex同步变量减1。如果同步变量变为0，则没有竞争发生， 进程照常执行。如果同步变量是个负数，则意味着有竞争发生，需要调用futex系统调用的futex_wait操作休眠当前进程。
> 当进程释放锁或 者要离开互斥区的时候，对futex进行"up"操作，即原子性的给futex同步变量加1。如果同步变量由0变成1，则没有竞争发生，进程照常执行。如 果加之前同步变量是负数，则意味着有竞争发生，需要调用futex系统调用的futex_wake操作唤醒一个或者多个等待进程。

**OS mutex is generally a bad idea inside of DBMSs as it is managed by OS and has large overhead.**

![9.jpg](C:/Users/xf/Desktop/CMU15445/pictures/9-16610731990176.jpg)

+ **优点：**Simple to use and requires no additional coding in DBMS

+ **缺点：**太慢，每一次lock/unlock的调用需要约25ns，所以不能大规模扩展。

###### 1.3.2 Test-and-Set Spin Latch（TAS）

std::atomic<T> 就是这种实现

Spin latches是一种比os mutex更有效的选择，因为它由DBMS控制。

Spin latches本质上是内存中线程试图更新的位置(例如，将布尔值设置为true)。线程执行CAS尝试更新内存位置。如果它不能，那么它就在一个while循环中旋转，永远尝试更新它。

+ **优点：**Latch/unlatch operations are **efficient (single instruction to lock/unlock)**。非常高效（单指令执行latch/unlatch）

+ **缺点：**有时候会让CPU看起来很忙，其实只是一直在循环等待罢了

![10.jpg](C:/Users/xf/Desktop/CMU15445/pictures/10-16610731990178.jpg)

另外针对这个 while 循环，我们可以从中做决定:

- Retry: 如果等待的时间不长，继续等待
- Yield, Abort: 如果等待的时间很长，直接放弃等待。做其他的事情，做完以后资源可能就空闲了

> spin 本质上就是在循环执行 test ( set )，若test返回成功就set (不成功则继续test), 表示获得latch执行下一步代码，完成后test and set 复位。

###### 1.3.3 Reader-Writer Latches

Mutexes与Spin Latches不区分读与写（它们不支持不同的模式）。我们需要一种允许并发读取的方法，这样，如果应用程序有大量的读取，它就会有更好的性能，因为读取器可以共享资源，而不是等待。

Reader-Writer Latch允许latch以读或写模式保持。它跟踪在每种模式下有多少线程持有latch或等待获得latch。

+ 例子：这是在Spin latch之上实现的。
+ 优点：允许并发读取。
+ 缺点：DBMS必须管理读/写队列以避免饥饿。由于额外的元数据，存储开销比Spin latch大。

下面的图示中，我们能看到读写锁会记录:

- 正在读的数量
- 正在写与否 (0 或 1)
- 等待读的数量
- 等待写的数量

![11.jpg](C:/Users/xf/Desktop/CMU15445/pictures/11-166107319901710.jpg)

![12.jpg](C:/Users/xf/Desktop/CMU15445/pictures/12-166107319901712.jpg)

![13.jpg](C:/Users/xf/Desktop/CMU15445/pictures/13-166107319901714.jpg)

![14.jpg](C:/Users/xf/Desktop/CMU15445/pictures/14-166107319901716.jpg)

![15.jpg](C:/Users/xf/Desktop/CMU15445/pictures/15.jpg)

- 上图情况是有一个读和一个写在等，目前锁模式的读。后面可能发生如下两种情况:
  - 可能 1: 读的锁不等待，直接和其他两个线程一起读。这样会让写线程一直等，如果一直有新的读线程生成并*插队*的话。即 读线程优先。
  - 可能 2: 读的锁需要等待写线程的完成。这样会让本可以共享的三次读操作不能同时完成。即 写线程优先。
- 具体是哪一种应对方式需要看锁的具体实现，也需要看程序中是并行读还是并行写多，需要给谁优先权。

#### 2 Hash Table Latching

由于线程访问数据结构的方式有限，在静态哈希表中很容易支持并发访问。例如，当从slot移动到下一个slot时，所有线程都沿相同的方向移动。线程一次也只访问一个page/slot。

因此，在这种情况下不可能出现死锁，因为没有两个线程可以竞争另一个线程持有的latch。要调整表的大小，请在整个表上设置一个全局latch。

这部分中我们主要讨论 static hash scheme 中的 linear probing hashing。

![17.jpg](C:/Users/xf/Desktop/CMU15445/pictures/17-166107319901719.jpg)

+ Page Latches：这降低了并行性，因为一次可能只有一个线程可以访问一个页面，但是访问一个页面中的多个slot将会很快，因为一个线程只需要获得一个latch。
+ Slot latches：这增加了并行性，因为两个线程可以访问同一页中的不同slot。但是它增加了访问表的存储和计算开销，因为线程必须为它们访问的每个slot获取一个latch。DBMS可以使用单一模式latch(即Spin latch)来减少元数据和计算开销。

即使上面有两种方式，它们锁粒度的不一样，造成其他地方很多也不一样。但是对并行控制的核心想法是一致的：

- 所有的线程总是以**从前往后**固定的方向去获得数据 (获得下一个 page / 获得下一个 slot)
- 因此每一个线程可以保证线程安全:
  - 1. 先松开当前 page /slot 的 latch
  - 2. 再去获得下一个 page /slot 的 latch
- 上面两步之间可能会有其他的线程先拿走了下一个 page /slot, 但是这样的行为不影响我们的线程安全性。

我们下面有足够篇幅的例子去解释上面这部分的描述。例子中默认每个 page 有两个 slot，另外我们使用 read-write latch。

##### 2.1 Page Latches

![18.jpg](C:/Users/xf/Desktop/CMU15445/pictures/18-166107319901721.jpg)

![19.jpg](C:/Users/xf/Desktop/CMU15445/pictures/19-166107319901723.jpg)

![20.jpg](C:/Users/xf/Desktop/CMU15445/pictures/20-166107319901725.jpg)

![21.jpg](C:/Users/xf/Desktop/CMU15445/pictures/21-166107319901827.jpg)

![22.jpg](C:/Users/xf/Desktop/CMU15445/pictures/22-166107319901829.jpg)

- 上图中 `T1` 完成第一步: **先松开当前 page1 的 latch**

![23.jpg](C:/Users/xf/Desktop/CMU15445/pictures/23-166107319901831.jpg)

- 上图中 `T1` 完成第二步: **再去获得 page2 的 latch**
- 上图中 `T2` 获得 page1 的 latch

![24.jpg](C:/Users/xf/Desktop/CMU15445/pictures/24-166107319901833.jpg)

![25.jpg](C:/Users/xf/Desktop/CMU15445/pictures/25-166107319901835.jpg)

![26.jpg](C:/Users/xf/Desktop/CMU15445/pictures/26-166107319901837.jpg)

![27.jpg](C:/Users/xf/Desktop/CMU15445/pictures/27-166107319901839.jpg)

##### 2.2 Slot Latches

![28.jpg](C:/Users/xf/Desktop/CMU15445/pictures/28-166107319901841.jpg)

![29.jpg](C:/Users/xf/Desktop/CMU15445/pictures/29-166107319901843.jpg)

![30.jpg](C:/Users/xf/Desktop/CMU15445/pictures/30-166107319901845.jpg)

![31.jpg](C:/Users/xf/Desktop/CMU15445/pictures/31-166107319901847.jpg)

![32.jpg](C:/Users/xf/Desktop/CMU15445/pictures/32-166107319901849.jpg)

- 上图中 `T1` 完成第一步: **先松开当前 slot 的 latch**

![33.jpg](C:/Users/xf/Desktop/CMU15445/pictures/33-166107319901851.jpg)

![34.jpg](C:/Users/xf/Desktop/CMU15445/pictures/34-166107319901853.jpg)

- 上图中 `T1` 完成第二步: **再去获得下一个 slot 的 latch**

![35.jpg](C:/Users/xf/Desktop/CMU15445/pictures/35-166107319901855.jpg)

![36.jpg](C:/Users/xf/Desktop/CMU15445/pictures/36-166107319901857.jpg)

![37.jpg](C:/Users/xf/Desktop/CMU15445/pictures/37-166107319901859.jpg)



#### 3 B+ Tree Concurrency Control

![38.jpg](C:/Users/xf/Desktop/CMU15445/pictures/38-166107319901861.jpg)

- 上图中最后一点中：如果不线程安全，可能导致触碰到不再有效的内存区域 invalid memory location, 产生 segment fault 段错误。

##### 3.1 线程不安全

下面我们用一个线程不安全的反例，来说明在 B+ Tree 中线程安全的重要意义。

###### 3.1.1 T1 running

![40.jpg](C:/Users/xf/Desktop/CMU15445/pictures/40-166107319901863.jpg)

![41.jpg](C:/Users/xf/Desktop/CMU15445/pictures/41-166107319901865.jpg)

![42.jpg](C:/Users/xf/Desktop/CMU15445/pictures/42-166107319901867.jpg)

- 直到上图，`T1` 从 leaf node 中删除了 `44`，`T1` 发现 leaf node 没有满足一半填满 (less half full) 的状态，需要继续 rebalance (from sibling)。这时 `T1: delete 44` 尚未完成。但是 thread scheudler 选择暂停 `T1`。

###### 3.1.2 T1 stalled, T2 running

![43.jpg](C:/Users/xf/Desktop/CMU15445/pictures/43-166107319901869.jpg)

![44.jpg](C:/Users/xf/Desktop/CMU15445/pictures/44-166107319901871.jpg)

- 直到上图，`T2` 找到了 `Node D`，并从它中得知并获得了 `41` 对应 leaf node 的指针，即指向 `Node H` 的指针 (图中的蓝色箭头)。这时 `T2: find 41` 尚未完成。但是 thread scheudler 选择暂停 `T2`。

###### 3.1.3 T1 running,T2 stalled

![45.jpg](C:/Users/xf/Desktop/CMU15445/pictures/45-166107319901873.jpg)

![46.jpg](C:/Users/xf/Desktop/CMU15445/pictures/46-166107319901875.jpg)

- 直到上图，`T1` 继续 rebalance, 将 `41` 从 sibling `Node H` 中换到 `Node I`。这时 `T1: delete 44` 尚未完成，但是 delete 部分已经完成，只是还没有 return 回主函数。这时 thread scheudler 选择暂停 `T1`。

###### 3.1.4 T1 stalled,T2 running

![47.jpg](C:/Users/xf/Desktop/CMU15445/pictures/47-166107319901877.jpg)

- 直到上图，`T2` 使用之前获得的指向 `Node H` 的指针 (图中的蓝色箭头)， 并成功到达 `Node H`。只是 `Node H` 中没有 `41`，这时 `T2: find 41` 完成，只是 return false，说明 B+ Tree 中不存在 `41`。
- 这明显是 **False Negitive**, 因为树中真实存在 `41`, `T2` 判断错误。

##### 3.2 Latch Crabbing / Coupling Protocol

> Lock crabbing / coupling is a protocol to allow multiple threads to access/modify B+Tree at the same time.

![48.jpg](C:/Users/xf/Desktop/CMU15445/pictures/48-166107319901879.jpg)

**要确保先取得 child latch 并确认它是 safe, 才能松开 parent latch。**就是说存在一个时刻，一个线程会有多于一个 latch 的可能。

###### 3.2.1 Basic Latch Crabbing Protocol

![49.jpg](C:/Users/xf/Desktop/CMU15445/pictures/49-166107319901981.jpg)

**Example1 - Find 38**

![50.jpg](C:/Users/xf/Desktop/CMU15445/pictures/50-166107319901983.jpg)

![51.jpg](C:/Users/xf/Desktop/CMU15445/pictures/51-166107319901985.jpg)

![52.jpg](C:/Users/xf/Desktop/CMU15445/pictures/52-166107319901987.jpg)

![53.jpg](C:/Users/xf/Desktop/CMU15445/pictures/53-166107319901989.jpg)

![54.jpg](C:/Users/xf/Desktop/CMU15445/pictures/54-166107319901991.jpg)

![55.jpg](C:/Users/xf/Desktop/CMU15445/pictures/55-166107319901993.jpg)

**Example2 - Delete 38**

![56.jpg](C:/Users/xf/Desktop/CMU15445/pictures/56-166107319901995.jpg)

![57.jpg](C:/Users/xf/Desktop/CMU15445/pictures/57-166107319901997.jpg)

![58.jpg](C:/Users/xf/Desktop/CMU15445/pictures/58-166107319901999.jpg)

- 在上图中，我们找到了 `38`, 发现删除了它，`Inner Node D` 依然是 safe 的。这之后我们决定松开 `Node A` 的 latch，和松开 `Node B` 的 latch。
- 松开 `Node A` 和 `Node B`latch 的先后并不影响正确性。但是 `Node A` 下覆盖的节点更多，潜在地有更多线程想要获得 `Node A` 的 latch。因此先松开 `Node A` 的 latch，再松开 `Node B` 的 latch，性能更好。

![59.jpg](C:/Users/xf/Desktop/CMU15445/pictures/59-1661073199019101.jpg)

![60.jpg](C:/Users/xf/Desktop/CMU15445/pictures/60-1661073199019103.jpg)

![61.jpg](C:/Users/xf/Desktop/CMU15445/pictures/61-1661073199019105.jpg)

**Example3 - Insert 45**

![62.jpg](C:/Users/xf/Desktop/CMU15445/pictures/62-1661073199019107.jpg)

![63.jpg](C:/Users/xf/Desktop/CMU15445/pictures/63-1661073199019109.jpg)

- 在上图中，我们发现 `Node B` 是 safe 的。这之后我们决定松开 `Node A` 的 latch。

![64.jpg](C:/Users/xf/Desktop/CMU15445/pictures/64-1661073199019111.jpg)

- 在上图中，我们发现 `Node D` 是满的，对 insertion 不是 safe 的。我们决定保持 `Node B` 的 latch。

![65.jpg](C:/Users/xf/Desktop/CMU15445/pictures/65-1661073199019113.jpg)

- 在上图中，我们发现 `Node I` 是 safe 的。这之后我们决定松开 `Node B` 的 latch，和松开 `Node D` 的 latch。
- 松开 `Node B` 和 `Node D`latch 的先后并不影响正确性。但是 `Node B` 下覆盖的节点更多，潜在地有更多线程想要获得 `Node D` 的 latch。因此先松开 `Node B` 的 latch，再松开 `Node D` 的 latch，性能更好。

![66.jpg](C:/Users/xf/Desktop/CMU15445/pictures/66-1661073199019115.jpg)

**Example4 - Insert 25**

![67.jpg](C:/Users/xf/Desktop/CMU15445/pictures/67-1661073199019117.jpg)

![68.jpg](C:/Users/xf/Desktop/CMU15445/pictures/68-1661073199019119.jpg)

![69.jpg](C:/Users/xf/Desktop/CMU15445/pictures/69-1661073199019121.jpg)

![70.jpg](C:/Users/xf/Desktop/CMU15445/pictures/70-1661073199019123.jpg)

![71.jpg](C:/Users/xf/Desktop/CMU15445/pictures/71-1661073199019125.jpg)

- 在上图中，我们发现 `Node F` 是满的，对 insertion 不是 safe 的。我们决定保持 `Node C` 的 latch。

![73.jpg](C:/Users/xf/Desktop/CMU15445/pictures/73-1661073199019127.jpg)

- 在上图中，只有当前线程才能访问新建的节点，其他的线程甚至只能排队等去访问 `Node C`。这种情况下，新建节点不需要上锁，可以节省开销。

从上面所有的例子中， 我们可以发现，我们需要有一个 **FIFO list** 去记录一路上获得的 latch。先获得的 latch， 可以先松开， 因此是使用 FIFO 数据结构。

###### 3.2.2 Improved Lock Crabbing Protocol

我们发现对应写操作 (Insert 和 Delete) 一开始，必须获得 root node 的 latch。那样的话，其他的想要进行读操作或者写操作的线程必须等待这个获得 root node latch 的线程松开这个 latch。这个地方会造成很多线程的等待，是并行 B+ Tree 性能的一个瓶颈 bottleneck。

![74.jpg](C:/Users/xf/Desktop/CMU15445/pictures/74-1661073199019129.jpg)

+ Search： Same algorithm as before
+ Insert/Delete：**Set READ latches as if for search, go to leaf, and set WRITE latch on leaf. If leaf is not safe, release all previous latches, and restart transaction using previous Insert/Delete protocol**

**Example2 - Delete 38 - New Version**

![77.jpg](C:/Users/xf/Desktop/CMU15445/pictures/77.jpg)

![78.jpg](C:/Users/xf/Desktop/CMU15445/pictures/78-1661073199019132.jpg)

![79.jpg](C:/Users/xf/Desktop/CMU15445/pictures/79-1661073199019134.jpg)

![80.jpg](C:/Users/xf/Desktop/CMU15445/pictures/80-1661073199019136.jpg)

![81.jpg](C:/Users/xf/Desktop/CMU15445/pictures/81-1661073199019138.jpg)

完成！不需要重头开始！

**Example4 - Insert 25 - New Version**

![83.jpg](C:/Users/xf/Desktop/CMU15445/pictures/83-1661073199019140.jpg)

- 在上图中，我们发现 `Node F` 是满的，对 insertion 不是 safe 的。这个线程需要松开 `Node F` 的 latch。重新开始，从 Root Node 开始，用 write mode latch 去进行 `Insert 25` 这个操作。

##### 3.3 Leaf Node Scans

![85.jpg](C:/Users/xf/Desktop/CMU15445/pictures/85.jpg)

这些协议中的线程以“自顶向下”的方式获得latch。这意味着线程只能从低于其当前节点的节点获得latch。如果所需的latch不可用，线程必须等待，直到它可用为止。考虑到这一点，永远不可能出现死锁。

叶节点扫描容易受到死锁的影响，因为现在我们有线程试图同时获得两个不同方向的锁(即从左到右和从右到左)。Index latches不支持死锁检测或避免。

因此，我们处理这个问题的唯一方法是通过编码规则。叶节点兄弟latch获取协议必须支持“无等待”模式。也就是说，B+树代码必须应对latch获取失败。这意味着，如果一个线程试图获得叶节点上的latch，但该latch不可用，那么它将立即终止其操作(释放它持有的任何latch)，然后重新启动该操作。

![image-20220504165107602](C:/Users/xf/Desktop/CMU15445/pictures/image-20220504165107602.png)

###### 3.3.1 Leaf Node Scan Example1

right-to-left leaf node scan. Read mode latch

![86.jpg](C:/Users/xf/Desktop/CMU15445/pictures/86.jpg)

![87.jpg](C:/Users/xf/Desktop/CMU15445/pictures/87.jpg)

![88.jpg](C:/Users/xf/Desktop/CMU15445/pictures/88.jpg)

![89.jpg](C:/Users/xf/Desktop/CMU15445/pictures/89.jpg)

![90.jpg](C:/Users/xf/Desktop/CMU15445/pictures/90.jpg)

###### 3.3.2 Leaf Node Scan Example2

T1: right-to-left leaf node scan. Read mode latch
T2: left-to-right leaf node scan. Read mode latch

![91.jpg](C:/Users/xf/Desktop/CMU15445/pictures/91.jpg)

![92.jpg](C:/Users/xf/Desktop/CMU15445/pictures/92.jpg)

![93.jpg](C:/Users/xf/Desktop/CMU15445/pictures/93.jpg)

![94.jpg](C:/Users/xf/Desktop/CMU15445/pictures/94.jpg)

![95.jpg](C:/Users/xf/Desktop/CMU15445/pictures/95.jpg)

![96.jpg](C:/Users/xf/Desktop/CMU15445/pictures/96.jpg)

###### 3.3.3 Leaf Node Scan Example3

T1: top-to-bottom tree traversal scan. Write mode latch
T2: left-to-right leaf node scan. Read mode latch

![97.jpg](C:/Users/xf/Desktop/CMU15445/pictures/97.jpg)

![98.jpg](C:/Users/xf/Desktop/CMU15445/pictures/98.jpg)

![99.jpg](C:/Users/xf/Desktop/CMU15445/pictures/99.jpg)

![100.jpg](C:/Users/xf/Desktop/CMU15445/pictures/100.jpg)

![101.jpg](C:/Users/xf/Desktop/CMU15445/pictures/101.jpg)

死锁！

+ 因为T2并不和T1进行线程通信，因为它们不知道对方在进行什么操作。T2可能有几种不同可能的做法：
  + `T2` 立即放弃获得 `Node B` 的 latch, 重新开始，重头开始。
  + `T2` 让 `T1` 放弃 `Node C` 的 latch, `T2` 获得 `Node C` 的 latch。而 `T1` 重新开始，重头开始。
  + `T2` 等待 `Node C` 的 latch。但是为了避免 deadlock。等完 timeout 就放弃获得 `Node B` 的 latch, 重新开始，重头开始。

一般是第一种做法。

##### 3.4 Delayed Parent Updates

![103.jpg](C:/Users/xf/Desktop/CMU15445/pictures/103.jpg)

###### 3.4.1 Example4 - Insert 25 - New New Version

**T1: Insert 25**

![104.jpg](C:/Users/xf/Desktop/CMU15445/pictures/104.jpg)

![105.jpg](C:/Users/xf/Desktop/CMU15445/pictures/105.jpg)

![106.jpg](C:/Users/xf/Desktop/CMU15445/pictures/106.jpg)

![107.jpg](C:/Users/xf/Desktop/CMU15445/pictures/107.jpg)

![108.jpg](C:/Users/xf/Desktop/CMU15445/pictures/108.jpg)

![110.jpg](C:/Users/xf/Desktop/CMU15445/pictures/110.jpg)

- 上图中: `T1` 推迟对 `Node C` 中写入 `31` 这个操作。`T1` 将这件事情记录在一个 global table 中，推迟到下一次对 `Node C` 有 Write latch 的时候。

**T2: Find 31**

![111.jpg](C:/Users/xf/Desktop/CMU15445/pictures/111.jpg)

![112.jpg](C:/Users/xf/Desktop/CMU15445/pictures/112.jpg)

**T3: Insert 33**

![113.jpg](C:/Users/xf/Desktop/CMU15445/pictures/113.jpg)

![114.jpg](C:/Users/xf/Desktop/CMU15445/pictures/114.jpg)

![115.jpg](C:/Users/xf/Desktop/CMU15445/pictures/115.jpg)

![116.jpg](C:/Users/xf/Desktop/CMU15445/pictures/116.jpg)

- 上图中: `T3` 从 global table 中发现，在对 `Node C` 有 Write latch 的时候有一个操作被推迟过。`T3` 完成这个事件：向 `Node C` 中写入 `31`. 这样我们捎带 piggyback 一个之前的插入操作到一个新的插入操作，均摊了性能花销.

**另外一个发现是 Parent Node 的信息可以延迟更新，即处在一个信息过期的状态，这不影响在 Leaf Node 的查找.**

- 因此我们删除的时候也可以延迟更新：先删除 Leaf Node 中的元素，之后延迟再删除 Parent Node 的元素 (如果有对应的话)