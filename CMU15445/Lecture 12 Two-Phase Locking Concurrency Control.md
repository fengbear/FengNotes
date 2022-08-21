### Lecture 12 Two-Phase Locking Concurrency Control

上节课介绍了通过 WW、WR、RW conflicts 来判断一个 schedule 是否是 serializable 的方法，但使用该方法的前提是预先知道所有事务的执行流程，这与真实的数据库使用场景并不符合，主要原因在于：

+ 请求连续不断。时时刻刻都有事务在开启、中止和提交
+ 显式事务中，客户端不会一次性告诉数据库所有执行流程

因此我们需要一种方式来保证数据库最终使用的 schedule 是正确的 (serializable)。不难想到，保证 schedule 正确性的方法就是合理的加锁 (locks) 策略，2PL 就是其中之一。

#### 1 Lock Types

DBMS 中锁通常被分为两种，Locks 和 Latches，二者的主要区别如下：

![image-20220527233958770](C:/Users/xf/Desktop/CMU15445/pictures/image-20220527233958770.png)

本节关注的是事务级别的锁，即 Locks。Locks 有两种基本类型：

- S-LOCK：共享锁 (读锁)
- X-LOCK：互斥锁 (写锁)

二者的兼容矩阵如下表所示：

|                    | S-LOCK (shared) | X-LOCK (exclusive) |
| ------------------ | --------------- | ------------------ |
| S-LOCK (shared)    | ✅               | ❌                  |
| X-LOCK (exclusive) | ❌               | ❌                  |

如下图所示：DBMS 中有个专门的模块，lock manager，负责管理系统中的 locks，每当事务需要加锁或者升级锁的时候，都需要向它发出请求，lock manager 内部维护着一个 lock table，上面记录着当前的所有分配信息，lock manager 需要根据这些来决定赋予锁还是拒绝请求，以保证事务操作重排的正确性和并发度。

![img](C:/Users/xf/Desktop/CMU15445/pictures/assets%252F-LMjQD5UezC9P8miypMG%252F-M59XqWh_wkYVvgTgf4X%252F-M59jgyRV9xDsF_I4pOs%252FScreen%20Shot%202020-04-18%20at%208.44.43%20AM.jpg)

一个典型的 schedule 执行过程如下所示：

![img](C:/Users/xf/Desktop/CMU15445/pictures/assets%252F-LMjQD5UezC9P8miypMG%252F-M59kZk0TdbK2qYZBeoo%252F-M59kpyfCd_510nTyHvf%252FScreen%20Shot%202020-04-18%20at%208.49.42%20AM.jpg)

但仅仅在需要访问或写入数据时获取锁无法保证 schedule 的正确性，举例如下：

![img](C:/Users/xf/Desktop/CMU15445/pictures/assets%252F-LMjQD5UezC9P8miypMG%252F-M59kZk0TdbK2qYZBeoo%252F-M59lYW_8vrfMxGw92B5%252FScreen%20Shot%202020-04-18%20at%208.52.49%20AM.jpg)

事务 T1 前后两次读到的数据不一致，出现了幻读，这与顺序执行的结果并不一致。于是我们需要更强的加锁策略，来保证 schedule 的正确性。

#### 2 Two-Phase Locking

PL 是一种并发控制协议，它帮助数据库在运行过程中决定某个事务是否可以访问某条数据，并且 2PL 的正常工作并不需要提前知道所有事务的执行内容，仅仅依靠已知的信息即可。

2PL，顾名思义，有两个阶段：growing 和 shrinking：

![img](C:/Users/xf/Desktop/CMU15445/pictures/assets%252F-LMjQD5UezC9P8miypMG%252F-M59r-JWvAfm2eb3HQQ_%252F-M59trOGqijZ0pqYdsTM%252FScreen%20Shot%202020-04-18%20at%209.29.06%20AM.jpg)

在 growing 阶段中，事务可以按需获取某条数据的锁，lock manager 决定同意或者拒绝；在 shringking 阶段中，事务只能释放之前获取的锁，不能获得新锁，即一旦开始释放锁，之后就只能释放锁。下图就违背了 2PL 的协议：

![img](C:/Users/xf/Desktop/CMU15445/pictures/assets%252F-LMjQD5UezC9P8miypMG%252F-M59r-JWvAfm2eb3HQQ_%252F-M59uYhCxQraMXM-_Baw%252FScreen%20Shot%202020-04-18%20at%209.32.09%20AM.jpg)

2PL 本身已经足够保证 schedule 是 serializable，通过 2PL 产生的 schedule 中，各个 txn 之间的依赖关系能构成有向无环图。但 2PL 可能导致级联中止 (cascading aborts)，举例如下：

