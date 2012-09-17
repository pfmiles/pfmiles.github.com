---
layout: post
title: "Prime Numbers Less Than N"
date: 2012-09-03 13:57
comments: true
categories: 
---
* Sieve方法: http://en.wikipedia.org/wiki/Sieve_of_Eratosthenes
* 优化1：初始化list时，预先排除2,3,5,7的倍数, 得到一个初始化list
* 优化2：在筛除合数时，不必一个一个挨着筛，只需在列表中剔除： p * x 即可(x是从p开始，初始list中的数字), 直到p*p > 初始化list中的最大数

资料：
用作测试，素数个数对照表： http://primes.utm.edu/howmany.shtml

扩展：
也许下一篇文章介绍一下“素数测试”？ http://en.wikipedia.org/wiki/Primality_tests

TODO 后面再完成本文，反正有版本管理～
