

> 之前看了一篇文章，里面提到了使用@Async注解的两个问题，第一个是Spring中实现@Async注解的线程池的阻塞队列是无界队列LinkedBlockingQueue，这就导致最大线程数的配置是无效的，如果异步任务很多且执行时间较长，会导致任务一直堆积在队列中，任务延迟很大。第二个问题是在SpringBoot中，如果没有自定义线程池实例，那么SpringBoot会使用默认的线程池，这个默认线程池是SimpleAsyncTaskExecutor，这种线程池是会为每个任务创建一个线程去执行，可能会引起资源问题。
>
> 因为项目中也用到了@Async注解，为了了解@Async的原理，决定从头开始撸一遍SpringBoot中的@Async注解的源码。



# 应用启动阶段

## 实例化AsyncConfigurationSelector

先看@EnableAsync注解

```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Import(AsyncConfigurationSelector.class)
public @interface EnableAsync {
	/**
	 * 自定义异步注解，@Async和@javax.ejb.Asynchronous默认是会被检测到的
	 */
	Class<? extends Annotation> annotation() default Annotation.class;
	/**
	 * 表示是否使用子类代理（CGLIB）还是基于接口的代理（JDK代理）
	 */
	boolean proxyTargetClass() default false;
	/**
	 * 表示使用哪种advice，PROXY是基于代理的，另外一种是切面织入形式的
	 */
	AdviceMode mode() default AdviceMode.PROXY;
	/**
	 * 表示AsyncAnnotationBeanPostProcessor这个后置处理器的应用顺序
	 */
	int order() default Ordered.LOWEST_PRECEDENCE;
}
```

@Import注解导入了AsyncConfigurationSelector类，继承自AdviceModeImportSelector

```java
public class AsyncConfigurationSelector extends AdviceModeImportSelector<EnableAsync> {
	private static final String ASYNC_EXECUTION_ASPECT_CONFIGURATION_CLASS_NAME =
			"org.springframework.scheduling.aspectj.AspectJAsyncConfiguration";
	/**
	 * 根据@EnableAsync的mode属性返回不同配置类
	 */
	@Override
	public String[] selectImports(AdviceMode adviceMode) {
		switch (adviceMode) {
			case PROXY:
				return new String[] { ProxyAsyncConfiguration.class.getName() };
			case ASPECTJ:
				return new String[] { ASYNC_EXECUTION_ASPECT_CONFIGURATION_CLASS_NAME };
			default:
				return null;
		}
	}
}

/**
 * 基于注解的mode属性来获取imports的基类
 */
public abstract class AdviceModeImportSelector<A extends Annotation> implements ImportSelector {

	public static final String DEFAULT_ADVICE_MODE_ATTRIBUTE_NAME = "mode";

	protected String getAdviceModeAttributeName() {
		return DEFAULT_ADVICE_MODE_ATTRIBUTE_NAME;
	}

	/**
	 * 获取泛型注解的mode属性，调用子类的selectImports(AdviceMode adviceMode)方法获取import配置类
	 * AnnotationMetadata importingClassMetadata是SpringBoot启动类的上获取的注解（我当前项目上就@EnableAsync, @SpringBootApplication两个注解）, ConfigurationClassParser的processImports方法传进来的
	 */
	@Override
    public final String[] selectImports(AnnotationMetadata importingClassMetadata) {
        //获取当前类的泛型参数（我自己Debug时就是@EnableAsync, getClass()获取到的是AsyncConfigurationSelector的class对象）
        Class<?> annoType = GenericTypeResolver.resolveTypeArgument(getClass(), AdviceModeImportSelector.class);
        //获取指定当前泛型注解属性和值
        AnnotationAttributes attributes = AnnotationConfigUtils.attributesFor(importingClassMetadata, annoType);
        if (attributes == null) {
            throw new IllegalArgumentException(String.format(
                "@%s is not present on importing class '%s' as expected",
                annoType.getSimpleName(), importingClassMetadata.getClassName()));
        }
		//获取mode属性值
        AdviceMode adviceMode = attributes.getEnum(this.getAdviceModeAttributeName());
        //调用子类获取import配置类
        String[] imports = selectImports(adviceMode);
        if (imports == null) {
            throw new IllegalArgumentException(String.format("Unknown AdviceMode: '%s'", adviceMode));
        }
        return imports;
    }

    /**
	 * 根据mode值返回import类完全限定名的数组
	 */
    protected abstract String[] selectImports(AdviceMode adviceMode);

}

```

看完AsyncConfigurationSelector类的结构，再看下这个类是在哪实例化的，然后再看调用了哪些方法。

