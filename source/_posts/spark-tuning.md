---
title: Spark并行化基本分析
date: 2019-05-08 23:18:28
tags: distributed system
---

#### shuffle

在有宽依赖（一个父节点的数据会被多个子节点用到）的时候会发生shuffle。在MR里，shuffle连接了map和reduce，它本身的实现在不断优化，所以在spark里，弱化了map和reduce的细节，把shuffle的概念提高了。

简单实现的情况，因为map任务和reduce任务里数据的存储方式是不清楚的，map会生成数量等于reduce任务个数的文件，每个reduce会从上游**所有**的节点去取数据。但这其实有优化空间，比如相同key的数据已经在一个机器上了呢？是不需要从所有节点拉数据的。

从另一个方面，简单理解，若数据分组key改变了，就会完全shuffle：如果下一个分组的方式和上一个分组的方式没有关系，会导致完全shuffle发生。

#### partition是数据的逻辑映射

Spark和MR在容错上不一样，Spark不用在HDFS上多副本，如果一个节点出错了，理论上就重新从父RDD计算出来。

partition是逻辑概念，partition和任务相关，用于计算。在数据层面，文件块是block的概念，但block是专门的StoreManager去管理。虽然不是一回事，但总之是有对应关系的。

在缩小一个rdd的partition时，是会发生shuffle的，只是不是完全shuffle。

#### 算子和MR

- Spark的算子是要对应上MR的，对于满足交换律和结合律操作，可以用类似combiner的处理，在mapper节点完成。

  以关系代数的为例：

  | 操作  | map              | reduce |
  | ----- | ----------------| ------ |
  | where | t若满足条件才输出(t,t) |        |
  | select | t选择对应列,(t', t') |        |
  | select distinct | t选择对应列(t', t') | (t', [t',t'...])去重 (t', t') |
  | union all | t直接输出(t,t) |  |
  | union |  t直接输出(t,t) |   (t, [t,t...])输出 (t, t)    |
  | intersection | t 输出 (t,t) | 满足(t, [t,t]) 才输出 |
  | difference | t 输出 (t, x)或(t, y) | 满足(t, [x]) 才输出 |
  | join | (a,b)和(b,c) 输出 (b, (a, x))或(b, (c,y)) | (b,[(a,x),(c,y),(d,x),(e,y)...])将x的和y的分开，然后两边做笛卡尔积，输出[（a,b,c）,(a,b,e), (d,b,c),(d,b,e)...] |
  | groupby | 按key t输出 (t, a)，(t, b)... |  |

#### 并行度

partition是计算的逻辑单位，应该有多少paritition就有多少task，就能有多大的并行。默认情况，map的并行就是输入的partition数量，这个和文件大小有关，reduce的并行是父rdd的最大partition个数。

每次做map操作，分区信息都会丢失，所以map类操作提供了preservedPartition来保留分区。这也是为什么对pair rdd更推荐用mapValues，因为它保留了分区，filter也是一个保留了分区的算子。

spark.default.parallelism：这个参数适用于RDD，设置了任务的并行度，对于map和reduce的task都有用。

spark.sql.shuffle.partitions：这参数针对dataframe的。shuffle的个数其实反应的也是reduce的个数，如果shuffle partition多了，那reduce的并行度就会高。

#### 调度

FAIR调度：比较TaskSetManager（一个stage中所有的task）中runningTasks和其minShare值，优先执行runningTasks比较少的，使得资源不被长时间的任务占据了。

本地化调度：采用延迟调度算法，先判别task的locality，看executor是否满足（任务的可容忍的延迟），不满足则逐步退化。

#### 动态资源申请

同时开启：spark.dynamicAllocation.enabled和ShuffleService，可以不用配置executor-num

