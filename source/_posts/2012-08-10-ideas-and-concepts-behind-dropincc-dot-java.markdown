---
layout: post
title: "Ideas and Concepts behind Dropincc.java"
date: 2012-08-10 00:40
comments: true
categories: [dropincc, DSL]
keywords: "design, idea, concept, dropincc"
---
### DSLs in java is hard.

DSLs are rarely discussed in java community. Or, at least comparing to other dynamic languages' world, DSLs in java is much harder to create.  

Why does this happen?  

According to [Martin Fowler's comments on DSLs](http://martinfowler.com/bliki/DomainSpecificLanguage.html), they are separated mainly to two kinds: internal and external DSLs.  

I believe that, the ability of how a specific general purposed programming language could be customized to an internal DSL relies on the language's [meta-programming](http://en.wikipedia.org/wiki/Metaprogramming) features.  

Some languages, especially dynamic typed ones, have ability to customize its syntactical appearance at runtime. By intercepting their class defining life-cycles, executing fail-over strategies when method or field missings, or doing presentation substitutions via sophisticated macro systems or even, directly change their parse tree at runtime. All those incredible features are the magic behind internal DSLs.  

But, we sadly found that java has none of those magic stuff to help building internal DSLs. There's little meta-programming features in java. Annotations? Perhaps one rare creature, but its not enough.  
Maybe the only way to create internal DSLs in java is to design a set of [Fluent Interface](http://www.martinfowler.com/bliki/FluentInterface.html) API. It's great, but you won't have much space on customizing the syntax of your DSL.  

Then, seems that the possible ways to create real-world, sophisticated DSLs in java are always turns out to be external-DSLs. But, this won't be easy, always, since the creation of computer world.  
<!-- more -->
Creating external DSLs means doing lexical & syntactical analyzing all by ourselves. There is a whole bunch of open sourced tools, say, [parser generators](http://en.wikipedia.org/wiki/Compiler-compiler), to help doing so, but there's too much unnecessary efforts must be made before you get your grammar run.  

Those traditional parser generators are great but in fact not suitable for DSL creating. Like lex&bison, javacc, jcup and antlr etc. They usually have their own grammar you must learn to define grammar rules, this is some what big learning cost.  
After you have done writing your grammar, they usually requires you to run a specific compiling tool in command-line to 'compile' the rules you just wrote. And finally gives you a couple of source files in your host language and telling you to place them into somewhere of your project directory. Then you could run your program. If errors occur, you would re-do those steps by frequently switching between command-line and your IDE.  

These steps are troublesome in practice. Especially for those first using a parser generator. I've seen one of my colleague struggled against antlr for one month, in order to build a SQL-like DSL. And much of his time was spent on getting familiar with antlr's grammar, its 'antlrWorks' IDE and some command-line tools. It was painful when recall on this month.  

So I think one must not have to carry so much burden when he just want to create a relatively simple DSL. And [dropincc.java](https://github.com/pfmiles/dropincc.java) is something I create to ease the building process of external DSL in java.  

### If all done in one place.
* I don't think additional grammar must be created to describe grammar rules, this could be done in internal DSL in java, a.k.a fluent interfaces. Creating a [EBNF](http://en.wikipedia.org/wiki/Extended_Backus%E2%80%93Naur_Form) like internal DSL in java is affordable.
* I don't think additional command-line tools or individual specific IDE is needed since we are adopting internal DSL to describe grammar rules. Just a commonly used java IDE is sufficient.
* Also, I don't think we must repeat the compile-copy, and run cycle when we trying and debugging our new creating language. Grammar may be run directly just after we defined it. Like we edit and test java program instantly in most modern IDEs.

And, all in all, in summary... dropincc.java ends up such a tool with those missions. I hope this small lib could help a lot in building external DSLs.  

For a starter dummy example, please refer to [this post](/blog/dropincc-dot-java-the-ultimate-tool-to-create-dsls-in-java/).
