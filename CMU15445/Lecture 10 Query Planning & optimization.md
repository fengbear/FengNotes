### Lecture 10 Query Planning & Optimization

记住SQL是声明性的。

+ 用户告诉DBMS他们想要的答案，而不是如何去得到这个答案

根据所使用的查询计划，性能可能会有很大的差异。

SQL 语句让我们能够描述想要获取的数据，而 DBMS 负责来根据用户的需求来制定高效的查询计划。不同的查询计划的效率可能出现多个数量级的差别，如 Join Algorithms 一节中的 Simple Nested Loop Join 与 Hash Join 的时间对比 (1.3 hours vs. 0.45 seconds)。

Query Optimizer 第一次出现在 IBM System R，那时人们认为 DBMS 指定的查询计划永远无法比人类指定的更好。System R 的 optimizer 中的一些理念至今仍在使用。

#### 1 Query Optimization 

Heuristics/Rules

- Rewrite the query to remove stupid/inefficient things
- These techniques may need to examine catalog，but they do not need to examine data

Cost-based Search

- Use a model to estimate the cost of executing a plan
- Evaluate multiple equivalent plans for a query and pick  the one with the lowest cost

> 1. 基于规则的： 通过重写QUERY来消除不高效，不需要一个成本模型。
> 2. 基于成本的： 使用成本模型来评估多种等价计划然后选择成本最小的。

![image-20220520103112617](C:/Users/xf/Desktop/CMU15445/pictures/image-20220520103112617.png)

**Logical vs. Physical plans**

优化器生成逻辑代数表达式到最佳等价物理代数表达式的映射。

Physical operators define a specific execution  strategy using an access path.

+ They can depend on the physical format of the data that  they process
+ Not always a 1:1 mapping from logical to physical

这里的 Rewriter 负责 Heuristics/Rules，Optimizer 则负责 Cost-based Search。

#### 2 Heuristics/Rules

##### 2.1 Query Rewriting

如果两个关系代数表达式 (Relational Algebra Expressions) 如果能产生相同的 tuple 集合，我们就称二者等价。DBMS 可以通过一些 Heuristics/Rules 来将关系几何表达式转化成成本更低的等价表达式，从而达到查询优化的目的。这些规则通常试用于所有查询，如：

+ Predicate Pushdown
+ Projections Pushdown

**Predicate Pushdown**

Predicate 通常有很高的选择性，可以过滤掉许多无用的数据。将 Predicate 推到查询计划的底部，可以在查询开始时就更多地过滤数据，举例如下：

![image-20220524101609826](C:/Users/xf/Desktop/CMU15445/pictures/image-20220524101609826.png)

![image-20220524101624614](C:/Users/xf/Desktop/CMU15445/pictures/image-20220524101624614.png)

![image-20220524101736850](C:/Users/xf/Desktop/CMU15445/pictures/image-20220524101736850.png)

核心思想如下：

- 越早过滤越多数据越好
- 重排 predicates，使得选择性大的排前面
- 将复杂的 predicate 拆分，然后往下压，$\boldsymbol{\sigma}_{\mathrm{p} 1 \wedge \mathrm{p} 2 \wedge \ldots \mathrm{p} n}(\mathbf{R})=\boldsymbol{\sigma}_{\mathrm{p} 1}\left(\boldsymbol{\sigma}_{\mathrm{p} 2}\left(\ldots \boldsymbol{\sigma}_{\mathrm{p} n}(\mathbf{R})\right)\right)$
- 简化复杂predicate，如 `X=Y AND Y=3`
- 
- 可以修改成 `X=3 AND Y=3`

**Projections Pushdown**

本方案对列存储数据库不适用。在行存储数据库中，越早过滤掉不用的字段越好，因此将 Projections 操作往查询计划底部推也能够缩小中间结果占用的空间大小，举例如下：

![img](C:/Users/xf/Desktop/CMU15445/pictures/assets%252F-LMjQD5UezC9P8miypMG%252F-LapLaibKEqy0uOzxt1N%252F-LapOlNeBS4k6SZ45deL%252FScreen%20Shot%202019-03-25%20at%2011.10.13%20PM.jpg)

##### 2.2 More Examples

###### 2.2.1 **Impossible/Unnecessary Predicates**

![image-20220524125002412](C:/Users/xf/Desktop/CMU15445/pictures/image-20220524125002412.png)

可以变为

![image-20220524125254246](C:/Users/xf/Desktop/CMU15445/pictures/image-20220524125254246.png)

###### 2.2.2 **Join Elimination**

![image-20220524125226643](C:/Users/xf/Desktop/CMU15445/pictures/image-20220524125226643.png)

可以变为

![image-20220524125312748](C:/Users/xf/Desktop/CMU15445/pictures/image-20220524125312748.png)

###### 2.2.3 **Ignoring Projections**

![image-20220524125339829](C:/Users/xf/Desktop/CMU15445/pictures/image-20220524125339829.png)

