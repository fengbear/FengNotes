### Lecture 4 Trees Indexes



上节提到，DBMS 使用一些特定的数据结构来存储信息：

- Internal Meta-data
- Core Data Storage
- Temporary Data Structures
- Table Indexes

本节将介绍存储 table index 最常用的树形数据结构：B+ Tree，Skip Lists，Radix Tree

table index 为提供 DBMS 数据查询的快速索引，它本身存储着某表某列排序后的数据，并包含指向相应 tuple 的指针。DBMS 需要保证表信息与索引信息在逻辑上保持同步。用户可以在 DBMS 中为任意表建立多个索引，DBMS 负责选择最优的索引提升查询效率。但索引自身需要占用存储空间，因此在索引数量与索引存储、维护成本之间存在权衡。

#### 1 B-Tree Family

B-Tree 中的 B 指的是 Balanced，实际上 B-Tree Family 中的 Tree 都是 Balanced Tree。B-Tree 就是其中之一，以之为基础又衍生出了 $B^+ Tree、B^{link} Tree、B^*Tree$等一系列数据结构。

#### 2 B+ Tree

B+ Tree 是一种自平衡树，它将数据有序地存储，且在 search、sequential access、insertions 以及 deletions 操作的复杂度上都满足 O(logn)O(logn)O(logn) ，其中 sequential access 的最终复杂度还与所需数据总量有关。

B+ Tree 可以看作是 BST (Binary Search Tree) 的衍生结构，它的每个节点可以有多个 children，这特别契合 disk-oriented database 的数据存储方式，每个 page 存储一个节点，使得树的结构扁平化，减少获取索引给查询带来的 I/O 成本。其基本结构如下图所示：

![img](C:/Users/xf/Desktop/CMU15445/pictures/assets%252F-LMjQD5UezC9P8miypMG%252F-LZz8wddPpPJxP1xxb5Y%252F-LZykS00HETGUoHxtn62%252FScreen%20Shot%202019-03-02%20at%2010.14.35%20PM.jpg)

以 M-way B+tree 为例，它的特点总结如下：

- B+ Tree 是 perfectly balanced，即每个 leaf node 的深度都一样
- 除了 root 节点，所有其它节点中至少处于半满状态，即 $M/2−1≤keys≤M−1$ 
- 假设每个 inner node 中包含 k 个 keys，那么它必然有 k+1 个 children
- B+ Tree 的 leaf nodes 通过双向链表串联，从而为 sequential access 提供更高效的支持

![12.jpg](C:/Users/xf/Desktop/CMU15445/pictures/12-166107316953822.jpg)

- inner node: 由 `node*` 和 `key` 组成
- leaf node: 由 `value` 和 `key` 组成

##### 2.1 Node

B+ Tree 中的每个 node 都包含一列按 key 排好序的 key/value pairs，key 就是 table index 对应的 column，value 的取值与 node 类型相关，在 inner nodes 和 leaf nodes 中存的内容不同。

leaf node 的 values 取值在不同数据库中、不同索引优先级中也不同，但总而言之，通常有两种做法：

- Record/Tuple Ids：存储指向最终 tuple 的指针
- Tuple Data：直接将 tuple data 存在 leaf node 中，但这种方式对于二级索引不适用，因为 DBMS 只能将 tuple 数据存储到一个 index 中，否则数据的存储就会出现冗余，同时带来额外的维护成本。

![18.jpg](C:/Users/xf/Desktop/CMU15445/pictures/18-166107316953824.jpg)

- `sortd keys` 排列在一块儿。因为 scan 的时候，只是检查 `key`。另外 `key` 的数据类型大小一致，　`values` 大小很可能不一致，如字符串。

##### 2.2 B-Tree vs. B+Tree

![20.jpg](C:/Users/xf/Desktop/CMU15445/pictures/20-166107316953826.jpg)

##### 2.3 B+Tree Operations

Insert、Delete见MySQL黑马课程的讲解

##### 2.4 B+Tree In Practice

![24.jpg](C:/Users/xf/Desktop/CMU15445/pictures/24-166107316953828.jpg)

