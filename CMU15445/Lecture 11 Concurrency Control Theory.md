### Lecture 11 Concurrency Control Theory

回顾本课程的路线图：

<img src="C:/Users/xf/Desktop/CMU15445/pictures/image-20220525094542280.png" alt="image-20220525094542280" style="zoom: 33%;" />

在前面的课程中介绍了 DBMS 的主要模块及架构，自底向上依次是 Disk Manager、Buffer Pool Manager、Access Methods、Operator Execution 及 Query Planning。但数据库要解决的问题并不仅仅停留在功能的实现上，它还需要具备：

- 满足多个用户同时读写数据，即 Concurrency Control，如：
  - 两个用户同时写入同一条记录
- 面对故障，如宕机，能恢复到之前的状态，即 Recovery，如：
  - 你在银行系统转账时，转到一半忽然停电

Concurrency Control 与 Recovery 都是 DBMSs 的重要特性，它们渗透在 DBMS 的每个主要模块中。而二者的基础都是具备 ACID 特性的 Transactions，因此本节的讨论从 Transactions 开始。

#### 1 Transactions

> A transaction is the execution of a sequence of one or more operations (e.g., SQL queries) on a shared database to perform some higher-level function.

事务其实就是，通过在数据库系统中执行一系列操作来执行某种更高级的功能，你可以将这些操作想象成SQL查询或者是我们对数据库所做的读和写操作。更高级的功能例如银行转账， 你不可能写一个函数就能完成。

所以，事务就是我们数据库管理系统中关于修改操作方面的一个基本单位，所有的要做的修改会被包裹在一个事务中。

**用一句白话说，transaction 是 DBMS 状态变化的基本单位，一个 transaction 只能有执行和未执行两个状态，不存在部分执行的状态。**具有原子性。

一个典型的 transaction 实例就是：从 A 账户转账 100 元至 B 账户：

- 检查 A 的账户余额是否超过 100 元
- 从 A 账户中扣除 100 元
- 往 B 账户中增加 100 元

##### 1.1 Simple System

系统中只存在一个线程负责执行 transaction：

- 任何时刻只能有一个 transaction 被执行且每个 transaction 开始执行就必须执行完毕才轮到下一个
- 在 transaction 启动前，将整个 database 数据复制到新文件中，并在新文件上执行改动
  - 如果 transaction 执行成功，就用新文件覆盖原文件
  - 如果 transaction 执行失败，则删除新文件即可

Strawman System 的缺点很明显，无法利用多核计算能力并行地执行相互独立的多个 transactions，从而提高 CPU 利用率、吞吐量，减少用户的响应时间，但其难度也是显而易见的，获得种种好处的同时必须保证数据库的正确性和 transactions 之间的公平性。

显然我们无法让 transactions 的执行过程在时间线上任意重叠，因为这可能导致数据的永久不一致。于是我们需要一套标准来定义数据的正确性。

##### 1.2 Formal Definitions

**Database：A fixed set of named data objects (e.g., A, B, C, ...)** 

**Transaction：A sequence of read and write operations (e.g., R(A), W(B), ...)**

**Transaction in SQL**：

A new txn starts with the BEGIN command.

The txn stops with either COMMIT or ABORT:

+ If commit, the DBMS either saves all the txn's changes or aborts it
+ If abort, all changes are undone so that it’s like as if the  txn never executed at all

**Transaction 的正确性标准称为 ACID：**

+ Atomicity: All actions in the txn happen, or none  happen
+ Consistency: "it looks correct to me"
+ Isolation: Execution of one txn is isolated from  that of other txns
+ Durability: If a txn commits, its effects persist.

##### 1.3 Atomicity

Transaction 执行只有两种结果：

- 在完成所有操作后 Commit
- 在完成部分操作后主动或被动 Abort

DBMS 需要保证 transaction 的原子性，即在使用者的眼里，transaction 的所有操作要么都被执行，要么都未被执行。回到之前的转账问题，如果 A 账户转 100 元到 B 账户的过程中忽然停电，当供电恢复时，A、B 账户的正确状态应当是怎样的？--- 回退到转账前的状态。如何保证？

**Logging**