![img](C:/Users/xf/Desktop/CMU15445/pictures/assets%252F-LMjQD5UezC9P8miypMG%252F-M59r-JWvAfm2eb3HQQ_%252F-M59vtGC4yjt8NmMPVZ5%252FScreen%20Shot%202020-04-18%20at%209.37.58%20AM.jpg)

由于 T1 中止了，T2 在之前读到 T1 写入的数据，就是所谓的 "脏读"。为了保证整个 schedule 是 serializable，DBMS 需要在 T1 中止后将曾经读取过 T1 写入数据的其它事务中止，而这些中止可能进而使得其它正在进行的事务级联地中止，这个过程就是所谓的级联中止

事实上 2PL 还有一个增强版变种，Rigorous 2PL，后者**每个事务在结束之前，其写过的数据不能被其它事务读取或者重写**，如下图所示：

![img](C:/Users/xf/Desktop/CMU15445/pictures/assets%252F-LMjQD5UezC9P8miypMG%252F-M59r-JWvAfm2eb3HQQ_%252F-M59yHOICavPTSY7Uj-Y%252FScreen%20Shot%202020-04-18%20at%209.48.27%20AM.jpg)

Rigorous 2PL 可以避免级联中止，而且回滚操作很简单。

下面我们以转账为例，对 Non-2PL、2PL 和 Rigorous 2PL 分别举例：

![img](C:/Users/xf/Desktop/CMU15445/pictures/assets%252F-LMjQD5UezC9P8miypMG%252F-M59r-JWvAfm2eb3HQQ_%252F-M59z-_Vfu9jcExYcAqq%252FScreen%20Shot%202020-04-18%20at%209.51.33%20AM.jpg)

- T1：从 A 向 B 转账 100 美元
- T2：计算并输出 A、B 账户的总和

Non-2PL 举例：

![img](C:/Users/xf/Desktop/CMU15445/pictures/assets%252F-LMjQD5UezC9P8miypMG%252F-M59r-JWvAfm2eb3HQQ_%252F-M5A0I-sYxFJ6Fhf_Po0%252FScreen%20Shot%202020-04-18%20at%2010.01.34%20AM.jpg)

由于 T2 读到了 T1 写到一半的数据，结果不正确，输出的是 1900。可以看到它的并发程度很高。

2PL 举例：

![img](C:/Users/xf/Desktop/CMU15445/pictures/assets%252F-LMjQD5UezC9P8miypMG%252F-M59r-JWvAfm2eb3HQQ_%252F-M5A0hdIWHMtA3-KxmDp%252FScreen%20Shot%202020-04-18%20at%2010.03.15%20AM.jpg)

2PL 输出的结果正确，为 2000，同时可以看到它的并发程度比 Non-2PL 的差一些，但看着还算不错。

Rigorous 2PL 举例：

![img](C:/Users/xf/Desktop/CMU15445/pictures/assets%252F-LMjQD5UezC9P8miypMG%252F-M59r-JWvAfm2eb3HQQ_%252F-M5A11L2oa9ZiF5Tb-dG%252FScreen%20Shot%202020-04-18%20at%2010.04.49%20AM.jpg)

Rigorous 2PL 输出的结果同样是正确的，可以看到它的并发程度比 2PL 更差一些。

#### 3 DeadLock Detection & Prevention

2PL 无法避免的一个问题就是死锁：

![img](C:/Users/xf/Desktop/CMU15445/pictures/assets%252F-LMjQD5UezC9P8miypMG%252F-M59r-JWvAfm2eb3HQQ_%252F-M5A1useSGsNutWpjp8R%252FScreen%20Shot%202020-04-18%20at%2010.08.41%20AM.jpg)

死锁其实就是事务之间互相等待对方释放自己想要的锁。解决死锁的办法也很常规：

- Detection：事后检测
- Prevention：事前阻止

##### 3.1 Deadlock Detection

死锁检测是一种事后行为。为了检测死锁，DBMS 会维护一张 waits-for graph，来跟踪每个事务正在等待 (释放锁) 的其它事务，然后系统会定期地检查 waits-for graph，看其中是否有成环，如果成环了就要**决定**如何打破这个环。

waits-for graph 中的节点是事务，从 $T_i$ 到 $T_j$ 的边就表示 $T_i$ 正在等待 $T_j$ 释放锁，举例如下：

![img](C:/Users/xf/Desktop/CMU15445/pictures/assets%252F-LMjQD5UezC9P8miypMG%252F-M59r-JWvAfm2eb3HQQ_%252F-M5A3fE43PPcx7lno-DH%252FScreen%20Shot%202020-04-18%20at%2010.16.19%20AM.jpg)

***Deadlock Handling***

