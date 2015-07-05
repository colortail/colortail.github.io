---
layout: post
title: 浅析二分查找
categories: dsa
---

对数组这种数据结构而言，因为下标都是整型，所以对它可以使用**二分查找**来查询其中元素的索引值。

当然大前提是数组都已经是有序的了！

函数接口表示为:**binSearch(key, list, lo, hi)**

虽然查找范围是[lo, hi),但某些实现还是对[lo, hi - 1]直接查询，做了三次判断，如*Java JDK*中的*Arrays和Collections*类，这里认为是基本实现。

一般而言，基本的二分查找方法对含有相等元素的数组是比较不确定的。

设list = [1,2,3,3,3,3,4]，则在Java默认接口中，二分查找3会返回下标3,而非第一个3（2），或最后一个3（5）。


这里的问题源于*Algorithm 4th*中，对符号表介绍了几种典型操作：
{% highlight java %}
class ST<Key extends Comparable<key>, Value> {
	...
	int rank(Key key);	//小于key值的元素个数，也就是key的排名
	Key select(int k);
	Key floor(Key key);	//小于等于key的最大值，即key的下整
	Key ceiling(Key key); //大于等于key的最小值，即key的上整
}
{% endhighlight %}
虽然符号表的键值唯一，使用数组时可以直接使用二分查找，但这里希望将rank等操作通过二分查找推广到更广义的含有重复元素的数组上。

也就是对list查3，可以返回多个相同元素中的最后一个或者第一个。

###最右

对含有重复元素的数组使用二分查找，语义比较直观的是返回查找到元素的最右下标，在计算[a,b)之间有多少个元素等时，这样很直接。

二分查找算法的两个细节问题，一是，lo和hi的变更条件，二是，查找终止的条件。

这里查询直接以[lo, hi)的范围来看，mi = (lo + hi) / 2时，原区间被分为[lo, mi) + [mi, hi)

1.因为list[mi]被排除在左侧之外，所以进入左侧区间的条件即是：key < list[mi]，反之则进入右侧,这个条件和区间的划分是吻合的。没有第三种情况。

2.因为问题被分为两个子问题，所以结束条件其实退化为，某个区间只含有一个元素了，假设取到的值为[e)。此时，lo + 1 == hi

3.若e == key，则表示查询到（查询的是最右侧元素），否则e的下标实际是小于key的最大元素.(这里直接返回-1)
{% highlight python %}
def binSearch(key, list, lo, hi):
	while lo + 1 != hi:
	mi = (lo + hi)/2
	if key < list[mi]:
		hi = mi
	else:
		lo = mi
	return lo if key == list[lo] else -1
{% endhighlight %}

多个元素之所以得到最右的原因是当存在==情况时，排除了左侧可能存在的相同元素，但因为是连同被选中的mi一起跳转到右侧区间，所以一定有一个key值存在于被选中区间。

###最左

若求取rank，则直接改变一下跳转条件为：key <= list[mi],则多个元素在相等时，改为跳转到左侧，因为区间为[a,mi),所以在区间[a, mi]（右侧改闭区间）全是小于等于该元素的。

这种情况和区间划分是有冲突的，这导致终止时会有多种不同的情况。

一般情况是，左侧区间的hi值停留在key所在的最左侧元素，[x, key)内的元素始终满足x <= key，所以hi不会下降，x慢慢收缩。此时无论key值是否存在，最终的hi都是rank值。

非常特殊的情况是，key值是数组里第一个元素，这时lo == 0,hi == 1,最终结果是[key),但是实际的rank应该为lo，而不是hi。此时key == list[lo]。另外若元素不存在，则key < list[lo]的。
{% highlight python %}
def rank(key, list, lo, hi):
	while lo + 1 != hi:
		mi = (lo + hi)/2
		if key <= list[mi]:
			hi = mi
		else:
			lo = mi
	return lo if key <= list[lo] else hi
{% endhighlight %}

比如下面得到 "5 0"
{% highlight python %}
if __name__ == '__main__':
	list = [1,2,3,3,3,3,4]
	print binSearch(3,list,0,len(list))
	print rank(1,list,0,len(list))		
{% endhighlight %}