DBMS 在日志中按顺序记录所有 actions 信息，然后 undo 所有 aborted transactions 已经执行的 actions，出于审计和效率的原因，几乎所有现代系统都使用这种方式。

**Shadow Paging**

transaction 执行前，DBMS 复制相关 pages，让 transactions 修改这些复制的数据，仅当 transaction Commit 后，这些 pages 才对外部可见。现代系统中采用该方式的不多，包括 CouchDB, LMDB。

##### 1.4 Consistency

The "world" represented by the database is  logically correct. All questions asked about the data  are given logically correct answers.

###### 1.4.1 Database Consistency

数据库精确地模拟真实世界，并遵循完整性约束。（比如年龄没有负值）

将来的事务会在数据库中看到过去提交的事务的效果。

###### 1.4.2 Transaction Consistency

If the database is consistent before the transaction  starts (running alone), it will also be consistent  after.

Transaction consistency is the application’s  responsibility.

##### 1.5 Isolation

Users submit txns, and each txn executes as if it  was running by itself.

但是DBMS通过交错txns的操作(对DB对象的读/写)来实现并发性。

我们需要一种方法来交错txns，但仍然使它看起来好像它们一次运行一个。

并发控制协议是 DBMS 如何决定对来自多个事务的操作进行适当的交错。

用户提交 transactions，不同 transactions 执行过程应当互相隔离，互不影响，每个 transaction 都认为只有自己在执行。但对于 DBMS 来说，为了提高各方面性能，需要恰如其分地向不同的 transactions 分配计算资源，使得执行又快又正确。这里的 “恰如其分” 的定义用行话来说，就是 concurrency control protocol，即 DBMS 如何认定多个 transactions 的重叠执行方式是正确的。总体上看，有两种 protocols：

+ 悲观：假设我们的事务在执行时会产生冲突，导致问题的出现
+ 乐观：假设冲突很少发生，在冲突发生后再处理

举例如下：假设 A, B 账户各有 1000 元，当下有两个 transactions：

- T1：从 A 账户转账 100 元到 B 账户
- T2：给 A、B 账户存款增加 6% 的利息

```sql
// T1
BEGIN
A = A - 100
B = B + 100
COMMIT

BEGIN
A = A * 1.06
B = B * 1.06
COMMIT
```

那么 T1、T2 发生后，可能的合理结果应该是怎样的？

尽管有很多种可能，但 T1、T2 发生后，A + B 的和应为 2000 * 1.06 = 2120 元。DBMS 无需保证 T1 与 T2 执行的先后顺序，如果二者同时被提交，那么谁先被执行都是有可能的，但执行后的净结果应当与二者按任意顺序分别执行的结果一致，如：

- 先 T1 后 T2：A = 954， B = 1166 => A+B = 2120
- 先 T2 后 T1：A = 960， B = 1160 => A+B = 2120

执行过程如下图所示：

![img](C:/Users/xf/Desktop/CMU15445/pictures/assets%252F-LMjQD5UezC9P8miypMG%252F-Lcut8zTn7Akdy-vPFlR%252F-LcvARiJ8r94h05pln1D%252FScreen%20Shot%202019-04-20%20at%2010.36.50%20PM.jpg)

当一个 transaction 在等待资源 (page fault、disk/network I/O) 时，或者当 CPU 存在其它空闲的 Core 时，其它 transaction 可以继续执行，因此我们需要将 transactions 的执行重叠 (interleave) 起来。

Example 1 (Good)

![img](C:/Users/xf/Desktop/CMU15445/pictures/assets%252F-LMjQD5UezC9P8miypMG%252F-Ld2Z54nuQcC1eqJxEsD%252F-Ld2Z7ms19nsCGVkc6pk%252FScreen%20Shot%202019-04-22%20at%201.40.36%20PM.jpg)

Example 2 (Bad)

![img](C:/Users/xf/Desktop/CMU15445/pictures/assets%252F-LMjQD5UezC9P8miypMG%252F-Ld2Z54nuQcC1eqJxEsD%252F-Ld2ZH2NbX1aUNYulZzC%252FScreen%20Shot%202019-04-22%20at%201.42.08%20PM.jpg)

从 DBMS 的角度分析第二个例子：

- A = A - 100 => R(A), W(A)
- ...

