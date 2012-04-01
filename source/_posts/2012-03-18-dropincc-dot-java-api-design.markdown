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
### 设计FluentInterface风格API过程中的经验总结
对于这个接口设计，我的经验就是：

1. 传入参数尽量“泛化”、“统一接口化”;这个“统一接口”最好不要包含任何方法(也就是说只是一个标记接口)，否则会给FluentInterface带来一些令人困惑的方法选项(当用户敲入'.'操作符的时候将会在IDE工具的方法提示里多出很多不必要的方法)。
1. 每个串接方法的返回结果可以随意“具体化”，因为这个返回结果的具体类名是不必展示给用户看的，所以你打算让用户下一步还能串接出什么样的方法，就尽管去返回一个具体的子类；但要注意返回结果不应该有太深的继承层次，否则“下一步串接”(当用户敲入'.'操作符的时候)将会在IDE工具里列出令人困惑的方法列表 —— 一般就一层继承关系 —— 继承自Object —— 一个接口实现 —— 就是1中所述的“泛化接口”，这样做方便你将返回值继续传入另一个方法中 —— 实现为所欲为的cascading & composition...

### 以闭包的形式嵌入动作代码

几乎所有的CC工具都提供一种“在match到特定的节点或树模式时允许执行一段用户自定义代码”的功能；这种自定义代码我把它叫做“动作代码”, “action”。无论是lex & yacc或是antlr都是这样。  
这个功能几乎是必须的，这提供了一种自由度相当高的形式让用户可以自定义AST节点，或是直接在parse的过程当中做一些自定义的计算，然后当整个parse结束时直接输出的就是整个的计算结果。  
动作代码一般来讲，都是用该CC工具所要生成的程序所使用的语言来写的；比如lex要生成C代码，他们的动作代码就是C来写的；antlr要生成java代码，那么它的动作代码自然就是java写的（当然antlr也能生成其它语言代码，相对应的，就使用那种语言来编写动作代码）。
dropincc.java既然设计目标是要嵌入到java语言中去使用，自然就是要用java语言来写动作代码了；  
但是，跟lex & yacc或者antlr不同的是，我并不打算让用户写一些残缺不全的动作代码放在某个约定的位置然后在生成目标程序的过程中将它们“编织”到目标程序中，这样的话就跟传统的CC工具没有任何区别了，没有什么实质上的改进；  
既然dropincc.java是“嵌入”到普通的java应用程序代码中间工作的，那么它完全有理由使用“闭包(closure)”的形式来组织“动作代码”！为什么用“闭包”？因为dropincc.java是“嵌入”到业务代码中使用的，用闭包来写动作代码可以capture一些业务代码上下文信息，这对构建DSL来讲是非常大的便利！  
比如，你的业务代码上下文里面有个`OrderService`对象，它负责一些"订单"业务逻辑，而当你编写一个闭包作为动作代码的时候，你完全可以"capture"上下文中的这个`OrderService`对象，从而使得你创造的语言在执行逻辑上与订单业务紧密、无缝地结合 —— 这是一种非常平滑的构建DSL的方式。  

### 最终的样子
> 注：此段代码只代表示意的风格、API情况，虽然能编译通过、运行，但不一定是行为完全正确四则运算表达式计算器

	// 3.define lexical rules
	Lang calculator = new Lang();
	Token DIGIT = calculator.addToken("\\d+");
	Token ADD = calculator.addToken("\\+");
	Token SUB = calculator.addToken("\\-");
	Token MUL = calculator.addToken("\\*");
	Token DIV = calculator.addToken("/");
	Token LEFTPAREN = calculator.addToken("\\(");
	Token RIGHTPAREN = calculator.addToken("\\)");
	// 2.define grammar rules and corresponding actions
	Grule expr = new Grule();
	Grule term = new Grule();
	Element mulTail = calculator.addGrammarRule(MUL.or(DIV), term).action(
			new Action() {
				public Object act(Object... params) {
					return params;
				}
			});
	term.fillGrammarRule(DIGIT, mulTail).action(new Action() {
		public Object act(Object... params) {
			int factor = Integer.parseInt((String) params[0]);
			Object[] mulTailReturn = (Object[]) params[1];
			String op = (String) mulTailReturn[0];
			int factor2 = (Integer) mulTailReturn[1];
			if ("*".equals(op)) {
				return factor * factor2;
			} else if ("/".equals(op)) {
				return factor / factor2;
			} else {
				throw new RuntimeException("Unsupported operator: " + op);
			}
		}
	}).alt(LEFTPAREN, expr, RIGHTPAREN).action(new Action() {
		public Object act(Object... params) {
			return params[1];
		}
	}).alt(DIGIT).action(new Action() {
		public Object act(Object... params) {
			return Integer.parseInt((String) params[0]);
		}
	});
	Element addendTail = calculator.addGrammarRule(ADD.or(SUB), term)
			.action(new Action() {
				public Object act(Object... params) {
					return params;
				}
			});
	expr.fillGrammarRule(term, addendTail, CC.EOF).action(new Action() {
		public Object act(Object... params) {
			int addend = (Integer) params[0];
			Object[] addendTailReturn = (Object[]) params[1];
			String op = (String) addendTailReturn[0];
			int addend2 = (Integer) addendTailReturn[1];
			if ("+".equals(op)) {
				return addend + addend2;
			} else if ("-".equals(op)) {
				return addend - addend2;
			} else {
				throw new RuntimeException("Unsupported operator: " + op);
			}
		}
	});
	// 1.compile it!
	calculator.compile();
	// 0.FIRE!!!
	System.out.println(calculator.exe("1+2+3+(4+5*6*7*(64/8/2/(2/1)/1)*8+9)+10"));

上面约60行java代码实现了一个相当复杂的多项式计算器语言：支持加减乘除四则运算以及括号调整优先级。而以往我们使用antlr来实现这个功能的话，最终生成、放到项目中工作的java代码会以千行计。  

* 这段代码是能够完全嵌入到任何普通的java业务系统程序代码当中的，是完全语法合格、能编译通过的，这是与传统CC工具的“代码残片”表示的动作代码所不同的地方
* 其中用到了上面介绍的“以正则语言来定义词法规则”、“以FluentInterfaces来定义语法规则”，以及“以闭包的形式来书写动作代码”（java中的闭包以“匿名内部类”的形式表示，道理一样）
* 其中，动作代码是一段段由`Action`接口实现的闭包（匿名内部类），里面的代码显然可以随意引用当前上下文中的所有变量、对象，这样实现了对上下文的capture，跟业务系统的无缝结合。
* `Action`闭包带有一个方法，参数是`Object... params`，这个参数里的值以位置顺序的方式对应于该action代码所附着的产生式里面的语法元素在parsing的时候所match到的值，听起来比较绕口，但实际上很简单，比如上面的 : 

带括号的表达式的action:
    .alt(LEFTPAREN, expr, RIGHTPAREN).action(new Action() {
        public Object act(Object... params) {
            return params[1];
        }
它的`params`的长度为3，分别是`["(", expr子规则匹配所返回的值, ")"]`，而这里我们的“业务(四则运算)”只需要关心`expr`的返回值，所以动作代码直接将`params[1]`（expr的返回值）返回给上一级匹配。  
就是这样，实际上你可以在动作代码里面做任何事情，只要符合java语法的、你能想到的。  

第一个版本的API设计基本就是这样了，往后可能还会加入一些语法糖，比如“Kleen闭包”来进一步方便语法规则的定义，不过会很慎重，因为dropincc.java是以小巧、易用为设计目标的，语法糖太多很可能会帮倒忙。
