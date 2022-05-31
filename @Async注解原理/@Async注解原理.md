@Async黑盒化的封装方便了使用，但是如果使用不当可能包含一些隐患，





```java

ConfigurationClassParser.processImports(ConfigurationClass configClass, SourceClass currentSourceClass,
			Collection<SourceClass> importCandidates, Predicate<String> exclusionFilter,
			boolean checkForCircularImports)
```



@Import注解参考https://www.jianshu.com/p/6b2f672e2446







![](http://img-blog.csdnimg.cn/7ca57c3406a84375a240ebd8213284d5.png)



BeanPostProcessor

https://www.jianshu.com/p/369a54201943

Aware接口

https://www.jianshu.com/p/c5c61c31080b

https://www.jianshu.com/p/da55c5e9f4f2



AopInfrastructureBean

标记接口，指示作为Spring AOP基础设施一部分的bean。特别是，这意味着任何这样的bean都不受自动代理的约束，即使切入点匹配。

ProxyConfig

用于创建代理的便利配置父类，用来确保所有的代理创建器使用相同的配置

ProxyProcessorSupport

具有代理处理器通用功能的基类，提供了evaluateProxyInterfaces(Class<?> beanClass, ProxyFactory proxyFactory)方法用来判断被代理对象是否有可以代理的接口，如果有就把这个接口加入代理工厂，否则就使用基于类的代理



AbstractAdvisingBeanPostProcessor

AbstractAdvisingBeanPostProcessor是BeanPostProcessor接口实现的一个基类，给指定的Bean提供一个Spring AOP的Advisor。

* 首先他的属性 Advisor

  源码注释翻译过来是这样的："包含AOP的Advice（在连接点采取的操作）和确定Advice适用性的过滤器（例如切入点）的基本接口。这个接口不是供Spring用户使用的，而是为了支持不同类型的Advice而提供通用性。”代表被拦截方法需要增强的逻辑。
  
  参考 https://blog.csdn.net/u012422440/article/details/87924776
  
  //TODO 实际注入的是哪个Advisor

AbstractBeanFactoryAwareAdvisingPostProcessor

在AbstractAdvisingBeanPostProcessor基础上实现了BeanFactoryAware的接口，有一个ConfigurableListableBeanFactory属性，在setBeanFactory方法里面进行初始化，重写了父类的prepareProxyFactory(Object bean, String beanName)和isEligible(Object bean, String beanName)两个方法



AsyncAnnotationBeanPostProcessor

Bean后处理器，通过向公开的代理（现有AOP代理或新生成的实现所有目标接口的代理）添加相应的{@link AsyncAnnotationAdvisor}，自动将异步调用行为应用于在类或方法级别携带{@link Async}注释的任何Bean。

有三个属性Supplier<Executor> executor，Supplier<AsyncUncaughtExceptionHandler> exceptionHandler，Class<? extends Annotation> asyncAnnotationType，重写了setBeanFactory(BeanFactory beanFactory)方法



在AsyncAnnotationBeanPostProcessor的setBeanFactory(BeanFactory beanFactory)方法，里面实例化了AsyncAnnotationAdvisor，这个类的结构如下图所示





![](https://img-blog.csdnimg.cn/877a3aef9c874261939f4891902df56a.png)

https://blog.csdn.net/f641385712/article/details/89303088

PointcutAdvisor

所有切入点驱动的Advisors的父接口，

AbstractPointcutAdvisor

实现PointcutAdvisor的抽象类，可以子类化以返回特定的PointCut/Advice或可自由配置的PointCut/Advice。

重写了getOrder、equals、hashCode方法。

AsyncAnnotationAdvisor

是一个Advisor，通过@Async注解激活异步方法执行。该注解可以在方法、实现类以及服务接口使用。这个Advisor也检测EJB 3.1 javax.ejb.Asynchronous注解

有两个属性Advice、Pointcut，主要的方法是这个构造方法，用于初始化Advice和Pointcut

```java
public AsyncAnnotationAdvisor(
      @Nullable Supplier<Executor> executor, @Nullable Supplier<AsyncUncaughtExceptionHandler> exceptionHandler) {

   Set<Class<? extends Annotation>> asyncAnnotationTypes = new LinkedHashSet<>(2);
   asyncAnnotationTypes.add(Async.class);
   try {
      asyncAnnotationTypes.add((Class<? extends Annotation>)
            ClassUtils.forName("javax.ejb.Asynchronous", AsyncAnnotationAdvisor.class.getClassLoader()));
   }
   catch (ClassNotFoundException ex) {
      // If EJB 3.1 API not present, simply ignore.
   }
   this.advice = buildAdvice(executor, exceptionHandler);
   this.pointcut = buildPointcut(asyncAnnotationTypes);
}

protected Advice buildAdvice(
			@Nullable Supplier<Executor> executor, @Nullable Supplier<AsyncUncaughtExceptionHandler> exceptionHandler) {
    AnnotationAsyncExecutionInterceptor interceptor = new AnnotationAsyncExecutionInterceptor(null);
    interceptor.configure(executor, exceptionHandler);
    return interceptor;
}
protected Pointcut buildPointcut(Set<Class<? extends Annotation>> asyncAnnotationTypes) {
	ComposablePointcut result = null;
    for (Class<? extends Annotation> asyncAnnotationType : asyncAnnotationTypes) {
        Pointcut cpc = new AnnotationMatchingPointcut(asyncAnnotationType, true);
        Pointcut mpc = new AnnotationMatchingPointcut(null, asyncAnnotationType, true);
        if (result == null) {
            result = new ComposablePointcut(cpc);
        }
        else {
            result.union(cpc);
        }
        result = result.union(mpc);
    }
    return (result != null ? result : Pointcut.TRUE);
}
```





![](https://img-blog.csdnimg.cn/70feb83580bf44a58e522ec99a3ef4e1.png)

Interceptor

代表一个通用的拦截器，可以截获基本程序中发生的运行时事件。这些事件通过连接点具体化。运行时连接点可以是调用、字段访问、异常。。。

MethodInterceptor

在方法调用前后处理一些逻辑，下面是源码注释提供的案例

```java
class TracingInterceptor implements MethodInterceptor {
  Object invoke(MethodInvocation i) throws Throwable {
    System.out.println("method " + i.getMethod()+" is called on "+ i.getThis()+" with args "+i.getArguments());
    Object ret=i.proceed();
    System.out.println("method " + i.getMethod()+" returns " + ret);
    return ret;
  }
}
```

可以看下这篇文章完整的案例 https://blog.csdn.net/u012834750/article/details/71773887

AsyncExecutionAspectSupport

这个类主要有几个属性，Map<Method, AsyncTaskExecutor> executors是方法与执行器的一个Map缓存，SingletonSupplier<Executor> defaultExecutor执行器的提供者，SingletonSupplier<AsyncUncaughtExceptionHandler> exceptionHandler异常处理的提供者，BeanFactory beanFactory是实现BeanFactoryAware接口，Spring会注入这个对象；

主要方法有构造方法和configure方法初始化默认执行器和异常处理两个属性，还有一个AsyncTaskExecutor determineAsyncExecutor(Method method)来确定执行给定方法时要使用的特定执行器，使用了前面executors属性，如果里面没有，就从beanFactory按要求去获取。还有一个doSubmit(Callable<Object> task, AsyncTaskExecutor executor, Class<?> returnType)根据执行任务的返回类型使用对应的执行器执行任务。最后一个是handleError(Throwable ex, Method method, Object... params)处理异步调用指定方法时出现的异常。



AsyncExecutionInterceptor

这个类主要有一个invoke方法，因为实现了MethodInterceptor接口，invoke方法逻辑就是获取目标方法，根据方法调用AsyncExecutionAspectSupport的determineAsyncExecutor方法获取到执行器，调用目标方法构造一个执行任务，调用AsyncExecutionAspectSupport的doSubmit方法，又返回值就返回，有异常就处理异常。

上面两个类可以参考下这篇文章  https://www.guoyuchuan.com/springboot/enableasync%E6%BA%90%E7%A0%81/2020/07/12/springboot%E5%BC%82%E6%AD%A5%E7%BA%BF%E7%A8%8B(%E4%B8%89)%E6%BA%90%E7%A0%81%E8%A7%A3%E6%9E%90(%E4%BA%8C)/



AnnotationAsyncExecutionInterceptor

这个类基本就是使用父类方法，主要有一个方法重写方法，getExecutorQualifier(Method method)，这个方法返回异步方法线程池实例在Spring容器的名字。逻辑就是获取@Async注解的value值，如果注解不是在方法上，就去类上找，如果没有返回null



上面是Advice的初始化，返回了AnnotationAsyncExecutionInterceptor实例，下面是Pointcut的初始化，

Pointcut

是一个切入点的抽象，两个抽象方法，分别返回ClassFilter和MethodMatcher

ClassFilter

筛选匹配切入点的类。

AnnotationClassFilter

主要是matches(Class<?> clazz)方法，匹配指定类是否有指定注解修饰

MethodMatcher

检查目标方法是否符合advice条件。

AnnotationMethodMatcher

主要是matches(Method method, Class\<?\> targetClass)方法，匹配指定方法是否有指定注解修饰，如果是抽象方法，看重写方法是否有指定注解修饰

ComposablePointcut

有两个属性，ClassFilter classFilter，MethodMatcher methodMatcher;

首先是构造方法，根据传进来的参数初始化，如果没传使用默认的实现，然后就是union(ClassFilter other)方法，就是将多个ClassFilter合并到一起，初始化classFilter属性，只要满足其中一个ClassFilter就可以，intersection(ClassFilter other)也是合并两个ClassFilter，但是必须两个ClassFilter都要满足，对于methodMatcher属性，类型的右union和intersection方法，总的来说就是用不同的方式初始化两个属性。

最后是重写了hashCode、equals、toString三个方法



AnnotationMatchingPointcut

基于注解的一个切入点，两个属性就是AnnotationMethodMatcher，AnnotationClassFilter



protected Pointcut buildPointcut(Set<Class<? extends Annotation>> asyncAnnotationTypes)方法就是将传进来的注解，生成两个基于类和方法的切点并组合成一个ComposablePointcut返回









参考 https://my.oschina.net/lixin91/blog/684918













实践：

Spring源码中关于@Async的测试；

带返回值和不带返回值如何做到的；

异步方法使用当前登录信息，导致Token丢失；重写Executor；使用InheritThreadLocal；

为什么@Async需要public，不在同一个类

实现AsyncConfigurer 接口，自定义executor和exceptionHandler；

反驳《新来一个技术经理，谁要是再用@Async就不要再来了》

关于@Async继承的问题

实现类似@EnableXX自定义功能