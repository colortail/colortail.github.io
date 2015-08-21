---
layout: post
title: 关系数据库是如何工作的
categories: interpret
---

译自：[How does a relational database work](http://coding-geek.com/how-databases-work/)

翻译：[找不到工作的colortail](www.colortail.com)

When it comes to relational databases, I can’t help thinking that something is missing. They’re used everywhere. There are many different databases: from the small and useful SQLite to the powerful Teradata. But, there are only a few articles that explain how a database works. You can google by yourself “how does a relational database work” to see how few results there are. Moreover, those articles are short.  Now, if you look for the last trendy technologies (Big Data, NoSQL or JavaScript), you’ll find more in-depth articles explaining how they work.

Are relational databases too old and too boring to be explained outside of university courses, research papers and books?

![main_databases](http://pic.yupoo.com/tan91319/ETrtyLaY/medish.jpg)

As a developer, I HATE using something I don’t understand. And, if databases have been used for 40 years, there must be a reason. Over the years, I’ve spent hundreds of hours to really understand these weird black boxes I use every day. **Relational Databases are** very interesting because they’re **based on useful and reusable concepts**. If understanding a database interests you but you’ve never had the time or the will to dig into this wide subject, you should like this article.

Though the title of this article is explicit, **the aim of this article is NOT to understand how to use a database. Therefore, you should already know how to write a simple join query and basic CRUD queries**; otherwise you might not understand this article. **This is the only thing you need to know**, I’ll explain everything else.

I’ll start with some computer science stuff like time complexity. I know that some of you hate this concept but, without it, you can’t understand the cleverness inside a database.  
鉴于这个题目的庞大，我将专注于我认为是核心基础的部分：数据库处理SQL查询的方式。  我也只会呈现数据库背后的基本概念，这样在文章的末尾，你就能对表面下正在发生什么有很好的理解。

Since it’s a long and technical article that involves many algorithms and data structures, take your time to read it. Some concepts are more difficult to understand; you can skip them and still get the overall idea.

For the more knowledgeable of you, this article is more or less divided into 3 parts: 

* 对底层和高层数据库组件的概览
* 对查询优化过程的概览
* 对事务和缓冲池管理的概览

##<a id="index"></a>目录
* [1 追溯基础](#back_to_basics)
	* [1.1 O(1) vs O(n2)](#o1vson2)
		* [1.1.1 概念](#concept)
		* [1.1.2 例子](#example)
		* [1.1.3 深入](#going_deeper)
	* [1.2 归并排序](#mergesort)
		* [1.2.1 归并](#merge)
		* [1.2.2 分解阶段](#division_phase)
		* [1.2.3 排序阶段](#sorting_phase)
		* [1.2.4 归并排序之强](#power_of_mergesort)
	* [1.3 数组，树和哈希表](#array_tree_hashtable)
		* [1.3.1 数组](#array)
		* [1.3.2 树和数据库索引](#tree_index)
		* [1.3.3 哈希表](#hashtable)
* [2 整体概览](#global)
* [3 客户端管理](#client)
* [4 查询管理](#query)
	* [4.1 查询解析器](#query_parser)
	* [4.2 查询重写器](#query_rewriter)
	* [4.3 统计](#statistics)
	* [4.4 查询优化器](#query_optimizer)
		* [4.4.1 索引](#indexs)
		* [4.4.2 访问路径](#access_path)
		* [4.4.3 连接运算符](#join_operators)
		* [4.4.4 简化示例](#simplified_example)
		* [4.4.5 动态规划，贪婪和启发性算法](#dp_greedy_heuristic)
		* [4.4.6 真正的优化器](#real_optimizers)
		* [4.4.7 缓存查询计划](#query_plan_cache)
	* [4.5 查询执行器](#query_executor)
* [5 数据管理](#data)
	* [5.1 缓存管理](#cache_manager)
		* [5.1.1 预读取](#prefetching)
		* [5.1.2 缓冲区置换策略](#buffer_replacement)
		* [5.1.3 写缓冲](#write_buffer)
	* [5.2 事务管理](#transaction_manager)
		* [5.2.1 ACID原则](#acid)
		* [5.2.2 并发控制](#concurrency)
		* [5.2.3 锁管理](#lock)
		* [5.2.4 日志管理](#log)
* [6 总结](#conclude)

##<a id="back_to_basics"></a>追溯基础

很久很久以前（大概宇宙大爆炸那时吧），程序员们得精确地知道他们的代码将产生的指令数量，他们得对数据结构和算法烂熟于心，因为那时的电脑很慢，一点对CPU和内存的浪费他们都承担不起。

在这一部分，我将提向你及这些概念，因为它们对理解数据库是很基本的。我也将介绍**数据库索引**的概念。

###<a id="o1vson2"></a>O(1) vs O(n<sup>2</sup>)

现在，许多开发者不太关心时间复杂度了...不过，他们是对的。

但是，当你处理大量数据（不是几千而已），或者当你对程序的需求是毫秒级的话，理解这些概念就变得很关键了。而且你猜怎么了，数据库就得处理这所有的情形。我不会一直讨论这些枯燥的问题，但现在是理解这些思想的时候。他们将帮助我们在接下来理解**基于代价的优化（cost based optimization）**这一概念。

####<a id="concept"></a>概念

**时间复杂度用于看出，一个算法需要花多长
时间来处理给定数量的数据**。 为了描述时间复杂度，计算机科学家使用了大O标记。这个标记以一个函数的形式描述对于给定数量的输入，一个算法需要进行多少次运算。

例如，当我们说“这个算法是O(some_function())”，这表示对于特定数量的数据，算法需要some_function(a_certain_amount_of_data)次操作来完成它的工作。

重要的并不是这个数据的数量，而是当数据量增加时，运算次数的增长方式。时间复杂度无法给出运算次数的精确值，但这是一个很好的想法。

![TimeComplexity](http://pic.yupoo.com/tan91319/ETsxLmem/medish.jpg)

在图中，你能看到不同类型复杂度的演化，我是以对数的比例来绘制的，换句话说，数据的数量从1到10亿快速增长，我们能看到：

* O(1)或者也叫常数复杂度会保持不变（否则也不叫常数复杂度）[译注1][annotation]。
* O(log(n))即使面对十亿级的数据，值也保持得很低
* 最糟糕的复杂度是O(n<sup>2</sup>)，运算的次数会爆炸式增长。
* 另外的两种复杂度增长得也很快

####例子<a id="example"></a>
数据量很小的时候，O(1)和O(n<sup>2</sup>)的差别微乎其微。举个例子，假如你的算法需要处理2000个元素。

* 一个O(1)的算法将花费1次操作
* 一个O(log(n))的算法将花费7次操作
* 一个O(n)的算法将花费2 000次操作
* 一个O(n * log(n))的算法将花费14 000次操作
* 一个O(n<sup>2</sup>)的算法将花费你4 000 000次操作

O(1)和O(n)的不同看起来很多（4百万），但是你最多只会损失2ms，这只是一眨眼的时间。实际上，现在的处理器每秒可以做上亿次的运算，这就是为什么性能和优化并不是许多IT项目需要考虑的问题。

我说过了，当面对庞大的数据量时，知道这些概念仍然很重要。如果这次算法需要处理1 000 000个元素（这对数据库来说还不算很大）。

* 一个O(1)的算法将花费1次操作
* 一个O(log(n))的算法将花费14次操作
* 一个O(n)的算法将花费1 000 000次操作
* 一个O(n * log(n))的算法将花费14 000 000次操作
* 一个O(n<sup>2</sup>)的算法将花费你1 000 000 000 000次操作

不用算也知道，n<sup>2</sup>算法所花费的时间够你喝杯咖啡了（其实够两杯）。如果你在数据量上再加个0，你就有时间去睡个好觉了。

####深入<a id="going_deeper"></a>
向你灌输一些知识点：

* 在一个理想的哈希表上，对元素的一次查找是O(1)
* 在一颗平衡树上，一次查找操作是O(log(n))
* 在一个数组上，查找操作是O(n)
* 最好的排序算法，复杂度是O(n * log(n))
* 糟糕的排序算法，复杂度是O(n<sup>2</sup>)

注意：下一部分，我们将看到这些算法和数据结构

有多种类型的时间复杂度

* 平均情况的时间复杂度
* 最好情况的时间复杂度
* 以及最坏情况的时间复杂度

时间复杂度通常指的是最坏情况的时间复杂度。

我只谈论时间复杂度，但复杂度也可以指：

* 算法的内存消耗
* 算法的磁盘IO消耗

当然，也有比O(n<sup>2</sup>)更糟糕的复杂度，如：

* n<sup>4</sup>: 简直可怕，我将会提到拥有这类复杂度的算法
* 3<sup>n</sup>: 天啦噜，在本文中间我们将看到这类复杂度的某个算法（这在许多数据库里确实用到了）
* n! （factorial n）：即使数据量很小，你都永远得不到结果
* n<sup>n</sup>：如果你正在和这种复杂度纠缠，你应该扪心自问，你真的是IT界的人吗...

注意：我没有给出大O记号的实际定义，而是一些思想，想要了解真实定义，你能阅读[Wikipedia][bigO]上的这篇文章。

[bigO]: https://en.wikipedia.org/wiki/Big_O_notation "Big O Notation"

###归并排序<a id="mergesort"></a>
当你想对一个集合排序时该怎么做呢？你能调用sort()函数...嗯！不错哦...但是对数据库而言，你需要理解这个sort()函数是怎么工作的。

好的排序算法有好几种，所以我将重点放在最重要的一种上：**归并排序（the merge sort）**。  
虽然你现在可能不理解为什么对数据排序很有用，但在查询优化一节后那你会理解的。而且，理解归并排序可以在之后帮助我们理解一种常见的数据库连接操作，**归并连接（merge join）**。

####归并<a id="merge"></a>
像许多有用的算法一样，归并排序基于一个技巧：将2个已排序过，且规模为 N/2的数组合成为一个拥有N个数据的已排序数组。该技巧只需要N个操作，这就被称为**归并（merge）**。

以一个简单的例子，让我们看看这到底是什么意思：

![merge](http://pic.yupoo.com/tan91319/ETsxKLJa/HnQ31.png)

你能在这张图上看出，为了构造出最终已排序的8元素（8-element）数组，只需要将两个4元素（4-element）数组迭代一次。因为两个4元素数组是已排序好的：

* 1）比较在两个数组中的当前元素（第一次比较的当前元素就是第一个元素）
* 2）取较小的那个当前元素，并把它放置到8元素的数组中
* 3）在取了元素的那个数组中，将当前元素定位到它的下一个元素上
* 重复步骤1,2,3直到取出了其中一个数组的最后一个元素
* 然后将另外一个数组中剩余的元素，统统放进8元素数组中

这个算法能够工作。因为所有的4元素数组是排序过的，因此对数组的访问不需要回退。

现在，我们理解了这个技巧，下面是我归并排序的伪代码。

    array mergeSort(array a)
	   if(length(a)==1)
	      return a[0];
	   end if

	   //递归调用
	   [left_array right_array] := split_into_2_equally_sized_arrays(a);
	   array new_left_array := mergeSort(left_array);
	   array new_right_array := mergeSort(right_array);

	   //将两个小的有序数组归并成一个大的
	   array result := merge(new_left_array,new_right_array);
	   return result;

归并排序将问题分解成较小的问题，找到小问题的解，然后再获得原问题的解。  
（注意：这类算法被称为分治法，divide and conquer）。  
如果你不理解这个算法，别担心，我第一次看到它的时候也一样。但是我将这个算法看成两个阶段，这样能帮你理解。

* 将数组分成两个小数组的 分解阶段
* 将小数组(使用归并)合并成一个大数组的 排序阶段

####分解阶段<a id="division_phase"></a>

![merge_sort_1](http://pic.yupoo.com/tan91319/ETsxJRDa/IUdR9.png)

在分解阶段，经过3步分解后，数组被分割成一个个单元格。严格来说，分解的步数是log(N)步（N = 8，log(N) = 3）。  

我为什么会知道呢？

<span style="text-decoration: line-through;">我是天才!</span>总之一句话： 数学。这个思路是，在每一步，数组的大小都会被除以2。所以步数就是初始的数组大小能被2除的次数。而这就是对数（以2为底）的准确定义。

####排序阶段<a id="sorting_phase"></a>

![merge_sort_2](http://pic.yupoo.com/tan91319/ETsxK3ss/wGckA.png)

在排序阶段，需要从单元格开始。每一步你可能会应用多次归并，而一步里总共需要N = 8次操作：

* 第一步里，总共需要4次归并，每次归并都需要2次操作
* 第二步里，总共需要2次归并，每次归并需要4次操作
* 第三步里，需要1次归并，这需要8次操作

因为有log(N)步，总代价是 N * log(N)次操作

####归并排序之强<a id="power_of_mergesort"></a>

这个算法为什么这么强大？    
因为：

* 可以修改它以减小内存占用(memory footprint)，方式是不必创建新数组而是直接修改原输入数组    
注意：这类算法也叫[原地算法](https://en.wikipedia.org/wiki/In-place_algorithm) (in-place)

* 可以修改它以便能同时使用磁盘空间和少量的内存，而不用大量的磁盘I/O损耗。这个思想是只将当前需要处理的部分载入内存。在你想对好几个G的表做排序，但内存的缓冲只有100M的时候非常重要。    
注意：这类算法也叫[外部排序](https://en.wikipedia.org/wiki/External_sorting) (external sorting)

* 可以修改它以便能在多进程/多线程/多机上运行  
例如，分布式归并排序是[Hadoop](https://hadoop.apache.org/docs/stable/api/org/apache/hadoop/mapreduce/Reducer.html) (THE framework in Big Data)的关键组件之一

* 这个算法能"化腐朽为神奇"[译注2][annotation]（实事求是哦！）

这个排序算法用在大多数（尽管不是全部）的数据库上，但是它不是唯一的一个。如果你想知道更多，你能读这篇[论文](http://wwwlgis.informatik.uni-kl.de/archiv/wwwdvs.informatik.uni-kl.de/courses/DBSREAL/SS2005/Vorlesungsunterlagen/Implementing_Sorting.pdf)，它讨论了在数据库中常用的排序算法的优劣。

###数组，树和哈希表<a id="array_tree_hashtable"></a>
既然我们理解了时间复杂度和排序这样的想法，我将谈论三个数据结构了。这是很重要的，因为它们是**现代数据库的支柱**。我也将介绍**数据库索引**这个术语。

####数组<a id="array"></a>
二维数组是最简单的数据结构，一个表能被看成一个数组。    
比如：

![array](http://pic.yupoo.com/tan91319/ETsxIw1C/E8nE5.png)

这个2维数组是一个有行和列的表：

* 每一行表示一个主体(subject)
* 列用于描述主体的特征
* 每一列存储的都是特定类型的数据（整型，字符串，日期）

尽管存储和可视化数据很方便，但当你想要查询特定值的时候就糟糕了。

举例而言，若你想找到所有曾经在UK工作过的人，你就不得不去考虑每一行，然后看看是否这一行属于UK。这将花费N次操作（N 表示行数），这样做尽管不赖，但是否可以有一个更快的方式？而这就是树（tree）将要登场的地方。

注意：大多数现代数据库提供了高级数组来有效地存储表，比如堆组织表（heap-organized tables）和索引组织表（index-organized tables）。但是在一组列上对特定条件的快速查找这一问题，并没有任何改变。

####树和数据库索引<a id="tree_index"></a>
二叉查找树（Binary Search Tree）是特别的二叉树，每个节点Key必须是：

* 大于所有存储在它左子树上的Keys
* 小于存储在它右子树上的Keys

我们形象地看一看这是什么意思

**思想**
![BST](http://pic.yupoo.com/tan91319/ETuugany/medish.jpg)

这棵树有N = 15个元素，现在假设我想找208

* 从Key为136的根开始，因为136 < 208，所以现在查看节点136的右子树
* 398 > 208，所以查看节点398的左子树
* 250 > 208，所以查看节点250的左子树
* 200 < 208，所以查看节点200的右子树，但是200没有右子树，所以值不存在（若节点存在，则一定是在200的右子树上）。

现在假设我们查找40

* 从Key为136的根开始，因为136 > 40，所以现在查看节点136的左子树
* 80 > 40，所以查看节点80的左子树
* 40 = 40，节点存在。我取出在这个节点中记录的行id（这在图中没画出来）然后根据给出的行id再查看表。
* 知道了行id，可以使我知道数据在表中的精确位置，因此我能立即获取它。

最后，查找的代价是树的高度，如果认真读了归并排序的话，你会发现树高应该是log(N)，所以查找的代价就是log(N)，还不错！

**回到我们的问题**

直接深入相关的问题或许会非常抽象，所以我们先回到刚才的问题。将整数替换掉，想象在这个表里的是表示某人国籍的字符串。假设，有一颗树包含表里的“country”这一列。

* 如果想知道谁在UK工作
* 搜索树以获取表示UK的节点
* 在这个“UK”节点里，你能找到这一行的位置，然后就找到了这个在UK工作的人。

这个操作只花费了log(N)次操作，而直接使用数组则是N次操作。这想象的就是**数据库索引**。

只要有能够比较两个key（也就是一组列）的函数，你都能为任意一组列（字符串，整数，两个字符串，一个整数和一个字符串，日期等等）建立一个树形索引，这样的话，就能够在一组key上建立一个序列了，这里的key就是一组数据库基本类型。

**B+树索引**



####哈希表<a id="hashtable"></a>


##整体概览<a id="global"></a>


##客户端管理<a id="client"></a>


##查询管理<a id="query"></a>


###查询解析器<a id="query_parser"></a>


###查询重写器<a id="query_rewriter"></a>


###统计<a id="statistics"></a>


###查询优化器<a id="query_optimizer"></a>


####索引<a id="indexs"></a>


####访问路径<a id="access_path"></a>


####连接运算符<a id="join_operators"></a>


####简化示例<a id="simplified_example"></a>


####动态规划，贪婪和启发性算法<a id="dp_greedy_heuristic"></a>


####真正的优化器<a id="real_optimizers"></a>


####缓存查询计划<a id="query_plan_cache"></a>


###查询执行器<a id="query_executor"></a>


##数据管理<a id="data"></a>


###缓存管理<a id="cache_manager"></a>


####预读取<a id="prefetching"></a>


####缓冲区置换策略<a id="buffer_replacement"></a>


####写缓冲<a id="write_buffer"></a>


###事务管理<a id="transaction_manager"></a>


####ACID原则<a id="acid"></a>


####并发控制<a id="concurrency"></a>


####锁管理<a id="lock"></a>


####日志管理<a id="log"></a>


##总结<a id="conclude"></a>

##译注<a id="annotation"></a>
[annotation]: #annotation
[1]: constant 作名词表示常量，形容词表示恒定不变
[2]: 原文为 turn lead into gold
