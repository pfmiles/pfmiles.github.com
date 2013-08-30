---
layout: post
title: "Growing a Full-Featured Calculator in 15 Minutes Using Dropincc.java"
date: 2012-08-12 15:58
comments: true
categories: [dropincc, DSL]
keywords: "dropincc, java, calculator"
---
First, let's create a empty java project in eclipse IDE, download newest playable dropincc.java jar file from [here](https://github.com/pfmiles/dropincc.java/releases). Adding it to the build path of the project, like this(note: jdk1.6 or above required):  
![emptyProjWithJar](/images/dropincc/calc/emptyProjWithJar.png)  
And then, create a new java source file to build our grammar, for example, `Calculator.java`.  
We start by specifying the EBNF form of our expected calculator language. The calculator expression would do addition and subtraction operations and also multiplication and division operations which have a greater priority.  
Also, we expect the calculator could handle parentheses to do priority adjustment.  
We could write the EBNF grammar rules in a top-down pattern:  

    calc ::= expr $
    expr ::= addend (('+'|'-') addend)*
    
We specified that the overall `expression` consists one or more `addend` sub-expressions with `'+'` or `'-'` symbol delimited.  
And for every single `addend` expression:  

    addend ::= factor (('\*'|'/') factor)*
    
consists one or more `factor` separated by `'*'` or `'/'`.  
And finally, for any single `factor`, is either a single digit or a quoted expression:  

    factor ::= '\d+(\.\d+)?'
             | '(' expr ')'
     
Ok, we've done defining the EBNF of our calculator. This is what it look like all over:  

    calc ::= expr $
    expr ::= addend (('+'|'-') addend)*
    addend ::= factor (('\*'|'/') factor)*
    factor ::= '\d+(\.\d+)?'
             | '(' expr ')'
             
<!-- more -->
It's quite short because it utilized the kleene notation to represent 'zero or more'. And a shorter EBNF ends up a shorter dropincc.java program!  
Now let's 'translate' the EBNF into dropincc.java program. I say 'translate' here because it's a really straightforward step:  

    // some elements' pre-definitions in order to call the methods
    Lang calc = new Lang("Calc");
    TokenDef a = calc.newToken("\\+");
    TokenDef m = calc.newToken("\\*");
    Grule expr = calc.newGrule();
    Grule addend = calc.newGrule();
    Grule factor = calc.newGrule();

    // grammar definition section start
    calc.defineGrule(expr, CC.EOF);
    expr.define(addend, CC.ks(a.or("\\-"), addend));
    addend.define(factor, CC.ks(m.or("/"), factor));
    factor.define("\\d+(\\.\\d+)?")
    .alt("\\(", expr, "\\)");
    // grammar definition end
    
The 'grammar definition' part is as many lines as the EBNF. In fact it's a line-by-line translation of the EBNF(please look at the 'grammar definition' section carefully).  

Just a little more explanation:  

* The basic elements of grammar rule is `TokenDef`s and `Grule`s.(In fact `Grule` means 'grammar rule') `TokenDef`s are defined using java built-in regular expression currently.
* Rule production's match sequence is represented as comma separated elements list. Like `expr, CC.EOF` means `expr $`, `addend, CC.ks(a.or("\\-"), addend)` means `addend (('+'|'-') addend)*`.
* Note the `CC.EOF` constant, it's a predefined constant provided by dropincc.java to represent `EOF` token definition.
* And the `CC.ks` method call means all elements passed in as parameters are enclosed in a kleene star node(kleene star node means zero or more multiplication of its enclosing elements, the same as you learnt from using a regular expression). And `ks` in fact means 'kleene star'.
* Likewise, there are `CC.kc` method and `CC.op` method which means `kleene cross`(one or more) and `optional`(zero or one) respectively but not used in the above example, you can browse them from source code.
* Rule alternative productions are represented by an `alt` method call. Look at the `factor` rule definition, the `factor` rule has two productions. And, respectively, in java code, it has an additional `alt` call besides `define`.

'Comma separated match sequence', 'CC.EOF', 'CC.ks/kc/op' and 'alt', those are almost all you need to know to define a non-trivial DSL grammar in dropincc.java.  

Ok, after we have defined the grammar, it's time to add `Action`s to process the elements the parser matched while parsing. It's also achieved by a very intuitive way:  