可以变为

![image-20220524125403909](C:/Users/xf/Desktop/CMU15445/pictures/image-20220524125403909.png)

###### 2.2.4 Merging Predicates

![image-20220524125451500](C:/Users/xf/Desktop/CMU15445/pictures/image-20220524125451500.png)

可以变为

![image-20220524125504922](C:/Users/xf/Desktop/CMU15445/pictures/image-20220524125504922.png)

**We can use static rules and heuristics to optimize a  query plan without needing to understand the  contents of the database.**

#### 3 Cost-Based Search

除了 Predicates 和 Projections 以外，许多操作没有通用的规则，如 Join：Join 操作既符合交换律又符合结合律，等价关系代数表达式数量庞大，这时候就需要一些成本估算技术，将过滤性大的表作为 Outer Table，小的作为 Inner Table，从而达到查询优化的目的。

##### 3.1 **Plan Cost Estimation**

一个查询需要花费多长时间，取决于许多因素 

- CPU: Small cost; tough to estimate
- Disk: #block transfers
- Memory: Amount of DRAM used
- Network: #messages

但本质上取决于：**整个查询过程需要读入和写出多少 tuples**



###### 3.1.1 Statistics 

Statistics catalog  统计目录 

用于估算成本

DBMS 需要保存每个 table 的一些统计信息在他们内部的catalog中，如 attributes、indexes 等信息，有助于估计查询成本。值得一提的是，不同的 DBMS 的搜集、更新统计信息的策略不同。

对于每个关系R，DBMS会维护以下信息

**假设数据是均匀分布的**，且：

+ N(R)：代表关系R的tuple数量
+ V(A, R)：代表关系R中attribute A的value数量（不重复的），**即A有多少种取值**

由于数据均匀分布，我们可以计算Attribute A种每个值所包含的tuple数为： $$S C(A, R)=N(R) / V(A, R)$$，SC称为selection cardinality（选择基数）。

**Selection Statistics**

如果在一个unique key上使用等价谓词（判断）。我们能很容易知道选择基数，就是1。

但是如果是更加复杂的谓词，范围条件或者交集运算之类的东西，如何算选择基数呢？

![image-20220524232202482](C:/Users/xf/Desktop/CMU15445/pictures/image-20220524232202482.png)

选择率sel就是一个函数，对于针对该表的一个给定条件来说，它会算出该表有多少符合该条件的tuple。

那么针对不同的谓词，其sel为

- **Equality：**$\operatorname{sel}(A=\text { constant })=S C(P) / N_{R}$
- **Range Predicate：**![[公式]](C:/Users/xf/Desktop/CMU15445/pictures/equation.svg+xml)
- **Negation Query：**![[公式]](C:/Users/xf/Desktop/CMU15445/pictures/equation-16610733675461.svg+xml)
- **Conjunction Query：**![[公式]](C:/Users/xf/Desktop/CMU15445/pictures/equation-16610733675472.svg+xml) 。这里各个谓词需要独立
- **Disjunction Query：**![[公式]](C:/Users/xf/Desktop/CMU15445/pictures/equation-16610733675473.svg+xml) 。同上

以下是一些例子：

![image-20220524232711126](C:/Users/xf/Desktop/CMU15445/pictures/image-20220524232711126.png)

![image-20220524232843489](C:/Users/xf/Desktop/CMU15445/pictures/image-20220524232843489.png)

![image-20220524233001534](C:/Users/xf/Desktop/CMU15445/pictures/image-20220524233001534.png)

![image-20220524233115046](C:/Users/xf/Desktop/CMU15445/pictures/image-20220524233115046.png)

![image-20220524233148748](C:/Users/xf/Desktop/CMU15445/pictures/image-20220524233148748.png)

###### 3.1.2 Cost Estimations

**HISTOGRAMS WITH QUANTILES**

我们的公式很好，但我们假设数据值是均匀分布的。

最简单的直方图，只需要统计列中不同值的出现次数

![image-20220524234142543](C:/Users/xf/Desktop/CMU15445/pictures/image-20220524234142543.png)

![image-20220524234203013](C:/Users/xf/Desktop/CMU15445/pictures/image-20220524234203013.png)

但是上述给每个不同的值都存储一个该值有多少个的话，太大了。解决方法就是划分bucket，一个bucket中我们只保存一个值，我们不会为一个bucket中的每一个元素都保存一个值。

![image-20220524235654974](C:/Users/xf/Desktop/CMU15445/pictures/image-20220524235654974.png)

当bucket = 3，会转变为以下的直方图：

![image-20220524235706256](C:/Users/xf/Desktop/CMU15445/pictures/image-20220524235706256.png)

但上述方法也不太好，等宽bucket可能有的bucket中的值很多，有的很少。所以使用变宽bucket

![image-20220524235902418](C:/Users/xf/Desktop/CMU15445/pictures/image-20220524235902418.png)