![img](C:/Users/xf/Desktop/CMU15445/pictures/assets%252F-LMjQD5UezC9P8miypMG%252F-Ld2Z54nuQcC1eqJxEsD%252F-Ld2ZbHnJxKOA4xrREA9%252FScreen%20Shot%202019-04-22%20at%201.43.44%20PM.jpg)

如何判断一种重叠的 schdule 是正确的？

**如果某个schedule的执行结果等同于按顺序执行的结果，那么我们就会说这种执行顺序的sechedule是正确的。**

这里明确几个概念：

- Serial Schedule: 不同 transactions 之间没有重叠
- Equivalent Schedules: 对于任意数据库起始状态，若两个 schedules 分别执行所到达的数据库最终状态相同，则称这两个 schedules 等价
- Serializable Schedule: 如果一个 schedule 与 transactions 之间的某种 serial execution 的效果一致，则称该 schedule 为 serializable schedule（能够序列化的schedule）

###### 1.5.1 Conflicting Operations

在对 schedules 作等价分析前，需要了解 conflicting operations。当两个 operations 满足以下条件时，我们认为它们是 conflicting operations：

+ 来自不同的 transactions
+ 对同一个对象操作
+ 两个 operations 至少有一个是 write 操作

可以穷举出这几种情况：

- Read-Write Conflicts (R-W)
- Write-Read Conflicts (W-R)
- Write-Write Conflicts (W-W)

***Read-Write Conflicts/Unrepeatable Reads*（不可重复读）**

![img](C:/Users/xf/Desktop/CMU15445/pictures/assets%252F-LMjQD5UezC9P8miypMG%252F-Ld2Z54nuQcC1eqJxEsD%252F-Ld2bdGLZFHxKn4j5jyu%252FScreen%20Shot%202019-04-22%20at%201.56.59%20PM.jpg)

一个事务先后读取同一条记录，但两次读取的数据不同。

***Write-Read Conflicts/Reading Uncommitted Data/Dirty Reads*（脏读）**

![img](C:/Users/xf/Desktop/CMU15445/pictures/assets%252F-LMjQD5UezC9P8miypMG%252F-Ld2Z54nuQcC1eqJxEsD%252F-Ld2byQOw-SjbrFm58nI%252FScreen%20Shot%202019-04-22%20at%201.58.01%20PM.jpg)

一个事务都到另一个事务还没有提交的数据

***Write-Write Conflicts/Overwriting Uncommitted Data***

![img](C:/Users/xf/Desktop/CMU15445/pictures/assets%252F-LMjQD5UezC9P8miypMG%252F-Ld2Z54nuQcC1eqJxEsD%252F-Ld2c5sHk911dFVCtMqg%252FScreen%20Shot%202019-04-22%20at%201.58.59%20PM.jpg)

从以上例子可以理解 **serializability 对一个 schedule 意味着这个 schedule 是否正确**。serializability 有两个不同的级别：

Conflict Serializability (Most DBMSs try to support this)：

- 两个 schedules 在 transactions 中有相同的 actions，且每组 conflicting actions 按照相同顺序排列，则称它们为 conflict equivalent
- 一个 schedule  S 如果与某个 serial schedule 是 conflict equivalent，则称 S 是 conflict serializable
- 如果通过交换不同 transactions 中连续的 non-conflicting operations 可以将 S 转化成 serial schedule，则称 S 是 conflict serializable

View Serializability (No DBMS can do this)

例 1：将 conflict serializable schedule S 转化为 serial schedule

![img](C:/Users/xf/Desktop/CMU15445/pictures/assets%252F-LMjQD5UezC9P8miypMG%252F-LdCdP-burZWkhWdoMaP%252F-LdCgSVEXsultXNNZFLA%252FScreen%20Shot%202019-04-24%20at%2012.54.14%20PM.jpg)

R(B) 与 W(A) 不矛盾，是 non-conflicting operations，可以交换它们



![img](C:/Users/xf/Desktop/CMU15445/pictures/assets%252F-LMjQD5UezC9P8miypMG%252F-LdChJLDEgOtT9muvCHq%252F-LdCgaTtPUzY__K0-n76%252FScreen%20Shot%202019-04-24%20at%2012.54.31%20PM.jpg)

