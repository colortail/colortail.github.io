---
layout: post
title: 关系数据库是如何工作的
categories: interpret
---

译自：[How does a relational database work](http://coding-geek.com/how-databases-work/)

翻译：[哭也找不到工作的colortail](www.colortail.com)

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
		* [4.4.7 查询计划缓存](#query_plan_cache)
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

注：下一部分，我们将看到这些算法和数据结构

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

注：我没有给出大O记号的实际定义，而是一些思想，想要了解真实定义，你能阅读[Wikipedia][bigO]上的这篇文章。

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
（注：这类算法被称为分治法，divide and conquer）。  
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
注：这类算法也叫[原地算法](https://en.wikipedia.org/wiki/In-place_algorithm) (in-place)

* 可以修改它以便能同时使用磁盘空间和少量的内存，而不用大量的磁盘I/O损耗。这个思想是只将当前需要处理的部分载入内存。在你想对好几个G的表做排序，但内存的缓冲只有100M的时候非常重要。    
注：这类算法也叫[外部排序](https://en.wikipedia.org/wiki/External_sorting) (external sorting)

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

注：大多数现代数据库提供了高级数组来有效地存储表，比如堆组织表（heap-organized tables）和索引组织表（index-organized tables）。但是在一组列上对特定条件的快速查找这一问题，并没有任何改变。

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

* 视图合并（view merging）：如果在查询里使用视图，视图变量会被转换成定义视图的SQL代码
* 子查询矫正（subquery flattening）：有子查询时是很难优化的，所以重写器会尝试修改包含子查询的语句，去掉子查询    
比如：  

	SELECT PERSON.*
	FROM PERSON
	WHERE PERSON.person_key IN
	(SELECT MAILS.person_key
	FROM MAILS
	WHERE MAILS.mail LIKE 'christophe%');

将被替换成：    

	SELECT PERSON.*
	FROM PERSON, MAILS
	WHERE PERSON.person_key = MAILS.person_key
	and MAILS.mail LIKE 'christophe%';

* 去掉不必要的操作符（Removal of unnecessary operators）：例如，在使用DISTINCT关键字时，若数据字段已经通过UNIQUE做了唯一化约束，那么DISTINCT关键字就会被移除
* 消除冗余的连接（Redundant join elimination）：如果有两个相同的连接条件，比如有一个连接条件隐藏在视图里或者连接条件经过传递后等价，那就有一个连接是没用的，需要移除掉。
* 算数常量计算（Constant arithmetic evaluation）：如果写了需要计算的东西，那么就会只在重写的时候计算一次，比如 WHERE AGE > 10 + 2将被转换成 WHERE > 12，或者TODATE("时间字符串")会被转换成datetime格式的时间
* （以下为高级特性）分区剪枝（Partition Pruning）：如果使用了分区表，重写器能找到用的是哪一个分区
* 物化视图重写（Materialized view rewrite）：如果查询谓词（即where，having等用于限定查询条件的语句）里的某一部分可以匹配到一个物化视图上，那么重写器将会检查视图是不是更新了，并修改查询以使用物化视图而不是原始的表。
* 自定义规则（Custom rules）：如果有自定义规则可以修改查询（比如Oracle policies），重写器也会执行这些规则
* 连接分析处理转换（OLAP transformations）：analytical/windowing functions, star joins, rollup...也会被转换（但是我不是很确定这些工作是在重写器还是在优化器里完成的，因为这些过程的关系都很紧密，实际情况取决于具体的数据库实现）

被重写的查询之后会被传递给查询优化器，接下来就是见证奇迹的时刻！

###统计<a id="statistics"></a>
在我们看数据库如何优化一个查询之前，我们先来谈谈**统计量**，因为没有统计量数据库就会变得很笨。如果你不告知数据库要分析它自己的数据，它就不会分析，这样它就会做出非常错误的计划。

但是，数据库需要的是哪种信息呢？

我简单说说数据库和操作系统是如何存储数据的吧，他们使用的最小单元是页（Page）或者块（Block），默认是4K或8K。这表示，假如你需要的只是1KB的数据，这也得需要一个页，如果一个页是8KB的话，这时就浪费了7KB的空间。

回到统计量，当你要求数据库收集统计量的时候，它会计算下面类似的值：

* 一个表的行数或页数
* 对表中的每一列：
	* 不同的数据值
	* 数据的存储长度（最小，最大，平均值）
	* 数值的范围信息（最小，最大，平均值）
* 表索引信息

**这些统计量将有助于优化器评估一条查询的磁盘I/O，CPU和内存使用。**

对每一列的统计都是非常重要的，举例来说，如果一个PERSON表需要在两列上做连接：LAST_NAME和FIRST_NAME。有了统计量，数据库就知道，哦，在FIRST_NAME上只有1 000个不同的值，而在LAST_NAME上有1 000 000个不同的值。因此数据库会将数据以 LAST_NAME，FITST_NAME的方式做连接，而不是FITST_NAME，LAST_NAME。因为LAST_NAME不太可能相同，大部分对LAST_NAME的前2个或3个字符的比较就已经很足够，所以前者比较的次数会远小于后者，

但是这只是基本统计量，你也可以要求数据库计算称为**直方图（histograms）**的高级统计量。直方图是能够表示表列中数据值分布的统计量。比如：    

* 最频繁出现的值
* 分位数
* ...

这些特别的统计量将帮助数据库找到更好的查询计划，特别对等值谓词（equality predicate），比如：WHERE AGE = 18，或者范围谓词（range predicates），比如WHERE AGE > 10 AND AGE < 40。通过考虑这些谓词，数据库对要选择的行的数目会有更好的认识。（注：这个概念的技术名词是选择度 selectivity）

统计数据将被存储在数据库的元数据里。例如你能在（非分区）表里查看统计量：

* Oracle的USER/ALL/DBA_TABLES和USER/ALL/DBA_TAB_COLUMNS
* DB2的SYSCAT.TABLES和SYSCAT.COLUMNS

**统计量需要及时更新**。没有什么比以为表只有500行结果却有1 000 000行更糟糕的了。统计唯一的坏处是，需要时间来计算它们。这就是为什么在大多数的数据库里它们不自动计算。当有百万级的数据需要计算还是挺难的。在这种情况下，可以选择只计算基本统计量，或者在数据库的一个样本下计算统计量。

举例来说，有一次我的项目需要处理的表里基本都是亿级数据，我只选择了其中10%的数据做统计，这在时间上能获得巨大收益。这个故事最终还是一个坏决定，因为非常偶然地，Oracle 10g在取那张表的列里10%的数据时，和取100%的分布非常不一样（这是一个一亿行的表里不太可能出现）。这个糟糕的统计导致了一个通常只要三十秒的查询，有次花了八小时。找原因的过程简直是恶梦。这个例子说明统计是多么重要额！

注：当然，对于特定的数据库还有更多的高级统计量。如果你想知道更多，读读各数据库的文档吧。话虽这么说，我也尝试过去理解那些使用到的统计量，然而我找到的最好的官方文档还是来自[PostgreSQL的](http://www.postgresql.org/docs/9.4/static/row-estimation-examples.html)。

###查询优化器<a id="query_optimizer"></a>
所有的现代数据库都在使用基于代价的优化CBO（Cost Based Optimization）来优化查询。这里面的思想是对每个操作设置一个代价，那么减少查询开销的最好方式，就是使用最简便的一套操作去获得结果。

为了理解一个代价优化器是如何工作的，我想，举个例子，直接去“感受”任务的复杂度或许最好。在这一小节，我将向你展示对两个表做连接的3种常见的方式，我们能很快看到，即使是最简单的连接操作，对优化来讲也是个噩梦。在那之后，我们能看到真实的优化器是如何做这项工作的。

对连接操作来说，我将把重点放在时间复杂度上，但是数据库优化器实际还会计算CPU代价，磁盘I/O代价以及内存开销。时间复杂度和CPU代价是大致相同的时间开销（特别对于我这种不拘小节的人来讲）。对于CPU代价，我将计算像“加法”，“if语句”，“乘法”，“迭代”一样的运算操作。甚至：
  
* 每条高层的代码，都对应特定数量的底层CPU操作
* CPU的代价对不同的机器比如Core i7，Intel Pentium 4或者AMD Opteron也不相同（就CPU周期来讲），换句话说，它依赖于CPU架构

至少对我来讲，使用时间复杂度会比较轻松，而且这样我们能继续贴近CBO的概念。我也会偶尔说起磁盘I/O，因为这也是一个重要概念。需要记住的一件事是**最大的瓶颈是磁盘I/O花的时间，而不是CPU花的**。。

####索引<a id="indexs"></a>
我们之前在B+树那谈到了索引，需要记住的是**索引是已经排序了的**。

顺便提及，仅供参考，还有其他类型的索引比如**位图索引**。他们不需要像B+树那样的CPU，磁盘I/O和内存开销。

甚至，在能改善执行计划的代价时，许多现代数据库还能为当前查询动态创建临时索引。


####访问路径<a id="access_path"></a>
在使用连接运算前，第一步是获取数据，下面是获取数据的方法。

注：因为对所有的访问路径而言，真正的问题是磁盘I/O，我就不谈论太多的时间复杂度了。

**全扫描**

如果你曾经读过一个查询的执行计划，你一定见过这个词**full scan（或者是scan）**。简单来讲，全扫描就是数据库完整地读入一张表，或者一个索引。就磁盘I/O来讲，全扫描一个表显然比全扫描一个索引要花费得多。

**索引范围扫描**

其他类型的扫描有**索引范围扫描（index-range-scan）**，举个例子，当你使用类似于“WHERE AGE > 20 AND AGE <40”这样的谓词时，就会使用到索引范围扫描。

不用说，你必须得在字段AGE上加索引才能使用索引范围扫描。

我们在第一部分看过，范围查询的时间开销类似于 log(N) + M的，N是在索引中的数据量，而M则是在该范围中的数据的估计量。N和M的值由于统计的关系都是已知的。    
（注，M就是谓词AGE >20 AND AGE<40的选择度）

而且，范围扫描都不需要完整地读一个索引，所以就磁盘I/O来讲，他的花费也比全扫描要小。

**索引唯一扫描**

如果你只需要从索引里查一个值，你就可以使用索引唯一扫描

**通过行id的访问**

如果数据库使用了索引，在大多数时候，他需要查找和索引相关联的行。为了这项任务，就需要使用通过行id的访问。

举例而言，如果你想做这样的查询：

	SELECT LAST_NAME, FIRST_NAME from PERSON WHERE AGE = 28

如果在PERSON表的AGE列上有索引，优化器将使用索引去找所有28岁的人，然后它会请求在表中关联的行，因为在索引里只有有关AGE的信息，而你想知道的还有LAST_NAME和FIRST_NAME。

但是，假如你希望的是这样：

	SELECT TYPE_PERSON.CATEGORY from PERSON ,TYPE_PERSON
	WHERE PERSON.AGE = TYPE_PERSON.AGE

在PERSON上的索引将和TYPE_PERSON做连接，但是表PERSON将不会通过行id被访问，因为这次并没有向表请求信息。

尽管对于少量的访问是有效的，但这些操作真正的问题是磁盘I/O，如果有大量需要通过行id的访问，数据库可能还是会选择全扫描的。

**其他方式**
我还没列出所有的访问方式。如果你想知道更多，你能读[Oracle文档](https://docs.oracle.com/database/121/TGSQL/tgsql_optop.htm)。这些方式的名字可能和其他数据库里的不一样，但底层的概念是一致的。

####连接运算符<a id="join_operators"></a>
好，我们已经知道如何获取数据了，那么开始对他们做连接吧！

我将描述三种常见的连接运算，归并连接（merge join），哈希连接（hash join）以及嵌套循环连接（nested loop join）。但在开始前，我还得介绍些新词：内关系（inner relation）和外关系（outer relation）。关系（relation）可以指：

* 表
* 索引
* 从前一个操作里产生的中间结果（比如之前连接的结果）

当你连接两个关系时，连接算法会处理这左右两个不同的关系。在本文的剩余部分，我将假设：

* 外关系是左边的数据集
* 内关系是右边的数据集

例如：A JOIN B是作用在A和B上的连接，A是外关系，而B就是内关系。

大多数情况下，A JOIN B的代价和B join A的代价可不相同哦。

在这个部分，我将假设，外关系有N个元素，内关系有M个元素。要记住的是，实际的优化器是会依据统计量，从而知道N和M的具体值的。

注：N和M是关系的基数。

**嵌套循环连接**
嵌套循环连接是最简单的一种。

![nested_loop_join](http://pic.yupoo.com/tan91319/ETsxKj9e/d7d5H.png)

它的思想是：

* 循环外关系中的每一行
* 查看内关系中的所有行，是否有行能匹配

下面是伪代码：

	nested_loop_join(array outer, array inner)
	  for each row a in outer
	    for each row b in inner
	      if (match_join_condition(a,b))
	        write_result_in_output(a,b)
	      end if
	    end for
	   end for

因为这是个二重循环，所以时间复杂度是O(N * M)

就磁盘I/O而言，对外关系中的N行，内部循环都需要从内关系里读取M行，算法需要读取磁盘 N + N * M次。但是假如内关系足够小，你就能将关系放入内存中，也就只需要M + N次读取了。根据这个改进思路，内关系应该是最小的那个，因为这样就有更多的机会将它放进内存里了。

就CPU而言，代价则没有区别，但是就磁盘I/O来说，更好的方式是一次把所有的关系都读进内存来。

当然，内关系可以被索引替代，这样对磁盘I/O就更有利了。

既然算法很简单，那如果内关系太大，而内存根本不能满足，这里也有另外一个对磁盘I/O更友好的例子。思路是这样：

* 不再一行一行（row by row）地读关系
* 而是一批一批（bunch by bunch）地读，同时在内存里把每个关系都放一批数据
* 比较两批数据里的行，看行是否匹配
* 再从磁盘上载入新的一批数据，并做比较
* 循环操作，直到没有数据可以载入

下面是可行的算法：

	// 减少磁盘I/O的改进版本.
	nested_loop_join_v2(file outer, file inner)
	  for each bunch ba in outer
	  // ba 现在在内存中
	    for each bunch bb in inner
		    // bb 现在在内存中
			for each row a in ba
	          for each row b in bb
	            if (match_join_condition(a,b))
	              write_result_in_output(a,b)
	            end if
	          end for
	       end for
	    end for
	   end for

无论哪个版本，时间复杂度都是一样的，但是访问磁盘的次数是在减少：

* 之前的版本需要 N + N * M次访问（每次访问都获取一行）
* 新的版本，磁盘访问的次数变成 number_of_bunches_for(outer)+ number_of_ bunches_for(outer)* number_of_ bunches_for(inner)
* 随着一批数据行的增加，磁盘访问的数量就会减小

注：和之前的算法不一样，每次磁盘访问获取更多的数据并没有关系，因为磁盘是顺序访问的，实际的问题是机械硬盘得花时间找到第一笔数据。

**哈希连接**

哈希连接更复杂，但在许多情况下，他比嵌套循环连接的代价小。

![hash_join](http://pic.yupoo.com/tan91319/ETsxJcpo/13GzjV.png)

哈希连接的思想是：

* 获取内关系的所有元素
* 构造一个驻于内存的哈希表
* 从外连接中一个一个地（one-by-one）获取所有元素
* 计算每个元素的哈希值（使用哈希表的哈希函数）以找到内关系中相关的哈希桶
* 查找在哈希桶内的元素和外关系内的元素是否匹配

就时间复杂度来说，我需要一些假设来简化问题：

* 内关系被分成X个桶
* 哈希函数将哈希值均匀地分布到所有的关系上，换句话说，桶的大小是均等的
* 外关系中的某元素和在哈希桶内的所有哦元素的匹配过程，代价是桶内元素的个数

时间复杂度是 (M / N) * (N / X) + cost_to_create_hash_table(M) + cost_of_hash_function * N

若哈希函数创建的哈希桶足够小，时间复杂度就是O(M + N)

另一个版本的哈希连接是更内存友好，但是磁盘I/O则没有上一个代价小。

* 计算出内外两个关系的哈希表
* 将他们放置在磁盘上
* 依次比较两个关系桶（bucket by bucket），将一个载入内存，另一个则逐行读取。

**归并连接**

**归并连接是唯一个会排序的连接**

注：在这个简化的归并连接里，没有内关系和外关系，他们的角色是一样的，但真实的实现是有很大不同的，比如在处理冗余上。

归并连接能被分成两步：

* （可选的）排序连接操作：所有的输入，以连接的key排序
* 归并连接操作：排序后的的输入被归并在一起

**排序**

我们已经说过归并排序，在这个情况下归并排序是一个好的算法（但当内存不是问题的时候，它不是还最好的）

但是有时候，数据集本身已经有序了：

* 如果表本身就是有序的，比如对连接条件而言，表是索引组织表
* 如果关系是一个加在连接条件上的索引
* 如果连接是作用在查询的中间结果上，而这个中间结果已经在处理过程中排过序

归并连接

![merge_join](http://pic.yupoo.com/tan91319/ETsxJRZH/Y6ryC.png)

这部分和之前见过的归并排序里的归并操作很相似。但是这次，我们只选出两个关系里的相等元素，而不是每一个元素。

* 比较在两个关系中的当前元素（第一次比较时，当前元素就是第一个元素）
* 如果它们相等，将他们的元素都放入结果中，当前元素均指向下一个元素
* 如果不相等，将较小的那个当前元素指向它的下一个元素（因为下一个元素可能匹配）
* 重复1，2，3，直到某一个关系的最后一个元素已经访问完

这个算法也是能工作的，因为有序的关系也不需要回退

这个算法是一个简化的版本，因为它不能处理在数组里有相同数据的情况（即多重匹配）。真实的版本对图上的例子而言更加复杂，所以我这里选了一个简单的版本。

若所有的关系都已排序，那么时间复杂度应该是O(N + M)

若所有的关系都需要被排序，那么时间复杂度是排序所有关系的代价：    
**O(N * log(N) + M * log(M))**

对CS狂人来说，这里有个可能的算法能处理多重匹配的情况（注，我不能100%保证我算法的正确性）

	mergeJoin(relation a, relation b)
	  relation output
	  integer a_key:=0;
	  integer  b_key:=0;

	  while (a[a_key]!=null and b[b_key]!=null)
	     if (a[a_key] < b[b_key])
	      a_key++;
	     else if (a[a_key] > b[b_key])
	      b_key++;
	     else //Join predicate satisfied
	      write_result_in_output(a[a_key],b[b_key])
	      //We need to be careful when we increase the pointers
	      if (a[a_key+1] != b[b_key])
	        b_key++;
	      end if
	      if (b[b_key+1] != a[a_key])
	        a_key++;
	      end if
	      if (b[b_key+1] == a[a_key] && b[b_key] == a[a_key+1])
	        b_key++;
	        a_key++;
	      end if
	    end if
	  end while


**哪种才是最好？**

如果有最好的连接算法，那么就不用提这么多种类型出来了，这个问题之所以难回答是因为需要考虑的因素有很多：

* 可用内存的数量：没有足够的内存，就可以直接和强大的哈希连接说再见了（至少完全驻于内存的哈希连接是不行的）
* 两个数据集的大小：举例来说，如果有一个大表和一个小表，嵌套循环连接可能比哈希连接快，因为哈希连接还有一个创建哈希表的动作。如果两个表都很大，那嵌套循环连接将很费CPU。
* 索引的存在：如果有两个B+树索引，最机智的方法应该就是归并连接了
* 结果集是否需要被排序：即使是处理无序的数据集，有时也希望使用很费时间的归并连接（花时间是因为有一个排序的过程），这是因为，若连接的最终结果被排序，就能链式调用另外一个归并连接了，或者查询本身就通过ORDER BY/GROUP BY/DISTINCT关键字显式或隐式地指出需要结果被排序。
* 是否结果已经有序：在这种情况下，归并连接是最好的选择
* 取决于你要做的连接类型：是否是一个等连接**equijoin**（tableA.col1 = tableB.col2)）？是否是一个内连接**inner join**，一个外连接**outer join**，一个笛卡尔积**cartesian product**还是一个自连接**self-join**。一些连接在某些情况下是不可行的。
* 数据的分布情况：如果在连接条件下的数据是有偏**skewed**的（比如，你想对人的名字做连接，但是很多人同名），假如使用哈希连接就完蛋了，因为哈希函数会常见病态分布（ill-distributed）的桶。
* 如果希望连接操作是在多线程/多进程环境下

想获取更多信息，你可以阅读[DB2](https://www-01.ibm.com/support/knowledgecenter/SSEPGG_9.7.0/com.ibm.db2.luw.admin.perf.doc/doc/c0005311.html)，[ORACLE](http://docs.oracle.com/cd/B28359_01/server.111/b28274/optimops.htm#i76330)或者[SQL Server](https://technet.microsoft.com/en-us/library/ms191426(v=sql.105).aspx)的文档。

####简化示例<a id="simplified_example"></a>
我们看过了三种连接操作后，现在假设我们需要连接5张表，来了解一个人的全部信息。一个PERSON有：

* 多个手机MOBILES
* 多个邮箱MAILS
* 多个地址ADDRESSES
* 多个银行账号BANK_ACCOUNTS

换个角度看，我们需要快速响应下面查询：

	SELECT * from PERSON, MOBILES, MAILS,ADRESSES, BANK_ACCOUNTS
	WHERE
	PERSON.PERSON_ID = MOBILES.PERSON_ID
	AND PERSON.PERSON_ID = MAILS.PERSON_ID
	AND PERSON.PERSON_ID = ADRESSES.PERSON_ID
	AND PERSON.PERSON_ID = BANK_ACCOUNTS.PERSON_ID

查询优化器需要找到处理数据的最好方式，但是这有两个问题：

* 对每个连接，我要使用哪一种连接操作呢？

有三种可能的连接方式（哈希连接，归并连接，嵌套连接），每种连接都可能处理没有，有一个或两个索引的情况（先不考虑索引的类型）。

* 计算连接应该选择怎么样的顺序呢？

举例来说，下面的图显示出可能在四张表上做三次连接操作的不同计划

![join_ordering_problem](http://pic.yupoo.com/tan91319/ETsxJ5XW/chHQu.png)

下面是我给出的方案：

* 1）使用暴力破解法

使用数据库的统计量，**计算每种可能计划的代价**，然后使用最好的那一个。但是可能有很多种可能。对于一个给定顺序的连接，每次连接都有三种可能：哈希连接，归并连接，嵌套连接，所以就会有3<sup>4</sup>种可能。而连接顺序是一个[二叉树的排列问题](https://en.wikipedia.org/wiki/Catalan_number),应该有(2 * 4)! / (4 + 1)!种可能的顺序。就这个简化的问题，依然有3<sup>4</sup> * (2 * 4)! / (4 + 1)!种可能。

也就是说，有27 216种不同的可能，如果我把有B+树索引的情况考虑进归并排序里，可能的情况就增加到210 000种。还记得我说过，这个查询情况是非常非常简单的吗？

* 2）哭着说不玩了呗

这问题是很有趣，但是得不到答案有什么用，我还要挣奶粉钱的[译注4][annotation]

* 3）只尝试少量的执行计划，然后找出代价最小的那个

我又不是超人，我不可能计算出每种计划的代价，但是退而求其次，我能任意选出可能计划的一部分，计算出它们的代价，在这个子集里选出最好的那个计划

* 4）使用一些聪明的规则减少可能的计划数量

规则的类型有两类：

使用一些“合乎逻辑”的规则移除不可能的情况，但是又不会过滤掉很多可能的情况。比如说，嵌套循环连接的内关系必须是数据集里最小的那个

接受并不是最好的解决方案，而且使用一些比较激进的规则可以减少大量的情况。比如说，如果关系规模小，那就用嵌套循环连接，不要使用归并连接和哈希连接

在这个简单的例子里，我已经得处理许多种可能了，但是真实的查询可能还有**其他的关系运算**呢，比如OUTER JOIN, CROSS JOIN, GROUP BY, ORDER BY, PROJECTION, UNION, INTERSECT, DISTINCT，这都意味着更多的可能。

所以数据库是怎么做的？

####动态规划，贪婪和启发性算法<a id="dp_greedy_heuristic"></a>
关系数据库尝试了之前提到的多种方法。真实的优化器得**在限定的时间里找到好的解决方案**。

**大多数时间里，优化器不能找到最好的解决方案，而是一个“好的”解决方案**

对于小查询来讲，可以使用暴力破解法。而且，因为有些方法可以避免不必要的计算，所以中等规模的查询也可以使用暴力破解法。这些避免不必要计算的方法被称为动态规划（Dynamic Programming）。

**动态规划DP**
在Dynamic Programming两个词的背后，隐藏的内涵是“许多执行计划是相似的”。如果你看看下面的执行计划计划。

![overlapping_trees](http://pic.yupoo.com/tan91319/ETsxKFwJ/QN1xY.png)

他们都共享了一个相同的子树（A JOIN B），所以**不在每个执行计划里都计算一次子树的代价，取而代之的是，只计算一次并存储该值，然后等再见到该子树的时候才重用它**。更正式的说法是，我们遇到的是个重复问题（overlapping problem）。为了避免对部分结果的重复计算，我们做了一个备忘。

使用了这个技巧后，不再是 (2*N)!/(N+1)! 的时间复杂度了，我们“只”需要3<span>N</span>的时间花费。对之前4次连接的例子来说，原来是336现在就要81了。如果一个稍大的查询需要八次连接（实际上这也不算大），这意味着时间复杂度可以从57 657 600变成6561。

对CS高手来讲，我在之前提到的[正规课程](http://codex.cs.yale.edu/avi/db-book/db6/slide-dir/PPT-dir/ch13.ppt)里找到了一个算法。我不解释这个算法，在你懂动规或者你的算法很厉害后，你再去读这个算法吧！（这算个告诫吧）

	procedure findbestplan(S)
	if (bestplan[S].cost infinite)
	   return bestplan[S]
	// else bestplan[S] has not been computed earlier, compute it now
	if (S contains only 1 relation)
	         set bestplan[S].plan and bestplan[S].cost based on the best way
	         of accessing S  /* Using selections on S and indices on S */
	     else for each non-empty subset S1 of S such that S1 != S
	   P1= findbestplan(S1)
	   P2= findbestplan(S - S1)
	   A = best algorithm for joining results of P1 and P2
	   cost = P1.cost + P2.cost + cost of A
	   if cost < bestplan[S].cost
	       bestplan[S].cost = cost
	      bestplan[S].plan = “execute P1.plan; execute P2.plan;
	                 join results of P1 and P2 using A”
	return bestplan[S]

对更大的查询仍然可以使用动态规划的方法来找到最优执行计划，但是需要一些额外的（也叫启发式heuristic）规则来去掉一些可能的方案

* 如果我们仅对特定类型的执行计划作分析（比如，左深连接数 left-deep trees），我们会以 n * 2<sup>n</sup>来代替3<sup>n</sup>

![left-deep-tree](http://pic.yupoo.com/tan91319/ETsxJzwR/medish.jpg)

* 如果我们添加了一些逻辑规则来避免一些模式（比如“如果对给出的查询谓词，某个表相当于一个索引，那么我们不用对这个表做归并连接，而只用对索引做归并连接就好”），这将减少可能方案的数量，而不会有损最好的可能方案

* 如果我们对处理流（处理的先后顺序，比如“在其他的关系操作前，要先做连接操作”）这样也会减少大量的可能方案

* ...

**贪心算法**

但是对于一个大查询来讲，或者优化需要快速响应（快速获得优化响应，而不是获得查询结果），另外一种类型的算法就会被用到，也就是贪心算法（greedy algorithm）

贪心的思想是跟着一个规则（也是启发性规则），以渐近递增的方式建立一个查询计划。根据这个规则，贪心算法的每一步都会找到当前的最优解。算法从只有一个连接操作的查询计划开始，然后每一步，算法都会根据那个规则往查询计划里添加一个新的连接。

看这个例子。假设我们要在五个表上做四次连接（表A，B，C，D，E）。为了简化问题，我们只使用嵌套连接。使用的规则是“选择代价最低的那个连接”。

* 我们从任意的一张表开始（比如A）
* 计算所有和A做连接的代价（A可以是内关系，也可以是外关系）
* 计算得出A和B连接代价最小
* 然后计算每个可以和A JOIN B做的连接（A JOIN B可以是内关系，也可以是外关系）
* 计算得出（A JOIN B）和C做连接代价最小
* 然后计算每个可以和（A JOIN B）JOIN C的代价
* ...
* 最后能得到查询计划(((A JOIN B) JOIN C) JOIN D) JOIN E)

因为我们是随机的从A开始，所以也可以把相同的算法使用在B，C，D，E上，然后获得最小代价的那个执行计划。

顺便提一下，这个算法有一个名字，叫[最近邻算法](https://en.wikipedia.org/wiki/Nearest_neighbour_algorithm)

我不深究里面的细节，但是一个好的模型和一个在N * log(N)下的排序可以让这个问题很轻松地被解决。[参考](http://www.cs.bu.edu/~steng/teaching/Spring2004/lectures/lecture3.ppt)

这个算法的完全动规版本的代价是 O(N * log(N)) 而不是 O(3<sup></sup>)。如果有一个查询有20个连接，这意味着26和3 486 783 401的区别，这差别就大了！

这个算法的问题是，假设找到两个表的连接最优，而试图加一个最优连接其结果也是最优的。但是：

* 即使A JOIN B是在 A，B和C上最优的连接
* （A JOIN C） JOIN B 还是可能比 （A JOIN B） JOIN C要好

为了改善这个结果，可以使用不同的规则多次运用贪心算法，这样能获得最好的结果

**其他的算法**

【如果你很讨厌“Algorithm”这东西，那么跳过这一节吧，我要说的东西对本文的其他部分不是很重要】

找最优解（the best possible plan）对许多计算机学者来说是一个活跃的研究主题。他们通常会想找到更多问题或模式的更好解法。比如说：

* 如果一个查询是一个星连接（star join，是一种多连接查询的特定类型）许多数据库会使用特别的算法
* 如果查询是并行查询，许多数据库也会使用特别的算法
* ...

也有许多算法正在被研究以在大规模查询中取代动态规划。贪心算法其实是属于启发式算法（heuristic algorithms）的大家庭里的。贪心算法根据规则（或称启发式规则）保持它在上一步计算得到的解，然后在当前步添加条件找到新的解。有一些算法也依赖某个规则，并使用逐步迭代（step-by-step）的方式，但是他们不一定是保持之前步骤里的最优解，他们都称为启发式算法。

比如，遗传算法（genetic algorithm）遵循了规则，但是迭代时上一步的最优解就不是总被保持着。

* 一个解表示一个可能的完整查询计划
* 不是每次都只保持一个最优解，遗传算法每步都保持着P个解
* 0）P个查询计划被随机创建
* 1）只有最优代价的计划会被保持
* 2）最优计划被搅碎，然后产生P个新的计划
* 3）P个新计划被随机修改
* 4）1,2,3被重复运行T次
* 5）从P个解里找到最好的那个

循环的次数越多，得到的解就越好

这很神奇吧？不，这是自然界的法则：适者生存（only the fittest survives!）

仅供参考，遗传算法在[PostgreSQL](http://www.postgresql.org/docs/9.4/static/geqo-intro.html)里被实现了，但是我也不清楚他是否会被默认使用

在数据库里还有其他的启发式算法，比如模拟退火（Simulated Annealing），迭代改进（Iterative Improvement），两段式优化（Two-Phase Optimization）...但是我不知道这些算法现在有没有用在企业级数据库上，还是说只用在了研究性数据库里

####真正的优化器<a id="real_optimizers"></a>
【这一部分你也可以跳过，因为也并不是很重要】

这里巴拉巴拉得都很理论，我是一个开发人员不是一个研究人员，我更喜欢具体的例子。

让我们看看[SQLite 优化器](https://www.sqlite.org/optoverview.html)具体是怎么做的。这是一个轻量级数据库，所以他使用的是基于贪心算法，又带一点额外规则的很简单的优化：

* 在CROSS JOIN操作上，SQLite不会重排表顺序
* **连接操作通过嵌套连接实现**
* 外连接按照它们出现的顺序评估
* ...
* 在3.8.0版本前，SQLite在搜索最优查询计划时，用的是“最近邻”贪心算法。

等等...这不就是之前看过的那个算法吗？好巧哦！

* 3.8.0之后（2015年发布），SQLite在搜索最优查询计划时，则改用“N近邻”贪心算法

我们来看看其他的优化器是怎么做的。IBM DB2像所有其他的企业数据库一样，但是我会把重点放在它身上，因为这是我在转到大数据领域前用的最后一个企业级数据库。

如果我们看看[官方文档](https://www-01.ibm.com/support/knowledgecenter/SSEPGG_9.7.0/com.ibm.db2.luw.admin.perf.doc/doc/r0005278.html)，我们就能知道DB2优化器让你可以使用七个不同层级的优化：

* 在连接时使用贪婪算法
	* 0 - 最小等级优化，使用索引扫描，嵌套循环连接，避免查询重写（Query Rewrite）
	* 1 - 低等级优化
	* 2 - 完全优化
* 在连接时使用动态规划
	* 3 - 适当地优化，并作粗略的估计
	* 5 - 完全优化，使用和启发规则相关的所有技巧
	* 7 - 和5相似的完全优化，但是不使用启发式规则
	* 9 - 最大化优化，不遗余力地考虑所有的连接顺序，甚至包括笛卡尔积

DB2使用了贪心算法和动态规划。当然，他不会分享具体的启发规则，因为查询优化是数据库的核心技术。

另外，默认的等级是 5，在默认的优化器下，可以使用下列特性：

* 所有可用的统计量，包括频率值和分位统计量
* 除了只在很少的情况下使用的一些计算密集型规则，使用了所有的查询重写规则（包括物化查询表路由 materialized query table routing）
* 使用动态规划连接枚举（dynamic programming join enumeration），同时：
	* 限制使用内关系的合成
	* 限制对包括查找表（lookup table）在内的星形模式（star schemas）做笛卡尔积
* 考虑了大量的访问方式，包括列表预取（list prefetch 注：马上会看到这是什么意思），index ANDing（注：一种在索引上的特殊操作），以及物化查询表路由。

默认情况下，DB2使用带启发式规则的动态规划做连接排序。

其他的条件（GROUP BY，DISTINCT...）是以很简单的方式处理的。

####查询计划缓存<a id="query_plan_cache"></a>
因为创建一个查询计划很费时，大多数的数据库系统都会将计划存储到一个查询计划缓存（query_plan_cache）中，以避免对同一个查询计划的重复计算。这是一个很大的主题，因为数据库得知道什么时候该更新那些过时的查询计划了。

更新的思路是，设置一个阈值，如果表统计量的变化高于了阈值，就将和该表相关的所有查询计划从缓存中移除。

###查询执行器<a id="query_executor"></a>
进入执行器后，就有一个优化后的执行计划了。这个执行计划被编译成一段可执行的代码，若有足够的资源（内存，CPU），他就会被查询执行器执行。这些在执行计划中的操作（连接，排序等）可以串行执行，也可以并行执行，这取决于执行器。为了获取或者写入这些数据，查询执行器要和数据管理器交互，也就是本文的下一部分。

##数据管理器<a id="data"></a>

![data_manager](http://pic.yupoo.com/tan91319/ETsxIOf1/medish.jpg)

在这一步，查询管理器会执行查询，同时需要获取来自表和索引的数据。这会请求数据管理器以获取数据，但是也有两个问题。

* 关系数据库使用了事务模型，所以，如果你在某个时候不能获取数据，是因为别人在同时使用或者修改该数据。

* **数据检索在数据库里是最慢的操作**。因此数据管理器得足够智能地将获取的数据保存在内存缓冲里。

在本部分，我们将看到关系数据库是如何处理这两个问题的，我不会谈论数据管理器是怎么获取数据的，因为这不是最重要的（而且这文章已经够长了）

###缓存管理器<a id="cache_manager"></a>
就我刚才说的，数据库的主要瓶颈是磁盘I/O，为了改善性能，现代数据库用到了缓存管理器。

![cache_manager](http://pic.yupoo.com/tan91319/ETsxIB6q/medish.jpg)

并不直接从文件系统获取数据，查询执行器先向缓存管理器请求数据。缓存管理器有一块驻于内存的缓存称为**缓冲池（buffer pool）**。从内存获取数据能剧烈地提升数据库速度。但是能从缓存中获取的数据数量是难以指定的，因为它取决于你需要做的操作。

* 是顺序访问（全扫描）还是随机访问（通过行id访问）
* 读还是写

以及数据库使用的磁盘类型

* 7.2k/10k/15k rpm HDD
* SSD
* RAID 1/5/...

但是我得说，内存比硬盘快了100 - 100k倍

但是这导致另外一个问题（和具体的数据库系统有关），缓存管理器需要在查询执行器使用前，将数据获取到内存里，否则查询管理器就得从慢速磁盘里等待数据。

####预读取<a id="prefetching"></a>
这个问题称为预读取。一个查询执行器知道它需要哪些数据，因为它知道查询的整个流程，而且通过统计量对在磁盘上的数据有很好的了解。所以想法就是：

* 当查询执行器在处理它的第一笔数据的时候
* 它要缓存管理器预先载入第二笔数据
* 当它处理第二笔数据的时候
* 他要求缓存处理器预先载入第三笔数据，并通知缓存处理器第一笔数据可以从缓存中移走了
* ...

缓存处理器在缓冲池里存储所有这些数据。为了得知数据是否被需要，缓存处理器会向这些缓存的数据添加额外的信息（称为latch）。

有时候，查询执行器不知道它需要什么数据，一些数据库也不提供这种功能。作为替代方案，他们可以使用一个推测性预取（speculative prefetching），比如当查询执行器请求数据1,3,5的时候，它也可能在不久的将来请求7,9,11。或者一个顺序预取，在这种情况下，当请求完了一个数据，缓存管理器就从硬盘中载入下一个连续的数据。

现代数据库提供了一个称为缓冲/缓存**命中率（buffer/cache hit radio）**的单位，来监控预取的效果。命中率表示数据请求在不访问磁盘的情况下，多长时间能在缓存里找到一次数据。

注： 糟糕的缓存命中率并不表示缓存没有很好得起到作用。更多资料，请阅读[Oracle文档](http://docs.oracle.com/database/121/TGDBA/tune_buffer_cache.htm)

但是，缓冲还是会被内存数量所限制。因此，有必要移除一些数据，以让新的数据进入缓冲池。考虑到磁盘和网络I/O，载入和移除缓存也是有代价的。如果一个查询经常被执行，那不停地载入和移除它需要的数据根本就不高效。为了处理这个问题，现代数据库使用了缓冲区置换策略（buffer-replacement strategies）.

####缓冲区置换策略<a id="buffer_replacement"></a>
大多数的现代数据库（起码 SQL Server，MySQL，Oracle和DB2）都是用LRU算法。

**LRU**

**LRU**表示**L**east **R**ecently **U**sed(最近最少使用)。算法背后的思想是，将最近在使用的数据保持在缓存里，这样它更可能被再次用到。

下面是个直观的例子：

![LRU](http://pic.yupoo.com/tan91319/ETsxK7Gq/medish.jpg)

为了好理解，我假设在缓冲里的数据是不会因latches而被锁的（这样他们就能被移出缓存）。在这个简单的例子里，缓冲存储了三个元素：

* 1：缓存管理器（CM）使用了数据1，然后将它放入空的缓冲区
* 2：CM使用了数据4，然后将其放入半负载（half-loaded）的缓冲区
* 3：CM使用了数据3，然后将其放入半负载的缓冲区
* 4：CM使用了数据9，缓冲区满，所以数据1被移除，因为它是最后一个**最近被用到**的元素。数据9被加到缓冲里。
* 5：CM使用了数据4，数据4已经在缓冲区里，所以它又成为了第一个**最近被用到**的元素。
* 6：CM使用了数据1，缓冲区满，数据9被移出，因为它是最后一个**最近被用到**的元素
* ...

这个算法能很好地工作，但也会有一些限制。假使对一个大表做全扫描会发生什么呢？换句话说，当表或索引的大小高于缓冲区的大小会发生什么呢？使用这个算法会移除在缓存里所有的值，然而来自全扫描的数据可能只会用到一次。

**改进**

为了防止这种事情的发生，一些数据库添加了特定的条件，比如根据[Oracle文档](http://docs.oracle.com/database/121/CNCPT/memory.htm#i10221)

> “For very large tables, the database typically uses a direct path read, which loads blocks directly […], to avoid populating the buffer cache. For medium size tables, the database may use a direct read or a cache read. If it decides to use a cache read, then the database places the blocks at the end of the LRU list to prevent the scan from effectively cleaning out the buffer cache.”(对很大的表来说，数据库通常使用直接路径的读取，这种方式直接载入数据块，而不必将它先转移到缓冲区的缓存里。对于中等大小的表，数据库可能使用直接读取，也可能使用缓存读。如果使用缓存读，数据库将在LRU表尾放一个数据块，防止扫描动作过于高效地清除了缓冲区缓存)

有一些其他的数据库会使用LRU的高级版本，称为LRU-K。比如SQL Server使用了K = 2的LRU-K。

在该算法背后的思想是考虑更多的历史记录。简单的LRU（也可以认为是LRU-K，K = 1的版本），算法只考虑了最近一次的数据使用。而LRU-K：

* 考虑了最近K次的数据使用
* 根据数据使用的次数，给出了一个权重
* 如果新的数据载入缓存，但是经常使用的旧的数据不会被移除（因为他的权重很高）
* 但是如果数据不再使用了，算法是不会把旧数据一直放在内存里
* 而数据若没有使用，它的权重会随着时间降低

权重的计算也是有代价的，所以SQL-Server只使用了K=2的情况。这个值表现得很好，而且多余的开销可是可以接受的。

对LRU-K更深度的介绍，可以参阅它原始的研究论文（1993）：针对数据库磁盘缓冲的LRU-K页置换算法[The LRU-K page replacement algorithm for database disk buffering](http://www.cs.cmu.edu/~christos/courses/721-resources/p297-o_neil.pdf)

**其他算法**

当然，还是有其他管理缓存的算法的，比如：

* 2Q (和LRU-K类似的algorithm)
* CLOCK (和LRU-K 类似的algorithm)
* MRU (most recently used最近最常使用, 和LRU逻辑相同但规则不同)
* LRFU (Least Recently and Frequently Used，最近最少和最不频繁使用算法)
* ...

一些数据库有机会使用其他的算法，而不是默认的那个LRU

####写缓冲<a id="write_buffer"></a>

我只谈论了在使用数据前载入数据的读缓冲。但是在数据库里，当向磁盘存储数据，或者清空数据（flush）的时候，为了不产生许多单次磁盘访问，使用批量代替的一个个数据的读写，也用写缓冲。

要记住的是，缓冲存储的是页（数据的最小单位），而不是行（人们在逻辑上审视数据的方式）。在缓冲池中，若某一页的数据被修改，但是还没有写入到磁盘上的时候，称这一页为**脏数据（dirty）**。

有几个算法能决定将脏数据页写到磁盘上的最好时机，他们和事务的概念紧密相关，这也就是下一节要讨论的话题。

###事务管理器<a id="transaction_manager"></a>
最后，也很重要的是事务管理器。我们将看到如何确保每个查询都在它自己的事务里被执行。但是在那之前，我们还要理解一下事务的ACID概念

####ACID原则<a id="acid"></a>
一个满足ACID的事务是一项工作的基本单元，这确保了四件事：

* 原子性Atomicity：事务是要么全做，要么全不做的，即使得持续十个小时。如果事务在处理过程里崩溃，数据库状态将回到事务开始前（也叫事务回滚**rolled back**）。
* 隔离性Isolation：如果两个事务A和B在同时进行，则不管事务A是在B之前/之后/进行中完成，事务A和B的结果都没有变化。
* 持续性Durability：一旦事务提交**committed**（也就是成功完成了）。数据就会在数据库里保存着，无论发生什么（系统崩溃或错误）
* 一致性Consistency：只有有效的数据（就关系约束relational constraints和函数约束functional constraints来讲）能被写入数据库。一致性和原子性以及隔离性是相关的。

在同一个事务里，能够运行多个SQL查询对数据进行增删改查。当两个事务要同时使用到同一数据时，就有点混乱了。一个经典的例子就是资金转账，从一个账户转到另一个账户上的情况。想象有两个事务：

* 事务T1，将100块（美元）从A账户转到B
* 事务T2，将50块从A转到B

如果我们回顾一下**ACID**的性质：

* 原子性确保无论在T1进行时发生了什么（服务器崩溃，网络失败等等），都不可能出现钱从A的账户了扣了，但是却没打到B的账户上
* 隔离性确保了如果T1和T2同时发生，A最后将被转走150块，同时B将受到150块，或者都没有。举例来说，A少了150但B只拿到了50，因为T2被T1部分覆盖了（这也被称为不一致状态）。
* 持久性确保T1在提交了后，不会因为数据库崩溃就消失掉
* 一致性确保系统不会创建多余的钱，也不会吞掉一笔钱

【如果你希望的话，可以跳过这一部分，我这里要讲的东西对文章的其他部分不太重要】

许多现代数据库系统默认不使用完全隔离（pure isolation），因为这是极大的性能开销。SQL规范定义了4个级别的隔离。

* 串行化Serializable：最高级别的隔离性，两个事务同时发生时100%隔离，每个事务都在自己的世界里完成
* 可重复读Repeatable Read：在这个级别，每个事务也像是在自己的世界里运行，除了一种情况。

当一个事务成功结束并添加了一个新数据的时候，这些数据对其他正在运行的事务是可见（visible）的。但是，当一个数据成功添加了新数据，这个修改是不应该对其他正在运行的事务可见的。所以在有新数据这种情况时，打破了事务之间的隔离性，但对已存在的数据没有影响。

举例来说，如果事务A做了“SELECT count(1) from TABLE_X”的操作，然后事务B添加了一个新数据并成功提交，那么在事务A中再一次count(1)，这个值就和之前的不相同了。这种情况称为幻读（**phantom read**）。

* 已提交读Read Committed：这是在大多数数据库里的默认行为（Oracle，MySQL，PostgreSQL以及SQL Server）。这是可重复读再加上一个对隔离性的破坏。如果事务A读了一个数据D，然后在事务B里这个数据被修改（或者被删除了），事务B成功提交后，如果A再次读取数据D，他会发现数据被B修改了

这称为不可重复读 Non-repeatable read

* 未提交读Read Uncommitted:这是最低的隔离级别，是已提交读加上新的对隔离性的破坏。如果事务A读取了数据D，然后数据D被事务B修改（这时事务B还没有提交），如果A再读取D，它会发现数据更改了。但如果事务B回滚，则事务A中第二次读取的数据D将毫无意义，因为它被事务B修改的那个事件相当于没有发生（因为他回滚了）

这叫做脏读 dirty read

大多数数据库会添加自己对隔离性的自定义等级（比如PostgreSQL，Oracle和SQL Server使用的快照隔离性snapshot isolation）。甚至大多数数据库都没有实现在SQL规范中定义的所有等级（特别是未提交读）

隔离性的默认等级可以被用户/开发者在连接的时候重写（override），这只需要添加几行简单的代码。

####并发控制<a id="concurrency"></a>
要保证隔离性，一致性和原子性，真实原因是有对同一个数据的写操作（增删改）

* 如果所有的事务只读数据，它们可以同时工作，而且也不需要修改其他事务的行为
* 但是如果（至少一个）事务在修改某个其他事务正在读取的数据时，数据库需要找到一种方式向其他的事务隐藏这个修改。甚至，也需要确保这个修改不会被另外一个事务给擦除（erase）掉，因为另外的事务是看不到修改数据这个行为的。

这个问题称作**并发控制（concurrency control）**

处理问题最简单的方式是让每个事务一个接一个地运行（也就是串行化）。但是这样的可拓展性不好，如果在多核处理器上的话，只会有一个核在工作，效率也不高。

解决问题的理想方式则是，每次有一个事务被创建或者被取消的话：

* 监控所有事务的所有操作
* 检查有没有两个（或更多）的事务因为读取/修改同一数据而起了冲突
* 对事务里冲突部分的指令重新排序，以减少冲突部分的大小
* 以特定的顺序执行冲突部分的代码，而不冲突的部分仍然并行化就好
* 考虑事务被取消这种可能性

正式说来，这是带冲突计划的调度问题（scheduling problem with conflicting schedules）。更具体一点，这是一个非常困难且耗费CPU的优化问题。企业数据库是不会为每个新的事务花几个小时去找最佳调度的。因此，他们使用不那么理想化的方式，这样在两个事务发生冲突的时候会费点时间。[译注5][annotation]

####锁管理器<a id="lock"></a>
为了处理这个问题，大多数数据库使用的是锁（locks）以及数据版本控制（data versioning）。因为这个话题很大，我将重点放在锁机制上，然后稍微提一下数据版本控制的问题。

**悲观锁**

在这种锁背后的思想是：

* 如果事务需要数据
* 那么就对数据加锁
* 如果另外一个事务也需要这个数据
* 那么得等到第一个事务释放数据

这也称为**排他锁（exclusive lock）**

但是如果对只需要读数据的事务使用排他锁就很浪费，因为它使得其他仅仅希望读这个数据的事务也在等待。这就是为什么会有另外一种类型的锁，**共享锁（shared lock）**。

共享锁：

* 如果事务仅需要读数据A
* 则对数据加共享锁，然后读数据
* 如果第二个事务也只需要读数据
* 它也对数据A加共享锁，然后读数据
* 如果第三个事务需要写数据A
* 它对数据加排他锁，但是得等到另外两个事务释放了他们的共享锁，才能对数据A加排他锁

不变的是，当一个数据被加上了排他锁，那些只想读数据的事务也需要等到排他锁释放，才能对数据加共享锁。

![lock_manager](http://pic.yupoo.com/tan91319/ETsxJpYe/medish.jpg)

锁管理是加锁是释放锁的进程。它在内部把锁存储在一个哈希表里，key是锁住的数据，这样就能知道：

* 锁住该数据的是哪个事务
* 哪些事务在等待这个数据

**死锁**

但是锁的使用，可能会导致这样一种情况，两个事务在永久地等待一个数据

![deadlock](http://pic.yupoo.com/tan91319/ETsxIXYh/medish.jpg)

在这个图里：

* 事务A有个排他锁锁住了data1，同时在等待data2
* 事务B有个排他锁锁住了data2，但是呢，它在等待data1

这就叫**死锁（deadlock）**

在死锁时，锁管理器会取消一个事务（也就是回滚），以移除这种死锁状态。但这个决定也不是那么简单：

* 是杀死那个修改数据量最小的事务好呢？（这样回滚产生的代价最小）
* 还是杀死那个运行时间最小的事务好呢？因为其他的事务都等得比他久
* 亦或是杀死那个马上就完成的那个事务好呢？（这样避免一定可能的饥饿 starvation）
* 万一回滚，将有多少事务被这个回滚所影响？

但在做决定前，我们需要先检测是否有死锁的情况存在。

哈希表可以看做一个图（graph），像之前的图表那样。如果有死锁，则在图里会有一个环。因为检查图中是否有环的代价较高（有锁的图还是很大的），一个简单实用的方法是：使用**超时机制（timeout）**。如果一个锁在给定的时限内没释放，那么事务就会进入死锁状态。

锁管理器会在加锁前查看该锁是否会造成死锁。但是，再提一遍，若完美地解决这个问题，计算量实在太高，因此，预先的检查通常就是一组基本规则。

**两段锁**
确保完全隔离性**最简单的方式**就是：是否一个事务在开始前和结束后需要加锁和释放锁。这意味着事务在开始前得等待它所有的锁，而在结束后才释放它所有的锁。这是可行的，但是**会产生大量的时间浪费**,因为他得等待所有的锁。

一个更快的方式是在DB2和SQL Server中使用的**两段锁协议（Two-Phase Locking Protocol）**，它将一个事务分成了两个阶段：

* **增长阶段（growing phase）**：事务获取锁，但不释放任何锁的阶段
* **缩减阶段（shrinking phase）**：事务能释放锁（释放那些已经处理过，且不会再处理的数据上的锁），但是不会获取新的锁的阶段

![two-phase-locking](http://pic.yupoo.com/tan91319/ETsxLaQZ/medish.jpg)

在这简单的两条规则的背后逻辑是：

* 释放不会再使用到的锁，减少了其他事务等待这些锁的时间
* 防止在事务开始后，在事务里多次获取数据，有时获取到的是被修改的数据，它们和之前获取的数据不一致

这个协议能很好的工作，除非修改了数据的事务在释放对应锁的时候取消掉（回滚了）。你会面对这种情况：另外一个事务读取了这个被修改的值，但是这个值最后还是回滚了。为了避免这个问题，**所有的排他锁都必须在事务结束时才释放。**

**其他的话**

当然，真实的数据库使用了更多更复杂的系统，包括更多类型的锁（比如目的锁 intention locks），以及更多的粒度（granularity），比如行锁，页锁，分区锁，表锁，以及表空间锁，但是它们的思想是一样的。

我只描述了完全基于锁的方法，数据版本控制（data versioning）是处理该问题的另外一个方式。

在版本控制背后的思路是：

* 多个事务能在同时修改同一个数据
* 每个事务都有一个数据的副本（也称版本）
* 如果两个事务同时修改了一个数据，只有一个修改是被接受的，另外一个修改将被拒绝，同时相关的事务将回滚（也可能是重新开始）。

这提高了性能，因为：

* 读事务不会阻塞写事务
* 写事务不会阻塞读事务
* 没有来自“臃肿而龟速的”锁管理器的开销

除了当两个事务同时对某一个数据写的时候外，这种方案的每一点都要比锁机制要强。此外，你也能快速地处理一次巨大的磁盘开销。

数据版本控制和锁机制是两种不同的类型：**乐观锁（optimistic locking）和悲观锁（pessimistic locking）。他们都有利有弊，而这取决于它们的使用场景（是更多读还是更多写）。对数据版本控制的描述我推荐这个[写的很好的文档](http://momjian.us/main/writings/pgsql/mvcc.pdf)，它对PostgreSQL怎样多实现并发控制做了很好的描述。

一些数据库，比如DB2（直到DB2 9.7）和SQL Server（除去快照隔离性）只使用了锁。其他的数据库，像PostgreSQL，MySQL和Oracle则使用了锁和数据版本控制混合的方式。我还不知道有只用版本控制的数据库（如果你知道只使用数据版本控制的数据库，放轻松，然后告诉我，嗯！）。

【2015/08/20更新】一个读者告诉我

> Firebird and Interbase use versioning without record locking.    
Versioning has an interesting effect on indexes: sometimes a unique index contains duplicates, the index can have more entries than the table has rows, etc.    
(Firebird and Interbase只用了版本控制，没有使用记录锁。    
有个版本控制对索引的影响非常有意思，有时，一个唯一索引会包含了冗余，这个索引能比表里的行有更多的实体数据)

如果你阅读了隔离性等级的那一部分，你可以想象一下，当你提升隔离等级的时候，你就会提升锁的数量，因此时间就会花费在事务等待锁这件事上了。这就是为什么大多数的数据库不默认使用最高的隔离等级（可串行化）的原因。

照旧的是，你可以查看主要数据库相关的文档，比如[MySQL](http://dev.mysql.com/doc/refman/5.7/en/innodb-transaction-model.html)，[PostgreSQL](http://www.postgresql.org/docs/9.4/static/mvcc.html)，[Oracle](http://docs.oracle.com/cd/B28359_01/server.111/b28318/consist.htm#i5337)

####日志管理器<a id="log"></a>
我们已经知道，为了增加性能，数据库会在内存缓冲里存储数据。但是如果在事务提交的时候，数据库崩溃了，在内存中的数据还是会丢失，这就破坏了事务的持久性。

假如你是把所有的东西直接写到硬盘里，那么当数据库崩溃的时候，你将面对的是只有一半的数据在磁盘里面，而这又破坏了事务的原子性。

应当认为**事务由于修改而做出的任何写操作，可以是完成的，也可能未完成。**

为了处理这个问题，有两种方法：

* 影子拷贝/影页（**shadow copies/pages**）：每个事务都创建它关于数据库的副本（或者数据库的一个部分），然后在这个副本上工作。万一出错了，副本就直接移除，如果成功，数据库就使用一些文件系统的技巧，把数据从副本及时的转移过来，然后再移走旧数据。

* 事务日志（**transaction log**）：事务日志是一个存储空间。在每次写磁盘前，数据库都要往事务日志里写一部分信息，这样以防事务崩溃或者回滚，数据库知道怎么移除或者完成未完成的事务。

**WAL**

当在很大的数据库上使用事务时，影子拷贝/影页技术会产生巨大的磁盘开销。这就是为什么现代数据库使用事务日志的原因。事务日志必须被存储在稳定的存储空间里。我不打算深入存储技术，但是起码要强制使用RAID磁盘，这样才能防止磁盘故障。

大多数数据库（起码Oracle，[SQL Server](https://technet.microsoft.com/en-us/library/ms186259(v=sql.105).aspx)，[DB2](http://www.ibm.com/developerworks/data/library/techarticle/0301kline/0301kline.html)，[PostgreSQL](http://www.postgresql.org/docs/9.4/static/wal.html)，MySQL和[SQLite](https://www.sqlite.org/wal.html)）处理事务日志使用的是**预写日志规则WAL（Write-Ahead Logging protocol）**。WAL有三条规则：

* 1）每次对数据库的修改都产生一条日志记录，日志记录必须在数据写入磁盘前被写入事务日志里。
* 2）日志记录必须顺序写入，日志记录A在日志记录B之前发生，那么A就要在B之前被写入
* 3）当事务提交时，提交命令必须在事务成功之前被写入事务日志

![log_manager](http://pic.yupoo.com/tan91319/ETsxJro5/medish.jpg)

这是日志管理器的工作。简单来看，日志管理器在缓存管理器和数据访问管理器（它将数据写入磁盘）之间，它将每一个更新/删除/添加/提交/回滚操作都在写入磁盘之前写入事务日志里。简单吧？

大错特错！！！

我们之前看过那么多东西，你也该知道:和数据相关的所有东西都逃不脱“数据库效应”的诅咒。严肃点将，问题是要找到在写日志的同时还要保持高性能。如果写事务日志很慢，那它将拖累其他所有的事情。

**ARIES**

1992年，IBM的研究人员“发明”了一个WAL的加强版本，ARIES。ARIES或多或少地使用在了最现代的数据库上。或许逻辑不太一样，但ARIES背后的概念使用得非常广泛。

我之所以在发明上打了引号，是因为它来自于这个[MIT课程](http://db.csail.mit.edu/6.830/lectures/lec15-notes.pdf)，IBM研究者做的只是很好地实现了一个事务恢复系统而以，没别的了。在ARIES的文章发表的时候，我才5岁，我才不关心那些屌丝研究员之间的八卦呢。事实上，我说这个故事是为了让你们在最后一个技术章节开始前小小地喘息一会。不过我已经读过那个宏伟的[对ARIES的研究了](http://www.cs.berkeley.edu/~brewer/cs262/Aries.pdf)，我发现真的非常有趣！在这一部分，我只说一点对ARIES的概览，但是如果你真的想了解的话，我强烈推荐你读读这篇文章。

ARIES代表**A**lgorithms for **R**ecovery and **I**solation **E**xploiting **S**emantics。

这项技术的目的有两个：

* 1）当写日志的时候保持一个好的性能
* 2）有一个快速而可靠的恢复系统

一个数据库要回滚一个事务有几个原因：

* 用户要取消它
* 服务器或者网络故障
* 事务破坏了数据库内部的完整性（比如，你在某个列上加了UNIQUE约束，但是还是插入了一个冗余的数据）
* 因为死锁

有时候（比如，网络故障的情况），数据库能从事务里恢复

这可能吗？为了回答这个问题，我们需要理解那些存储在日志记录里的信息。

**日志**

在事务里的每一个操作（添加/移除/修改）都会产生一个日志。日志记录由以下内容组成：

* LSN：一个唯一的日志序列号（a unique **L**og **S**equence **N**umber），LSN给出了一个时间序列，这表示如果一个操作A在操作B前发生，A的LSN记录将会小于B的LSN记录。
* TransID：产生该操作的事务ID
* PageID：被修改数据的磁盘位置。因为在磁盘上，数据的最小单位是页，所以数据的位置就是包含了该数据的页的位置。
* PrevLSN：对同一事务的之前一条日志记录的连接
* UNDO：这就是去掉操作所带来影响的方式

比如：如果操作是一个更新，那么UNDO可能存储更新前的值（也称状态），这是“物理UNDO”，或者存储要回到之前状态的反向操作，这是“逻辑UNDO”

* REDO：这是回放操作（replay）的方式

同样的，也有两种方式做。或者存储元素在操作之后的值或状态，又或者存储回放操作本身

* ... 另外，ARIES日志还有两个其他的字段，UndoNxtLSN和Type

甚至，每个磁盘上的页（存储数据的，而不是存日志的）里都存有最后一个更改其上数据的日志LSN。

LSN的产生方式挺复杂的，因为他和日志的存储方式有关，但是它们的思想是一致的。

ARIES只使用逻辑UNDO，因为要处理物理UNDO的话实在有点乱

注：就我知道的而言，只有PostgreSQL没有使用UNDO，它使用垃圾收集器的Daemon来移除旧版本的数据。这和PostgreSQL里通过实现数据版本控制来做并发管理有关。

为了给你一个更好的理解，这里有个直观而简单的日志记录的例子，它由查询“UPDATE FROM PERSON SET AGE = 18;”产生，假设该查询是在事务18里进行的。

![ARIES_logs](http://pic.yupoo.com/tan91319/ETsxIKCg/medish.jpg)

每个日志都有一个唯一的LSN，互相连接的日志记录来自相同的事务。日志以时间序连接，也就是说，日志链表里的最后一条，就是最后一个操作的记录。

**日志缓冲**

为了避免日志的写入成为主要的瓶颈，可以使用日志缓冲（**log buffer**）。

![ARIES_log_writing](http://pic.yupoo.com/tan91319/ETsxIgjQ/medish.jpg)

当查询执行器请求一个修改的时候：

* 1）缓存管理器在它的缓冲里存储这个修改
* 2）日志管理器在它的缓冲里存储相关的日志
* 3）在这一步，查询执行器认为操作已经完成，因此可以请求另外一个修改操作
* 4）之后（可能稍微后面一点）日志管理器将日志写到事务日志里。什么时候写日志由算法决定
* 5）再之后（也是稍后）缓存管理器将修改写入磁盘。什么时候将数据写入磁盘也是由算法决定

当事务提交，表示事务里的所有的操作，也就是步骤1,2,3,4,5都已经完成了。在事务里写日志是很快的，因为只是在事务日志的某个地方添加了一行日志而已，相比而言，往磁盘里写数据的方式就会复杂一点，因为写进去的数据还要能快速的读出来。

**STEAL和FORCE策略**

为了性能原因，**第5步也可以在提交之后才完成**，因为万一崩溃了，仍然可以根据REDO日志恢复事务。这叫做**NO-FORCE策略**。

数据库也可以使用FORCE策略，也就是说，步骤5必须在提交前完成，这样就降低了恢复时的工作量。

另外一个问题是选择多次分步地将数据写入磁盘（**STEAL策略**）呢，还是等到有了提交命令，缓冲管理器才把数据一次写入（**NO-STEAL**）呢？在STEAL和NO-STEAL之间的选择依赖于你想怎么做：快速地写，但是恢复过程很漫长呢，亦或是快速的恢复呢？

这里有一些对这些恢复策略的总结：

* STEAL/NO-FORCE需要UNDO和REDO：性能最好，但是日志和恢复过程会比较复杂（比如ARIES）。**这是大多数数据库使用的方式**。注：我在很多研究论文和课程里看到这个说法，但是我在官方文档里没有明显地看到他们提过。
* STEAL/ FORCE 只需要UNDO
* NO-STEAL/NO-FORCE 只需要REDO.
* NO-STEAL/FORCE 什么都不需要: 性能最糟糕，也需要大量的存储空间。

**恢复**

好了现在有了完备的日志，我们得用上他们。

让我们假设新的实习生把数据库搞崩溃了（铁律n°1，老是实习生的问题哦）。你重启了数据库，然后恢复了进程，重新开始。

ARIES 用三个过程从崩溃里恢复：

* 1）分析过程（**The Analysis pass**）:恢复进程将读取完整的事务日志，以重建在崩溃时发生事件的时间线。这决定了哪些事务是要回滚的（所有没有提交的事务都要求回滚），而且在崩溃的时候，哪些数据需要被写入磁盘。
* 2）重做过程（**The Redo pass**）：这一过程从分析过程决定的一个日志记录开始，然后使用REDO更新数据库到崩溃前的状态。

在重做过程里，REDO日志通过LSN按照时间序列被处理。

对每一条日志，恢复进程都会在磁盘上找到被修改数据的页，然后读取页上面记录的LSN

如果磁盘页上的LSN >= 日志里的LSN，这表示在系统崩溃前，数据已经被修改了（因为崩溃前，磁盘上的值被在该日志之后的某个操作修改过），这时就什么都不做

如果磁盘页上的LSN < 日志里的LSN，则更新磁盘上的页

等重做完成，然后就可以回滚了，因为这样简化了恢复的处理。（注：我不确定现代数据库是否真的是这样做的）

* 3）撤销过程（**The Undo pass**）：这一过程回滚所有在崩溃时未完成的事务，回滚以每个事务的最后一个日志开始，然后以逆向时间序列处理日志的UNDO（通过日志记录的PrevLSN）。

在恢复过程中，事务日志必须提醒恢复进程的动作，使得写在磁盘上的数据和在事务日志中的保持同步。一个解决办法是移除正在撤销的事务日志记录，但是相对而言有点困难。ARIES是向事务日志里写补偿日志，然后以逻辑删除的方式移除事务记录。

当事务被人为的取消，或者被锁管理器取消（为了终止死锁），或者仅仅因为网络故障，此时的分析过程都不必要。事实上，REDO和UNDO的信息在驻于内存的两个表里即可用到。

* 事务表（transaction table），存储所有当前事务的的状态
* 脏页表（dirty page table），存储那些需要被写入硬盘的数据

这些表会根据每个新的事务，而被缓存管理器和事务管理器所更新。因为它们是存储在内存中的，所以当数据库崩溃，它们就会消失掉。

而分析过程的工作就是在崩溃之后，根据在事务日志中的信息重建这两个表。而为了加速这个分析过程，ARIES还提供了检查点（checkpoint）这一术语。checkpoint的思路是时不时地将事务表，脏页表以及当时最后的一个LSN写到磁盘里去，这样进入进行分析过程的时候，只有在该LSN后面的日志才会被分析。

##总结<a id="conclude"></a>
在写这篇文章之前，我还是知道这个主题有多大的，而且我也知道要深入地写一篇得花不少时间。最后我还是很乐观，而且我花的时间比我想的还要多两倍多，但是我也学了很多。

如果你想要一个关于数据库的概览，我推荐你阅读这篇研究性论文：数据库系统架构[Architecture of a Database System](http://db.cs.berkeley.edu/papers/fntdb07-architecture.pdf)。这是一个对数据库很好的介绍（110页），而且这个是仅有的对非计算机科学专业的人员可读读物。我在构思这篇文章的时候，这篇论文帮了我很多，而且它跟我的文章很相似，都没有过多地把重点放在数据结构和算法上，而是更着重于体系架构。

如果你认真读完了这篇文章，你现在应该理解了数据库是多么强大了。因为这是一篇很长的文章，让我帮你回忆一下我们都看过什么东西吧！

* 对B+树索引的综述
* 数据库的整体概览
* 对基于代价的优化的介绍，重点是连接操作
* 缓冲池管理的导论
* 事务管理的概述

但是数据库里其实还含有更多智慧结晶，比如说，我没有触及的问题：

* 如何管理数据库集群，如何维护全局事务
* 在数据库运行时如何做快照
* 如何有效的存储和压缩数据
* 如何管理内存

所以啊，当你在满是bug的NoSQL数据库和坚如磐石的关系数据库之间选择的时候，三思而后行！不要误会，一些NoSQL数据库也是很棒的，但是他们还太年轻，还只在一部分应用中处理特定的问题。

作为总结，如果有人问你数据库是怎么工作的，不要走开，你现在能这样回答他了！

![magic_low_9](http://pic.yupoo.com/tan91319/EU5xR9bZ/medish.jpg)

或者给他看这篇文章吧！

##译注<a id="annotation"></a>
[1]： constant 作名词表示常量，形容词表示恒定不变

[2]： 原文 turn lead into gold

[3]： 原文 all or nothing

[4]： 原文 I need money to pay the bills

[5]： 理想的方式是在事务运行前做调度，这样花的时间不实际，而实际是在事务运行时，在事务间解决冲突

[annotation]: #annotation "译注"
