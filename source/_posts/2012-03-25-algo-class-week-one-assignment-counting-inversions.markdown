---
layout: post
title: "[algo-class]第一周作业——Count Inversions"
date: 2012-03-25 15:41
comments: true
categories: [algo-class, algorithm]
---
[algo-class](http://algo-class.org)还是挺有意思的，一上来就讲merge sort；介绍了分治法以及"piggyback"的思路...当作是复习吧：
### 分治法(divide and conquer)
1. divide into smaller problems
1. conquer sub-problems recursively
1. combine solutions of problems into one for the original problems

而"piggyback"的意思就是：记住一些经典算法的实现，遇到类似问题的时候，可以将要解决的问题“搭载”到那些经典算法上面，一般都具有和那些经典算法相同的平均时间复杂度；相当于是把那些经典算法当作一个个“模式”或者“骨架”了(所以说平时经常听到的经典算法还是要熟悉熟悉的，很多地方能派上用场)。

这一周围绕merge sort来讲divide and conquer真是再合适不过了；而作业“Count Inversions”就是一个将场景piggyback到merge sort上的一个典型例子；  作业给出了一个包含10W行数据的文本文件，每一行一个随机数字，要求找出这10W个数字中的所有Inversions。我用python实现如下：
	def countInversions(arr):
	    length = len(arr)
	    if length == 0 or length == 1:
		return (0, arr) 
	    mid = int(length / 2)
	    (leftCount, leftSortedArr) = countInversions(arr[0:mid])
	    (rightCount, rightSortedArr) = countInversions(arr[mid:length])
	    (mergedCount, mergedSortedArr) = mergeCount(leftSortedArr, rightSortedArr)
	    return (leftCount + rightCount + mergedCount, mergedSortedArr)

	def mergeCount(lArr, rArr):
	    count = 0
	    mergedArr = []
	    i = 0
	    j = 0
	    lenl = len(lArr)
	    lenr = len(rArr)
	    while i < lenl and j < lenr:
		if lArr[i] > rArr[j]:
		    count += lenl - i # key line
		    mergedArr.append(rArr[j])
		    j += 1
		else:
		    mergedArr.append(lArr[i])
		    i += 1
	    if i < lenl:
		mergedArr.extend(lArr[i:lenl])
		
	    if j < lenr:
		mergedArr.extend(rArr[j:lenr])
	    return (count, mergedArr)

	import sys
	if len(sys.argv) < 2:
	    print 'usage: python countInversionInAFile path_to_file'
	    exit(1)
	numbers = [int(line) for line in open(sys.argv[1], 'r')]
	print countInversions(numbers)[0]

程序或许想起来简单但实际上要正确还是要费点功夫的；有些地方没想到的还非得debug才能看清楚为什么，比如程序中标注为"key line"那里，本来我写的是`count += 1`的但怎么都不对，后来debug才看出来应该写`count += lenl - i`。

另外，本来打算跟2门课程的，但现在看来我这忙碌的程序员完全没有时间...只好放弃“机器人汽车”那门课目前专门跟这一门了，毕竟“算法”的可操作性要强一些...
