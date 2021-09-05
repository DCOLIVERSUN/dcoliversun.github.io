# 列式存储 vs 行式存储：它们之间的本质区别在哪里？


{{< admonition type=tip title="论文链接" open=true >}}
[Column-Stores vs. Row-Stores: How Different Are They
Really?](http://www.cs.umd.edu/~abadi/papers/abadi-sigmod08.pdf)
{{< /admonition >}}

## 概述

该文发表在 2008 年的 SIGMOD 会议上。从论文标题可以看出，论文主要内容不是陈述一种新的技术、架构，而是偏向议论、验证。其主要目的在于阐述清楚在 OLAP 下为什么<ruby>列式存储<rt>Column-Store</rt></ruby>优于<ruby>行式存储<rt>Row-Store</rt></ruby>。

在 OLAP 场景下，基准测试的结论都是列式存储比行式存储快一个数量级。而大家普遍理解的原因是

> column-stores are more I/O efficient for read-only queries since they only have to read from disk (or from memory) those attributes accessed by a query.
> 
> 对于只读查询，列式存储的 I/O 效率更高，因为它们只需要从磁盘（或内存）中读取查询所需要的那些字段。

从这个理解出发，大家想到即使是行式存储，也可以通过一些优化手段达到列式存储的性能，包括：

* <ruby>垂直分表<rt>Vertically Partitioning</rt></ruby>
* <ruby>全列索引<rt>Indexing Every Column</rt></ruby>

## 简介

作者认为当年面向列的系统性能测试方面过于“传统”——仍然以面向行的系统数据结构的设计思路来设计性能测试方案。虽然看出了列式存储的潜力，但没有回答本质问题：

{{< admonition type=question title="问题" open=true >}}
Are these performance gains due to something fundamental about the way column-oriented DBMSs are internally architected, or would such gains also be possible in a conventional system that used a more column-oriented physical design?

列式存储带来的性能提升是由于它内部架构的基本原理导致的，还是采用更加面向列式物理设计的传统系统中也有可能实现这些性能提升？
{{< /admonition >}}

为了验证这个问题，作者采用了垂直分表、<ruby>索引查询<rt>Index-only Plans</rt></ruby>、<ruby>物化视图<rt>Materialized Views</rt></ruby>三种设计范式来实现更加面向列式物理设计的行式系统：

* 垂直分表，将表结构拆分为二元组的形式，来实现对于一个查询，只需要访问特定列的目标；
* 索引查询，为每个表创建一组索引，以确保可以覆盖所有查询中所需要的列，保证查询过程中直接从索引中提取字段的值；
* 物化视图，针对所有查询做物化视图，以获取最佳的查询性能。

经过模拟实验，作者得出结论，**即使在面向行的系统中应用了面向列的思路来设计存储范式，在分析场景下，性能仍然无法与面向列的系统抗衡。** 

在得到上述结论后，作者又抛出下一个问题：

{{< admonition type=question title="问题" open=true >}}
Which of the many column-database speciﬁc optimizations proposed in the literature are most responsible for the signiﬁcant performance advantage of column-stores over row-stores on warehouse workloads?

在数仓场景下，究竟是哪些优化手段让面向列的系统性能优于面向行的系统性能？
{{< /admonition >}}

论文直接给出结论，面向列的系统中优化主要有：

* <ruby>延迟物化<rt>Late Materialization</rt></ruby>
* <ruby>块遍历<rt>Block Iteration</rt></ruby>
* <ruby>压缩<rt>Compression</rt></ruby>
* <ruby>隐式连接<rt>Invisible Joins</rt></ruby>（本文创新优化点）

到这里，作者提出了第三个问题：

{{< admonition type=question title="问题" open=true >}}
However, because each of these techniques was described in a separate research paper, no work has analyzed exactly which of these gains are most signiﬁcant.

定量来看，这几种优化手段分别可以为面向列的系统提升多少性能呢？
{{< /admonition >}}

接下来的工作中，作者逐个移除 C-Store 数据库中的上述优化手段来进行测试，最终发现：

* 压缩带来的性能提升要看具体的数据，最多时可以提升一个数量级
* 延迟物化可以提升 3 倍
* 块遍历和隐式连接可以提升约 1.5 倍

## 面向行的执行

### 垂直分表

垂直分表将表结构拆分成二元组，通过某种机制再将所有字段组合成之前的数据。第一反应可能需要在表中增加一个<ruby>主键字段<rt>Primary Key</rt></ruby>，但是主键字段可能会比较大，并且很多时候主键是复合主键，所以通常需要向每个表添加一个整数 "position" 列。但是查询中如果需要多个列，就得基于 position 列做额外的 join 操作。如果为了加速 join 再引入索引，则又会增加存储和 I/O 的开销，很难有优势。

### 索引查询

垂直分表会引入两个新问题：

* 它需要在每一列上增加一个 position 字段用来还原之前的记录，这会浪费大量的磁盘空间和磁盘带宽；
* 大多数行式存储在每个元组上会存储一个大的头部数据，这又进一步浪费了磁盘空间。

索引查询可以避免上述问题。为了不发生回表查询，势必要将 query 中任意条件的字段，都通过对应字段的索引来过滤出各种主键列表，然后做合并计算。如果某些字段对应的条件无法被其索引快速过滤数据的话，就会导致索引的全扫描，且这样的扫描可能会有多次。最终造成多个索引扫描，合并主键列的速度还不如一趟扫描原始表数据并过滤的速度。另外，大量冗余的元信息和头信息也会造成巨大的性能损失。因此，整体的存储、I/O 开销都很大。

### 物化视图

完全根据预定义的 SQL 来生成确定的物化视图，且其中不会关联多余的列。显然这种方式查询性能很好（插入性能差），I/O 效率高，但这种方法又只能应付极其有限的场景。

## 面向列的执行

### 压缩

压缩，就是将相似度很高、信息熵很低的数据放在一起，用更小的空间表达相同的信息量。列存的数据往往类型相同，相同的特征更多，更容易被压缩，所以列存系统的压缩优化比行存系统更加有效。

压缩优化带来的优势有：

* 压缩后的数据量更小，可以减少硬盘存储空间，同时硬盘的数据量变少在读取时就可以减少 I/O 压力；
* 有些时候解压缩的过程可以省略掉，从而直接对压缩后的数据进行操作。比如使用 Run-Length 编码方式进行压缩的数据，就可以直接进行某些运算。

{{< admonition type=info title="Run-Length 压缩" open=false >}}
对于原始序列为 1, 1, 2, 2, 3, 3, 3 的数据，压缩后表达为 1 * 2,  2 * 2, 3 * 3。当我们要对这一列进行 sum 或者 count 运算时，原始数据可以直接转换为 $sum(1 * 2 + 2 * 2 + 3 * 3) $ 和 $ count(2 + 2 + 3)$，不仅不需要解压缩，而且还提高了计算效率。
{{< /admonition >}}

另外，压缩优化最好可以配合 `sort` 使用，如果数据是经过排序的，则更容易找到相邻数据的同质化特征，获得更好的压缩效果。

### 延迟物化

<ruby>物化<rt>Materialization</rt></ruby> 是指为了把底层面向列的存储格式跟要求的行式格式对上，需要在查询的某个阶段转换一下。

{{< figure src="/images/202109/column-store-vs-row-store/materialization.png" title="Materialization" height="500" width="700">}}

理解物化的概念后，延迟物化就很好理解了，意思就是把物化的时机尽量地拖延到整个查询声明的后期。延迟物化意味着在查询执行的前一段时间内，查询执行的模型不是关系代数，而是基于 Column 的。

例如下面的查询：

```sql
select name 
from person
where id > 10 
  and age > 20
```

一般的做法是从文件系统读出三列的数据，马上物化成一行行的 `person` 数据，然后应用两个过滤条件：$id > 10$ 和 $age > 20$，过滤完了之后从数据里面抽出 `name` 字段，作为最后的结果，大致的转换过程如下图：

{{< figure src="/images/202109/column-store-vs-row-store/early-materialization.png" title="Early Materialization" height="600" width="800">}}

而延迟物化的做法则会先不拼出行式数据，直接在 Column 数据上分别应用两个过滤条件，从而得到两个满足过滤条件的 bitmap，然后再把两个 bitmap 做位与的操作得到同时满足两个条件的所有的 bitmap，因为最后用户需要的只是 `name` 字段而已，因此下一步我们拿着这些 `position` 对 `name` 字段的数据进行过滤就得到了最终的结果。如下图：

{{< figure src="/images/202109/column-store-vs-row-store/late-materialization.png" title="Late Materialization" height="600" width="800">}}

延迟物化有四个方面的好处：

* 关系代数里面的 `selection` 和 `aggregation` 操作下其实不需要整行数据，此时过早物化会浪费；
* 如果数据是被压缩过的，物化的过程就必须对数据进行解压，这会影响压缩带来的好处；
* 列式的内存组织形式对 CPU Cache 非常友好，从而提高计算效率，相反行式的组织形式因为非必要的列占用了 Cache Line 的空间，Cache 效率低；
* 块遍历的优化手段对 Column 类型的数据效果更好，因为数据以 Column 形式保存在一起，数据是定长的可能性更大，而如果 Row 形式保存在一起数据是定长的可能性非常小（因为你一行数据里面只要有一个是非定长的，比如 VARCHAR，那么整行数据都是非定长的）。

### 块遍历

行存模型中，每一个算子在处理数据的时候，都要先迭代一条数据，然后通过定义的接口从数据中获取到某个字段的值，然后再对值进行操作。这个流程使得每处理一条数据就得额外调用一两次用来获取数据的函数（一般称为火山模型）。

而在列式存储中，每一次块迭代都可以获取到多条数据，并且当需要对某一列操作时，可以将一整块列的值传递给处理函数。同时不需要额外调用函数获取值，并且如果列是<ruby>等宽的<rt>fixed-width</rt></ruby>，可以直接作为数组来迭代。

使用上述方法时，可以充分利用 CPU 的很多优势（SIMD加速、cache line适配、CPU pipeline等）。

### 隐式连接

最后本文提出了一个具有创新性的性能优化措施：隐式连接。隐式连接针对的场景是数仓里面的<ruby>星型模型<rt>Star Schema</rt></ruby>， 如果用户查询符合下面的模式就可以应用隐式连接：

{{< figure src="/images/202109/column-store-vs-row-store/star-schema.png" title="Star Schema" height="600" width="600">}}

针对这个模型，我们给出一个 `Join` 的场景，例如以下的 SQL

```sql
SELECT c.nation, s.nation, d.year, sum(lo.revenue) as revenue
FROM customer AS c, lineorder AS lo, supplier AS s, dwdate AS d
WHERE lo.custkey = c.custkey
	AND lo.suppkey = s.suppkey
	AND lo.orderdate = d.datekey
	AND c.region = ’ASIA’
	AND s.region = ’ASIA’
	AND d.year >= 1992 and d.year <= 1997
GROUP BY c.nation, s.nation, d.year
ORDER BY d.year asc, revenue desc;
```

**按照 Selectivity 依次 Join**

这种方式很简单，就是按照谓词的选择性来依次执行 join。

例如，`c.region = 'ASIA'` 的选择性很强，则首先使用这个条件过滤 `customer` 表，然后使用 `customer` 表去 `Join` `LineOrder` 表，所以 `customer` 表中的 `nation` 字段就被增加到了 `customer-order` 表中。依次类推，去完成 `supplier` 和 `date` 表的 `Join`。

这种方式的缺点为：一开始就开始做 `Join` ，后续无法享受上面提到的延迟物化的好处。

**传统延迟物化的 Join**

这个方式可以规避一开始做 `Join` 的行为，具体方法为：

1. 用 `c.region = 'ASIA'` 过滤 `custom` 表，并拿到满足条件的 `custom key` 的集合，同时记录 `custom` 表中满足条件的记录的位置
2. 用 1 中获得的 `custom key` 来过滤 `orderline` 表，并拿到满足条件的记录的位置
3. 遍历 2 中获得的位置列表，提取 `suppplier key`、`order date` 和 `revenue` 并且借助 `custom key` 和 1 中获取到的位置信息，提取 `custom` 表中的 `c.nation` 字段
4. `supplier` 和 `date` 表的 `Join` 类似处理

这种方式的缺点为：在 3 中提取 `c.nation` 的操作为**随机访问**，会产生较大的开销。

**隐式连接**

隐式连接是本文新提出的一种方法，用于上文中提到的星型模型的 `Join` 场景。它优化了传统延迟物化 `Join` 的缺点，尽可能减少随机读取的数据，从而提高性能。具体的执行步骤为：

1. 在每个维度表上应用对应的过滤条件，得到每个<ruby>维度表<rt>Dimension Table</rt></ruby>满足条件的记录的 `key`，同时这个 ` key` 也应该是<ruby>事实表<rt>Fact Table</rt></ruby>的<ruby>外键<rt>Foreign Key</rt></ruby>。

{{< figure src="/images/202109/column-store-vs-row-store/first-phase.png" title="The first phase of the joins needed to execute Query 3.1 from the Star Schema benchmark on some sample data" height="400" width="400">}}

2. 遍历事实表的各个外键列，使用 1 中得到的 `key` 来判断是否满足条件，生成一个满足条件的记录的位置信息的 `bitmap`，并将这些 `bitmap` 做 `AND` 操作，生成最终过滤结果的 `bitmap`。

{{< figure src="/images/202109/column-store-vs-row-store/second-phase.png" title="The second phase of the joins needed to execute Query 3.1 from the Star Schema benchmark on some sample data" height="400" width="400">}}

3. 利用 2 中得到的 `bitmap` 依次提取各个维度表的外键，使用维度表的键来提取维度表中查询所需要的其他列。如果维度表的键是排过序的、从 1 开始连续的值，意味着维度表里面的列可以通过类似访问数组一样的方式提取出来（这一点会比传统的延迟物化方法快很多）。

{{< figure src="/images/202109/column-store-vs-row-store/third-phase.png" title="The third phase of the joins needed to execute Query 3.1 from the Star Schema benchmark on some sample data" height="400" width="400">}}

相比于延迟物化，隐式连接可以减少 I/O 的随机访问，但仅是这样，无法大幅度提升性能。论文认为在很多维度表内符合过滤条件的数据往往是连续的，这样就可以把`Lookup Join`变成范围查询。如果维度表的数据不连续，可以通过 `Dictionary Encoding` 方式，重新按序编码。

{{< admonition type=info title="Dictionary Encoding" open=false >}}
Referennces doc: https://docs.kinetica.com/7.1/concepts/dictionary_encoding/
{{< /admonition >}}

## 总结
读这篇论文最大的收获是首先明确了是哪些因素促使列式存储对 OLAP 性能可以大幅提升——压缩、延迟物化、块遍历、隐式连接。

论文三位作者系统系统解答了列式存储与行式存储的区别，通过实验告诉我们，列式存储是因为其内部架构而具有更好的性能，而不是理所当然的理由——更少的 I/O。不仅仅限于内部架构，查询引擎层的各种优化也同样是列式存储性能提升的关键。