![](https://img-blog.csdnimg.cn/ae764480475e439dbd6a3eab0bf9f74d.jpeg)

AsyncConfigurationSelector是在ConfigurationClassParser的processImports方法实例化的，而获取所有Import是通过ConfigurationClassParser的Set<SourceClass> getImports(SourceClass sourceClass)方法，从启动类注解开始，递归遍历所有注解上的@Import注解，获取@Import注解的value值。ConfigurationClassParser的processImports方法循环遍历所有Import的value值，对每一个调用上图578行代码实例化selector，实例化每个selector后，会继续调用每个selector的selectImports(adviceMode)方法，获取到每个selector配置的imports配置类，对这些配置类继续递归调用ConfigurationClassParser的processImports方法实例化每个import配置类。这样就实现了@Import注解的功能，根据配置导入相应的配置类。

关于@Import可以参考这篇文章  [@Import注解使用](https://www.jianshu.com/p/6b2f672e2446)

## 实例化ProxyAsyncConfiguration

```java
@Configuration
@Role(BeanDefinition.ROLE_INFRASTRUCTURE)
public class ProxyAsyncConfiguration extends AbstractAsyncConfiguration {

	@Bean(name = TaskManagementConfigUtils.ASYNC_ANNOTATION_PROCESSOR_BEAN_NAME)
	@Role(BeanDefinition.ROLE_INFRASTRUCTURE)
	public AsyncAnnotationBeanPostProcessor asyncAdvisor() {
		Assert.notNull(this.enableAsync, "@EnableAsync annotation metadata was not injected");
		AsyncAnnotationBeanPostProcessor bpp = new AsyncAnnotationBeanPostProcessor();
		Class<? extends Annotation> customAsyncAnnotation = this.enableAsync.getClass("annotation");
		if (customAsyncAnnotation != AnnotationUtils.getDefaultValue(EnableAsync.class, "annotation")) {
			bpp.setAsyncAnnotationType(customAsyncAnnotation);
		}
		if (this.executor != null) {
			bpp.setExecutor(this.executor);
		}
		if (this.exceptionHandler != null) {
			bpp.setExceptionHandler(this.exceptionHandler);
		}
		bpp.setProxyTargetClass(this.enableAsync.getBoolean("proxyTargetClass"));
		bpp.setOrder(this.enableAsync.<Integer>getNumber("order"));
		return bpp;
	}

}
```

ProxyAsyncConfiguration这个类主要功能是声明了AsyncAnnotationBeanPostProcessor这个Bean，就是异步注解后置处理器，首先需要看下AsyncAnnotationBeanPostProcessor这个类的结构

![](https://img-blog.csdnimg.cn/7ca57c3406a84375a240ebd8213284d5.png)

Serializable序列化接口，ProxyConfig代理相关的统一配置类，Ordered顺序优先级接口，AopInfrastructureBean是标识当前这个类是基础的类不允许被代理。

Aware接口是应用程序感知Spring框架一些功能的接口。比如BeanFactoryAware接口能获取Spring的BeanFactory，实现了该接口的类在实例化时，Spring框架会将BeanFactory传到重写的setBeanFactory(BeanFactory beanFactory)方法里面。BeanClassLoaderAware接口能够获取Spring的BeanClassLoader。关于Aware可以参考这篇文章 [Spring中的aware接口](https://www.jianshu.com/p/c5c61c31080b)

BeanPostProcessor接口是Bean的后置处理器，在Bean实例化后、属性设置完毕，自定义初始化方法执行之前和之后进行bean的处理。关于Spring后置处理器可以看这篇文章 [后置处理器的使用](https://www.jianshu.com/p/369a54201943)

ProxyProcessorSupport类主要有两个功能，第一个是实现了BeanClassLoaderAware接口设置类加载器，第二个是protected void evaluateProxyInterfaces(Class<?> beanClass, ProxyFactory proxyFactory)方法，用来判断bean是否使用基于类的代理，如果不是，把需要代理的接口加到代理工厂里面。

下面重点讲解下面三个类

### AbstractAdvisingBeanPostProcessor

```java
/**
 * 
 */
public abstract class AbstractAdvisingBeanPostProcessor extends ProxyProcessorSupport implements BeanPostProcessor {

    /**
     * Advisor是一个基本接口，持有一个Advice（在连接点采取的操作）和一个确定Advice使用范围的过滤器（比如切入点），是为了支持不同类型Advice提供的抽象
     * 实例化AsyncAnnotationBeanPostProcessor，初始化该属性的时候，传进来的实际类型是AsyncAnnotationAdvisor
     */
	@Nullable
	protected Advisor advisor;

    /**
     * 如果当前传进来的bean是Advised类型，且持有很多Advisor，是否把当前Advisor放在最前面
     */
	protected boolean beforeExistingAdvisors = false;

    /**
     * bean对象和是否能被当前后置处理器的advisor处理的一个Map缓存，因为同一bean会走项目所有的后置处理器，当前这个抽象类有很多子类，使用缓存可以加快效率
     */
	private final Map<Class<?>, Boolean> eligibleBeans = new ConcurrentHashMap<>(256);

	public void setBeforeExistingAdvisors(boolean beforeExistingAdvisors) {
		this.beforeExistingAdvisors = beforeExistingAdvisors;
	}

	//重写的初始化之前的操作，直接返回
	@Override
	public Object postProcessBeforeInitialization(Object bean, String beanName) {
		return bean;
	}

	@Override
	public Object postProcessAfterInitialization(Object bean, String beanName) {
		if (this.advisor == null || bean instanceof AopInfrastructureBean) {
			// Ignore AOP infrastructure such as scoped proxies.
			return bean;
		}

        //如果当前传进来的Bean是Advised类型，把当前的advisor放到一起管理
		if (bean instanceof Advised) {
			Advised advised = (Advised) bean;
			if (!advised.isFrozen() && isEligible(AopUtils.getTargetClass(bean))) {
				//当前Advisor到Advised中， 根据beforeExistingAdvisors是否添加到前面
				if (this.beforeExistingAdvisors) {
					advised.addAdvisor(0, this.advisor);
				}
				else {
					advised.addAdvisor(this.advisor);
				}
				return bean;
			}
		}

        //当前传进来的Bean对象，是否能够被当前后置处理器的Advisor处理
		if (isEligible(bean, beanName)) {
            //为传进来的bean生成一个代理工厂
			ProxyFactory proxyFactory = prepareProxyFactory(bean, beanName);
            //如果是接口代理，调用父类ProxyProcessorSupport的方法把需要代理接口加入到代理工厂对象中
			if (!proxyFactory.isProxyTargetClass()) {
				evaluateProxyInterfaces(bean.getClass(), proxyFactory);
			}
            //将当前advisor加入到代理工厂中，ProxyFactory也是实现了Advised接口的
			proxyFactory.addAdvisor(this.advisor);
            //子类自定义处理代理工厂
			customizeProxyFactory(proxyFactory);
            //生成代理对象
			return proxyFactory.getProxy(getProxyClassLoader());
		}

		// No proxy needed.
		return bean;
	}

	protected boolean isEligible(Object bean, String beanName) {
		return isEligible(bean.getClass());
	}

	protected boolean isEligible(Class<?> targetClass) {
		Boolean eligible = this.eligibleBeans.get(targetClass);
		if (eligible != null) {
			return eligible;
		}
		if (this.advisor == null) {
			return false;
		}
        //根据不同类型的advisor中的ClassFilter和MethodMatcher，判断是否能处理当前bean的Class对象
		eligible = AopUtils.canApply(this.advisor, targetClass);
        //放入缓存
		this.eligibleBeans.put(targetClass, eligible);
		return eligible;
	}

	protected ProxyFactory prepareProxyFactory(Object bean, String beanName) {
		ProxyFactory proxyFactory = new ProxyFactory();
		proxyFactory.copyFrom(this);
		proxyFactory.setTarget(bean);
		return proxyFactory;
	}

	protected void customizeProxyFactory(ProxyFactory proxyFactory) {
	}

}
```

有关Advisor、Advice、Advised可以看这三篇文章 [Spring AOP(三) Advisor类架构](https://blog.csdn.net/u012422440/article/details/87924776)    [Advice、Advisor、Advised都是什么接口？](https://blog.51cto.com/laowang6/4854737)    [Spring AOP之Advisor、PointcutAdvisor、IntroductionAdvisor、IntroductionInterceptor](https://blog.csdn.net/f641385712/article/details/89303088)

### AbstractBeanFactoryAwareAdvisingPostProcessor

这个类只是多实现了BeanFactoryAware接口，为了获取BeanFactory，重写了父类的prepareProxyFactory和isEligible方法

```java
public abstract class AbstractBeanFactoryAwareAdvisingPostProcessor extends AbstractAdvisingBeanPostProcessor
		implements BeanFactoryAware {

    /**
     * 除了有ConfigurableBeanFactory的功能，为分析、修改bean的定义，或单例bean预初始化提供了便利
     */
	@Nullable
	private ConfigurableListableBeanFactory beanFactory;


    /**
     * BeanFactoryAware的功能，Spring框架会将BeanFactory传进来，调用这个方法
     */
	@Override
	public void setBeanFactory(BeanFactory beanFactory) {
        //如果是指定的类型就初始化beanFactory这个属性
		this.beanFactory = (beanFactory instanceof ConfigurableListableBeanFactory ?
				(ConfigurableListableBeanFactory) beanFactory : null);
	}

	@Override
	protected ProxyFactory prepareProxyFactory(Object bean, String beanName) {
		if (this.beanFactory != null) {
            //在BeanDefinition中设置bean的originalTargetClass属性
			AutoProxyUtils.exposeTargetClass(this.beanFactory, beanName, bean.getClass());
		}
		//调用父类方法生成代理工厂
		ProxyFactory proxyFactory = super.prepareProxyFactory(bean, beanName);
        //即使代理工厂是基于接口代理，但是如果给定的bean和BeanFactory需要基于类代理的，设置代理工厂为基于类代理
		if (!proxyFactory.isProxyTargetClass() && this.beanFactory != null &&
				AutoProxyUtils.shouldProxyTargetClass(this.beanFactory, beanName)) {
			proxyFactory.setProxyTargetClass(true);
		}
		return proxyFactory;
	}

    /**
     * 如果beanName表示是原始实例，那么跳过任何代理，否则调用父类isEligible方法
     */
	@Override
	protected boolean isEligible(Object bean, String beanName) {
		return (!AutoProxyUtils.isOriginalInstance(beanName, bean.getClass()) &&
				super.isEligible(bean, beanName));
	}
}
```



### AsyncAnnotationBeanPostProcessor

```java
public class AsyncAnnotationBeanPostProcessor extends AbstractBeanFactoryAwareAdvisingPostProcessor {

	/**
	 * 默认的TaskExecutor bean名称，值是"taskExecutor"
	 */
	public static final String DEFAULT_TASK_EXECUTOR_BEAN_NAME =
			AnnotationAsyncExecutionInterceptor.DEFAULT_TASK_EXECUTOR_BEAN_NAME;

	protected final Log logger = LogFactory.getLog(getClass());

    //线程池的一个Supplier接口，Supplier是一个函数式接口，通过他的Get方法可以获得Executor的实例
	@Nullable
	private Supplier<Executor> executor;

    /**
     * 处理异步方法引发的未捕获异常的策略。异步方法通常返回一个Future实例，该实例允许访问底层异常。
     * 当该方法不提供该返回类型时，可以使用该处理程序来管理此类未捕获的异常。
     */
	@Nullable
	private Supplier<AsyncUncaughtExceptionHandler> exceptionHandler;

    /**
     * 自定义的异步注解，@EnableAsync的annotation属性传进来的
     */
	@Nullable
	private Class<? extends Annotation> asyncAnnotationType;

    //设置当前后置处理器的advisor是否在其他advisor之前
	public AsyncAnnotationBeanPostProcessor() {
		setBeforeExistingAdvisors(true);
	}

	public void configure(
			@Nullable Supplier<Executor> executor, @Nullable Supplier<AsyncUncaughtExceptionHandler> exceptionHandler) {

		this.executor = executor;
		this.exceptionHandler = exceptionHandler;
	}

	public void setExecutor(Executor executor) {
		this.executor = SingletonSupplier.of(executor);
	}

	public void setExceptionHandler(AsyncUncaughtExceptionHandler exceptionHandler) {
		this.exceptionHandler = SingletonSupplier.of(exceptionHandler);
	}

	public void setAsyncAnnotationType(Class<? extends Annotation> asyncAnnotationType) {
		Assert.notNull(asyncAnnotationType, "'asyncAnnotationType' must not be null");
		this.asyncAnnotationType = asyncAnnotationType;
	}

	@Override
	public void setBeanFactory(BeanFactory beanFactory) {
        //调用父类方法，设置ConfigurableListableBeanFactory类型的BeanFactory
		super.setBeanFactory(beanFactory);
		//生成一个Advisor实例
		AsyncAnnotationAdvisor advisor = new AsyncAnnotationAdvisor(this.executor, this.exceptionHandler);
		if (this.asyncAnnotationType != null) {
            //如果有自定义的异步注解，使用自定义的异步注解
			advisor.setAsyncAnnotationType(this.asyncAnnotationType);
		}
		advisor.setBeanFactory(beanFactory);
        //初始化advisor属性
		this.advisor = advisor;
	}

}
```

生成AsyncAnnotationAdvisor实例需要继续往里面看，是非常关键的一步，里面有如何拦截异步方法，如何调用的逻辑，继续往下看。

#### AsyncAnnotationAdvisor

这个类的继承关系如下

![](http://img-blog.csdnimg.cn/877a3aef9c874261939f4891902df56a.png)

PointcutAdvisor接口是切入点驱动的Advisor，所以AsyncAnnotationAdvisor就是持有Advice和Pointcut的Advisor，类代码如下

```java
public class AsyncAnnotationAdvisor extends AbstractPointcutAdvisor implements BeanFactoryAware {

	private Advice advice;

	private Pointcut pointcut;

	public AsyncAnnotationAdvisor() {
		this((Supplier<Executor>) null, (Supplier<AsyncUncaughtExceptionHandler>) null);
	}

	@SuppressWarnings("unchecked")
	public AsyncAnnotationAdvisor(
			@Nullable Executor executor, @Nullable AsyncUncaughtExceptionHandler exceptionHandler) {

		this(SingletonSupplier.ofNullable(executor), SingletonSupplier.ofNullable(exceptionHandler));
	}

	@SuppressWarnings("unchecked")
	public AsyncAnnotationAdvisor(
			@Nullable Supplier<Executor> executor, @Nullable Supplier<AsyncUncaughtExceptionHandler> exceptionHandler) {

		Set<Class<? extends Annotation>> asyncAnnotationTypes = new LinkedHashSet<>(2);
        //添加@Async注解
		asyncAnnotationTypes.add(Async.class);
		try {
            //添加@Asynchronous注解
			asyncAnnotationTypes.add((Class<? extends Annotation>)
					ClassUtils.forName("javax.ejb.Asynchronous", AsyncAnnotationAdvisor.class.getClassLoader()));
		}
		catch (ClassNotFoundException ex) {
			// If EJB 3.1 API not present, simply ignore.
		}
        //初始化advice属性
		this.advice = buildAdvice(executor, exceptionHandler);
		//初始化pointcut属性
        this.pointcut = buildPointcut(asyncAnnotationTypes);
	}

	/**
	 * 自定义异步注解的切点，通过@EnableAsync的annotation属性传进来，默认的@Async和@javax.ejb.Asynchronous会失效
	 */
	public void setAsyncAnnotationType(Class<? extends Annotation> asyncAnnotationType) {
		Assert.notNull(asyncAnnotationType, "'asyncAnnotationType' must not be null");
		Set<Class<? extends Annotation>> asyncAnnotationTypes = new HashSet<>();
		asyncAnnotationTypes.add(asyncAnnotationType);
		this.pointcut = buildPointcut(asyncAnnotationTypes);
	}

	/**
	 * 设置advice的BeanFactory，在后面调用异步方法时，通过这个beanFactory根据executor的beanName来获取executor实例
	 */
	@Override
	public void setBeanFactory(BeanFactory beanFactory) {
		if (this.advice instanceof BeanFactoryAware) {
			((BeanFactoryAware) this.advice).setBeanFactory(beanFactory);
		}
	}

	@Override
	public Advice getAdvice() {
		return this.advice;
	}

	@Override
	public Pointcut getPointcut() {
		return this.pointcut;
	}

	protected Advice buildAdvice(
			@Nullable Supplier<Executor> executor, @Nullable Supplier<AsyncUncaughtExceptionHandler> exceptionHandler) {
		//生成一个advice实例，这个下面继续往里面看
		AnnotationAsyncExecutionInterceptor interceptor = new AnnotationAsyncExecutionInterceptor(null);
		interceptor.configure(executor, exceptionHandler);
		return interceptor;
	}

	protected Pointcut buildPointcut(Set<Class<? extends Annotation>> asyncAnnotationTypes) {
		ComposablePointcut result = null;
        //传进来的是@Async或@javax.ejb.Asynchronous注解；也可能是自定义的一个异步注解，遍历每一个注解
		for (Class<? extends Annotation> asyncAnnotationType : asyncAnnotationTypes) {
            //类级别的切点
			Pointcut cpc = new AnnotationMatchingPointcut(asyncAnnotationType, true);
            //方法级别的切点
			Pointcut mpc = new AnnotationMatchingPointcut(null, asyncAnnotationType, true);
			if (result == null) {
				result = new ComposablePointcut(cpc);
			}
			else {
				result.union(cpc);
			}
            //合并所有类级别、方法级别的切点
			result = result.union(mpc);
		}
        //如果为空，返回所有方法、类都匹配的切点
		return (result != null ? result : Pointcut.TRUE);
	}

}
```

下面需要讲下AnnotationAsyncExecutionInterceptor和AnnotationMatchingPointcut两个类，看完这两个类，整个异步处理就基本结束了

##### AnnotationAsyncExecutionInterceptor

AnnotationAsyncExecutionInterceptor类文件结构如下

![](http://img-blog.csdnimg.cn/70feb83580bf44a58e522ec99a3ef4e1.png)

Interceptor是通用拦截器接口抽象，是个空接口；

MethodInterceptor是方法拦截器，有一个Object invoke(MethodInvocation invocation)方法，重写该方法可以在方法调用的前后，加入自定义逻辑，下面是源码中提供的例子 （关于方法拦截器可以参考这篇文章 [Spring方法拦截器MethodInterceptor](https://blog.csdn.net/u012834750/article/details/71773887)）

```java
class TracingInterceptor implements MethodInterceptor {
  Object invoke(MethodInvocation i) throws Throwable {
    System.out.println("method "+i.getMethod()+" is called on "+
                     i.getThis()+" with args "+i.getArguments());
    Object ret=i.proceed();//调用目标方法
    System.out.println("method "+i.getMethod()+" returns "+ret);
    return ret;
  }
}
```

AsyncExecutionAspectSupport是异步方法执行切面的基类，实现了BeanFactoryWare接口，可以获取Bean工厂，源码如下

```java
public abstract class AsyncExecutionAspectSupport implements BeanFactoryAware {

	//线程池默认bean名称
	public static final String DEFAULT_TASK_EXECUTOR_BEAN_NAME = "taskExecutor";


	//CompletableFuture类是否存在，这个类是java8引入的，这个字段doSubmit方法有用到
	private static final boolean completableFuturePresent = ClassUtils.isPresent(
			"java.util.concurrent.CompletableFuture", AsyncExecutionInterceptor.class.getClassLoader());


	protected final Log logger = LogFactory.getLog(getClass());

    //异步方法和对应线程池实例的缓存，因为每个异步方法可以指定线程池实例
	private final Map<Method, AsyncTaskExecutor> executors = new ConcurrentHashMap<Method, AsyncTaskExecutor>(16);

    //默认的线程池实例
	private volatile Executor defaultExecutor;

    //未捕获异常的处理器
	private AsyncUncaughtExceptionHandler exceptionHandler;

	private BeanFactory beanFactory;

	public AsyncExecutionAspectSupport(Executor defaultExecutor) {
		this(defaultExecutor, new SimpleAsyncUncaughtExceptionHandler());
	}

	public AsyncExecutionAspectSupport(Executor defaultExecutor, AsyncUncaughtExceptionHandler exceptionHandler) {
		this.defaultExecutor = defaultExecutor;
		this.exceptionHandler = exceptionHandler;
	}

	public void setExecutor(Executor defaultExecutor) {
		this.defaultExecutor = defaultExecutor;
	}

	public void setExceptionHandler(AsyncUncaughtExceptionHandler exceptionHandler) {
		this.exceptionHandler = exceptionHandler;
	}

	/**
	 * 重写BeanFactoryAware接口的方法，设置Bean工厂
	 */
	@Override
	public void setBeanFactory(BeanFactory beanFactory) {
		this.beanFactory = beanFactory;
	}


	/**
	 * 根据指定异步方法获取对应的线程池实例
	 */
	protected AsyncTaskExecutor determineAsyncExecutor(Method method) {
        //从缓存里面获取，如果获取得到直接返回
		AsyncTaskExecutor executor = this.executors.get(method);
		if (executor == null) {
			Executor targetExecutor;
            //根据方法获取线程池实例的Bean名称，@Async的value属性的值
			String qualifier = getExecutorQualifier(method);
			if (StringUtils.hasLength(qualifier)) {
                //bean工厂根据bean名称获取线程池实例
				targetExecutor = findQualifiedExecutor(this.beanFactory, qualifier);
			}
			else {
				targetExecutor = this.defaultExecutor;
				if (targetExecutor == null) {
					synchronized (this.executors) {
						if (this.defaultExecutor == null) {
							this.defaultExecutor = getDefaultExecutor(this.beanFactory);
						}
						targetExecutor = this.defaultExecutor;
					}
				}
			}
			if (targetExecutor == null) {
				return null;
			}
            //如果不是AsyncListenableTaskExecutor类型的线程池实例，构造一个TaskExecutorAdapter实例，TaskExecutorAdapter是带一个TaskDecorator属性的线程池实例，可以对要执行的任务进行装饰，比如SpringSecurity进行权限管理时，创建异步任务会丢失父线程的权限信息，可以写一个类实现TaskDecorator接口，在decorate方法里面往SecurityContextHolder设置上下文信息
			executor = (targetExecutor instanceof AsyncListenableTaskExecutor ?
					(AsyncListenableTaskExecutor) targetExecutor : new TaskExecutorAdapter(targetExecutor));
			this.executors.put(method, executor);
		}
		return executor;
	}

	/**
	 * 抽象方法，根据异步方法获取线程池bean的名称
	 */
	protected abstract String getExecutorQualifier(Method method);

	/**
	 * 根据bean名称获取Executor类型的线程池实例
	 */
	protected Executor findQualifiedExecutor(BeanFactory beanFactory, String qualifier) {
		if (beanFactory == null) {
			throw new IllegalStateException("BeanFactory must be set on " + getClass().getSimpleName() +
					" to access qualified executor '" + qualifier + "'");
		}
		return BeanFactoryAnnotationUtils.qualifiedBeanOfType(beanFactory, Executor.class, qualifier);
	}
	
    //获取默认的线程池实例
	protected Executor getDefaultExecutor(BeanFactory beanFactory) {
		if (beanFactory != null) {
			try {
				// 找TaskExecutor类型的线程池实例
				return beanFactory.getBean(TaskExecutor.class);
			}
			catch (NoUniqueBeanDefinitionException ex) {
				logger.debug("Could not find unique TaskExecutor bean", ex);
				try {
                    //找名称为taskExecutor的线程池实例
					return beanFactory.getBean(DEFAULT_TASK_EXECUTOR_BEAN_NAME, Executor.class);
				}
				catch (NoSuchBeanDefinitionException ex2) {
					if (logger.isInfoEnabled()) {
						logger.info("More than one TaskExecutor bean found within the context, and none is named " +
								"'taskExecutor'. Mark one of them as primary or name it 'taskExecutor' (possibly " +
								"as an alias) in order to use it for async processing: " + ex.getBeanNamesFound());
					}
				}
			}
			catch (NoSuchBeanDefinitionException ex) {
				logger.debug("Could not find default TaskExecutor bean", ex);
				try {
                    //找名称为taskExecutor的线程池实例
					return beanFactory.getBean(DEFAULT_TASK_EXECUTOR_BEAN_NAME, Executor.class);
				}
				catch (NoSuchBeanDefinitionException ex2) {
					logger.info("No task executor bean found for async processing: " +
							"no bean of type TaskExecutor and no bean named 'taskExecutor' either");
				}
				// Giving up -> either using local default executor or none at all...
			}
		}
		return null;
	}


	/**
	 * 执行异步任务，参数分别是异步方法执行逻辑、线程池实例、异步方法返回结果
	 */
	protected Object doSubmit(Callable<Object> task, AsyncTaskExecutor executor, Class<?> returnType) {
        //如果是Java8，使用CompletableFuture来执行异步任务
		if (completableFuturePresent) {
			Future<Object> result = CompletableFutureDelegate.processCompletableFuture(returnType, task, executor);
			if (result != null) {
				return result;
			}
		}
        //如果返回类型是ListenableFuture调用submitListenable
		if (ListenableFuture.class.isAssignableFrom(returnType)) {
			return ((AsyncListenableTaskExecutor) executor).submitListenable(task);
		}
        //如果返回类型是其他的Future类型，直接交给线程池执行
		else if (Future.class.isAssignableFrom(returnType)) {
			return executor.submit(task);
		}
		else {
            //直接不返回结果
			executor.submit(task);
			return null;
		}
	}

	//异常处理
	protected void handleError(Throwable ex, Method method, Object... params) throws Exception {
        //带返回值的直接抛出异常
		if (Future.class.isAssignableFrom(method.getReturnType())) {
			ReflectionUtils.rethrowException(ex);
		}
		else {
			//异常处理器处理异常，即使再出现异常也不抛出
			try {
				this.exceptionHandler.handleUncaughtException(ex, method, params);
			}
			catch (Throwable ex2) {
				logger.error("Exception handler for async method '" + method.toGenericString() +
						"' threw unexpected exception itself", ex2);
			}
		}
	}


	/**
	 * Java8下执行异步任务的内部类
	 */
	@UsesJava8
	private static class CompletableFutureDelegate {

		public static <T> Future<T> processCompletableFuture(Class<?> returnType, final Callable<T> task, Executor executor) {
            //如果异步方法返回值不是CompletableFuture类型直接返回null
			if (!CompletableFuture.class.isAssignableFrom(returnType)) {
				return null;
			}
            //调用CompletableFuture的supplyAsync方法去执行task任务
			return CompletableFuture.supplyAsync(new Supplier<T>() {
				@Override
				public T get() {
					try {
						return task.call();
					}
					catch (Throwable ex) {
						throw new CompletionException(ex);
					}
				}
			}, executor);
		}
	}

}
```

关于CompletableFuture类可以参考这篇文章 [CompletableFuture原理解析](https://www.jianshu.com/p/abfa29c01e1d)

下面看AsyncExecutionInterceptor这个类，这个类是处理异步方法调用的方法拦截器

```java
public class AsyncExecutionInterceptor extends AsyncExecutionAspectSupport implements MethodInterceptor, Ordered {

	public AsyncExecutionInterceptor(Executor defaultExecutor) {
		super(defaultExecutor);
	}

	public AsyncExecutionInterceptor(Executor defaultExecutor, AsyncUncaughtExceptionHandler exceptionHandler) {
		super(defaultExecutor, exceptionHandler);
	}

	/**
	 * MethodInterceptor重写的方法，方法调用前后处理一些逻辑
	 */
	@Override
	public Object invoke(final MethodInvocation invocation) throws Throwable {
        //获取invocation的目标对象的class对象（被调用的异步方法所属对象的Class对象）
		Class<?> targetClass = (invocation.getThis() != null ? AopUtils.getTargetClass(invocation.getThis()) : null);
        //通过class对象和invocation的method获取Method
		Method specificMethod = ClassUtils.getMostSpecificMethod(invocation.getMethod(), targetClass);
        //没理解为什么还要再获取一次，我debug时候调用specificMethod.equlas(userDeclaredMethod)返回的是true
		final Method userDeclaredMethod = BridgeMethodResolver.findBridgedMethod(specificMethod);

        //通过method获取处理这个异步方法的线程池实例
		AsyncTaskExecutor executor = determineAsyncExecutor(userDeclaredMethod);
		if (executor == null) {
			throw new IllegalStateException(
					"No executor specified and no default executor set on AsyncExecutionInterceptor either");
		}

        //将异步方法封装成一个Callable对象
		Callable<Object> task = new Callable<Object>() {
			@Override
			public Object call() throws Exception {
				try {
					Object result = invocation.proceed();
					if (result instanceof Future) {
						return ((Future<?>) result).get();
					}
				}
				catch (ExecutionException ex) {
					handleError(ex.getCause(), userDeclaredMethod, invocation.getArguments());
				}
				catch (Throwable ex) {
					handleError(ex, userDeclaredMethod, invocation.getArguments());
				}
				return null;
			}
		};

        //把异步任务、线程池实例、返回值类型传进去，调用父类的AsyncExecutionAspectSupport的doSubmit方法
		return doSubmit(task, executor, invocation.getMethod().getReturnType());
	}

	/**
	 * 根据异步方法，获取处理该异步方法的线程池实例的bean名称，后续在BeanFactory里面根据这个名称获取线程池实例，本类中返回null，子类会重写这个方法，AnnotationAsyncExecutionInterceptor重写改方法是获取@Async注解的value值
	 */
	@Override
	protected String getExecutorQualifier(Method method) {
		return null;
	}

	/**
	 * 调用父类的获取默认线程池实例的方法，如果获取不到，使用SimpleAsyncTaskExecutor实例
	 * SimpleAsyncTaskExecutor这个线程池会为每个任务触发一个新线程，异步执行它，相当于没用线程池
	 */
	@Override
	protected Executor getDefaultExecutor(BeanFactory beanFactory) {
		Executor defaultExecutor = super.getDefaultExecutor(beanFactory);
		return (defaultExecutor != null ? defaultExecutor : new SimpleAsyncTaskExecutor());
	}

	@Override
	public int getOrder() {
		return Ordered.HIGHEST_PRECEDENCE;
	}

}
```

最后看最终实例化的AnnotationAsyncExecutionInterceptor这个类

```java
public class AnnotationAsyncExecutionInterceptor extends AsyncExecutionInterceptor {

	public AnnotationAsyncExecutionInterceptor(@Nullable Executor defaultExecutor) {
		super(defaultExecutor);
	}

	public AnnotationAsyncExecutionInterceptor(@Nullable Executor defaultExecutor, AsyncUncaughtExceptionHandler exceptionHandler) {
		super(defaultExecutor, exceptionHandler);
	}


	/**
	 * 返回执行该异步方法执行时用到的线程池实例bean名称
	 */
	@Override
	@Nullable
	protected String getExecutorQualifier(Method method) {
		//从方法上找是否有@Async修饰
		Async async = AnnotatedElementUtils.findMergedAnnotation(method, Async.class);
		if (async == null) {
            //如果方法上没有，该方法的类上找是否有@Async修饰
			async = AnnotatedElementUtils.findMergedAnnotation(method.getDeclaringClass(), Async.class);
		}
        //如果找的到，返回@Async注解的value属性值
		return (async != null ? async.value() : null);
	}

}
```

到这里AsyncAnnotationAdvisor的advice属性已经讲完了，下面再看另外一个重要属性pointcut

##### AnnotationMatchingPointcut

在AsyncAnnotationAdvisor类中的Pointcut buildPointcut(Set<Class<? extends Annotation>> asyncAnnotationTypes)这个方法，我们传进来了@Async和@javax.ejb.Asynchronous注解，对每个注解，在类和方法级别都创建了AnnotationMatchingPointcut对象，并通过ComposablePointcut对象把所有的切点组合在一起，AnnotationMatchingPointcut源码如下（关于ClassFilter可以参考这篇文章 [spring-aop 组件详解 ——ClassFilter 类过滤器](https://my.oschina.net/lixin91/blog/684918) ）

```java
public class AnnotationMatchingPointcut implements Pointcut {

    //类级别的过滤器
	private final ClassFilter classFilter;

	//方法级别的匹配器
	private final MethodMatcher methodMatcher;

	public AnnotationMatchingPointcut(Class<? extends Annotation> classAnnotationType) {
		this(classAnnotationType, false);
	}

	public AnnotationMatchingPointcut(Class<? extends Annotation> classAnnotationType, boolean checkInherited) {
		//实例化一个AnnotationClassFilter注解类过滤器，判断class对象是否有指定注解修饰，如果checkInherited为true还会往父类、父接口查找是否有指定注解修饰
        this.classFilter = new AnnotationClassFilter(classAnnotationType, checkInherited);
		this.methodMatcher = MethodMatcher.TRUE;
	}

	public AnnotationMatchingPointcut(@Nullable Class<? extends Annotation> classAnnotationType,
			@Nullable Class<? extends Annotation> methodAnnotationType) {

		this(classAnnotationType, methodAnnotationType, false);
	}

	public AnnotationMatchingPointcut(@Nullable Class<? extends Annotation> classAnnotationType,
			@Nullable Class<? extends Annotation> methodAnnotationType, boolean checkInherited) {

		Assert.isTrue((classAnnotationType != null || methodAnnotationType != null),
				"Either Class annotation type or Method annotation type needs to be specified (or both)");

		if (classAnnotationType != null) {
            //判断class对象是否有指定注解修饰，如果checkInherited为true还会往父类、父接口查找是否有指定注解修饰
			this.classFilter = new AnnotationClassFilter(classAnnotationType, checkInherited);
		}
		else {
            //判断class对象是否有指定注解修饰，但是不会判断父接口、父类是否有该接口修饰
			this.classFilter = new AnnotationCandidateClassFilter(methodAnnotationType);
		}

		if (methodAnnotationType != null) {
            //判断方法是否有指定注解修饰
			this.methodMatcher = new AnnotationMethodMatcher(methodAnnotationType, checkInherited);
		}
		else {
			this.methodMatcher = MethodMatcher.TRUE;
		}
	}


	@Override
	public ClassFilter getClassFilter() {
		return this.classFilter;
	}

	@Override
	public MethodMatcher getMethodMatcher() {
		return this.methodMatcher;
	}

	@Override
	public boolean equals(@Nullable Object other) {
		if (this == other) {
			return true;
		}
		if (!(other instanceof AnnotationMatchingPointcut)) {
			return false;
		}
		AnnotationMatchingPointcut otherPointcut = (AnnotationMatchingPointcut) other;
		return (this.classFilter.equals(otherPointcut.classFilter) &&
				this.methodMatcher.equals(otherPointcut.methodMatcher));
	}

	@Override
	public int hashCode() {
		return this.classFilter.hashCode() * 37 + this.methodMatcher.hashCode();
	}

	@Override
	public String toString() {
		return "AnnotationMatchingPointcut: " + this.classFilter + ", " + this.methodMatcher;
	}

	/**
	 * 类级别的AnnotationMatchingPointcut方法工厂
	 */
	public static AnnotationMatchingPointcut forClassAnnotation(Class<? extends Annotation> annotationType) {
		Assert.notNull(annotationType, "Annotation type must not be null");
		return new AnnotationMatchingPointcut(annotationType);
	}

    /**
	 * 方法级别的AnnotationMatchingPointcut方法工厂
	 */
	public static AnnotationMatchingPointcut forMethodAnnotation(Class<? extends Annotation> annotationType) {
		Assert.notNull(annotationType, "Annotation type must not be null");
		return new AnnotationMatchingPointcut(null, annotationType);
	}

    //判断class对象是否有指定注解修饰，和AnnotationClassFilter区别就是AnnotationCandidateClassFilter不会判断父接口、父类是否有该接口修饰
	private static class AnnotationCandidateClassFilter implements ClassFilter {

		private final Class<? extends Annotation> annotationType;

		AnnotationCandidateClassFilter(Class<? extends Annotation> annotationType) {
			this.annotationType = annotationType;
		}

		@Override
		public boolean matches(Class<?> clazz) {
			return AnnotationUtils.isCandidateClass(clazz, this.annotationType);
		}

		@Override
		public boolean equals(Object obj) {
			if (this == obj) {
				return true;
			}
			if (!(obj instanceof AnnotationCandidateClassFilter)) {
				return false;
			}
			AnnotationCandidateClassFilter that = (AnnotationCandidateClassFilter) obj;
			return this.annotationType.equals(that.annotationType);
		}

		@Override
		public int hashCode() {
			return this.annotationType.hashCode();
		}

		@Override
		public String toString() {
			return getClass().getName() + ": " + this.annotationType;
		}

	}

}
```



# 异步方法调用阶段

方法调用时，会进入DynamicAdvisedInterceptor这个类的Object intercept(Object proxy, Method method, Object[] args, MethodProxy methodProxy)方法，为什么会进这个方法，还需要往前看，在应用启动阶段生成代理对象时做的具体操作。在AbstractAdvisingBeanPostProcessor类中，

![](https://img-blog.csdnimg.cn/27c54c377a0a4ffda1b3cbf62b49c5f7.png)

红框处生成代理对象时，会再调用CglibAopProxy类的getProxy方法

![](https://img-blog.csdnimg.cn/a14ad839e6ed4c71aa5b54f57264e954.png)

里面会再调用getCallbacks方法获取回调

![](https://img-blog.csdnimg.cn/5db8f66b465e48de9103b1c2f2cd0db9.png)

在获取回调的方法中会直接new一个DynamicAdvisedInterceptor，把这个方法拦截器设置到回调中，在调用异步方法时会进入这个拦截器，DynamicAdvisedInterceptor的源码如下

```java
private static class DynamicAdvisedInterceptor implements MethodInterceptor, Serializable {

		private final AdvisedSupport advised;

		public DynamicAdvisedInterceptor(AdvisedSupport advised) {
			this.advised = advised;
		}

		@Override
		@Nullable
		public Object intercept(Object proxy, Method method, Object[] args, MethodProxy methodProxy) throws Throwable {
			Object oldProxy = null;
			boolean setProxyContext = false;
			Object target = null;
			TargetSource targetSource = this.advised.getTargetSource();
			try {
				if (this.advised.exposeProxy) {
					// Make invocation available if necessary.
					oldProxy = AopContext.setCurrentProxy(proxy);
					setProxyContext = true;
				}
				// Get as late as possible to minimize the time we "own" the target, in case it comes from a pool...
				target = targetSource.getTarget();
				Class<?> targetClass = (target != null ? target.getClass() : null);
				List<Object> chain = this.advised.getInterceptorsAndDynamicInterceptionAdvice(method, targetClass);
				Object retVal;
				// Check whether we only have one InvokerInterceptor: that is,
				// no real advice, but just reflective invocation of the target.
				if (chain.isEmpty() && Modifier.isPublic(method.getModifiers())) {
					// We can skip creating a MethodInvocation: just invoke the target directly.
					// Note that the final invoker must be an InvokerInterceptor, so we know
					// it does nothing but a reflective operation on the target, and no hot
					// swapping or fancy proxying.
					Object[] argsToUse = AopProxyUtils.adaptArgumentsIfNecessary(method, args);
					retVal = methodProxy.invoke(target, argsToUse);
				}
				else {
					// We need to create a method invocation...
					retVal = new CglibMethodInvocation(proxy, target, method, args, targetClass, chain, methodProxy).proceed();
				}
				retVal = processReturnType(proxy, target, method, retVal);
				return retVal;
			}
			finally {
				if (target != null && !targetSource.isStatic()) {
					targetSource.releaseTarget(target);
				}
				if (setProxyContext) {
					// Restore old proxy.
					AopContext.setCurrentProxy(oldProxy);
				}
			}
		}

		@Override
		public boolean equals(@Nullable Object other) {
			return (this == other ||
					(other instanceof DynamicAdvisedInterceptor &&
							this.advised.equals(((DynamicAdvisedInterceptor) other).advised)));
		}

		/**
		 * CGLIB uses this to drive proxy creation.
		 */
		@Override
		public int hashCode() {
			return this.advised.hashCode();
		}
	}
```

DynamicAdvisedInterceptor这个类有一个AdvisedSupport属性，这个属性其实是从异步注解后置处理器的advisor拿过来，然后执行new CglibMethodInvocation那一行代码，把之前设置在后置处理器中的Advice传进去，接着会跳到AsyncExecutionInterceptor的invoke方法去，把任务放到线程池汇中去执行，然后处理返回值，整个流程就结束了。



# 总结与启示

整个流程就是，首先在项目启动阶段，通过@EnableAsync注解导入一些配置类，最终实例化一个异步注解的后置处理器（AsyncAnnotationBeanPostProcessor），这个后置处理器有自己的切入点和处理逻辑，会拦截项目中所有的bean，如果某个bean符合该后置处理器的切入点，那么SpringBoot会通过AOP生成一个代理对象，生成代理对象时会设置一个回调，回调的内容就是后置处理器中的处理逻辑（实际逻辑就是将异步方法内容放入线程池中执行），并将这个代理对象注入到使用的地方。

当真正调用异步方法时，因为注入的是代理对象，那么调用到异步方法之前会进入之前设置的回调，去执行异步方法内容，执行完毕后会根据不同的返回值类型处理返回值，至此异步方法就执行完毕了。整篇文章还是有点抽象，过程描述不太详细，只是描述每个类的功能，之前想画一下时序图，但是因为方法调用太多画的图会很复杂，所以强烈建议读者在应用启动阶段和方法调用阶段设置断点，一步步Debug会理解得更深刻。

看完@Async源码感触还是挺多的，首先就是源码的类结构的设计真的很优秀，很多思路可以借鉴，引入到自己的项目中。其次看源码我的一个思路就是，从上往下，从父类到子类，父接口到子接口，对于每一个类，确定是如何实例化的，每个属性是如何设置的，有哪些方法，每个方法的每一行是做什么的，最后大致看完以后，设置断点一步步Debug走下去，边Debug边看每个类的结构，这样对某个类就有个整体把握。

当然源码很复杂，有些地方不可能看的很细，在我看@Async源码的时候有些地方也没有理解，比如AOP那一块，以后有机会再慢慢看看，总之看源码慢慢来，看完有收获就可以。