+ Fill-Factor 是指 `leaf node的个数 / 所有 node的个数`比值
+ 实际上 B + 树不需要５层以上。

##### 2.5 Clustered Indexes

聚集索引 规定了 table 本身的物理存储方式，通常即按 primary key 排序存储，因此一个 table 只能建立一个 cluster index。有些 DBMSs 对每个 table 都添加聚集索引，如果该 table 没有 primary key，则 DBMS 会为其自动生成一个；而有些 DBMSs 则不支持 clustered indexes。

这种 Clustered Indexes 在对于 range query 很有帮助。range query 的一个例子是要 `primary key` 在 0 到 100 的所有 tuple。对这个例子，如果我们需要 5 个 page。那这 5 个 page 在 clustered index 以后的硬盘上是连续的，一次 sequential I/O 就可以解决。如果不使用 clustered indexes, 那就可能出现 5 次 random I/O，对应的性能就很垃圾了。

##### 2.6 Compound Index

DBMS 支持同时对多个字段建立 table index（B+ Tree），即 compound index，如

```c++
CREATE INDEX compound_idx ON table (a, b, c);
```

##### 2.7 Selection Conditions

Selection Conditions 是 B+ tree 的优势，指我们可以搜索 index 对应的 search key 的 prefix 前缀，suffix 后缀，或者其中的部分。

这是 hash table 无法做到的，它只能搜索完整了 search key。

![26.jpg](C:/Users/xf/Desktop/CMU15445/pictures/26-166107316953830.jpg)

**前缀栗子**

通过前缀我们可以知道我们想要找的 tuple 出现在 index 中的区间，即从小于我们前缀的位置，扫描到大于我们前缀的位置。下面是两个前缀的例子:

![27.jpg](C:/Users/xf/Desktop/CMU15445/pictures/27-166107316953832.jpg)

![28.jpg](C:/Users/xf/Desktop/CMU15445/pictures/28-166107316953834.jpg)

![29.jpg](C:/Users/xf/Desktop/CMU15445/pictures/29-166107316953836.jpg)

**后缀栗子**

我们缺失一个前缀，那就将所有的可能都填入前缀。这样会产生好几个需要扫描的区间，这些区间的个数等于所有前缀可能的个数。如下图 `*` 可以是 A, B, C

![30.jpg](C:/Users/xf/Desktop/CMU15445/pictures/30-166107316953838.jpg)

![31.jpg](C:/Users/xf/Desktop/CMU15445/pictures/31-166107316953840.jpg)

![32.jpg](C:/Users/xf/Desktop/CMU15445/pictures/32-166107316953842.jpg)

##### 2.8 B+ Tree Design Choices

![33.jpg](C:/Users/xf/Desktop/CMU15445/pictures/33-166107316953844.jpg)

###### 2.8.1 Node Size

通常来说，disk 的数据读取速度越慢，node size 就越大：

| Disk      | Node Size |
| --------- | --------- |
| HDD       | ~1MB      |
| SSD       | ~10KB     |
| In-Memory | ~512B     |

###### 2.8.2 Merge Threshold

![35.jpg](C:/Users/xf/Desktop/CMU15445/pictures/35-166107316953846.jpg)

- 如果一个 node 中元素少于 `M/2 - 1`，即没有 half-full, 发生 underflow。我们可以让这个 node 故意保留存在，而不去 merge 它。然后每一段时间，批量处理 B+ tree 中所有这类的 node。

###### 2.8.3 Variable Length Keys

B+ Tree 中存储的 key 经常是变长的，通常有四种手段来应对：

![36.jpg](C:/Users/xf/Desktop/CMU15445/pictures/36-166107316953848.jpg)

- Approach 1: Pointers　太慢 (very rarely used).
- Approach 2: Variable Length Nodes 不适合 fix size page (also rare).
- Approach 3: Padding 太浪费空间
- Approach 4: Key Map / Indirection 性能较好，形式像 slotted pages (the most common approach)。内嵌一个指针数组，指向 node 中的 key/val list 

**Key Map / Indirection** 

将 vaiable lenght key 存在同一个 page

