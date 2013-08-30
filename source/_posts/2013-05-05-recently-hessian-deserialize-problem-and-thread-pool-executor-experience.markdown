---
layout: post
title: "近期hessian反序列化问题总结与ThreadPoolExecutor使用心得"
date: 2013-05-05 14:53
comments: true
categories: [troubleShooting, java]
keywords: "hessian, deserialize, ThreadPoolExecutor, RejectedExecutionHandler"
---
最近工作中遇到一个诡异的问题：别人远程调用我们的系统暴露的服务，同步调用，底层使用hessian协议做序列化；  
调用方系统报空指针，反序列化失败：  
<!-- more -->

    2013-04-18 16:52:10,308 [AvatarRuleChargeService.java:74] [com.alibaba.itbu.billing.biz.adaptor.avatar.AvatarRuleChargeService] ERROR com.alibaba.itbu.billing.biz.adaptor.crm.ChargeProxy :: avatar charge sys error
    com.alibaba.dubbo.rpc.RpcException: Failed to invoke the method match in the service com.alibaba.china.ruleservice.RuleService. Tried 3 times of the providers [172.22.6.83:20980, 172.22.6.80:20980, 172.22.9.76:20980] (3/3) from the registry dubbo-reg1.hst.xyi.cn.alidc.net:9090 on the consumer 172.30.118.26 using the dubbo version 2.4.9. Last error is: Failed to invoke remote method: match, provider: dubbo://172.22.6.83:20980/com.alibaba.china.ruleservice.RuleService?anyhost=true&application=billing&check=false&default.reference.filter=dragoon&dubbo=2.4.9&interface=com.alibaba.china.ruleservice.RuleService&methods=match&pid=18616&revision=1.0-SNAPSHOT&side=consumer&timeout=5000&timestamp=1366275108588&version=1.0.0, cause: com.alibaba.com.caucho.hessian.io.HessianProtocolException: 'com.alibaba.china.ruleservice.RuleServiceImpl$DynamicPluginInvocationMatchedResult' could not be instantiated
    com.alibaba.com.caucho.hessian.io.HessianProtocolException: 'com.alibaba.china.ruleservice.RuleServiceImpl$DynamicPluginInvocationMatchedResult' could not be instantiated
        at com.alibaba.com.caucho.hessian.io.JavaDeserializer.instantiate(JavaDeserializer.java:275)
        at com.alibaba.com.caucho.hessian.io.JavaDeserializer.readObject(JavaDeserializer.java:155)
        at com.alibaba.com.caucho.hessian.io.SerializerFactory.readObject(SerializerFactory.java:396)
        at com.alibaba.com.caucho.hessian.io.Hessian2Input.readObjectInstance(Hessian2Input.java:2070)
        at com.alibaba.com.caucho.hessian.io.Hessian2Input.readObject(Hessian2Input.java:2005)
        at com.alibaba.com.caucho.hessian.io.Hessian2Input.readObject(Hessian2Input.java:1990)
        at com.alibaba.com.caucho.hessian.io.Hessian2Input.readObject(Hessian2Input.java:1538)
        at com.alibaba.dubbo.common.serialize.support.hessian.Hessian2ObjectInput.readObject(Hessian2ObjectInput.java:94)
        at com.alibaba.dubbo.common.serialize.support.hessian.Hessian2ObjectInput.readObject(Hessian2ObjectInput.java:99)
        at com.alibaba.dubbo.rpc.protocol.dubbo.DecodeableRpcResult.decode(DecodeableRpcResult.java:83)
        at com.alibaba.dubbo.rpc.protocol.dubbo.DecodeableRpcResult.decode(DecodeableRpcResult.java:109)
        at com.alibaba.dubbo.rpc.protocol.dubbo.DubboCodec.decodeBody(DubboCodec.java:97)
        at com.alibaba.dubbo.remoting.exchange.codec.ExchangeCodec.decode(ExchangeCodec.java:128)
        at com.alibaba.dubbo.remoting.exchange.codec.ExchangeCodec.decode(ExchangeCodec.java:87)
        at com.alibaba.dubbo.rpc.protocol.dubbo.DubboCountCodec.decode(DubboCountCodec.java:49)
        at com.alibaba.dubbo.remoting.transport.netty.NettyCodecAdapter$InternalDecoder.messageReceived(NettyCodecAdapter.java:135)
        at org.jboss.netty.channel.SimpleChannelUpstreamHandler.handleUpstream(SimpleChannelUpstreamHandler.java:80)
        at org.jboss.netty.channel.DefaultChannelPipeline.sendUpstream(DefaultChannelPipeline.java:564)
        at org.jboss.netty.channel.DefaultChannelPipeline.sendUpstream(DefaultChannelPipeline.java:559)
        at org.jboss.netty.channel.Channels.fireMessageReceived(Channels.java:274)
        at org.jboss.netty.channel.Channels.fireMessageReceived(Channels.java:261)
        at org.jboss.netty.channel.socket.nio.NioWorker.read(NioWorker.java:349)
        at org.jboss.netty.channel.socket.nio.NioWorker.processSelectedKeys(NioWorker.java:280)
        at org.jboss.netty.channel.socket.nio.NioWorker.run(NioWorker.java:200)
        at org.jboss.netty.util.ThreadRenamingRunnable.run(ThreadRenamingRunnable.java:108)
        at org.jboss.netty.util.internal.DeadLockProofWorker$1.run(DeadLockProofWorker.java:44)
        at java.util.concurrent.ThreadPoolExecutor$Worker.runTask(ThreadPoolExecutor.java:886)
        at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:908)
        at java.lang.Thread.run(Thread.java:662)
    Caused by: java.lang.reflect.InvocationTargetException
        at sun.reflect.NativeConstructorAccessorImpl.newInstance0(Native Method)
        at sun.reflect.NativeConstructorAccessorImpl.newInstance(NativeConstructorAccessorImpl.java:39)
        at sun.reflect.DelegatingConstructorAccessorImpl.newInstance(DelegatingConstructorAccessorImpl.java:27)
        at java.lang.reflect.Constructor.newInstance(Constructor.java:513)
        at com.alibaba.com.caucho.hessian.io.JavaDeserializer.instantiate(JavaDeserializer.java:271)
        ... 28 more
    Caused by: java.lang.NullPointerException
        at com.alibaba.china.ruleservice.RuleServiceImpl$DynamicPluginInvocationMatchedResult.<init>(RuleServiceImpl.java:163)
        ... 33 more
    
        at com.alibaba.dubbo.rpc.cluster.support.FailoverClusterInvoker.doInvoke(FailoverClusterInvoker.java:101)
        at com.alibaba.dubbo.rpc.cluster.support.AbstractClusterInvoker.invoke(AbstractClusterInvoker.java:226)
        at com.alibaba.dubbo.rpc.cluster.support.wrapper.MockClusterInvoker.invoke(MockClusterInvoker.java:72)
        at com.alibaba.dubbo.rpc.proxy.InvokerInvocationHandler.invoke(InvokerInvocationHandler.java:52)
        at com.alibaba.dubbo.common.bytecode.proxy1.match(proxy1.java)
        at sun.reflect.NativeMethodAccessorImpl.invoke0(Native Method)
        at sun.reflect.NativeMethodAccessorImpl.invoke(NativeMethodAccessorImpl.java:39)
        at sun.reflect.DelegatingMethodAccessorImpl.invoke(DelegatingMethodAccessorImpl.java:25)
        at java.lang.reflect.Method.invoke(Method.java:597)
        at org.springframework.aop.support.AopUtils.invokeJoinpointUsingReflection(AopUtils.java:307)
        at org.springframework.aop.framework.ReflectiveMethodInvocation.invokeJoinpoint(ReflectiveMethodInvocation.java:182)
        at org.springframework.aop.framework.ReflectiveMethodInvocation.proceed(ReflectiveMethodInvocation.java:149)
        at org.springframework.aop.aspectj.MethodInvocationProceedingJoinPoint.proceed(MethodInvocationProceedingJoinPoint.java:88)
        at com.alibaba.itbu.billing.framework.aop.OpenApiLogAspect.logExecuteTime(OpenApiLogAspect.java:38)
        at sun.reflect.GeneratedMethodAccessor199.invoke(Unknown Source)
        at sun.reflect.DelegatingMethodAccessorImpl.invoke(DelegatingMethodAccessorImpl.java:25)
        at java.lang.reflect.Method.invoke(Method.java:597)
        at org.springframework.aop.aspectj.AbstractAspectJAdvice.invokeAdviceMethodWithGivenArgs(AbstractAspectJAdvice.java:627)
        at org.springframework.aop.aspectj.AbstractAspectJAdvice.invokeAdviceMethod(AbstractAspectJAdvice.java:616)
        at org.springframework.aop.aspectj.AspectJAroundAdvice.invoke(AspectJAroundAdvice.java:64)
        at org.springframework.aop.framework.ReflectiveMethodInvocation.proceed(ReflectiveMethodInvocation.java:171)
        at org.springframework.aop.interceptor.ExposeInvocationInterceptor.invoke(ExposeInvocationInterceptor.java:89)
        at org.springframework.aop.framework.ReflectiveMethodInvocation.proceed(ReflectiveMethodInvocation.java:171)
        at org.springframework.aop.framework.JdkDynamicAopProxy.invoke(JdkDynamicAopProxy.java:204)
        at $Proxy107.match(Unknown Source)
        at com.alibaba.itbu.billing.biz.adaptor.avatar.AvatarRuleChargeService.chargeByFactor(AvatarRuleChargeService.java:72)
        at com.alibaba.itbu.billing.biz.charge.times.RuleChargeByTimesProcessor.getChargeResult(RuleChargeByTimesProcessor.java:62)
        at com.alibaba.itbu.billing.biz.charge.times.ChargeByTimesProcessor.charge(ChargeByTimesProcessor.java:117)
        at com.alibaba.itbu.billing.biz.task.ChargeByIncInstantTimesTask.charge(ChargeByIncInstantTimesTask.java:174)
        at com.alibaba.itbu.billing.biz.task.ChargeByIncInstantTimesTask$1.run(ChargeByIncInstantTimesTask.java:109)
        at java.util.concurrent.ThreadPoolExecutor$Worker.runTask(ThreadPoolExecutor.java:886)
        at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:908)
        at java.lang.Thread.run(Thread.java:662)
    Caused by: com.alibaba.dubbo.remoting.RemotingException: com.alibaba.com.caucho.hessian.io.HessianProtocolException: 'com.alibaba.china.ruleservice.RuleServiceImpl$DynamicPluginInvocationMatchedResult' could not be instantiated
    com.alibaba.com.caucho.hessian.io.HessianProtocolException: 'com.alibaba.china.ruleservice.RuleServiceImpl$DynamicPluginInvocationMatchedResult' could not be instantiated
        at com.alibaba.com.caucho.hessian.io.JavaDeserializer.instantiate(JavaDeserializer.java:275)
        at com.alibaba.com.caucho.hessian.io.JavaDeserializer.readObject(JavaDeserializer.java:155)
        at com.alibaba.com.caucho.hessian.io.SerializerFactory.readObject(SerializerFactory.java:396)
        at com.alibaba.com.caucho.hessian.io.Hessian2Input.readObjectInstance(Hessian2Input.java:2070)
        at com.alibaba.com.caucho.hessian.io.Hessian2Input.readObject(Hessian2Input.java:2005)
        at com.alibaba.com.caucho.hessian.io.Hessian2Input.readObject(Hessian2Input.java:1990)
        at com.alibaba.com.caucho.hessian.io.Hessian2Input.readObject(Hessian2Input.java:1538)
        at com.alibaba.dubbo.common.serialize.support.hessian.Hessian2ObjectInput.readObject(Hessian2ObjectInput.java:94)
        at com.alibaba.dubbo.common.serialize.support.hessian.Hessian2ObjectInput.readObject(Hessian2ObjectInput.java:99)
        at com.alibaba.dubbo.rpc.protocol.dubbo.DecodeableRpcResult.decode(DecodeableRpcResult.java:83)
        at com.alibaba.dubbo.rpc.protocol.dubbo.DecodeableRpcResult.decode(DecodeableRpcResult.java:109)
        at com.alibaba.dubbo.rpc.protocol.dubbo.DubboCodec.decodeBody(DubboCodec.java:97)
        at com.alibaba.dubbo.remoting.exchange.codec.ExchangeCodec.decode(ExchangeCodec.java:128)
        at com.alibaba.dubbo.remoting.exchange.codec.ExchangeCodec.decode(ExchangeCodec.java:87)
        at com.alibaba.dubbo.rpc.protocol.dubbo.DubboCountCodec.decode(DubboCountCodec.java:49)
        at com.alibaba.dubbo.remoting.transport.netty.NettyCodecAdapter$InternalDecoder.messageReceived(NettyCodecAdapter.java:135)
        at org.jboss.netty.channel.SimpleChannelUpstreamHandler.handleUpstream(SimpleChannelUpstreamHandler.java:80)
        at org.jboss.netty.channel.DefaultChannelPipeline.sendUpstream(DefaultChannelPipeline.java:564)
        at org.jboss.netty.channel.DefaultChannelPipeline.sendUpstream(DefaultChannelPipeline.java:559)
        at org.jboss.netty.channel.Channels.fireMessageReceived(Channels.java:274)
        at org.jboss.netty.channel.Channels.fireMessageReceived(Channels.java:261)
        at org.jboss.netty.channel.socket.nio.NioWorker.read(NioWorker.java:349)
        at org.jboss.netty.channel.socket.nio.NioWorker.processSelectedKeys(NioWorker.java:280)
        at org.jboss.netty.channel.socket.nio.NioWorker.run(NioWorker.java:200)
        at org.jboss.netty.util.ThreadRenamingRunnable.run(ThreadRenamingRunnable.java:108)
        at org.jboss.netty.util.internal.DeadLockProofWorker$1.run(DeadLockProofWorker.java:44)
        at java.util.concurrent.ThreadPoolExecutor$Worker.runTask(ThreadPoolExecutor.java:886)
        at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:908)
        at java.lang.Thread.run(Thread.java:662)
    Caused by: java.lang.reflect.InvocationTargetException
        at sun.reflect.NativeConstructorAccessorImpl.newInstance0(Native Method)
        at sun.reflect.NativeConstructorAccessorImpl.newInstance(NativeConstructorAccessorImpl.java:39)
        at sun.reflect.DelegatingConstructorAccessorImpl.newInstance(DelegatingConstructorAccessorImpl.java:27)
        at java.lang.reflect.Constructor.newInstance(Constructor.java:513)
        at com.alibaba.com.caucho.hessian.io.JavaDeserializer.instantiate(JavaDeserializer.java:271)
        ... 28 more
    Caused by: java.lang.NullPointerException
        at com.alibaba.china.ruleservice.RuleServiceImpl$DynamicPluginInvocationMatchedResult.<init>(RuleServiceImpl.java:163)
        ... 33 more
    
        at com.alibaba.dubbo.remoting.exchange.support.DefaultFuture.returnFromResponse(DefaultFuture.java:190)
        at com.alibaba.dubbo.remoting.exchange.support.DefaultFuture.get(DefaultFuture.java:110)
        at com.alibaba.dubbo.remoting.exchange.support.DefaultFuture.get(DefaultFuture.java:84)
        at com.alibaba.dubbo.rpc.protocol.dubbo.DubboInvoker.doInvoke(DubboInvoker.java:96)
        at com.alibaba.dubbo.rpc.protocol.AbstractInvoker.invoke(AbstractInvoker.java:144)
        at com.alibaba.dubbo.rpc.listener.ListenerInvokerWrapper.invoke(ListenerInvokerWrapper.java:74)
        at com.alibaba.dubbo.monitor.dragoon.filter.DragoonFilter.invoke0(DragoonFilter.java:82)
        at com.alibaba.dubbo.monitor.dragoon.filter.DragoonFilter.invoke(DragoonFilter.java:34)
        at com.alibaba.dubbo.rpc.protocol.ProtocolFilterWrapper$1.invoke(ProtocolFilterWrapper.java:91)
        at com.alibaba.dubbo.monitor.support.MonitorFilter.invoke(MonitorFilter.java:75)
        at com.alibaba.dubbo.rpc.protocol.ProtocolFilterWrapper$1.invoke(ProtocolFilterWrapper.java:91)
        at com.alibaba.dubbo.rpc.protocol.dubbo.filter.FutureFilter.invoke(FutureFilter.java:53)
        at com.alibaba.dubbo.rpc.protocol.ProtocolFilterWrapper$1.invoke(ProtocolFilterWrapper.java:91)
        at com.alibaba.dubbo.rpc.filter.ConsumerContextFilter.invoke(ConsumerContextFilter.java:48)
        at com.alibaba.dubbo.rpc.protocol.ProtocolFilterWrapper$1.invoke(ProtocolFilterWrapper.java:91)
        at com.alibaba.dubbo.rpc.protocol.InvokerWrapper.invoke(InvokerWrapper.java:53)
        at com.alibaba.dubbo.rpc.cluster.support.FailoverClusterInvoker.doInvoke(FailoverClusterInvoker.java:77)
        ... 32 more