![image-20220524235909482](C:/Users/xf/Desktop/CMU15445/pictures/image-20220524235909482.png)

**SAMPLING**

维护一张样本表，根据该样本来衍生出统计信息。

![image-20220525000327386](C:/Users/xf/Desktop/CMU15445/pictures/image-20220525000327386.png)

![image-20220525000340002](C:/Users/xf/Desktop/CMU15445/pictures/image-20220525000340002.png)

**通过以上两种方法，我们可以(粗略地)估计谓词的sel，那么我们实际上可以用它们做什么呢?**



##### 3.2 Plan Enumeration 计划枚举

After performing rule-based rewriting, the DBMS  will enumerate different plans for the query and  estimate their costs.

+ Single relation
+ Multiple relations
+ Nested sub-queries

It chooses the best plan it has seen for the query  after exhausting all plans or some timeout.

###### 3.2.1 Single relation query planning

首先是access method的选择

Pick the best access method.

+ Sequential Scan
+ Binary Search (clustered indexes)
+ Index Scan

其次是评估条件时的顺序，当一个评估条件可以提前丢掉更多数据，我肯定先评估这个条件。（这时就可以用到sel了）

![image-20220525001703172](C:/Users/xf/Desktop/CMU15445/pictures/image-20220525001703172.png)

**sargable**

+ 可以选择最好的index
+ Join几乎总是在具有较小基数的外键关系上。

###### 3.2.2 Multi-Relation query planning

![image-20220525002032826](C:/Users/xf/Desktop/CMU15445/pictures/image-20220525002032826.png)

![image-20220525002123461](C:/Users/xf/Desktop/CMU15445/pictures/image-20220525002123461.png)

这样不需要延后结果，A与Bjion的结果可以直接去到下一个jion operator 与C jion

像最右边，先C、Djion，然后A，Bjion。（？）

如何枚举我们的query plan？

![image-20220525002456230](C:/Users/xf/Desktop/CMU15445/pictures/image-20220525002456230.png)

**Dynamic Programming**

![image-20220525002527604](C:/Users/xf/Desktop/CMU15445/pictures/image-20220525002527604.png)

![image-20220525002536286](C:/Users/xf/Desktop/CMU15445/pictures/image-20220525002536286.png)

![image-20220525002544741](C:/Users/xf/Desktop/CMU15445/pictures/image-20220525002544741.png)

![image-20220525002554382](C:/Users/xf/Desktop/CMU15445/pictures/image-20220525002554382.png)

![image-20220525002606257](C:/Users/xf/Desktop/CMU15445/pictures/image-20220525002606257.png)

举一个具体的例子（完整版）：

![image-20220525002844303](C:/Users/xf/Desktop/CMU15445/pictures/image-20220525002844303.png)

![image-20220525002918944](C:/Users/xf/Desktop/CMU15445/pictures/image-20220525002918944.png)

![image-20220525002906148](C:/Users/xf/Desktop/CMU15445/pictures/image-20220525002906148.png)

![image-20220525002959769](C:/Users/xf/Desktop/CMU15445/pictures/image-20220525002959769.png)

![image-20220525003042649](C:/Users/xf/Desktop/CMU15445/pictures/image-20220525003042649.png)



还有一种优化方法GEQO

![image-20220525003200104](C:/Users/xf/Desktop/CMU15445/pictures/image-20220525003200104.png)

![image-20220525003256405](C:/Users/xf/Desktop/CMU15445/pictures/image-20220525003256405.png)

![image-20220525003308371](C:/Users/xf/Desktop/CMU15445/pictures/image-20220525003308371.png)

![image-20220525003342758](C:/Users/xf/Desktop/CMU15445/pictures/image-20220525003342758.png)

![image-20220525003352401](C:/Users/xf/Desktop/CMU15445/pictures/image-20220525003352401.png)

![image-20220525003400962](C:/Users/xf/Desktop/CMU15445/pictures/image-20220525003400962.png)

![image-20220525003409422](C:/Users/xf/Desktop/CMU15445/pictures/image-20220525003409422.png)

###### 3.2.3 Nested sub-queries

DBMS将where子句中的嵌套子查询视为接受参数并返回单个值或一组值的函数。

两种方法：

+ Rewrite to de-correlate and/or flatten them
+ 分解嵌套查询并将结果存储到临时表中

**rewrite**

![image-20220525003627173](C:/Users/xf/Desktop/CMU15445/pictures/image-20220525003627173.png)

![image-20220525003655438](C:/Users/xf/Desktop/CMU15445/pictures/image-20220525003655438.png)

**Decomposing query**

![image-20220525003741134](C:/Users/xf/Desktop/CMU15445/pictures/image-20220525003741134.png)

对于更难的查询，优化器将查询分解成块，然后一次集中处理一个块

子查询被写入到一个临时表中，在查询完成后丢弃这些临时表。