![38.jpg](C:/Users/xf/Desktop/CMU15445/pictures/38-166107316953850.jpg)

![39.jpg](C:/Users/xf/Desktop/CMU15445/pictures/39-166107316953852.jpg)

可以在 `sorted key map` 这个区域存一个**首字母**，这样可以减少跳到 `key+values` 区域的次数。不必每一个 `key` 都去看它的全部内容

##### 2.9 Non-Unique Indexes

索引针对的 key 可能是非唯一的，通常有两种手段来应对：

1. Duplicate Keys：存储多次相同的 key
2. Value Lists：每个 key 只出现一次，但同时维护另一个链表，存储 key 对应的多个 values，类似 chained hashing

![40.jpg](C:/Users/xf/Desktop/CMU15445/pictures/40-166107316953856.jpg)

![41.jpg](C:/Users/xf/Desktop/CMU15445/pictures/41-166107316953854.jpg)

![42.jpg](C:/Users/xf/Desktop/CMU15445/pictures/42-166107316953858.jpg)

![43.jpg](C:/Users/xf/Desktop/CMU15445/pictures/43-166107316953860.jpg)

##### 2.10 Intra-Node Search

Intra-Node Search 指在 node 中搜索，可以想象成在 page 中那些已经排序完的数据中搜索：

###### 2.10.1 Linear

![44.jpg](C:/Users/xf/Desktop/CMU15445/pictures/44-166107316953962.jpg)

###### 2.10.2 Binary Search

###### 2.10.3 Interpolation

Interpolation 指数字的分布如果已知，可以直接猜到想搜的 `key` 的位置。这个属于特例，而且不适用于字符串。

![49.jpg](C:/Users/xf/Desktop/CMU15445/pictures/49-166107316953964.jpg)

##### 2.11 Optimization

![50.jpg](C:/Users/xf/Desktop/CMU15445/pictures/50.jpg)

###### 2.11.1 Prefix Compression

Prefix Compression 即字符串共有的前缀只需要存储一次，能高效节省出存储的空间。

![51.jpg](C:/Users/xf/Desktop/CMU15445/pictures/51-166107316953967.jpg)

###### 2.11.2 Suffix Truncation

我们在最开始说过 leaf node 起到存储信息的作用，inner node 起到 lookup 中导航的作用。对于字符串，inner node 中没有必要保存全部，只需要保存足够的前缀，**保证大于小于的关系和导航的作用**。见下面的例子中，实际上 `a` 和 `l` 也已经足够：

![52.jpg](C:/Users/xf/Desktop/CMU15445/pictures/52-166107316953969.jpg)

![53.jpg](C:/Users/xf/Desktop/CMU15445/pictures/53-166107316953971.jpg)

###### 2.11.3 Bulk Insert

一次次 insert 来获得一个 B+ Tree 很慢。Bulk Insert 是将 keys 先排序，再完成 leaf node, 再根据 leaf node 去完成 inner node。Bulk Insert 让我们能更快去获得一个 B+ Tree：

![54.jpg](C:/Users/xf/Desktop/CMU15445/pictures/54.jpg)

![55.jpg](C:/Users/xf/Desktop/CMU15445/pictures/55-166107316953974.jpg)

###### 2.11.4 Pointer Swizzling

我们每一次从 B+ Tree 中需要一个 page，都需要调用 buffer pool 中的函数，用 `pageid` 获得对应 page 的指针。这种间接 indirection，带来固定的开销。而我们可以节省这一部分，特别在 B+ Tree 的上面几层。因为每次查询都需要上面几层的 page，它们属于 hot page，即它们经常需要有被访问，每一次都找 buffer pool 显得花销更大了。我们完全可以把这些 page 一直留着内存里，然后在 B+Tree 中直接带上它们内容的指针，这样访问它们不需要经过 buffer pool。而这些 hot page 的数量不是很多，完全可以长久留着内存中。

![57.jpg](C:/Users/xf/Desktop/CMU15445/pictures/57-166107316954076.jpg)

![58.jpg](C:/Users/xf/Desktop/CMU15445/pictures/58-166107316954078.jpg)