![img](C:/Users/xf/Desktop/CMU15445/pictures/assets%252F-LMjQD5UezC9P8miypMG%252F-LdChJLDEgOtT9muvCHq%252F-LdCgfImzjYRRD9Emw86%252FScreen%20Shot%202019-04-24%20at%2012.55.10%20PM.jpg)

R(B) 与 R(A) 是 non-conflicting operations，交换它们

![img](C:/Users/xf/Desktop/CMU15445/pictures/assets%252F-LMjQD5UezC9P8miypMG%252F-LdChJLDEgOtT9muvCHq%252F-LdCgpJ4EgHsX5Tu0VNh%252FScreen%20Shot%202019-04-24%20at%2012.55.52%20PM.jpg)

依次类推，分别交换 W(B) 与 W(A)，W(B) 与 R(A) 就能得到：

![img](C:/Users/xf/Desktop/CMU15445/pictures/assets%252F-LMjQD5UezC9P8miypMG%252F-LdChJLDEgOtT9muvCHq%252F-LdChPSq4OxUpbUnFKTH%252FScreen%20Shot%202019-04-24%20at%2012.57.24%20PM.jpg)

因此 S 为 conflict serializable。

例 2：S 无法转化成 serial schedule

![img](C:/Users/xf/Desktop/CMU15445/pictures/assets%252F-LMjQD5UezC9P8miypMG%252F-LdChJLDEgOtT9muvCHq%252F-LdChnA4mnmOn4_514pc%252FScreen%20Shot%202019-04-24%20at%201.00.05%20PM.jpg)

由于两个 W(A) 之间是矛盾的，无法交换，因此 S 无法转化成 serial schedule。

上文所述的交换法在检测两个 transactions 构成的 schedule 很容易，但要检测多个 transactions 构成的 schedule 是否 serializable 则显得别扭。有没有更快的算法来完成这个工作？

**Dependency Graphs 依赖图**

每个事务有一个节点

$T_i$的一个操作$O_i$与$T_j$的一个操作$O_j$冲突且$O_i$比$O_j$要早。

<img src="C:/Users/xf/Desktop/CMU15445/pictures/image-20220527132211093.png" alt="image-20220527132211093" style="zoom: 50%;" />

如果一个调度的依赖图是无循环的，那么它就是conflict serializable 的。

例1:

![image-20220527132311255](C:/Users/xf/Desktop/CMU15445/pictures/image-20220527132311255.png)

![image-20220527132323851](C:/Users/xf/Desktop/CMU15445/pictures/image-20220527132323851.png)

例2：

![image-20220527132339583](C:/Users/xf/Desktop/CMU15445/pictures/image-20220527132339583.png)

![image-20220527132357645](C:/Users/xf/Desktop/CMU15445/pictures/image-20220527132357645.png)

###### 1.5.2 View Serializability

Schedules $S_1$ and $S_2$ are view equivalent if：

+ If $T_1$ reads initial value of A in $S_1$ , then $T_1$ also reads initial  value of A in $S_2$ .
+ If $T_1$ reads value of A written by $T_2$ in $S_1$ , then $T_1$ also  reads value of A written by $T_2$ in $S_2$ .
+ If $T_1$ writes final value of A in $S_1$ , then $T_1$ also writes final  value of A in $S_2$ .

![image-20220527162724819](C:/Users/xf/Desktop/CMU15445/pictures/image-20220527162724819.png)

![image-20220527162743276](C:/Users/xf/Desktop/CMU15445/pictures/image-20220527162743276.png)

![image-20220527162800284](C:/Users/xf/Desktop/CMU15445/pictures/image-20220527162800284.png)

![image-20220527162831012](C:/Users/xf/Desktop/CMU15445/pictures/image-20220527162831012.png)

![image-20220527162845558](C:/Users/xf/Desktop/CMU15445/pictures/image-20220527162845558.png)

![image-20220527162911071](C:/Users/xf/Desktop/CMU15445/pictures/image-20220527162911071.png)

**In practice, Conflict Serializability is what  systems support because it can be enforced  efficiently.**



##### 1.6 Durability

所有已提交事务的更改都应该是持久的。

+ 没有被更新
+ 失败的事务没有更改

The DBMS can use either logging or shadow  paging to ensure that all changes are durable.