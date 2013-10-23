---
layout: post
title: "动态的Java - 无废话JavaCompilerAPI中文指南"
date: 2013-09-24 11:54
comments: true
categories: [java]
keywords: "java, dynamic, 动态, compilerAPI, treeAPI, jsr199, eval, annotation"
---
# JavaCompiler API
1.6之后JDK提供了一套compiler API，定义在[JSR199](http://jcp.org/en/jsr/detail?id=199)中, 提供在运行期动态编译java代码为字节码的功能  
简单说来，这一套API就好比是在java程序中模拟javac程序，将java源文件编译为class文件；其提供的默认实现也正是在文件系统上进行查找、编译工作的，用起来感觉与javac基本一致;  
不过，我们可以通过一些关键类的继承、方法重写和扩展，来达到一些特殊的目的，常见的就是“与文件系统解耦”(就是在内存或别的地方完成源文件的查找、读取和class编译)  
需要强调的是，compiler API的相关实现被放在tools.jar中，JDK默认会将tools.jar放入classpath而jre没有，因此如果发现compiler API相关类找不到，那么请检查一下tools.jar是否已经在classpath中；  
当然我指的是jdk1.6以上的版本提供的tools.jar包  
<!-- more -->
## 基本使用
### 一个基本的例子

    public static CompilationResult compile(String qualifiedName, String sourceCode,
                                            Iterable<? extends Processor> processors) {
        JavaStringSource source = new JavaStringSource(qualifiedName, sourceCode);
        List<JavaStringSource> ss = Arrays.asList(source);
        List<String> options = Arrays.asList("-classpath", HotCompileConstants.CLASSPATH);
        JavaCompiler compiler = ToolProvider.getSystemJavaCompiler();

        MemClsFileManager fileManager = null;
        Map<String, JavaMemCls> clses = new HashMap<String, JavaMemCls>();
        Map<String, JavaStringSource> srcs = new HashMap<String, JavaStringSource>();
        srcs.put(source.getClsName(), source);
        try {
            fileManager = new MemClsFileManager(compiler.getStandardFileManager(null, null, null), clses, srcs);
            DiagnosticCollector<JavaFileObject> diagnostics = new DiagnosticCollector<JavaFileObject>();
            StringWriter out = new StringWriter();
            CompilationTask task = compiler.getTask(out, fileManager, diagnostics, options, null, ss);
            if (processors != null) task.setProcessors(processors);
            boolean sucess = task.call();
            if (!sucess) {
                for (Diagnostic<? extends JavaFileObject> diagnostic : diagnostics.getDiagnostics()) {
                    out.append("Error on line " + diagnostic.getLineNumber() + " in " + diagnostic).append('\n');
                }
                return new CompilationResult(out.toString());
            }
        } finally {
            try {
                fileManager.close();
            } catch (Exception e) {
                throw new RuntimeException(e);
            }
        }
        // every parser class should be loaded by a new specific class loader
        HotCompileClassLoader loader = new HotCompileClassLoader(Util.getParentClsLoader(), clses);
        Class<?> cls = null;
        try {
            cls = loader.loadClass(qualifiedName);
        } catch (ClassNotFoundException e) {
            throw new RuntimeException(e);
        }
        return new CompilationResult(cls, loader);
    }

解释一下这段程序：  
这个static方法提供这样一种功能：输入希望的类名和String形式的java代码内容，动态编译并返回编译好的class对象; 其中`CompilationResult`只是一个简单的pojo封装，用于包装返回结果和可能的错误信息  
类名和源码首先被包装成一个`JavaStringSource`对象, 该对象继承自`javax.tools.SimpleJavaFileObject`类，是compiler API对一个“Java文件”(即源文件或class文件)的抽象；将源文件包装成这个类也就实现了“将java源文件放在内存中”的想法  
`JavaCompiler compiler = ToolProvider.getSystemJavaCompiler();`这是取得一个默认的JavaCompiler工具的实例  
由于我打算将编译好的class文件直接存放在内存中，因此我自定义了一个`MemClsFileManager`: 

    public class MemClsFileManager extends ForwardingJavaFileManager<StandardJavaFileManager> {

        private Map<String, JavaMemCls>       destFiles;
        private Map<String, JavaStringSource> srcFiles;
    
        protected MemClsFileManager(StandardJavaFileManager fileManager, Map<String, JavaMemCls> destFiles,
                                    Map<String, JavaStringSource> srcFiles){
            super(fileManager);
            this.destFiles = destFiles;
            this.srcFiles = srcFiles;
        }
    
        public JavaFileObject getJavaFileForOutput(Location location, String className, Kind kind, FileObject sibling)
                                                                                                                      throws IOException {
            if (!(Kind.CLASS.equals(kind) && StandardLocation.CLASS_OUTPUT.equals(location))) return super.getJavaFileForOutput(location,
                                                                                                                                className,
                                                                                                                                kind,
                                                                                                                                sibling);
            if (destFiles.containsKey(className)) {
                return destFiles.get(className);
            } else {
                JavaMemCls file = new JavaMemCls(className);
                this.destFiles.put(className, file);
                return file;
            }
        }
    
        public void close() throws IOException {
            super.close();
            this.destFiles = null;
        }
    
        public Iterable<JavaFileObject> list(Location location, String packageName, Set<Kind> kinds, boolean recurse)
                                                                                                                     throws IOException {
            List<JavaFileObject> ret = new ArrayList<JavaFileObject>();
            if ((StandardLocation.CLASS_OUTPUT.equals(location) || StandardLocation.CLASS_PATH.equals(location))
                && kinds.contains(Kind.CLASS)) {
                for (Map.Entry<String, JavaMemCls> e : destFiles.entrySet()) {
                    String pkgName = resolvePkgName(e.getKey());
                    if (recurse) {
                        if (pkgName.contains(packageName)) ret.add(e.getValue());
                    } else {
                        if (pkgName.equals(packageName)) ret.add(e.getValue());
                    }
                }
            } else if (StandardLocation.SOURCE_PATH.equals(location) && kinds.contains(Kind.SOURCE)) {
                for (Map.Entry<String, JavaStringSource> e : srcFiles.entrySet()) {
                    String pkgName = resolvePkgName(e.getKey());
                    if (recurse) {
                        if (pkgName.contains(packageName)) ret.add(e.getValue());
                    } else {
                        if (pkgName.equals(packageName)) ret.add(e.getValue());
                    }
                }
            }
            // 也包含super.list
            Iterable<JavaFileObject> superList = super.list(location, packageName, kinds, recurse);
            if (superList != null) for (JavaFileObject f : superList)
                ret.add(f);
            return ret;
        }
    
        private String resolvePkgName(String fullQualifiedClsName) {
            return fullQualifiedClsName.substring(0, fullQualifiedClsName.lastIndexOf('.'));
        }
    
        public String inferBinaryName(Location location, JavaFileObject file) {
            if (file instanceof JavaMemCls) {
                return ((JavaMemCls) file).getClsName();
            } else if (file instanceof JavaStringSource) {
                return ((JavaStringSource) file).getClsName();
            } else {
                return super.inferBinaryName(location, file);
            }
        }
    
    }

其中最主要的步骤就是重写了`getJavaFileForOutput`方法，使其使用内存中的map来作为生成文件(class文件)的输出位置  

    CompilationTask task = compiler.getTask(out, fileManager, diagnostics, options, null, ss);
    boolean sucess = task.call();
    
上面这两行是创建了一个编译task，并调用  
最后使用自定义ClassLoader来加载编译好的类并返回：  

    HotCompileClassLoader loader = new HotCompileClassLoader(Util.getParentClsLoader(), clses);
    Class<?> cls = null;
    try {
        cls = loader.loadClass(qualifiedName);
    } catch (ClassNotFoundException e) {
        throw new RuntimeException(e);
    }
    return new CompilationResult(cls, loader);
    
而该ClassLoader的实现关键在于“到内存中(即之前存放编译好的class的map中)加载字节码”：  

    public class HotCompileClassLoader extends ClassLoader {

        private Map<String, JavaMemCls> inMemCls;
    
        public HotCompileClassLoader(ClassLoader parent, Map<String, JavaMemCls> clses){
            super(parent);
            this.inMemCls = clses;
        }
    
        protected Class<?> findClass(String name) throws ClassNotFoundException {
            byte[] b = this.inMemCls.get(name).getClsBytes();
            return defineClass(name, b, 0, b.length);
        }
    
    }
    
之后只要调用方法`compile(className, source, null)`这样就算完成了一个基本的、不依赖实际的文件系统的动态编译过程  
### JavaFileManager的意义
一个广义的、管理“文件”资源的接口，并不一定指“操作系统的磁盘文件系统”  
其实`JavaFileManager`只是一个接口，只要行为正确，那么就无所谓“文件”到底以何种形式、实际被存放在哪里  
## StandardJavaFileManager的默认行为
这是基于磁盘文件的JavaFileManager实现, 所有的文件查找、新文件输出位置都在磁盘上完成；也就是说，如果直接使用默认的`StandardJavaFileManager`来做动态编译，那么得到的效果就跟命令行中直接使用javac编译差不多  
## 继承ForwardingJavaFileManager类，让compiler API脱离对文件系统的依赖
如果想要将编译好的class文件放在内存中而不是磁盘上，那么需要使用一个`ForwardingJavaFileManager`来包装默认的`StandardJavaFileManager`并重写`getJavaFileForOutput`方法，将其实现改为内存操作；这个实现可参考上面的`MemClsFileManager`类  
不过`ForwardingJavaFileManager`还有许多别的方法，没有文档说明动态编译过程中到底那些方法会被调用,原则上讲，所有方法都有可能被调用  
但具体哪些方法被调用了可以被实测出来，这样可以有选择性地重写其中一些方法  
比如`MemClsFileManager`中，重写了`inferBinaryName`, `list`, `close`, `getJavaFileForOutput`方法，因为这些方法都会被“class文件放在内存中”这一策略所影响，所以需要兼容  
# JavaCompiler Tree API
下面想介绍的其实是另一个更进一步的话题：如何在动态编译的过程中分析被编译的源代码  
因为几乎所有打算用到java动态编译的应用场景，都会想要对被编译的代码做一些检查、review，以防编译运行了让人出乎意料的代码从而对系统造成破坏  
那么这个事情，除了人工review之外，一些简单的验证，完全可以在编译期自动地做到；而JavaCompiler Tree API就是用来做这个静态编译期检查的  
它的基本思路很简单，就是hook进编译的过程中，在java源码被parse成[AST](http://en.wikipedia.org/wiki/Abstract_syntax_tree)的时候，使用visitor模式对该AST做遍历分析，以找出你需要定位的语法结构，这样来达到验证目的; 比如"如果发现assert语句就报错"或“不允许定义嵌套类”这样的检查都很容易做  
这之中的关键，当然是java的AST和其对应的visitor的实现了  
## Java的AST: 
<http://docs.oracle.com/javase/7/docs/jdk/api/javac/tree/com/sun/source/tree/package-summary.html>  
上述链接给出了java的AST类结构，所有语法元素的节点都有  
而对应的visitor接口`TreeVisitor<R,P>`与AST形成了标准的visitor模式  
`TreeVisitor<R,P>`的默认实现及其用途:  

1. SimpleTreeVisitor: 简单的、“消极”的visitor实现，当访问一个分支节点时不会默认“往下”继续遍历它的所有子节点, 若想要遍历所有子节点需要继承、编码实现
1. TreeScanner: 默认会遍历所有子节点的“积极”的visitor实现，还能让当前节点的遍历逻辑取到上一个被访问的邻接兄弟节点被访问的返回结果(换句话说能够拿到最近的、刚才被访问过的兄弟节点对象，或其它等效的自定义返回值)
1. TreePathScanner: 跟TreeScanner一样“积极”，且除了能拿到前一个邻接兄弟节点的访问返回结果外，还能拿到父亲节点(这样来追踪该节点的路径)  

有了上面这个visitor模式的脚手架，我们就能通过实现一个visitor来达到对java源码的分析了  
比如下面这个visitor, 它继承自`TreePathScanner`(`ExprCodeChecker`是我自定义的一个`TreePathScanner`的子类)：

    public class ForbiddenStructuresChecker extends ExprCodeChecker<Void, FbdStructContext> {

        private StringBuilder errMsg = new StringBuilder();
    
        public String getErrorMsg() {
            return this.errMsg.toString();
        }
    
        public FbdStructContext getInitParam() {
            return new FbdStructContext();
        }
    
        /**
         * 禁止定义内部类
         */
        public Void visitClass(ClassTree node, FbdStructContext p) {
            if (p.isInClass) {
                // 已经位于另一个外层class定义中了，直接报错返回，不再继续遍历子节点
                this.errMsg.append("Nested class is not allowed in api expressions. Position: " + resolveRowAndCol(node)).append('\n');
                return null;
            } else {
                boolean oldInClass = p.isInClass;
                p.isInClass = true;
                // 继续遍历子节点
                super.visitClass(node, p);
                p.isInClass = oldInClass;
                return null;
            }
    
        }
    
        /**
         * 禁止定义'assert'语句
         */
        public Void visitAssert(AssertTree node, FbdStructContext p) {
            this.errMsg.append("Assertions are not allowed in api expressions. Position: " + this.resolveRowAndCol(node)).append('\n');
            return null;
        }
    
        /**
         * 禁止定义goto(break or continue followed by a label)语句
         */
        public Void visitBreak(BreakTree node, FbdStructContext p) {
            if (node.getLabel() != null) {
                this.errMsg.append("'break' followed by a label is not allowed in api expressions. Position: "
                                           + this.resolveRowAndCol(node)).append('\n');
                return null;
            } else {
                return super.visitBreak(node, p);
            }
        }
    
        public Void visitContinue(ContinueTree node, FbdStructContext p) {
            if (node.getLabel() != null) {
                this.errMsg.append("'continue' followed by a label is not allowed in api expressions. Position: "
                                           + this.resolveRowAndCol(node)).append('\n');
                return null;
            } else {
                return super.visitContinue(node, p);
            }
        }
    
        // *************禁止定义goto end*************
    
        /**
         * 禁止定义死循环，for/while/do-while loop, 只限制常量类型的循环条件造成的明显死循环; 这种静态校验是不完善的，要做完善很复杂，没必要加大投入；若要做到更精确的控制应从动态期方案的方向考虑
         */
        public Void visitDoWhileLoop(DoWhileLoopTree node, FbdStructContext p) {
            boolean condTemp = p.isConstantTrueCondition;
            boolean isLoopExpTemp = p.isLoopConditionExpr;
    
            p.isLoopConditionExpr = true;
            node.getCondition().accept(this, p);
    
            if (p.isConstantTrueCondition) {
                // 死循环
                this.errMsg.append("Dead loop is not allowed in api expressions. Position: " + this.resolveRowAndCol(node)).append('\n');
            }
            p.isConstantTrueCondition = condTemp;
            p.isLoopConditionExpr = isLoopExpTemp;
    
            return super.visitDoWhileLoop(node, p);
        }
    
        public Void visitForLoop(ForLoopTree node, FbdStructContext p) {
            if (node.getCondition() == null) {
                // 无条件，相当于'true'
                this.errMsg.append("Dead loop is not allowed in api expressions. Position: " + this.resolveRowAndCol(node)).append('\n');
            } else {
                boolean condTemp = p.isConstantTrueCondition;
                boolean isLoopExpTemp = p.isLoopConditionExpr;
    
                p.isLoopConditionExpr = true;
                node.getCondition().accept(this, p);
    
                if (p.isConstantTrueCondition) {
                    // 死循环
                    this.errMsg.append("Dead loop is not allowed in api expressions. Position: "
                                               + this.resolveRowAndCol(node)).append('\n');
                }
                p.isConstantTrueCondition = condTemp;
                p.isLoopConditionExpr = isLoopExpTemp;
            }
            return super.visitForLoop(node, p);
        }
    
        public Void visitWhileLoop(WhileLoopTree node, FbdStructContext p) {
            boolean condTemp = p.isConstantTrueCondition;
            boolean isLoopExpTemp = p.isLoopConditionExpr;
    
            p.isLoopConditionExpr = true;
            node.getCondition().accept(this, p);
    
            if (p.isConstantTrueCondition) {
                // 死循环
                this.errMsg.append("Dead loop is not allowed in api expressions. Position: " + this.resolveRowAndCol(node)).append('\n');
            }
    
            p.isConstantTrueCondition = condTemp;
            p.isLoopConditionExpr = isLoopExpTemp;
    
            return super.visitWhileLoop(node, p);
        }
    
        // 处理循环条件, 需要关心结果为boolean值的表达式
        // 二元表达式
        public Void visitBinary(BinaryTree node, FbdStructContext p) {
            boolean isLoopCondTemp = p.isLoopConditionExpr;
    
            // 求左值
            p.isLoopConditionExpr = false;
            node.getLeftOperand().accept(this, p);
            p.isLoopConditionExpr = isLoopCondTemp;
            Object leftVal = p.expValue;
    
            // 求右值
            p.isLoopConditionExpr = false;
            node.getRightOperand().accept(this, p);
            p.isLoopConditionExpr = isLoopCondTemp;
            Object rightVal = p.expValue;
    
            // 求整体值
            Object val = null;
            if (leftVal != null && rightVal != null) switch (node.getKind()) {
                case MULTIPLY:
                    val = ((Number) leftVal).doubleValue() * ((Number) rightVal).doubleValue();
                    break;
                case DIVIDE:
                    val = ((Number) leftVal).doubleValue() / ((Number) rightVal).doubleValue();
                    break;
                case REMAINDER:
                    val = ((Number) leftVal).intValue() % ((Number) rightVal).intValue();
                    break;
                case PLUS:
                    if (leftVal instanceof Number && rightVal instanceof Number) {
                        val = ((Number) leftVal).doubleValue() + ((Number) rightVal).doubleValue();
                    } else {
                        val = String.valueOf(leftVal) + String.valueOf(rightVal);
                    }
                    break;
                case MINUS:
                    val = ((Number) leftVal).doubleValue() - ((Number) rightVal).doubleValue();
                    break;
                case LEFT_SHIFT:
                    val = ((Number) leftVal).longValue() << ((Number) rightVal).intValue();
                    break;
                case RIGHT_SHIFT:
                    val = ((Number) leftVal).longValue() >> ((Number) rightVal).intValue();
                    break;
                case UNSIGNED_RIGHT_SHIFT:
                    val = ((Number) leftVal).longValue() >>> ((Number) rightVal).intValue();
                    break;
                case LESS_THAN:
                    val = ((Number) leftVal).doubleValue() < ((Number) rightVal).doubleValue();
                    break;
                case GREATER_THAN:
                    val = ((Number) leftVal).doubleValue() > ((Number) rightVal).doubleValue();
                    break;
                case LESS_THAN_EQUAL:
                    val = ((Number) leftVal).doubleValue() <= ((Number) rightVal).doubleValue();
                    break;
                case GREATER_THAN_EQUAL:
                    val = ((Number) leftVal).doubleValue() >= ((Number) rightVal).doubleValue();
                    break;
                case EQUAL_TO:
                    val = leftVal == rightVal;
                    break;
                case NOT_EQUAL_TO:
                    val = leftVal != rightVal;
                    break;
                case AND:
                    if (leftVal instanceof Number) {
                        val = ((Number) leftVal).longValue() & ((Number) rightVal).longValue();
                    } else {
                        val = ((Boolean) leftVal) & ((Boolean) rightVal);
                    }
                    break;
                case XOR:
                    if (leftVal instanceof Number) {
                        val = ((Number) leftVal).longValue() ^ ((Number) rightVal).longValue();
                    } else {
                        val = ((Boolean) leftVal) ^ ((Boolean) rightVal);
                    }
                    break;
                case OR:
                    if (leftVal instanceof Number) {
                        val = ((Number) leftVal).longValue() | ((Number) rightVal).longValue();
                    } else {
                        val = ((Boolean) leftVal) | ((Boolean) rightVal);
                    }
                    break;
                case CONDITIONAL_AND:
                    val = ((Boolean) leftVal) && ((Boolean) rightVal);
                    break;
                case CONDITIONAL_OR:
                    val = ((Boolean) leftVal) || ((Boolean) rightVal);
                    break;
                default:
                    val = null;
            }
    
            if (p.isLoopConditionExpr) {
                if (val != null && val instanceof Boolean && (Boolean) val) p.isConstantTrueCondition = true;
            } else {
                p.expValue = val;
            }
            return null;
        }
    
        // 3元条件表达式
        public Void visitConditionalExpression(ConditionalExpressionTree node, FbdStructContext p) {
            boolean isLoopCondTemp = p.isLoopConditionExpr;
    
            p.isLoopConditionExpr = false;
            node.getCondition().accept(this, p);
            p.isLoopConditionExpr = isLoopCondTemp;
    
            Object val = null;
            if (p.expValue != null) {
                if ((Boolean) p.expValue) {
                    // 取true expr值
                    p.isLoopConditionExpr = false;
                    node.getTrueExpression().accept(this, p);
                    p.isLoopConditionExpr = isLoopCondTemp;
                    val = p.expValue;
                } else {
                    // 取false expr值
                    p.isLoopConditionExpr = false;
                    node.getFalseExpression().accept(this, p);
                    p.isLoopConditionExpr = isLoopCondTemp;
                    val = p.expValue;
                }
            }
    
            if (p.isLoopConditionExpr) {
                p.isConstantTrueCondition = val != null && val instanceof Boolean && (Boolean) val;
            } else {
                p.expValue = val;
            }
            return null;
        }
    
        // 常量
        public Void visitLiteral(LiteralTree node, FbdStructContext p) {
            if (p.isLoopConditionExpr) {
                p.isConstantTrueCondition = Boolean.TRUE.equals(node.getValue());
            } else {
                p.expValue = node.getValue();
            }
            return null;
        }
    
        // 括起来的表达式
        public Void visitParenthesized(ParenthesizedTree node, FbdStructContext p) {
            boolean isLoopCondTemp = p.isLoopConditionExpr;
            if (p.isLoopConditionExpr) {
                // 求值子表达式
                p.isLoopConditionExpr = false;
                node.getExpression().accept(this, p);
                p.isLoopConditionExpr = isLoopCondTemp;
                if (p.expValue != null && Boolean.TRUE.equals(p.expValue)) p.isConstantTrueCondition = true;
            } else {
                // 直接以子表达式的结果作为括号表达式的结果
                p.isLoopConditionExpr = false;
                node.getExpression().accept(this, p);
                p.isLoopConditionExpr = isLoopCondTemp;
            }
            return null;
        }
    
        // 类型转换表达式
        public Void visitTypeCast(TypeCastTree node, FbdStructContext p) {
            boolean isLoopCondTemp = p.isLoopConditionExpr;
    
            if (p.isLoopConditionExpr) {
                p.isLoopConditionExpr = false;
                node.getExpression().accept(this, p);
                p.isLoopConditionExpr = isLoopCondTemp;
    
                if (p.expValue != null
                    && ("Boolean".equals(node.getType().toString()) || "boolean".equals(node.getType().toString()))) p.isConstantTrueCondition = true;
            } else {
                p.isLoopConditionExpr = false;
                node.getExpression().accept(this, p);
                p.isLoopConditionExpr = isLoopCondTemp;
            }
            return null;
        }
    
        // 一元表达式
        public Void visitUnary(UnaryTree node, FbdStructContext p) {
            boolean isLoopCondTemp = p.isLoopConditionExpr;
    
            // 求子表达式值
            p.isLoopConditionExpr = false;
            node.getExpression().accept(this, p);
            p.isLoopConditionExpr = isLoopCondTemp;
    
            Object val = null;
            if (p.expValue != null) {
                switch (node.getKind()) {
                    case POSTFIX_INCREMENT:
                    case POSTFIX_DECREMENT:
                    case PREFIX_INCREMENT:
                    case PREFIX_DECREMENT:
                        val = null;
                        break;
                    case UNARY_PLUS:
                        val = p.expValue;
                        break;
                    case UNARY_MINUS:
                        val = -((Number) p.expValue).doubleValue();
                        break;
                    case BITWISE_COMPLEMENT:
                        val = ~((Number) p.expValue).longValue();
                        break;
                    case LOGICAL_COMPLEMENT:
                        val = !((Boolean) p.expValue);
                        break;
                    default:
                        val = null;
                }
            }
            if (p.isLoopConditionExpr) {
                if (val != null && val instanceof Boolean && (Boolean) val) p.isConstantTrueCondition = true;
            } else {
                p.expValue = val;
            }
            return null;
        }
        // *************禁止定义死循环end*************
    }
    
* 上述visitor通过对`visitClass`的处理，对“定义嵌套类”这种行为进行了报错
* 通过对`visitAssert`的处理，凡是遇到代码中出现`assert`语句的，均给出错误信息
* 通过对`visitBreak`和`visitContinue`的处理禁止了goto语句(即带label的break和continue语句)
* 通过对`visitDoWhileLoop`等循环语法结构的访问，以及部分表达式结构的访问(如`visitBinary`、`visitLiteral`)，禁止了如`while(true)`、`for(;;)`或`while(1<2)`等明显死循环

## 在什么阶段能将Java代码转化为AST从而被TreeVisitor分析？
接下来的问题就是：我们有了处理AST的visitor，那么到底要在什么时候运行它呢？  
答案就是使用jdk1.6的PluggableAnnotationProcessor机制, 在创建compilerTask时设置对应的Processor, 然后在该Processor中调用我们的visitor  
下面是我们的processor实现：  

    @SupportedSourceVersion(SourceVersion.RELEASE_6)
    @SupportedAnnotationTypes("*")
    public class ExprCodeCheckProcessor extends AbstractProcessor {
    
        // 工具实例类，用于将CompilerAPI, CompilerTreeAPI和AnnotationProcessing框架粘合起来
        private Trees                       trees;
        // 分析过程中可用的日志、信息打印工具
        private Messager                    messager;
    
        // 所有的CodeChecker
        private List<ExprCodeChecker<?, ?>> codeCheckers = new ArrayList<ExprCodeChecker<?, ?>>();
    
        // 搜集错误信息
        private StringBuilder               errMsg       = new StringBuilder();
    
        // 代码检查是否成功, 若false, 则'errMsg'里应该有具体错误信息
        private boolean                     success      = true;
    
        /**
         * ==============在这里列出所有的checker实例==============
         */
        public ExprCodeCheckProcessor(){
            // 检查 —— 禁止定义一些不必要的结构，如内部类
            this.codeCheckers.add(new ForbiddenStructuresChecker());
        }
    
        public synchronized void init(ProcessingEnvironment processingEnv) {
            super.init(processingEnv);
            this.trees = Trees.instance(processingEnv);
            this.messager = processingEnv.getMessager();
            // 为所有checker置入工具实例
            for (ExprCodeChecker<?, ?> c : this.codeCheckers) {
                c.setTrees(trees);
                c.setMessager(messager);
            }
        }
    
        public boolean process(Set<? extends TypeElement> annotations, RoundEnvironment env) {
            if (!env.processingOver()) for (Element e : env.getRootElements()) {
                for (ExprCodeChecker<?, ?> c : this.codeCheckers) {
                    c.check(this.trees.getPath(e));
                    if (!c.isSuccess()) {
                        this.success = false;
                        this.errMsg.append(c.getErrorMsg()).append('\n');
                    }
                }
            }
            /*
             * 这里若return true将阻止任何后续可能存在的Processor的运行，因此这里可以固定返回false
             */
            return false;
        }
    
        /**
         * 获取代码检查的错误信息
         * 
         * @return
         */
        public String getErrMsg() {
            return errMsg.toString();
        }
    
        /**
         * 指示代码检查过程是否成功，若为false，则可调用getErrMsg取得具体错误信息
         * 
         * @return
         */
        public boolean isSuccess() {
            return success;
        }
    }
    
这里面关键的就是实现`init`方法和`process`方法  
`init`方法初始化了`Trees`工具并将其设置到了所有的`ExprCodeChecker`(其实就是visitor)中; Trees是一个很重要的工具类实例，它能帮助我们获取AST结构对应的行号、列号等重要信息  
`process`方法则是真正对源码进行处理，在这里，真正调用了所有的`ExprCodeChecker`(也就是visitor)  
然后在使用Compiler API的时候，将processor设置到`CompilationTask`中即可：
    
    CompilationTask task = compiler.getTask(out, fileManager, diagnostics, options, null, ss);
    if (processors != null) task.setProcessors(processors);
    
顺便贴下`ExprCodeChecker`的代码如下，它就是一个对`TreePathScanner`的简单继承，封装了一些代码分析过程中的基本属性和常用方法, 所有的visitor只要继承自它就可以了:

    public abstract class ExprCodeChecker<R, P> extends TreePathScanner<R, P> {

        // 当前被扫描代码对应的节点转换工具类, 运行时由Processor负责置入
        protected Trees    trees;
        // 错误信息打印、处理流程控制工具, 运行时由Processor负责置入
        protected Messager messager;
    
        /**
         * 取得代码检查的错误信息, 返回结果为null或空字符串串则表示无错误, 否则认为有错误发生
         * 
         * @return
         */
        public abstract String getErrorMsg();
    
        /**
         * 取得初始参数
         * 
         * @return 用于遍历代码树的初始参数
         */
        protected abstract P getInitParam();
    
        /**
         * 代码检查是否成功
         * 
         * @return true - 成功，无问题； false - 失败，调用getErrorMsg可获取错误信息
         */
        final boolean isSuccess() {
            String err = this.getErrorMsg();
            return err == null || err.length() == 0;
        }
    
        /**
         * package访问权限，专门用于由Processor置入Trees工具实例
         * 
         * @param trees
         */
        final void setTrees(Trees trees) {
            this.trees = trees;
        }
    
        /**
         * package访问权限，专门用于由Processor置入Messager工具实例
         * 
         * @param messager
         */
        final void setMessager(Messager messager) {
            this.messager = messager;
        }
    
        /**
         * 开始遍历处理传入的代码树节点
         * 
         * @param path
         */
        final void check(TreePath path) {
            this.scan(path, getInitParam());
        }
    
        /**
         * 获取指定语法节点缩在源文件中的行号和列号信息, 用于错误信息输出
         * 
         * @param node
         * @return
         */
        protected final String resolveRowAndCol(Tree node) {
            CompilationUnitTree unit = this.getCurrentPath().getCompilationUnit();
            long pos = this.trees.getSourcePositions().getStartPosition(unit, node);
            LineMap m = unit.getLineMap();
            return "row: " + m.getLineNumber(pos) + ", col: " + m.getColumnNumber(pos);
        }
    
    }
    
其中`check`方法是遍历分析的起点，由processor调用  
`resolveRowAndCol`则是获取AST节点对应的行号、列号的方法，用于输出错误信息  
## 使用trees.getSourcePositions()获取节点的源码位置,以及LineMap获得指定位置的行号、列号
在processor的init方法中被置入的Trees工具实例，最大的用处就是获取对应AST节点的行号、列号，具体代码参见上述`resolveRowAndCol`方法  
## JavaCompiler Tree API的限制
有必要讨论下什么样的控制，适合用Tree API来做？  
总结起来应该是：静态的、简单的  
比如“不能定义内部类”、“不能写annotation”、“不能写assert语句”等  
随着需求的复杂度增高，使用Tree API的编码成本也会增高，毕竟使用visitor来分析复杂的AST模式并非十分容易的事情  
比如上面的例子，“限制死循环”这种需求；如果说非常简单的死循环，比如`while(true)`，这种是非常好做的  
但如果稍微复杂一点, 比如`while(1<2)`，那么这里势必会牵涉到一个"计算"过程，我们需要在分析过程中对`1<2`这个表达式做出计算，从而知晓该循环语句是否死循环;虽然人眼对`1<2`的结果一目了然，但这里靠程序来做的话，增加的复杂度还是相当可观的  
如果在继续复杂下去，可以想象，其开发成本会越来越高，且这个分析过程本身的“运行”成本也会越来越接近真正运行这段被分析的代码的成本，这个时候使用Tree API来做分析就不划算了    
所以说，考虑到成本因素，Tree API并不适合做太复杂的分析  
其次就是"静态的"代码，才能在编译期做分析，如果是这样的代码:`while(x<1)`，而`x`又是从方法参数中传入，那么`x`的值就完全在运行期才能确定，那么Tree API就无法判断该循环是否是死循环  

还有就是Tree API很容易让人联想到一个问题：可否在遍历AST的过程中改变AST的结构？  
这是个激动人心的话题，运行期改变源码的AST是一个想象空间很大的想法，就像在groovy和ruby中能办到的那样，这能成为一种强大的元编程机制  
不过，从java的Tree API规范上讲，是不能在遍历AST过程中修改AST的结构的，但目前有一个bug可以做到：[Erni08b.pdf](http://scg.unibe.ch/archive/projects/Erni08b.pdf)  
并且目前的[Project Lombok](http://projectlombok.org/)就是基于此bug实现; 就未来的版本发展来讲，利用此bug来实现功能是不可靠的, Project Lombok的开发者对此也表示担心  
不过这个bug也不失为一种必要时的选择，毕竟通过它能实现的功能很酷  