---
layout: post
title: 关系数据库是如何工作的
categories: interpret
---

译自：[How does a relational database work](http://coding-geek.com/how-databases-work/)

翻译：[colortail](www.colortail.com)

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
* [3 客户端管理器](#client)
* [4 查询管理器](#query)
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
* [5 数据管理器](#data)
	* [5.1 缓存管理器](#cache_manager)
		* [5.1.1 预读取](#prefetching)
		* [5.1.2 缓冲区置换策略](#buffer_replacement)
		* [5.1.3 写缓冲](#write_buffer)
	* [5.2 事务管理器](#transaction_manager)
		* [5.2.1 ACID原则](#acid)
		* [5.2.2 并发控制](#concurrency)
		* [5.2.3 锁管理器](#lock)
		* [5.2.4 日志管理器](#log)
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
* 在一棵平衡树上，一次查找操作是O(log(n))
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

<span style="text-decoration: line-through;">我是天才!</span>

总之一句话:数学。这个思路是，在每一步，数组的大小都会被除以2。所以步数就是初始的数组大小能被2除的次数。而这就是对数（以2为底）的准确定义。

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

直接深入相关的问题或许会非常抽象，所以我们先回到刚才的问题。将整数替换掉，想象在这个表里的是表示某人国籍的字符串。假设，有一棵树包含表里的“country”这一列。

* 如果想知道谁在UK工作
* 搜索树以获取表示UK的节点
* 在这个“UK”节点里，你能找到这一行的位置，然后就找到了这个在UK工作的人。

这个操作只花费了log(N)次操作，而直接使用数组则是N次操作。这想象的就是**数据库索引**。

只要有能够比较两个key（也就是一组列）的函数，你都能为任意一组列（字符串，整数，两个字符串，一个整数和一个字符串，日期等等）建立一个树形索引，这样的话，就能够在一组key上建立一个序列了，这里的key就是一组数据库基本类型。

**B+树索引**


尽管在取得特定值时，这棵树能工作的很好，但仍有一个大问题，那就是当你需要**在两个值之间取得多个元素**的时候。  
这会花费O(N)的复杂度，因为你得查看在树中的每一个节点，然后查看它们是否在这两个值之间（比如，对这棵树使用中序遍历）。  
然而这个操作并不是I/O友好的，因为你得读取整棵树。所以呢，需要找出一种能做**范围查询（range query）**的有效方式。为了解决这个问题，现代数据库使用了之前那棵树的一个改进版本，也就是**B+树**。一棵B+树：

* 只有最低层的节点（叶子节点）**存储信息**（信息是在对应表中行的位置）。
* 其他的节点是在搜索过程中用来找到（to route）正确节点的。

![database_index](http://pic.yupoo.com/tan91319/ETsxJdFH/cKc6r.png)
*I made a mistake in this B+Tree for the node 70

正如你所见，这棵树比刚才的那棵节点要多（大约两倍）。的确，这里有一些附加的节点，“决策节点”用于找到正确的节点（那些在相关的表里存储的行的位置）。但是搜索的复杂度仍然是O(log(N))的，因为这里只是在树上多加了一层而已。  
另外一个很大的不同是：**最底层节点都被链接到它的后继节点上了**

使用B+树来查找在40到100内的值：

* 你只需要查找40，或者当40不存在的时候，查找在大于40的最小值
* 然后使用直接的链接来收集40的后继，直到到达了100

我们令树的节点有N个，而希望查找到的后继有M个。对特定节点的查找，像之前一样需要花费log(N)，但是一旦你获得了这个节点，你就能仅以M次操作得到它的M个后继。这个查找的复杂度仅为 M + log(N)，而之前的树则需要N次操作。    
另外，这样的话，根本不需要往内存读入整棵树，这就意味着很少的磁盘使用量，如果M很小（比如200）而N很大（1 000 000）的话，这作用就很明显了。

但这又有一个新问题。如果添加或者删除了数据库中的一行（这会关联到B+树索引上）：

* 得保证B+树节点之间的顺序，否则就无法再这堆杂乱无章的节点里找到正确的那一个
* 得尽可能保证B+树层次的值是最小的，否则O(log(N))的时间复杂度将变成O(N)

说得专业点，B+树需要**自排序**和**自平衡**。    

谢天谢地的是，依靠一些技巧性的插入和删除操作，这是有可能的。但是这也伴随着一定的代价：B+树上的插入和删除操作都是O(log(N))复杂度的。这就是人们常说**“不要太多地使用数据库索引”**的原因。事实上，这使得原本很快的数据库插入、更新和删除操作变慢了，因为对表里的每一个索引，更新它的代价都是O(log(N))。不仅如此，对事物管理器（本文的最后将介绍它）来讲，添加一个索引意味着更多的工作量。

查找更多的细节，你可以查阅有关[B+树](https://en.wikipedia.org/wiki/B%2B_tree)的维基百科。如果你想获取一个在数据库里B+树具体实现的例子，可以看看由MySQL核心开发人员写的[这篇](http://blog.jcole.us/2013/01/07/the-physical-structure-of-innodb-index-pages/)和[这篇](http://blog.jcole.us/2013/01/10/btree-index-structures-in-innodb/)。它们都只关注innoDB（它是MySQL引擎）是如何处理索引的。

####哈希表<a id="hashtable"></a>
最后一个重要的数据结构是哈希表。当你希望快速查找值的时候，它就很有用了。而且，理解哈希表能帮助我们在之后理解一个常见的数据库连接操作，也就是**哈希连接（hash join）**。

在数据库里，这个数据结构也被用于存储一些内部的组件，比如**锁表（lock table）**和**缓冲池（buffer pool）**，之后将会涉及这些概念。

哈希表是能根据key快速找到元素的数据结构。要建立一个哈希表，需要定义：

* 为元素定义一个**key**（键）
* 为所有的key定义统一的**哈希函数（hash function）**，由key计算出来的哈希值（也称哈希码）给出了元素的所在位置（这个位置称为**哈希桶 buckets**）
* 另外一个函数用于比较（哈希桶里的）key值。当找到了正确的哈希桶后，就需要在桶内使用这个比较函数来找出这个元素。

**一个简单的例子**
看看这个可视化的例子

![hash_table](http://pic.yupoo.com/tan91319/ETsxJPES/TTt4s.png)

这个哈希表有十个桶，但我很任性得只画出了五个，不过你一定懂得。我用的哈希函数是对key做模10运算。换句话说，我只保留了元素key值的最后一个数字。

* 如果最后一个数字是0，元素会放到0号哈希桶内
* 如果最后一个数字是1，元素会放到1号哈希桶内
* 如果最后一个数字是2，元素会放到2号哈希桶内
* ...

我使用的比较函数就是简单地对两个整数判等。

假设我们想获取元素78。

* 哈希表计算78的哈希码是8
* 查找桶8，然后找到的第一个元素就是78
* 再然后，就会返回78
* 这个查找只花费了两次操作（一次是计算哈希值，另一次是在桶里查找元素）

好了，假设我们想获取元素59

* 哈希表计算59的哈希码是9
* 查找桶9，然后找到的第一个元素是99，因为99 != 59，元素99还不是那个正确的元素
* 按照相同的思路，将查看第二个元素（9），第三个（79）...直到最后一个（29）
* 这个元素不存在
* 这个查找花费了7次操作

**一个好的哈希函数**
其实你可以想到，根据你所查找的值不同，查找花费的代价也是不一样的！

如果我现在把哈希函数改成对key模1 000 000（也就是取倒数六个数），第二次查找只会花费一次操作，因为不会有元素在000059号桶里。一个好的哈希函数会使得每个桶里包含元素的个数都很小，而真正的挑战实际是找到这个好的哈希函数。

在我的例子里，找到一个好的哈希函数很简单，但这毕竟只是一个简单的例子，当key是下面的情况时，找到一个好的哈希函数就比较复杂了：

* 一个字符串（比如一个人的姓）
* 两个字符串 (比如一个人的姓和他的名字)
* 两个字符串和一个日期时间（比如一个人的姓名，和他的生日）
* ...

**在拥有一个好的哈希函数的情况下，在哈希表中查找元素的复杂度是O(1)的**


**数组和哈希表**

为什么不使用数组

额，这个问题问得好

* 一个哈希表能够仅**部分载入内存（half load in memory）**，而其他的哈希桶则只在硬盘里
* 使用数组你必须得使用连续的内存空间，当你载入一个很大的哈希表时，很难有足够的连续内存。
* 使用哈希表，key的类型随意选择（比如一个人的国家以及他的姓）

想要了解更多信息，可以阅读我关于[Java HashMap](http://coding-geek.com/how-does-a-hashmap-work-in-java/)，这一个非常有效的哈希表实现的相关文章。

##整体概览<a id="global"></a>
我们现在看看数据库系统里的基本构件。现在得在高层鸟瞰一下这整体的框架。

数据库是信息的集合，这些信息是可以轻松访问和修改的。但是一组普通的文件也可以做到这样，事实上，最简单的数据库（比如SQLite）就比一堆文件多不了多少东西。但是SQLite是带着些技巧的文件，因为它允许：

* 使用事务来确保数据的安全性（safe）和一致性（coherent）
* 快速的数据处理，即使数据量是在百万级的

一般而言，一个数据库能够被看成下面的图：

![global_overview](http://pic.yupoo.com/tan91319/ETsxITBu/4cgc0.png)

在写这一部分之前，我读了很多书和论文，他们每一个都用自己的方式来描绘数据库。所以你也不要过多地纠结于，我怎样组织和命名这些数据库组件以及它里面的过程，因为我只是从那些资料里选了一些使得这篇文章符合我的计划。而且，重要的是这些不同的组件。整体的思路是：**数据库被分成多个组件，而且他们之间可以互相交互**。

核心的组件有：

* 进程管理器：许多数据库都需要管理**进程/线程池**，而且为了获得纳秒级的能力，一些现代数据库会用它自己的线程取代操作系统的线程。
* 网络管理器：网络I/O是个大问题，特别是对于分布式数据库，这就是为什么一些数据库有它自己的管理器
* 文件系统管理器：**磁盘I/O是数据库的首个瓶颈**。有个能完美解决甚至替代操作系统的文件系统是非常重要的。
* 内存管理器：为了避免磁盘I/O的损耗，就需要大量的RAM，但是如果要操纵大量的内存，那么就需要有效的内存管理器了。特别是在有多个内存查询同时进行的时候。
* 安全管理器：用于管理用户的认证（authentication）和授权（authorization）
* 客户端管理器：管理客户端连接
* ...

工具：

* 备份管理器: 用于保存和恢复数据库
* 复原管理器: 用于在数据库崩溃后以一致性状态重启
* 监控管理: 用于记录数据库活动，并提供监控数据库的工具
* 管理管理器: 存储元数据(比如表的名字和结构)，并提供管理数据库，数据库模式和表空间等的工具
* …

查询管理器:

* 查询解析器: 检测查询语句是否有效
* 查询重写器: 对查询做预优化
* 查询优化器: 优化查询
* 查询执行器: 编译并执行查询

数据管理器:

* 事务管理器: 处理事务
* 缓存管理器: 在使用数据前，以及准备将数据写入磁盘前，现将其放入内存中
* 数据访问管理器: 访问在磁盘上的数据
 
在本文的剩余部分, 我将探讨数据库是如何经由以下过程来把控一个SQL查询的

* 客户端管理器
* 查询管理器
* 数据管理器 (在这个部分我也会涵盖到恢复管理器)

##客户端管理器<a id="client"></a>

![client_manager](http://pic.yupoo.com/tan91319/ETsxIvuf/1AUnB.png)

客户端管理器用来处理和客户端的通信。客户端可以是一个Web服务器或者一个终端的应用。客户管理器通过一组众所周知的API，比如JDBC，ODBC，OLE-DB...，提供了访问数据库的不同方式。

客户端管理器也提供专用的数据库访问API。

当你连接到数据库的时候：

* 管理器首先会对你做认证（检查登录名和密码）然后检查你是否被授权使用数据库。访问权限则是由DBA设置的。
* 然后，它检查是否有一个可用的进程（或者线程）可以对你的查询做处理
* 它也会检查数据库是否不在一个高负载（heavy load）状态下
* 在获得需要的资源前，它会稍微等待，如果等待超时，它就会关闭连接，并返回一个可读的错误信息
* 否则，它会将你的查询请求发送给查询管理器，这样你的查询就会被处理
* 查询处理不会“闷不吭声”[注3][annotation]，一旦从查询处理器返回了数据，客户端管理器就会将它部分的结果存入缓冲并将它们发送给你
* 如果发生了问题，它会停止连接，给出一个**可读的解释（readable explanation）**，并释放资源

##查询管理器<a id="query"></a>

![query_manager](http://pic.yupoo.com/tan91319/ETsxKJxt/5dDYw.png)

这部分就是数据库厉害的地方了。在这里，即使一个查询语句写的很烂，也会被转换成一个速度很快的可执行代码。代码执行后结果会返回给客户端管理器。这是一个多步的操作。

* 查询会首先被解析，看是否有效
* 查询会被重写，并移除没用的操作，然后增加一些预先优化
* 查询会被优化从而提高性能，然后转换成可执行的数据访问计划（data access plan）
* 然后数据访问计划会被编译
* 最终执行

在这个小节，我不会谈论最后的两点，因为他们不太重要。

在读完这个小结后，如果你想更深入地了解，我建议你阅读：

* 那篇关于“基于代价的优化”这一问题的最初始研究论文：[Access Path Selection in a Relational Database Management System](http://www.cs.berkeley.edu/~brewer/cs262/3-selinger79.pdf)。这篇文章只有12页，只要有一般的计算机基础就能很好地理解。
* DB2 9.x是如何优化查询的一个很棒很有深度的说明[点这里](http://infolab.stanford.edu/~hyunjung/cs346/db2-talk.pdf)
* PostgreSQL 9.x是如何优化查询的一个很棒的描述[点这里](http://momjian.us/main/writings/pgsql/optimizer.pdf)。这是最通俗易懂的文档了，因为他更多地在介绍“在这些情况下，PostgreSQL的查询计划是怎样的”而不是“PostgreSQL使用的算法是什么”。
* 有关优化的官方[SQLite文档](https://www.sqlite.org/optoverview.html)。这个也很容易读，因为SQLite使用的规则都挺简单。而且也只有官方文档能真正说明他们是怎么工作的。
* 一个很好的对SQL Server 2005优化查询的说明[点这里](https://blogs.msdn.com/cfs-filesystemfile.ashx/__key/communityserver-components-postattachments/00-08-50-84-93/QPTalk.pdf)
* Oracle 12c和优化相关的白皮书[点这里](http://www.oracle.com/technetwork/database/bi-datawarehousing/twp-optimizer-with-oracledb-12c-1963236.pdf)
* 两个和优化相关的理论性课程，老师是“数据库系统概念（DATABASE SYSTEM CONCEPTS）”的作者。[这里](http://codex.cs.yale.edu/avi/db-book/db6/slide-dir/PPT-dir/ch12.ppt)[以及这里](http://codex.cs.yale.edu/avi/db-book/db6/slide-dir/PPT-dir/ch13.ppt)。他们是探讨磁盘I/O代价的好读物，但是需要比较好的CS基础。
* 另外一个比较好懂的[理论性课程](https://www.informatik.hu-berlin.de/de/forschung/gebiete/wbi/teaching/archive/sose05/dbs2/slides/09_joins.pdf)将重点放在联合操作和磁盘I/O上了。

###查询解析器<a id="query_parser"></a>
每个被送给解析器的SQL语句，都会被检查语法的正确性。如果查询里有错误，解析器会拒绝查询。比如说若把“SELECT”写成了“SLECT”，那事情到这就结束了。

但是不止如此，解析器也检查关键字是否以正确的顺序使用。比如一个“WHERE”关键字若跑到“SELECT”之前就会被拒绝。

然后在查询里和表以及表域（tables and fields）相关的问题会被分析，解析器将使用在数据库中的元数据检查：

* 表是否存在
* 表里的表域是否存在
* 对这些域的类型而言，操作是不是可行的（比如不能将一个整型和字符串做比较，也不能对整型使用substring()函数）

依据查询，解析器还会检查你是否有对该表读写的权限。再提一次，对某个表的访问权限是由DBA设置的。

在解析过程中，SQL查询会被转换成一个内部的数据表示（通常是一棵抽象语法树）

如果一切都顺利进行了，这个内部的表示将会被送到查询重写器里。

###查询重写器<a id="query_rewriter"></a>
在这一步，我们已经拥有一个查询的内部表示。而这里的目的是重写它以：

* 对查询做预优化
* 避免不必要的操作
* 尽可能帮助优化器找到最好的解决方案

重写器对查询执行了一系列已知的规则，如果查询满足某个规则的模式（pattern），将会应用这个规则来改写查询。下面给出一列不太详尽的（可选）优化规则。

* 视图归并（view merging）：如果

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


##数据管理器<a id="data"></a>


###缓存管理器<a id="cache_manager"></a>


####预读取<a id="prefetching"></a>


####缓冲区置换策略<a id="buffer_replacement"></a>


####写缓冲<a id="write_buffer"></a>


###事务管理器<a id="transaction_manager"></a>


####ACID原则<a id="acid"></a>


####并发控制<a id="concurrency"></a>


####锁管理器<a id="lock"></a>


####日志管理器<a id="log"></a>


##总结<a id="conclude"></a>

##译注<a id="annotation"></a>
[1]： constant 作名词表示常量，形容词表示恒定不变

[2]： 原文 turn lead into gold

[3]： 原文 all or nothing

[annotation]: #annotation "译注"
