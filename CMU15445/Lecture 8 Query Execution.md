### Lecture 8 Query Execution

![img](C:/Users/xf/Desktop/CMU15445/pictures/assets%252F-LMjQD5UezC9P8miypMG%252F-L_SJmRDBsMOcTdYyKNm%252F-L_SKqDH3suePXuli3Z0%252FScreen%20Shot%202019-03-08%20at%208.46.34%20PM.jpg)

如上图所示，通常一个 SQL 会被组织成树状的查询计划，数据从 leaf nodes 流到 root，查询结果在 root 中得出。而本节将讨论在这样一个计划中，如何为这个数据流动过程建模，大纲如下：

- Processing Models
- Access Methods
- Expression Evaluation

#### 1 Processing Models

DBMS 的 processing model 定义了系统如何执行一个 query plan，目前主要有三种模型:

- Iterator Model
- Materialization Model
- Vectorized/Batch Model

不同模型的适用场景不同。

处理模型分为3类，迭代模型，主要是自顶向下的。物化模型，是自底向上的。矢量模型，像迭代模型一样通过next来交互，但是是批量发送数据，也是自顶向下的。

##### 1.1 Iterator (Volcano / Pipeline) Model

query plan 中的每步 operator 都实现一个 next 函数，每次调用时，operator 返回一个 tuple 或者 null，后者表示数据已经遍历完毕。operator 本身实现一个循环，每次调用其 child operators 的 next 函数，从它们那边获取下一条数据供自己操作，这样整个 query plan 就被从上至下地串联起来，它也称为 Volcano/Pipeline Model：

![img](C:/Users/xf/Desktop/CMU15445/pictures/assets%252F-LMjQD5UezC9P8miypMG%252F-L_SJmRDBsMOcTdYyKNm%252F-L_SPAXpoLiDtZYjPQzb%252FScreen%20Shot%202019-03-08%20at%209.05.30%20PM.jpg)

Iterator 几乎被用在每个 DBMS 中，包括 sqlite、MySQL、PostgreSQL 等等，其它需要注意的是：

- 有些 operators 会等待 children 返回所有 tuples 后才执行，如 Joins, Subqueries 和 Order By
- Output Control 在 Iterator Model 中比较容易，如 Limit，只按需调用 next 即可。

> 所有的代数运算符(operator)都被看成是一个迭代器，它们都提供一组简单的接口：open()—next()—close()，查询计划树由一个个这样的关系运算符组成，每一次的next()调用，运算符就返回一行(Row)，每一个运算符的next()都有自己的流控逻辑，数据通过运算符自上而下的next()嵌套调用而被动的进行拉取。

##### 1.2 Materialization Model

每个 operator 处理完所有输入后，将所有结果一次性输出，DBMS 会将一些参数传递到 operator 中防止处理过多的数据，这也是一种从上至下的思路，示意如下：

![img](C:/Users/xf/Desktop/CMU15445/pictures/assets%252F-LMjQD5UezC9P8miypMG%252F-L_SJmRDBsMOcTdYyKNm%252F-L_SQydRFxzDN-0YMd92%252FScreen%20Shot%202019-03-08%20at%209.13.22%20PM.jpg)

materialization model：

- 更适合 OLTP 场景，因为后者通常指需要处理少量的 tuples，这样能减少不必要的执行、调度成本
- 不太适合会产生大量中间结果的 OLAP 查询



##### 1.3 Vectorization Model

Vectorization Model 是 Iterator 与 Materialization Model 折衷的一种模型：

- 每个 operator 实现一个 next 函数，但每次 next 调用返回一批 tuples，而不是单个 tuple
- operator 内部的循环每次也是一批一批 tuples 地处理
- batch 的大小可以根据需要改变（hardware、query properties）

![img](C:/Users/xf/Desktop/CMU15445/pictures/assets%252F-LMjQD5UezC9P8miypMG%252F-L_SJmRDBsMOcTdYyKNm%252F-L_SSpL7OtEU8ncA5L95%252FScreen%20Shot%202019-03-08%20at%209.21.24%20PM.jpg)

vectorization model 是 OLAP 查询的理想模型：

- 极大地减少每个 operator 的调用次数
- 允许 operators 使用 vectorized instructions (SIMD) 来批量处理 tuples



**三种方式都是从上到下的方法**

#### 2 Access Methods

access method 指的是 DBMS 从数据表中获取数据的方式，它并没有在 relational algebra 中定义。主要有三种方法：

- Sequential Scan
- Index Scan
- Multi-Index/"Bitmap" Scan

##### 2.1 Sequential Scan

顾名思义，sequential scan 就是按顺序从 table 所在的 pages 中取出 tuple，这种方式是 DBMS 能做的最坏的打算。

```sql
for page in table.pages:
    for t in page.tuples:
        if evalPred(t):
            # do something
```

DBMS 内部需要维护一个 cursor 来追踪之前访问到的位置（page/slot）。Sequential Scan 是最差的方案，因此也针对地有许多优化方案：

+ Prefetching
+ Parallelization
+ Buffer Pool Bypass
+ (本节) Zone Maps
+ (本节) Late Materialization
+ (本节) Heap Clustering

