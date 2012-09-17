---
layout: post
title: "用于代码生成的模板引擎应该是什么样子？"
date: 2012-09-13 15:27
comments: true
categories: [compiler, dropincc]
keywords: "代码生成, 模板引擎, codeGeneration, templateEngine"
---
以前在做一个涉及代码生成的项目的时候，使用了velocity模板引擎来做这个代码生成；  
而事后发现，velocity有大部分的语法是我在代码生成过程中没有用到的；  
且velocity的引入，也导致引入了一些看起来很多余的jar包；  
后来我看了看`StringTemplate`这个东西，发现它不仅包含velocity几乎全部的语法，还有模板继承机制！这岂不是更复杂？虽然标榜为“为了代码生成”的模板引擎...但鉴于`velocity`的复杂印象，我并没有使用它.    
再后来，我为了包管理上的简洁(对于某些项目我承认我有洁癖)，我直接使用了java标准库里的`java.text.MessageFormat`来做代码生成；这也确实可行！不过，缺点也是很快就能遇到的：  

1. 大括号`{`和`}`在`MessageFormat`中是关键字！所以输出时必须转义；这两个东西在目标代码中(比如java)很常见，遇到一个就要转义一个是很不爽的事情;  
1. 除了字符串替换之外，没有任何分支逻辑判断、循环结构支持，这使得所有的分支逻辑和循环结构都要写在调用这些`MessageFormat`的java代码里；虽然很多时候确实可以这么做并且很合适，但是还是有一些情况，如果能写在模板里会明显比写在java代码里好;
1. `MessageFormat`的placeholder是数字，渲染时是根据传入参数的位置来做值替换，这很不直观，更好的用户体验应该是依据名字来替换，就像所有的、普遍意义上人们所见过的模板引擎那样;

总结性地思考一下：我在做代码生成的时候，需要的模板引擎大概应该是：  

* 简单，这意味着只需要拥有简单的语法以及简单的jar包依赖(或者不需要jar包依赖)
* 高效，最好直接编译成字节码执行，而非像velocity那样每次渲染都遍历AST