* One `alternative production`(note: not one rule) could have only one action. That's, the `calc` rule could have only one action because it has only one alternative production. While `factor` rule could have two actions because it has two alternative productions.
* Let's add actions for the `factor` rule first:  

For the first alternative production:  

    factor.define("\\d+(\\.\\d+)?").action(new Action<String>(){
        public Double act(String matched) {
            return Double.parseDouble(matched);
        }})

The first alt of `factor` rule only matches a single string which have a 'digit' format. So the `act` method of the `Action` interface should be a single `String` object holding the matched digit string. And, also, the generic type argument of `Action` interface should be `String`, to provide a strong type safety for the actually matched element.  
We just parse the matched digit string to a number and returned it. So the return type of this action is `Double`.  

And the `Action` for the second alt:  

    .alt("\\(", expr, "\\)").action(new Action<Object[]>(){
        public Double act(Object[] ms) {
            return (Double) ms[1];
        }
    });

The second alt is an expression enclosed by parentheses. There are three element in the match sequence: `(`, enclosing `expr`, and `)`.  
The type argument of `Action` interface should be `Object[]` since the matched element will be passed in as an object array of length 3. With first element `(`, second element the returned value of the enclosing `expr` rule's action, and third element `)`.  
All we care about is the second element: the returned value of the enclosing `expr`'s action. So we just return `ms[1]` here, ignoring `(` and `)`. Note that we confidently converted the `ms[1]` to a `Double` object, that's because we knew the return value of `expr` rule's action is definitely a `Double` value.  

* Then, let's add action for the `addend` rule:  

It has only one production, so it could have only one corresponding action:

    addend.define(factor, CC.ks(m.or("/"), factor)).action(new Action<Object[]>() {
        public Double act(Object[] ms) {
            Double f0 = (Double) ms[0];
            Object[] others = (Object[]) ms[1];
            for (Object other : others) {
                Object[] opAndFactor = (Object[]) other;
                if ("*".equals(opAndFactor[0])) {
                    f0 *= (Double) opAndFactor[1];
                } else {
                    f0 /= (Double) opAndFactor[1];
                }
            }
            return f0;
        }
    });
    
The argument of `act` method is an object array of length 2. Its first element is the returned value from the matched `factor` rule's action,  
and the second are other operator & factor pairs.  
If there's only one factor matched here, the 'operator & factor' pairs would be an array of length 0, otherwise it contains all other matched operators and factors.  
You can see that the argument structure of method `act` is highly consistent to the defined match sequence of the corresponding alternative production.  

We wrote some logic to compute the total multiplication or division of all matched factors, and finally, returned the result as the result of this action.  

* Then, we add action for the `expr` rule. It's almost the same as the action of `addend` rule, but in an `add` or `subtract` manner when computing the matched elements:  

The `expr` rule also could have only one action:  

    expr.define(addend, CC.ks(a.or("\\-"), addend)).action(new Action<Object[]>() {
        public Double act(Object[] ms) {
            Double a0 = (Double) ms[0];
            Object[] others = (Object[]) ms[1];
            for (Object other : others) {
                Object[] opAndFactor = (Object[]) other;
                if ("+".equals(opAndFactor[0])) {
                    a0 += (Double) opAndFactor[1];
                } else {
                    a0 -= (Double) opAndFactor[1];
                }
            }
            return a0;
        }
    });
    
Finally, we add action for the `calc` rule. It's the starter rule. All computations should have been done at this point. So all we need to do in its action is return the final result, ignoring the `EOF`:  

    calc.defineGrule(expr, CC.EOF).action(new Action<Object[]>() {
        public Double act(Object[] ms) {
            return (Double) ms[0];
        }
    });
    
Now, we have done defining a calculator and added proper actions for it, we could `compile` it now:  

    Exe exe = calc.compile();
    
Then, enjoy with our new calculator:  

    // this should print the result: 3389.0
    System.out.println(exe.eval("1 +2+3+(4 +5*6*7*(64/8/2/(2/1 )/1)*8  +9  )+   10"));
    
It's quite simple isn't it? If you are already familiar with the API, 15 min is too much for defining such a calculator!  
And the follow is all the code in a gist, you could copy and try it in your own IDE:  
{% gist 3339604 %}
