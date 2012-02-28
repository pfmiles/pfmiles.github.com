---
layout: post
title: "跟踪递归函数路径的方法"
date: 2012-02-29 00:04
comments: true
categories: [programming, algorithm]
---

有些时候，需要追踪递归函数的路径；比如：递归地遍历一棵树，当找到符合要求的节点时，打印出从根节点到此节点的路径。

这种时候，若子节点不持有到父节点的指针，则需要对路径进行随时地记录，以便需要时能够获取；

* 一种方式就是建立“栈”结构，在每次递归时将当前节点压栈，并且记得在递归调用过后立刻***出栈***；
* 另一种方式就是直接利用函数递归调用的“调用栈”(也就是传参)了。

*例如有一个题目：输入一个整数和一棵二元树。
从树的根结点开始往下访问一直到叶结点所经过的所有结点形成一条路径。
打印出和与输入整数相等的所有路径。*

程序较简单，就是递归遍历，查找节点相加数值等于预期数值的情况：
	# define the data-structure first
	class Node(object):
	    def __init__(self, left=None, value=None, right=None):
		self.value = value
		self.left = left
		self.right = right
		
	root = Node(Node(Node(None, 4, None), 5, None), 10 , Node(None, 12, Node(None, 7, None)))

	found = []
	def findPath(tree, value, sm=0, paths=[]):
	    if tree.value + sm > value:
		return
	    elif tree.value + sm == value:
		paths.append(tree.value)
		found.append(paths[:])
		return
	    else:
		paths.append(tree.value)
		if tree.left != None:
		    findPath(tree.left, value, tree.value + sm, paths)
		if tree.right !=None:
		    findPath(tree.right, value, tree.value + sm, paths)
		paths.pop()
		    
	findPath(root, 22)
	print found

其中值得注意的是：
    paths.append(tree.value)
和：
    paths.pop()
这两行；
在递归方法前后进行对应地push和pop是正确记录路径必不可少的步骤。

其实，像上述这种每次递归调用之前都push，每次调用之后都pop的情况，其栈的层次、深度、变化规律跟函数调用栈是完全一致的，那么其实就完全可以考虑直接把父节点扔给下一层递归函数：
    findPath(tree.left, value, tree.value + sm, tree)
其实，findPath的函数签名现在就变成了：
    findPath(树, 预期值, 当前和, 父节点)
这样一来，其实路径上的每一步，都是记录在调用栈上的(就算子节点没有父节点的指针，函数参数也直接把父节点带过来了，也就是在调用栈中)，当程序想要输出路径时，也完全能够将路径还原。
