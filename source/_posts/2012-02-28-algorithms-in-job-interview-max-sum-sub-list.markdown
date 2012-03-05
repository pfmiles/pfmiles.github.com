---
layout: post
title: "[面试中的算法]求子数组的最大和"
date: 2012-02-28 22:48
comments: true
categories: [algorithm, interview]
description: "求子数组的最大和"
keywords: "algorithm, 算法, 子数组, 最大和"
---
此问题由[v\_JULY\_v](http://blog.csdn.net/v_JULY_v/article/details/6050133)整理发布并发表于[blog](http://blog.csdn.net/v_JULY_v)上, 版权归原作者所有。

**输入一个整形数组,数组里有正数也有负数。
数组中连续的一个或多个整数组成一个子数组,每个子数组都有一个和。
求所有子数组的和的最大值。要求时间复杂度为 O(n)。
例如输入的数组为 1, -2, 3, 10, -4, 7, 2, -5,和最大的子数组为 3, 10, -4, 7, 2, 因此输出为该子数组的和 18**

### 分析：
贪心算法能在此问题的任意一个子问题上找到最优解(最优子结构)，所以可以用贪心算法解决。

<!-- more -->

### 程序：
	# 1, -2, 3, 10, -4, 7, 2, -5, output the maximum summarized sub-list
	# greedy algorithm
	def greatestSumSubList(al):
	    mx = 0
	    sm = 0
	    for i in al:
		sm += i
		if sm < 0:
		    sm = 0
		if sm > mx:
		    mx = sm
	    return mx

	print greatestSumSubList([1, -2, 3, 10, -4, 7, 2, -5])
输出： 18
