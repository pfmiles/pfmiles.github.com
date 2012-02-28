---
layout: post
title: "[面试中的算法]把二元查找树转变成排序的双向链表"
date: 2012-02-27 23:26
comments: true
categories: [algorithm, interview]
---
此问题由[v\_JULY\_v](http://blog.csdn.net/v_JULY_v/article/details/6050133)整理发布并发表于[blog](http://blog.csdn.net/v_JULY_v)上, 版权归原作者所有。

**问题描述：输入一棵二元查找树,将该二元查找树转换成一个排序的双向链表。要求不能创建任何新的结点,只调整指针的指向**

比如二叉搜索树：
           10
          /   \
        4      12
       / \    /  \
      3   5 11    13
输出应为： 3, 4, 5, 10, 11, 12, 13

### 分析：
处理树状结构很容易想到递归，而二叉搜索树其实恰好是已经排序好的一个结构，而要把它变成数组，只需“中序遍历”即可: 3,4,5,10,11,12,13
<!--more-->
### 程序：
    # define the data-structure first
    class Node(object):
        def __init__(self, left=None, value=None, right=None):
            self.value = value
            self.left = left
            self.right = right

    # construct the bin-search tree 
    root = Node(None, 10, None)

    left = Node(Node(None, 3, None), 4, Node(None, 5, None))

    right = Node(Node(None, 11, None), 12, Node(None, 13, None))

    root.left = left
    root.right = right

    def show(node):
        rst = [node.value]
        leftCur, rightCur = node, node
        while leftCur.left != None:
            rst.insert(0, leftCur.left.value)
            leftCur = leftCur.left
        while rightCur.right != None:
            rst.append(rightCur.right.value)
            rightCur = rightCur.right
        print rst

    # 3,4,5,10,11,12,13 expected

    # the recursive mid-order traverse function
    def traverse(node, leftOrRight = None):
        l, r = None, None
        if node.left != None:
            l = traverse(node.left, "left")
            node.left = l
            l.right = node
        if node.right != None:
            r = traverse(node.right, "right")
            node.right = r
            r.left = node
        if leftOrRight == "left" and r != None:
            return r
        elif leftOrRight == "right" and l != None:
            return l
        else:
            return node
    
    n = traverse(root)

    show(n)
结果输出： [3, 4, 5, 10, 11, 12, 13]

要点除了搞清楚二叉搜索树的特性以及中序遍历的规律外，要注意traverse方法中对于该返回'l' 还是 'r' 或者是node的判断
