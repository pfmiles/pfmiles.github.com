---
layout: post
title: "dropincc.java API design"
date: 2012-03-18 16:29
comments: true
categories: [dropincc, design]
keywords: "dropincc, java, api, design"
description: "dropincc.java的API设计介绍"
---
[dropincc.java](https://github.com/pfmiles/dropincc.java)为了使得用户能够在java语言中直接定义新语言的词法、语法规则，API设计采用了串接与组合(cascading & composition)风格的[FluentInterface](http://martinfowler.com/bliki/FluentInterface.html)形式。

这种形式能够使得这一套API看起来更具有“业务语义”，而不需要像antlr等其它工具一样强迫用户去学习全新的另外一种含有业务语义的DSL。  
新语言的词法、语法规则被用户直接在java语句里书写、执行，其实得到的是dropincc.java这个CC工具的内部概念的AST。这个AST其实与“让用户学习一套DSL，然后编写，然后编译这些DSL得到的AST”是一样的。  
也就是说 —— dropincc.java里面很重要的一个理念就是：像书写字面量一样书写CC工具的AST(AST literal???)。
>注意：强调一下这里说的AST并非用户想要实现的新语言的AST，我指的是dropincc.java这个工具其内部表达词法、语法规则的内部形式。

经过一些初步的尝试，我认为在java中定义出这样一套API是完全可行的，其中一些值得列出的、具有代表性的点如下：
<!-- more -->
1. 词法规则强制用户限定在正则文法的表达能力之内(即使用正则表达式，例子略)
1. 上下文无关文法的表示主要是解决3种形式的表示：连接、选择、递归;
    1. 其中“连接”的表示, 若BNF为：`term ::= LEFTPAREN expr RIGHTPAREN`, 则右边的部分在java中表达为：`func(LEFTPAREN, expr, RIGHTPAREN)`;即：“以方法传参的自然按顺序排列表达‘连接’的意思”;而其中`expr`是一个non-terminal，也就是说这里要引用`expr`这个规则，这也就表达了“递归” —— 要递归地引用`expr`的话那么直接写在这里就好了；
    1. “选择”的表示，即一个产生式有好几个alternative的情况，若BNF是`term ::= DIGIT multail | LEFTPAREN expr RIGHTPAREN | DIGIT;`, 则java中表达为：`func(DIGIT, mulTail).alt(LEFTPAREN, expr, RIGHTPAREN).alt(DIGIT);`。
1. 上述3种基本元素的表达具备之后，其实已经足够表达出复杂的语法了，不过为了“更好的体验”，为了用得更方便，我认为还应该加入“子rule语法糖”。即让使用者能够以“加括号”的方式在产生式中直接定义子规则，而不必每次都单独写成一个产生式，这种做法应该能很受欢迎，比如：
    * 第一个产生式是：`MUL_DIV ::= MUL | DIV;`
    * 第二个产生式是：`mulTail ::= MUL_DIV term;`
    * 其实，若是有了前面说的语法糖，那么上述2个产生式就可以合并为：`mulTail ::= (MUL | DIV) term;`, 少了`MUL_DIV`这个没太多意义的产生式定义；
    * 在java中表达为：`func(MUL.or(DIV), term)`; 若子规则中想表达的不是“选择”关系，而是“连接”关系的话，也就是`mulTail ::= (MUL DIV) term;`的话，在java中表达为：`func(MUL.and(DIV), term)`; 其中`and`这个方法就是表达“连接”关系，只不过加了括号，成为了一个子规则而已。

OK, dropincc.java的核心FluentInterface形式的API应该就这些了，我相信这是一种很自然的方式，用户需要掌握的东西很少。  
当然，还有动作代码呢？待续...  
在定义这套富含语义的FluentInterface的时候，有总结什么样的模式与经验？待续...
