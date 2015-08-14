---
layout: post
title: Programming Contest Challenge Book
categories: dsa
---
<script>
	
</script>
##挑战程序设计竞赛（第二版）阅读笔记

------------

###2015/07/21

######poj 1852 Ants

因为在每笔数据来时没有做min和max结果的初始化，所以导致本次可能使用到了上一次结果。

TLE是因为使用了iostream进行IO，需要换成printf，scanf。老问题，看来IO对程序速度影响确实很大！使用预处理define了替代max和min函数，但bottle-neck不是这个，这是没有必要的。

这里不需要维护一个数组，只需要遍历一遍所有的蚂蚁位置。

------------