当 DBMS 检测到死锁时，它会选择一个 "受害者" (事务)，将该事务回滚，打破环形依赖，而这个 "受害者" 将依靠配置或者应用层逻辑重试或中止。这里有两个设计决定：

+ 检测死锁的频率
+ 如何选择合适的 "受害者"

检测死锁的频率越高，陷入死锁的事务等待的时间越短，但消耗的 cpu 也就越多。所以这是个典型的 trade-off，通常有一个调优的参数供用户配置。

选择 "受害者" 的指标可能有很多：事务持续时间、事务的进度、事务锁住的数据数量、级联事务的数量、事务曾经重启的次数等等。在选择完 "受害者" 后，DBMS 还有一个设计决定需要做：完全回滚还是回滚到足够消除环形依赖即可。

##### 3.2 Deadlock Prevention

Deadlock prevention 是一种事前行为，采用这种方案的 DBMS 无需维护 waits-for graph，也不需要实现 detection 算法，而是在事务尝试获取其它事务持有的锁时直接决定是否需要将其中一个事务中止。

通常 prevention 会按照事务的年龄来赋予优先级，事务的时间戳越老，优先级越高。有两种 prevention 的策略：

+ Old Waits for Young：如果 requesting txn 优先级比 holding txn 更高则等待后者释放锁；更低则自行中止
+ Young Waits for Old：如果 requesting txn 优先级比 holding txn 更高则后者自行中止释放锁，让前者获取锁，否则 requesting txn 等待 holding txn 释放锁

举例如下：

![img](C:/Users/xf/Desktop/CMU15445/pictures/assets%252F-LMjQD5UezC9P8miypMG%252F-M5A6zDD-oqyZS-XOaS_%252F-M5A83uVzgZ1nqIsY2XC%252FScreen%20Shot%202020-04-18%20at%2010.35.32%20AM.jpg)

无论是 Old Waits for Young 还是 Young Waits for Old，只要保证 prevention 的方向是一致的，就能阻止死锁发生，其原理类似哲学家就餐设定顺序的解决方案：先给哲学家排个序，遇到获取刀叉冲突时，顺序高的优先。

#### 4 Hierarchical Locking

上面的例子中所有的锁都是针对单条数据 (database object)，如果一个事务需要更新十亿条数据，那么 lock manager 中的 lock table 就要撑爆了。因此需要有一些手段能够将锁组织成树状/层级结构，减少一个事务运行过程中需要记录的锁的数量。

当我们说txn获得了一个“锁”，这实际上是什么意思? On an Attribute? Tuple? Page? Table?

使用不同粒度的lock 可以在tuple page  table 上都使用锁

![image-20220528000700699](C:/Users/xf/Desktop/CMU15445/pictures/image-20220528000700699.png)

例子：

![image-20220528000834490](C:/Users/xf/Desktop/CMU15445/pictures/image-20220528000834490.png)

##### 4.1 Intention Locks

意向锁允许将更高级别的节点锁定为共享或独占模式，而无需检查所有后代节点。

> An intention lock allows a higher level node to  be locked in shared or exclusive mode without  having to check all descendent nodes.

如果节点处于意向模式，则显式锁定在树中的较低级别完成。

> If a node is in an intention mode, then explicit locking is being done at a lower level in the tree.

**Intention-Shared (IS)**：

+ Indicates explicit locking at a lower level with shared locks

**Intention-Exclusive (IX)：**

+ Indicates locking at lower level with exclusive or shared  locks.

**Shared+Intention-Exclusive (SIX)：**

+ 以该节点为根的子树在共享模式下显式锁定，并且显式锁定在较低级别使用独占模式锁定完成。

  > The subtree rooted by that node is locked explicitly in  shared mode and explicit locking is being done at a lower level with exclusive-mode locks.

![image-20220528001459603](C:/Users/xf/Desktop/CMU15445/pictures/image-20220528001459603.png)

![image-20220528001538582](C:/Users/xf/Desktop/CMU15445/pictures/image-20220528001538582.png)

例子1：

![image-20220528001606958](C:/Users/xf/Desktop/CMU15445/pictures/image-20220528001606958.png)

![image-20220528001626206](C:/Users/xf/Desktop/CMU15445/pictures/image-20220528001626206.png)

例子2：

![image-20220528001642493](C:/Users/xf/Desktop/CMU15445/pictures/image-20220528001642493.png)

![image-20220528001748555](C:/Users/xf/Desktop/CMU15445/pictures/image-20220528001748555.png)

SIX下所有node都有隐含的shared lock

![image-20220528001821000](C:/Users/xf/Desktop/CMU15445/pictures/image-20220528001821000.png)

![image-20220528001848366](C:/Users/xf/Desktop/CMU15445/pictures/image-20220528001848366.png)



**Hierarchical locks are useful in practice as each txn only needs a few locks.**



