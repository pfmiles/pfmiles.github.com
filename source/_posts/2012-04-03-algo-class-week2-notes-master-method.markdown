---
layout: post
title: "Algo-class第二周小记: Master Method"
date: 2012-04-03 11:40
comments: true
categories: [algo-class, algorithm] 
keywords: "algo-class, master method, algorithm, 算法"
description: "Algo-class第二周小记: master method的理解与运用"
---
week2主要讲了master method，感觉从来没有听过如此清晰有条理的介绍master method的讲解，不得不佩服[Tim Roughgarden](http://theory.stanford.edu/~tim/)教授的功力。  
从一些随课笔记总结起来，master method是用来预估divide and conquer算法的大O时间复杂度的公式；它要求divide的时候“所有sub-problems的规模都相等”才能适用。  
要使用master method, 首先要将算法的时间消耗函数写成递推公式形式： 
$$T(n) &lt;= aT(\frac{n}{b}) + O(n^d)$$
其中T(n)是本层递归所消耗的时间，$T(\frac{n}{b})$自然就是下一层递归所消耗的时间了，b就是每次往“下层”递归调用时问题规模缩小的倍率，a就是代码中递归调用的个数；$O(n^d)$ 是将递归调用之外的其它代码的时间开销写成关于输入规模n的指数形式，d就是这个指数。  
基础情况： 当n足够小时，T(n) &lt;= 一个常数  
a,b,d与输入规模无关。  
那么，根据a,b,d的情况，T(n)可能被表示为三种情况：  
![Master Method](/images/masterMethod.png "Master Method")  
<!-- more -->
对于这个式子，我个人的、不严格的、直觉的理解就是：

1. 当$a = b^d$时，程序规模被divide之后，合并代价增大的幅度与程序规模缩小的幅度相当，所以复杂度公式中合并代价部分和切分、解决部分的复杂度都要算在里面
1. 当$a < b^d$时，合并代价增长快于切分、解决缩小幅度，所以复杂度公式中只算合并代价部分
1. 当$a > b^d$时，切分、解决部分时间代价占主导，忽略递归调用以外的项，只写出divide and conquer递归调用部分的时间开销

不过其实这个是有严格的证明的：  
设$T(n) = aT(\frac{n}{b}) + cn^d$  
对于第j层递归 ，运算量为：$a^j \times c \times (\frac{n}{b^j})^d$  
即：$cn^d(\frac{a}{b^d})^j$，所以a和$b^d$的大小关系是关键！  
进而总计算时间： $total work \leqslant cn^d \centerdot \sum\limits_{j=0}^{\log_bn}(\frac{a}{b^d})^j$, 然后通过这个式子，由a和$b^d$的大小关系可以较容易想通为什么master method会是上述这种三段分段函数的形式。  
Week2作业是快排...写个快排倒是简单，可是作业要求要count快排过程中的比较次数...在作业论坛里面看到哀嚎声一片...毕竟实现只要稍微有点偏差，虽然同样是快排，确实很容易比较次数不一样。  
值得一提的是讲解中给出的in-place的快排partition方法，就是不用建立额外的数组来进行partition，python示意代码如下：
	def partition(pivotIndex, arr):
	    swap(arr, 0, pivotIndex)
	    pivot = arr[0]
	    i = 1 # index 0...i < pivot
	    j = 1 # index i...j > pivot
	    for k in range(1, len(arr)):
		if arr[k] <= pivot :
		    swap(arr, i, k)
		    i += 1
		    j += 1
		else:
		    j += 1
	    return (arr[1:i], pivot, arr[i:j])
另外，从快排的例子可以看出，若divide and conquer算法是基于某种比较结果，将问题规模初步divide and conquer的话，每一次比较“能否将问题划分为规模差不多均等的子问题”将很大地影响算法实际的执行效率。