![59.jpg](C:/Users/xf/Desktop/CMU15445/pictures/59-166107316954080.jpg)

#### 3 Duplicate Keys

![4.jpg](C:/Users/xf/Desktop/CMU15445/pictures/4-166107316954082.jpg)

处理 Duplicate Keys 有两种方法: Append Record Id, Overflow Leaf Nodes。我们下面会用例子去了解这两种作法

##### 3.1 Append Record Id

Record Id 又称 RID，它代表对应元素存储的位置， 自然这个位置可以确定唯一的元素。

Append Record Id 方法中：我们即存储 Key (可以重复), 也存储 Record Id (不可能重复)。

比如下图中的 `1` 实际在 page 中的表示是 `1 | record id of 1`:

![5.jpg](C:/Users/xf/Desktop/CMU15445/pictures/5.jpg)

**例子: Insert 6**

树中已经有了一个 `6`, 我们假设它实际上是 `6 | record id x`。另外我们还希望再插入另外一个 `6`, 我们先将这个新的 `6` 存储在 `record id y`，那么它在树中对应的表达应该是 `6 | record id y`, 我们将这个对 插入树中:

![6.jpg](C:/Users/xf/Desktop/CMU15445/pictures/6-166107316954085.jpg)

![7.jpg](C:/Users/xf/Desktop/CMU15445/pictures/7.jpg)

![8.jpg](C:/Users/xf/Desktop/CMU15445/pictures/8.jpg)

![9.jpg](C:/Users/xf/Desktop/CMU15445/pictures/9.jpg)

![11.jpg](C:/Users/xf/Desktop/CMU15445/pictures/11-166107316954090.jpg)

##### 3.2 Overflow Leaf Nodes

Overflow Leaf Nodes 使用另外一种方式，我们不再存储 record id, 而是将重复的 key 存储在另外一个 page 上， 称为 overflow page。这个 page 上我们选择不排序，同时还允许数值重复出现。我们每次插入在这个 page 的最后端。如果我们需要在 overflow leaft nodes 搜索数值，需要采取 linear search, 原因是数值没有被排序。

**Insert 6, 7, 6**

![12.jpg](C:/Users/xf/Desktop/CMU15445/pictures/12-16610731695361.jpg)

![13.jpg](C:/Users/xf/Desktop/CMU15445/pictures/13-166107316954093.jpg)

![14.jpg](C:/Users/xf/Desktop/CMU15445/pictures/14-166107316954095.jpg)

#### 4 Additional Index Usage

##### 4.1 Implicit Indexes

许多 DBMSs 会自动创建 index，来帮助施行 integrity constraints，情形包括：

- Primary Keys
- Unique Constraints

如当我们创建下面的 foo table 时：

```sql
CREATE TABLE foo (
  id SERIAL PRIMARY KEY,
  val1 INT NOT NULL,
  val2 VARCHAR(32) UNIQUE
);

CREATE TABLE bar (
  id INT REFERENCES foo (val1),
  val VARCHAR(32)
);
```

```sql
CREATE UNIQUE INDEX foo_pkey ON foo (id);        /* primary keys */
CREATE UNIQUE INDEX foo_val2_key ON foo (val2);  /* Unique Constraints */
```

但是对 `REFERENCES` - foreign key 不会自动建立索引，因为 foreign key 对应的数值不一定是唯一的。比如下面的 `foo.val1` 是另外一个关系 `bar` 的外键 foreign key, 但不会给它新建索引，原因是 `foo.val1` 可以是有重复值的。

![18.jpg](C:/Users/xf/Desktop/CMU15445/pictures/18-16610731695362.jpg)

![19.jpg](C:/Users/xf/Desktop/CMU15445/pictures/19-166107316954098.jpg)

`foo.val1` 如果是 `UNIQUE` 的话，那对它建立索引也变得合理了

![20.jpg](C:/Users/xf/Desktop/CMU15445/pictures/20-16610731695363.jpg)

##### 4.2 Partial Indexes

Partial Indexes 并不对整个关系中的所有 tuple 建立索引，只是对其中的一部分建立索引。下列例子中，只是对符合 `foo.c = 'WuTang'` 的 tuple 建立索引，因此也只能对这些 tuple 应用索引加速。

