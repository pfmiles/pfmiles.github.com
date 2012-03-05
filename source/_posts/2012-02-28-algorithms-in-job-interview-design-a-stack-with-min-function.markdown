---
layout: post
title: "[面试中的算法]设计包含min函数的栈"
date: 2012-02-28 00:14
comments: true
categories: [algorithm, interview] 
description: "设计包含min函数的栈"
keywords: "algorithm, stack, min, 算法, 面试, interview"
---
此问题由[v\_JULY\_v](http://blog.csdn.net/v_JULY_v/article/details/6050133)整理发布并发表于[blog](http://blog.csdn.net/v_JULY_v)上, 版权归原作者所有。

**问题描述：定义栈的数据结构,要求添加一个 min 函数,能够得到栈的最小元素, 要求函数 min、push 以及 pop 的时间复杂度都是 O(1)**

### 分析：
主要是抓住栈的**“还原现场”**的能力 —— 由于随着元素的加入与弹出，“最小元素”随时可能变化；在栈的每一个状态，都要维护一个当前最小元素的记录;随着push或pop，“最小元素”也跟着更新或还原到上一次的状态；所以，“最小元素”应该随着栈元素被记录在栈的每一层。

<!--more-->
### 程序：
	class MinStack(object):
	    def __init__(self):
		self.arr = []
		
	    def push(self, ele):
		if len(self.arr) == 0:
		    # no elements yet
		    self.arr.append((ele, ele));# pushes (ele, minEle)
		else:
		    self.arr.append((ele, ele if ele < self.arr[-1][1] else self.arr[-1][1]))
		    
	    def pop(self):
		return self.arr.pop()[0]
	    
	    # using name 'myMin' instead of 'min' because 'min' is a built-in method
	    def myMin(self):
		return self.arr[-1][1]
	    
	stack = MinStack()
	stack.push(5)
	print stack.myMin() # 5 expected
	stack.push(-1)
	stack.push(10)
	stack.push(9)

	print stack.myMin() # -1 expected
	print stack.pop() # 9 expected
要点：利用栈的“还原上一次记录”的能力, 随时维护当前最小元素