而重点就在于第一点：简单，这需要我好好总结一份“用作代码生成的模板引擎所必须的语法特性”列表, 并且除此以外，不能有更多其它的东西;  
至于高效，只要确保最终被反复执行的逻辑是一个编译好的java类就行了，而非遍历AST，这很容易办到;  
而且..正好之前开发的[dropincc.java](https://github.com/pfmiles/dropincc.java)需要为用户提供一个模板引擎，因为语法解析与代码生成在很多任务中是一对好基友，常常是伴随着出现；那么在dropincc.java中内建一个针对代码生成的模板引擎也就是非常有必要的了；它之于dropincc.java，就好比StringTemplate之于antlr，都是为了方便用户做代码生成创造的;  
并且，这个模板引擎应该随着dropincc.java一起发布，不需要引入新的jar包依赖;  
计划中，这个模板引擎的基本特性：  
<!-- more -->

1. 编译为java字节码，常驻内存
1. 语法简单，只支持`if-else`语句和`foreach`语句, `foreach`语句支持数组和`Collection`的遍历, 也支持对"range常量"的遍历
1. 兼容velocity语法的一个子集, 但语义不一样(即只实现其`if-else`和`foreach`语法，一方面这是为了利用已有的、强大的velocity语法高亮插件)
1. 任何元素的输出结果总是与`String.valueOf(obj)`的结果一致
1. 逻辑判断只支持`==`、`!`, 支持`&&`或`||`和括号优先级，因为我认为，用于代码生成的模板引擎的语法应该全是“开关”形式的，即“已经是判断的结果了”，非`true`即`false`(顶多再加上`==`比较)，不应该有过多的动态的复杂逻辑判断，这些判断其实应该放到调用这个模板引擎的java代码里, 而在模板里，只应该存在对这些bool值的与、或、非的各种组合判断
1. `==`操作符的判断结果与java中`==` + `equals`函数的判断逻辑一致
1. 模板context支持map和javabean两种形式, 可使用`.`操作读取bean或map中的属性值，但不支持方法调用
1. 没有赋值语句，context中的所有东西，要么是要输出的元素，要么是作为输出控制的bool开关，它们在渲染过程中是不可变的
1. 模板引擎接口简单、清晰，只有4个static方法：
    * `public static String merge(String filePath, Object context)`
    * `public static byte[] merge(String filePath, Object context)`
    * `public static String merge(String filePath, Object context, String encoding)`
    * `public static byte[] merge(String filePath, Object context, String encoding)`
    * 其中，`filePath`是模板路径; `context`可是javabean或map，为模板上下文，包含要输出的元素和控制条件; `encoding`是指定模板所使用的编码格式; 返回`String`的方法即返回渲染后的字符串结果；返回`byte[]`的接口则是直接返回带编码格式的、`byte`的数组，这个结果可直接用作写入外部存储或流，而无需再应用编码格式;

而基本上就是这些，不需要太多东西，可能第一眼会觉得“功能是不是太少”？但是，我已经见过了许多过重的逻辑被放在模板语法里的情况，与其增加这种不必要的、很可能是错误的使用方式(因为这样做确实不好维护)，还不如一开始就不提供了；  
并且，代码生成所需要的模板引擎，相比起做web页面所需要的模板引擎来讲，一定要简单许多；因为大部分的逻辑是应该放在java代码中(比如负责翻译代码的visitor)而不是模板里。  

ok, 下面需要给出这个模板引擎的[EBNF](http://en.wikipedia.org/wiki/Extended_Backus%E2%80%93Naur_Form),这是最重要的准绳，几乎框定了绝大部分东西：  

    template ::= content $
    content ::= renderable*
    renderable ::= PLAIN_TEXT
              | '#if' '(' boolExpr ')' renderable ('#elseif' '(' boolExpr ')' renderable)* ('#else' renderable)? '#end'
              | '#foreach' '(' REF 'in' (REF|RANGE) ')' renderable '#end'
              | REF
    boolExpr ::= andExpr ('||' andExpr)*
    andExpr ::= term ('&&' term)*
    term ::= '!'* REF
           | value '==' value
           | '!'* '(' boolExpr ')'
    value ::= '-'?[0-9]+('.'[0-9]+)?
            | '\'' .* '\''
            
语法就是这样，应该已经足够;上述规则中，全大写字母的元素是terminal，一部分terminal的定义未在上面给出，补充在下面：  
    
    REF ::= '${' ID('.'ID)* '}'
    ID ::= [a-zA-Z_][a-zA-Z0-9_]*
    RANGE ::= '[' [0-9]+ '..' [0-9]+ ']'

总结起来讲，这个模板语法支持`if`和`foreach`语句，以及引用值的输出，其余的元素一律被看作`PLAIN_TEXT`直接输出;  
在`if`语句中，逻辑表达式支持`==`比较，以及“非”逻辑`!`;支持添加圆括号调整运算优先级;  
`foreach`语句中，可以支持对引用(必须是`Collection`或`Array`)或`RANGE`表达式的遍历; `RANGE`表达式即生成一个数字序列的表达式，步长为1，如`[1..10]`表示一个由1到10的数字序列;  

至于具体的实现方式，仍然采用与dropincc.java一致的jdk1.6 compiler API的动态编译的形式，编译为字节码运行。  
这个模板引擎将作为dropincc.java内部自己做一些代码生成，也同时提供给用户做代码生成工作，这样既能避免过多jar包的引入，又能施行一些最佳实践("最佳"的意思是我这里定义的，在我看来是在“什么逻辑放在模板中”和“什么逻辑放在java代码中”找到一个平衡点，当然不同人可以有不同理解)。