![22.jpg](C:/Users/xf/Desktop/CMU15445/pictures/22-1661073169540101.jpg)

##### 4.3 Covering Indexes

Covering Indexes 意味我们可以直接从 index 索引数据结构中获得数据，不需要跳到 table 存储的位置再读取信息。

如果 query 所需的所有 column 都存在于 index 中，则 DBMS 甚至不用去获取 tuple 本身即可得到查询结果，如下所示：

![24.jpg](C:/Users/xf/Desktop/CMU15445/pictures/24-16610731695364.jpg)

##### 4.4 Index Include Columns

Index Include Columns 在 Covering Indexes 的基础上，再存储了另外的字段。这个**另外的字段**并不进入 index 索引判断，只是为了避免跳到 table 存储的位置再读取信息，提高性能。

**另外的字段**也就是被捎带的。它不会被存储在 inner node, 因为不起到索引的作用。但是被存储在 leaf node 中。

![25.jpg](C:/Users/xf/Desktop/CMU15445/pictures/25-1661073169540104.jpg)

![28.jpg](C:/Users/xf/Desktop/CMU15445/pictures/28-16610731695365.jpg)

##### 4.5 Functional/Expression Indexes

index 中的 key 不一定是 column 中的原始值，也可以是通过计算得到的值

栗子

`EXTRACT(dow FROM login) = 2`:

- `dow` := day of week
- `EXTRACT(dow FROM login)` 从 `login` 这个时间点中获得 `dow`
- `EXTRACT(dow FROM login) = 2` 要求 login 时间点对应星期二

对于 `login` 内部读取出的 `dow` 数据，针对 `login` 的索引不能帮助我们。

![29.jpg](C:/Users/xf/Desktop/CMU15445/pictures/29-16610731695366.jpg)

Function/Expression Indexes 在这种情况能帮助我们，它可以针对 `login` 中读取出的 `dow` 建立索引。

另外这种情况，我们上面见到的 Pratial Index 也可以帮助我们。具体情况，具体分析。

![33.jpg](C:/Users/xf/Desktop/CMU15445/pictures/33-16610731695367.jpg)

#### 5 Trie Index

如果我们想在 B+ Tree 中搜索一个明知道不存在的 key， 我们需要从 root node 经过 inner node，到达 leaf node，在遍历 leaf node 完以后，才能最终发现这个 key 不存在。

Trie Index 是另外一种树型 tree-like 的数据结构。它能解决我们上面的问题，它能搜索的途中发现一个 key 不存在于树中，不必从 root 到 leaf。

Trie Index 也称为: Digital Search Tree, Prefix Tree。 **Radix Tree 是 Trie Tree 的一种。**

Key 不会以一个整体的形式直接出现，而是被解体分开 decomponent 至每一层。我们可以从路径 (从 root 至 leaf 方向) 中读出 key。

下图中的最后的红色标志：可以是对应的 Record ID

![36.jpg](C:/Users/xf/Desktop/CMU15445/pictures/36-16610731695368.jpg)

##### 5.1 Trie Index vs. B+Tree

|                                          | Trie Index                                                   | B+ Tree                                                      |
| :--------------------------------------- | :----------------------------------------------------------- | :----------------------------------------------------------- |
| **树形状**                               | 确定，不会根据 insert 的先后顺序变化。树形状的确定性由 key 的特征决定：长度 | 不确定，会根据 insert 的先后顺序变化。有不同形状的树，但是它们包含的 key 是一样的。 |
| **自平衡 re-balance**                    | 不需要                                                       | 需要                                                         |
| **确定 key 是否存在**                    | 不必须从 root 到 leaf                                        | 必须从 root 到 leaf                                          |
| **key 存储**                             | key 不直接 (implicitly stored) 出现在树中。需要从 root 读到 leaf, 才能间接地重建 re-constructkey | key 肯定直接存在于 leaf node 中，部分 key 另外还出现在 root node 和 inner node 中 |
| **lookup, insert, delete - point query** | O(k), k := length of the key， 大概率更快                    | O(log⁡(n)), n := number of keys， 大概率更慢                  |
| **sequential scan - range query**        | leaf node 间的 sequential scan 无意义。sequential scan 相对繁琐，而且效率一般。每次遇到一个分岔，需要记录 (track path), 遍历完一个分支以后，需要遍历分岔的其他分支。 — DFS 深度搜索 | 可以进行 leaf node 间的 sequential scan。非常简单且高效 (sequential I/O) |

