---
layout: post
title: "使用Kleene闭包语法糖消除语法中的左递归"
date: 2012-03-28 11:18
comments: true
categories: [compiler, dropincc]
keywords: "kleene, 闭包, closure, 左递归, 消除, leftRecursion, removal, LL分析"
description: "使用Kleene闭包语法糖消除语法中的左递归"
---
一般来讲，如果使用的解析器生成器是基于LL分析法来解析文本的话(比如antlr)，我们就需要在编写语法规则的时候避免[左递归](http://en.wikipedia.org/wiki/Left_recursion)。  
虽然有一套成熟的、公式化的方法来去除左递归，但它会影响原有文法的结合顺序、优先级，或者多出一些本不需要的、额外的产生式。  
例如，有左递归语法规则：
    addition ::= addition '+' factor
               | factor
这个式子其实是想要表达一个“一连串+号组成的和”的表达式，例如`1+2+3+4`；  
假设目前的输入就是`1+2+3+4`, 那么超前查看带来的选择应该是`addition '+' factor`这个产生式；于是在parser的代码中立刻就又直接调用了`addition()`方法，但注意，整个递归过程没有消耗掉任何token...也就是说，当流程再次递归到`addition`函数里时，输入`1+2+3+4`并没有任何改变，这又会进入下一轮递归...很快就会overflow。  
这个问题，除了机械性地按照公式来消除左递归外，还可以改写语法规则，使其成为右递归语法：
    addition ::= factor '+' addition
               | factor
这样就不用stackOverFlow了，但是带来另一个问题：这使得我们的'+'加法成为了一个'右结合'的运算...  
也就是说，`1+2+3+4`的计算次序是这样的：`1+(2+(3+4))`  
还好这里是“加法”，整数集合和加法运算构成“阿贝尔群(可交换群)”，满足交换律，所以就算实现成右结合的也能蒙混过关...但减法呢？除法呢？  
<!-- more -->
难道要先生成一个AST，再写个visitor，把节点顺序给rewrite一下，然后再写个visitor求值？这么麻烦就为了实现一个简单的四则运算功能也太坑爹了。  

其实，我们在这里使用递归，只是为了表达元素的“重复”这种意思；的确，递归能表示“重复”的概念，但这很容易让人联想到“循环”，因为平时的编程中，我们都通常使用循环来表达“重复”这种东西的。  
更重要的是，循环与[尾递归](http://en.wikipedia.org/wiki/Tail_call)在逻辑上是等价的，这使得我们可以将上述左递归改写成某种表示“循环”的形式，在这里，[Kleene Closure(克林闭包)](http://en.wikipedia.org/wiki/Kleene_star)就正是一种表达“循环”的标记形式。  
熟悉正则表达式的我们都知道，符号`*`(kleene star)和符号`+`(kleene cross)分别用来表示前面临接元素的“0个或多个”和“一个或多个”，如果这种标记用在我们的语法规则中那就酷毙了：
    addition ::= (factor '+')* factor
表达的意义和"左递归版本"的语法规则是一样的，但是，这种记法的重大意义就是：在生成的parser程序中可以直接以一个`while`语句来解析重复的`(factor '+')`而不用再递归回`addition`中！这样就避免了parser中的左递归实现带来的问题！  
由于kleene闭包所能表达的所有语法规则形式都能被递归所表达，所以引入kleene闭包并没有从形式上增加语法解析器生成器的表达能力，所以说kleene闭包只是一个“语法糖”。  
但它的实现意义却是重大的：解决了“为了表达‘元素重复’”的这一类左递归问题。
当然，[dropincc.java](https://github.com/pfmiles/dropincc.java)也会支持kleene闭包的表达形式，大致的api形式如下：
    addition.fillGrammarRule(addend, CC.ks(ADD.or(SUB), addend));
`CC.ks`方法就是kleene star的意思，当然还有`CC.kc`，顾名思义。