看到这个日志第一反映是觉得`RuleServiceImpl.java:163`数据错误，抛空指针，但检查那块代码发现那个地方根本不可能抛空指针 —— 所有用到的变量都是`new`出来的；  
而且，依据调用方提供的错误日志的抛出时间，我在被调用方系统的所有机器的日志里查找了一遍，没有发现对应的服务端日志；照理说服务端抛空指针，服务端也会有对应日志，但没有任何线索...  

因此只好翻开了`JavaDeserializer.java:275`作检查，发现有这么一段：  

      public JavaDeserializer(Class cl)
      {
        _type = cl;
        _fieldMap = getFieldMap(cl);
    
        _readResolve = getReadResolve(cl);
    
        if (_readResolve != null) {
          _readResolve.setAccessible(true);
        }
    
        Constructor []constructors = cl.getDeclaredConstructors();
        long bestCost = Long.MAX_VALUE;
    
        for (int i = 0; i < constructors.length; i++) {
          Class []param = constructors[i].getParameterTypes();
          long cost = 0;
    
          for (int j = 0; j < param.length; j++) {
        cost = 4 * cost;
    
        if (Object.class.equals(param[j]))
          cost += 1;
        else if (String.class.equals(param[j]))
          cost += 2;
        else if (int.class.equals(param[j]))
          cost += 3;
        else if (long.class.equals(param[j]))
          cost += 4;
        else if (param[j].isPrimitive())
          cost += 5;
        else
          cost += 6;
          }
    
          if (cost < 0 || cost > (1 << 48))
        cost = 1 << 48;
    
          cost += param.length << 48;
    
          if (cost < bestCost) {
            _constructor = constructors[i];
            bestCost = cost;
          }
        }
    
        if (_constructor != null) {
          _constructor.setAccessible(true);
          Class []params = _constructor.getParameterTypes();
          _constructorArgs = new Object[params.length];
          for (int i = 0; i < params.length; i++) {
            _constructorArgs[i] = getParamArg(params[i]);
          }
        }
      }
      