#####  5.2 Trie Key Span

Span:= 指的是 key 对应的进制编码

1-bit Span Trie:= 存储 2 进制的 key，每一个 node 的分支数 fan-out 是 2， 即 0 和 1 这两种可能。也叫 2-way Trie。

![38.jpg](C:/Users/xf/Desktop/CMU15445/pictures/38-16610731695369.jpg)

**栗子**

我们存储 3 个 key: 10, 25, 31 到下面的 1-bit Span Trie 中，具体 key 的二进制表达见下图：

![40.jpg](C:/Users/xf/Desktop/CMU15445/pictures/40-166107316953710.jpg)

![41.jpg](C:/Users/xf/Desktop/CMU15445/pictures/41-166107316953711.jpg)

![42.jpg](C:/Users/xf/Desktop/CMU15445/pictures/42-166107316953712.jpg)

![43.jpg](C:/Users/xf/Desktop/CMU15445/pictures/43-166107316953713.jpg)

![44.jpg](C:/Users/xf/Desktop/CMU15445/pictures/44-166107316953714.jpg)

![45.jpg](C:/Users/xf/Desktop/CMU15445/pictures/45-1661073169541116.jpg)

###### 5.2.1 优化1：水平压缩

因为是 1-bit Span Trie，我们**总是**有两种选择: 0 和 1。我们可以不必存储 0 和 1:

![46.jpg](C:/Users/xf/Desktop/CMU15445/pictures/46-1661073169541118.jpg)

###### 5.2.2 优化2：垂直压缩

另外如果当前路径已经确定 **专属于一个 key (single match)** 的话，我们可以在最先可以确定唯一性的地方提前终止，提供指向 tuple 的 record ID。这样能够减少存储量

![47.jpg](C:/Users/xf/Desktop/CMU15445/pictures/47-1661073169541120.jpg)

![48.jpg](C:/Users/xf/Desktop/CMU15445/pictures/48-1661073169541122.jpg)

##### 5.3 Insert & Delete

Radix Tree 允许多个字母一个节点，不同于 Trie Tree。

**Insert Hair**

![49.jpg](C:/Users/xf/Desktop/CMU15445/pictures/49-166107316953715.jpg)

![50.jpg](C:/Users/xf/Desktop/CMU15445/pictures/50-166107316953716.jpg)

**Delete Hat**

![51.jpg](C:/Users/xf/Desktop/CMU15445/pictures/51-166107316953717.jpg)

![52.jpg](C:/Users/xf/Desktop/CMU15445/pictures/52-166107316953718.jpg)

**Delete Have**

![53.jpg](C:/Users/xf/Desktop/CMU15445/pictures/53-166107316953719.jpg)

![54.jpg](C:/Users/xf/Desktop/CMU15445/pictures/54-166107316953720.jpg)

![56.jpg](C:/Users/xf/Desktop/CMU15445/pictures/56-1661073169541130.jpg)

最后一步中，我们尽可能的 merge 不必要的分岔，这样我们可以更早的去搜速到 `HAIR`

#### 6 Inverted Index

Tree Index 只适合做 point query 和 range query， 而不是适合做 keyword search：

![62.jpg](C:/Users/xf/Desktop/CMU15445/pictures/62-1661073169541132.jpg)

![63.jpg](C:/Users/xf/Desktop/CMU15445/pictures/63-1661073169541134.jpg)

上面的 SQL 并不正确：会搜索到 `Pavlote` 这样将 `Pavlo` 当做 substring 的词。而不是所有关键字为`Pavlo`的文本

![64.jpg](C:/Users/xf/Desktop/CMU15445/pictures/64.jpg)