###### 2.1.1 Zone Maps

预先为每个 page 计算好 attribute values 的一些统计值，DBMS 在访问 page 之前先检查 zone map，确认一下是否要继续访问，如下图所示：

![img](C:/Users/xf/Desktop/CMU15445/pictures/assets%252F-LMjQD5UezC9P8miypMG%252F-L_SJmRDBsMOcTdYyKNm%252F-L_SYzTzVdvka1QB0F-K%252FScreen%20Shot%202019-03-08%20at%209.48.23%20PM.jpg)

当 DBMS 发现 page 的 Zone Map 中记录 val 的最大值为 400 时，就没有必要访问这个 page。

**Zone Maps 可能放在单独的page中，也可能放在当前的page中。**

###### 2.1.2 Late Materialization

每个Operator传递下一个Operator所需的最少量信息（例如，Record ID）。这仅适用于列存储系统（即DSM）。

> 尽量延迟到后面的操作再去拿数据，适合DSM，因为DSM一个tuple分散在不同的page，想拿一个太麻烦，不如再不需要整个的时候传递一个Record ID就行了。

在列存储 DBMS 中，每个 operator 只选取查询所需的列数据，若该列数据在查询树上方并不需要，则仅需向上传递 offsets 即可：

> 比如a > 100 这个操作后 后面操作不需要a这一列，那就向上传一个Offset就行。

![img](C:/Users/xf/Desktop/CMU15445/pictures/webp.webp)



###### 2.1.3 Heap Clustering（聚集索引）

使用 clustering index 时，tuples 在 page 中按照相应的顺序排列，如果查询访问的是被索引的 attributes，DBMS 就可以直接跳跃访问目标 tuples。



##### 2.2 Index Scan

DBMS 选择一个 index 来找到查询需要的 tuples。使用哪个 index 取决于以下几个因素：

- index 包含哪些 attributes
- 查询引用了哪些 attributes
- attribute 的定义域
- predicate composition
- index 的 key 是 unique 还是 non-unique

这些问题都将在后面的课程中详细描述，本节只是对 Index Scan 作概括性介绍。

尽管选择哪个 Index 取决于很多因素，但其核心思想就是，越早过滤掉越多的 tuples 越好，如下面这个 query 所示：

```sql
SELECT * FROM students
 WHERE age < 30
   AND dept = 'CS'
   AND country = 'US';
```

![img](C:/Users/xf/Desktop/CMU15445/pictures/assets%252F-LMjQD5UezC9P8miypMG%252F-L_SJmRDBsMOcTdYyKNm%252F-L_ScefnfeWAkOd66WZf%252FScreen%20Shot%202019-03-08%20at%2010.08.28%20PM.jpg)

+ Scenario #1：使用 dept 的 index 能过滤掉更多的 tuples
+ Scenario #2：使用 age 的 index 能过滤掉更多的 tuples

##### 2.3 Multi-Index/"Bitmap" Scan

如果有多个 indexes 同时可以供 DBMS 使用，就可以做这样的事情：

- 计算出符合每个 index 的 tuple id sets
- 基于 predicates (union vs. intersection) 来确定是对集合取交集还是并集
- 取出相应的 tuples 并完成剩下的处理

Postgres 称 multi-index scan 为 Bitmap Scan。

仍然以上一个 SQL 为例，使用 multi-index scan 的过程如下所示：

![img](C:/Users/xf/Desktop/CMU15445/pictures/assets%252F-LMjQD5UezC9P8miypMG%252F-L_SJmRDBsMOcTdYyKNm%252F-L_SeJOQCHNWwzK9pYNo%252FScreen%20Shot%202019-03-08%20at%2010.16.01%20PM.jpg)

其中取集合交集可以使用 bitmaps, hash tables 或者 bloom filters。

**Index Scan Page Sorting**

当使用的不是 clustering index 时，实际上按 index 顺序检索的过程是非常低效的，DBMS 很有可能需要不断地在不同的 pages 之间来回切换。为了解决这个问题，DBMS 通常会先找到所有需要的 tuples，根据它们的 page id 来排序，完毕后再读取 tuples 数据，使得整个过程每个需要访问的 page 只会被访问一次。如下图所示：

![img](C:/Users/xf/Desktop/CMU15445/pictures/assets%252F-LMjQD5UezC9P8miypMG%252F-L_SJmRDBsMOcTdYyKNm%252F-L_ShA0GMpjwLpjmIrHN%252FScreen%20Shot%202019-03-08%20at%2010.28.29%20PM.jpg)

##### 2.4 Expression Evaluation

DBMS 使用 expression tree 来表示一个 WHERE 语句，如下图所示：

![img](C:/Users/xf/Desktop/CMU15445/pictures/assets%252F-LMjQD5UezC9P8miypMG%252F-L_SJmRDBsMOcTdYyKNm%252F-L_ShOP570xIvRR3bk4N%252FScreen%20Shot%202019-03-08%20at%2010.29.28%20PM.jpg)

然后根据 expression tree 完成数据过滤的判断，但这个过程比较低效，很多 DBMS 采用 JIT Compilation 的方式，直接将比较的过程编译成机器码来执行，提高 expression evaluation 的效率。