看完这段后，再结合远程调用的返回结果类后恍然大悟：  
上面这段代码，是hessian在反序列化的时候，用于在被反序列化的类里面找一个“得分最低”的构造函数，反序列化时会加以调用;  
构造函数的“得分”规则大致是：参数越少得分越低；参数个数相同时，参数类型越接近JDK内置类得分越低  
而我们的调用返回的类只有一个构造函数，当然只有这个构造函数会被选中  
但是，hessian反序列化调用被选中的构造函数时，是这样来创造该构造函数需要的参数的：  

    protected static Object getParamArg(Class cl)
      {
        if (! cl.isPrimitive())
          return null;
        else if (boolean.class.equals(cl))
          return Boolean.FALSE;
        else if (byte.class.equals(cl))
          return new Byte((byte) 0);
        else if (short.class.equals(cl))
          return new Short((short) 0);
        else if (char.class.equals(cl))
          return new Character((char) 0);
        else if (int.class.equals(cl))
          return new Integer(0);
        else if (long.class.equals(cl))
          return new Long(0);
        else if (float.class.equals(cl))
          return new Float(0);
        else if (double.class.equals(cl))
          return new Double(0);
        else
          throw new UnsupportedOperationException();
      }
      
可以看到，如果参数不是`primitive`类型，会被`null`代替；但不巧的是我们的构造函数内部对该参数调用了一个方法，因此抛出了空指针...  
也就是说这个空指针是抛在客户端反序列化的时候而不是服务端内部，因此服务端当然找不到对应的错误日志了；  
解决的办法也很简单：可以新增一个无参构造函数(无参构造函数肯定“得分”最低一定会被选中)；或者修改代码保证任意参数为`null`的时候都不会出问题  

---

下面这个问题更有意思，说的是`ThreadPoolExecutor`的`RejectedExecutionHandler`的使用：  

    threadPool = new ThreadPoolExecutor(5, maxThreadNum, 5, TimeUnit.MINUTES, new SynchronousQueue<Runnable>(), tf,
                new RejectedExecutionHandler() {
                    public void rejectedExecution(Runnable r, ThreadPoolExecutor e) {
                        logger.error("Ep thread pool exhausted, epId: " + id + "!!");
                        r.run();
                    }
                });
                
这个代码是说，当我这个ThreadPool不够用，又不能再新增线程数的时候，由调用方线程自己来执行这个`Runnable`任务...  
本来这看上去没什么问题，问题出在这个`Runnable`本身的实现上 —— 它内部将执行它的线程block到了一个blocking queue上面，当调用方主线程亲自来执行它时，使得主线程再也回不去做它自己该做的事情了，因此会出大问题...  
所以这么看来，“当线程池不够用就让调用方线程自己来干”的这个策略在实际使用时要非常谨慎  

这个问题是通过一个方便的thread dump分析工具: [tda](https://java.net/projects/tda)来查找的，因为出问题的这个应用是个多线程程序，有1600多个常驻线程，thread dump非常之大，肉眼直接看很不方便，但tda能将thread dump变得更友好易读，方便排查问题，在此推荐一下  
