# Spring - AOP

能在程序运行期间动态的将某段代码切入到指定方法指定位置运行的编程方式。

1. 导入aop模块，Spring-aspects
2. 定义一个业务逻辑类，在业务逻辑运行时将日志进行打印。（方法运行之间、之后、出现异常都进行打印。）
3. 定义一个切面类（LogAspects），动态感知到业务逻辑类的运行。
4. 通知方法：
    1. 前置通知
    2. 后置通知：无论方法正常结束还是异常结束
    3. 返回通知
    4. 异常通知
    5. 环绕通知
5. 给切面类标注何时何地运行，标注@Before,@After,@AfterReturning,@AfterThrowing
6. 将切面类与业务逻辑类加入到容器中。
7. 告诉Spring哪个是切面类（给切面类加一个注解@Aspect，表示当前类是一个切面类）
8. 在配置类中加入@EnableAspectJAutoProxy，开启基于注解的AOP模式

Spring中的很多EnableXXX注解都是用来开启基于注解的XXX功能的。

## AOP原理

### 分析源码基于的例子

例子是一个计算器，源码如下：

bean代码：

    public class MathCalculator {

        public int div(int i,int j) {
            return i/j;
        }
        
    }

配置类代码：

    @Configuration
    @EnableAspectJAutoProxy
    public class MathConfiguration {

        @Bean
        public MathCalculator mathCalculator() {
            return new MathCalculator();
        }
        
        @Bean 
        public LogAspects logAspects() {
            return new LogAspects();
        }
        
    }

切面类代码：

    @Aspect
    public class LogAspects {
        
        @Pointcut("execution(public int cn.edu.ynu.example.example7.MathCalculator.div(int,int))")
        public void pointCut() {}
        
        @Before("pointCut()")
        public void logStart(JoinPoint joinPoint) {
            Object[] args = joinPoint.getArgs();
            System.out.println(joinPoint.getSignature().getDeclaringTypeName()+"除法运行..参数列表是：{"+Arrays.asList(args)+"}");
        }
        @After("pointCut()")
        //JoinPoint必须放在参数表的第一位
        public void logEnd(JoinPoint joinPoint) {
            System.out.println(joinPoint.getSignature().getDeclaringTypeName()+"除法运行结束...");
        }
        @AfterReturning(value="pointCut()",returning="result")
        public void logReturn(JoinPoint joinPoint,int result) {
            System.out.println(joinPoint.getSignature().getDeclaringTypeName()+"除法运行..返回结果是：{}");
        }
        @AfterThrowing(value="pointCut()",throwing="expection")
        public void logException(JoinPoint joinPoint,Exception expection) {
            System.out.println(joinPoint.getSignature().getDeclaringTypeName()+"运行异常..异常信息是：{}"+expection);
        }
    }

测试类代码：

    public class Main {

        public static void main(String[] args) {
            AnnotationConfigApplicationContext context = new AnnotationConfigApplicationContext(MathConfiguration.class);
            MathCalculator cal = context.getBean(MathCalculator.class);
            System.out.println(cal.div(10, 2));
            context.close();
        }
        
    }

### @EnableAspectJAutoProxy

查看源码：

    @Target(ElementType.TYPE)
    @Retention(RetentionPolicy.RUNTIME)
    @Documented
    //通过AspectJAutoProxyRegistrar使Spring容器中注册该类中定义的bean
    @Import(AspectJAutoProxyRegistrar.class)
    public @interface EnableAspectJAutoProxy {

        /**
        * Indicate whether subclass-based (CGLIB) proxies are to be created as opposed
        * to standard Java interface-based proxies. The default is {@code false}.
        */

        //显示指定是否要创建CGLib的代理而不是基于Java接口的代理

        boolean proxyTargetClass() default false;

        /**
        * Indicate that the proxy should be exposed by the AOP framework as a {@code ThreadLocal}
        * for retrieval via the {@link org.springframework.aop.framework.AopContext} class.
        * Off by default, i.e. no guarantees that {@code AopContext} access will work.
        * @since 4.3.1
        */

        //指示代理应由AOP框架作为{@code ThreadLocal}公开，
        //以便通过{@link org.springframework.aop.framework.AopContext}类
        //进行检索。默认情况下，即不保证{@code AopContext} 访问将工作。

        boolean exposeProxy() default false;

    }

这里详细说一下这两个参数：

- proxy-target-class:Spring AOP部分使用JDK动态代理或者CGLIB来为目标对象创建代理。（建议尽量使用JDK的动态代理），如果被代理的目标对象实现了至少一个接口，则会使用JDK动态代理。所有该目标类型实现的接口都将被代理。若该目标对象没有实现任何接口，则创建一个CGLIB代理。如果你希望强制使用CGLIB代理，（例如希望代理目标对象的所有方法，而不只是实现自接口的方法）那也可以。但是需要考虑如下两个问题：
    - 无法通知（advise）Final方法，因为它们不能被覆写。
    - 你需要将CGLIB二进制发行包放在classpath下面。
    这里区分下两种动态代理的区别：
    - JDK动态代理：其代理对象必须是某个接口的实现，它是通过在运行期间创建一个接口的实现类来完成对目标对象的代理。
    - CGLIB代理：实现原理类似于JDK动态代理，只是它在运行期间生成的代理对象是针对目标类扩展的子类。CGLIB是高效的代码生成包，底层是依靠ASM（开源的Java字节码编辑类库）操作字节码实现的，性能比JDK强。
- expose-proxy:有时候目标对象内部的自我调用将无法实施切面中的增强，如下实例：
        
        public interface AService{
            public void a();
            public void b();
        }
        @Service()
        public class AServiceImpl1 implements AService{
            @Transactional(propagation = Propagation.REQUIRED)
            public void a(){
                this.b();
            }
            @Transactional(propagation = Propagation.REQUIRES_NEW)
            public void b(){
            }
        }

    此处的this指向目标对象，因此调用this.b()将不会执行b事务切面，则不会执行事务增强，因此b方法的事务定义“@Transactional(propagation = Propagation.REQUIRES_NEW)”将不会实施，为了解决这个问题，我们可以将expose-proxy属性设置为true。然后将以上代码中的“this.b();”修改为“((AService)AopContext.currentProxy()).b();”即可。通过以上的修改便可以完成对a和b方法的同时增强。
    
由于@EnableAspectJAutoProxy注解包含了@Import(AspectJAutoProxyRegistrar.class)，这里我们查看AspectJAutoProxyRegistrar类的源码：

    class AspectJAutoProxyRegistrar implements ImportBeanDefinitionRegistrar {

        /**
        * Register, escalate, and configure the AspectJ auto proxy creator based on the value
        * of the @{@link EnableAspectJAutoProxy#proxyTargetClass()} attribute on the importing
        * {@code @Configuration} class.
        */
        @Override
        public void registerBeanDefinitions(
                AnnotationMetadata importingClassMetadata, BeanDefinitionRegistry registry) {
            
            //注册一个AspectJAnnotationAutoProxyCreator，根据方法名我们知道该方法的作用是
            //如果必要的话注册AspectJAnnotationAutoProxyCreator类型的bean
            //仔细研究该方法我们知道bean的id为org.springframework.aop.config.internalAutoProxyCreator
            AopConfigUtils.registerAspectJAnnotationAutoProxyCreatorIfNecessary(registry);
            
            //这里获取@EnableAspectJAutoProxy注解的两个属性内容
            //从上面源码可知，该注解具有两个属性：
            //  -proxyTargetClass     默认为false
            //  -exposeProxy          默认为false


            AnnotationAttributes enableAspectJAutoProxy =
                    AnnotationConfigUtils.attributesFor(importingClassMetadata, EnableAspectJAutoProxy.class);

            //根据上述的两个属性为id为
            //org.springframework.aop.config.internalAutoProxyCreator的bean
            //即刚才注册的bean
            //设置属性

            if (enableAspectJAutoProxy != null) {
                if (enableAspectJAutoProxy.getBoolean("proxyTargetClass")) {
                    AopConfigUtils.forceAutoProxyCreatorToUseClassProxying(registry);
                }
                if (enableAspectJAutoProxy.getBoolean("exposeProxy")) {
                    AopConfigUtils.forceAutoProxyCreatorToExposeProxy(registry);
                }
            }
        }

    }

由于`AspectJAutoProxyRegistrar`实现了`ImportBeanDefinitionRegistrar`接口。

该接口的用法是，我们实现这个接口`可以在其中进行注册bean定义，然后通过@Import注解导入到配置类`，具体请查看`@Import`注解源码。

这里给容器中注入一个`AnnotationAwareAspectJAutoProxyCreator`类型的`bean`，`bean的id`是`org.springframework.aop.config.internalAutoProxyCreator`。为什么是`AnnotationAwareAspectJAutoProxyCreator`?我们跳入

     AopConfigUtils.registerAspectJAnnotationAutoProxyCreatorIfNecessary(registry);

这行代码进行仔细查看，经过debug，我们跳入了AopConfigUtils类，查看下面的方法

    @Nullable
	public static BeanDefinition registerAspectJAnnotationAutoProxyCreatorIfNecessary(BeanDefinitionRegistry registry) {
		return registerAspectJAnnotationAutoProxyCreatorIfNecessary(registry, null);
	}

	@Nullable
	public static BeanDefinition registerAspectJAnnotationAutoProxyCreatorIfNecessary(BeanDefinitionRegistry registry,
			@Nullable Object source) {
        
        //这里告诉我们注册了AnnotationAwareAspectJAutoProxyCreator类型的bean

		return registerOrEscalateApcAsRequired(AnnotationAwareAspectJAutoProxyCreator.class, registry, source);
	}

到现在为止，@EnableAspectJAutoProxy注解实现了如下逻辑：
1. `@EnableAspectJAutoProxy`注解中使用`@Import(AspectJAutoProxyRegistrar.class)`为spring-ioc中注册新的bean。
2. `AspectJAutoProxyRegistrar`类的`registerBeanDefinitions()`方法注册了id为`org.springframework.aop.config.internalAutoProxyCreator`的bean，其类型为`AnnotationAwareAspectJAutoProxyCreator`。
3. 根据`@EnableAspectJAutoProxy`中的属性值为`org.springframework.aop.config.internalAutoProxyCreator`这个bean的对应属性赋值，bean中的属性与注解中属性名字相同。

### AnnotationAwareAspectJAutoProxyCreator继承关系

上述步骤中我们为容器注册了一个id为`org.springframework.aop.config.internalAutoProxyCreator`的类型为`AnnotationAwareAspectJAutoProxyCreator`的bean，下面查看`AnnotationAwareAspectJAutoProxyCreator`类的继承关系：

<xmp>
AnnotationAwareAspectJAutoProxyCreator
  ->AspectJAwareAdvisorAutoProxyCreator
   ->AbstractAdvisorAutoProxyCreator    
     ->AbstractAutoProxyCreator
       ->ProxyProcessorSupport
       -<>SmartInstantiationAwareBeanPostProcessor
         ->InstantiationAwareBeanPostProcessor
           ->BeanPostProcessor
       -<>BeanFactoryAware
</xmp>

其中->指向了其父类，-<>表示实现了该接口。下面我们从底层往上查看。

### ProxyProcessorSupport

考察该类的声明：

    public class ProxyProcessorSupport extends ProxyConfig 
    implements Ordered, BeanClassLoaderAware, AopInfrastructureBean

不管如何我们发现这里实现了`Ordered`接口，这里我们只需记住`ProxyProcessorSupport`类实现了`Ordered`接口即可，后面会用到。

### SmartInstantiationAwareBeanPostProcessor

该接口继承了`BeanPostProcessor`接口，我们已知`BeanPostProcessor`接口要求我们实现

    @Nullable
	default Object postProcessBeforeInitialization(Object bean, String beanName) throws BeansException {
		return bean;
	}

    @Nullable
	default Object postProcessAfterInitialization(Object bean, String beanName) throws BeansException {
		return bean;
	}

上述两个接口，这两个接口用于在bean初始化之前和之后进行一系列处理，考察`InstantiationAwareBeanPostProcessor`接口，我们发现需要的实现方法如下：

    @Nullable
	default Object postProcessBeforeInstantiation(Class<?> beanClass, String beanName) throws BeansException {
		return null;
	}

    default boolean postProcessAfterInstantiation(Object bean, String beanName) throws BeansException {
		return true;
	}

    @Nullable
	default PropertyValues postProcessPropertyValues(
			PropertyValues pvs, PropertyDescriptor[] pds, Object bean, String beanName) throws BeansException {

		return pvs;
	}

`SmartInstantiationAwareBeanPostProcessor`接口继承了这两个接口，并且的`SmartInstantiationAwareBeanPostProcessor`注解描述如下：

    Extension of the {@link InstantiationAwareBeanPostProcessor} interface,
    adding a callback for predicting the eventual type of a processed bean.

    扩展{@link InstantiationAwareBeanPostProcessor}接口，添加一个回调函数来预测已处理bean的最终类型。

查看注解我们看到，这里知识用来预测已处理的bean，而具体的处理逻辑还是在父接口中，因此我们还是应该关注父接口要求实现的方法，对于这个接口的分析就到这里。

### BeanFactoryAware:注入了BeanFactory

`为了方便开发者使用Spring内部的bean，Spring为我们提供了许多命名为XXXAware的接口`。`BeanFactoryAware`为·注入了ioc容器中类型为`BeanFactory`类型的bean，这里只是一个接口方法，我们还需要查看子类，来查看其具体实现。

### AbstractAutoProxyCreator

因为`SmartInstantiationAwareBeanPostProcessor`的处理逻辑实际上是在其父接口上的，即`InstantiationAwareBeanPostProcessor`接口，因此我们
考察`AbstractAutoProxyCreator`的`InstantiationAwareBeanPostProcessor`的接口方法，可以看到`InstantiationAwareBeanPostProcessor`接口的接口方法如下：

    //前两个是InstantiationAwareBeanPostProcessor接口特有的方法
    @Override
	public Object postProcessBeforeInstantiation(Class<?> beanClass, String beanName) throws BeansException {
		Object cacheKey = getCacheKey(beanClass, beanName);

		if (!StringUtils.hasLength(beanName) || !this.targetSourcedBeans.contains(beanName)) {
			if (this.advisedBeans.containsKey(cacheKey)) {
				return null;
			}
			if (isInfrastructureClass(beanClass) || shouldSkip(beanClass, beanName)) {
				this.advisedBeans.put(cacheKey, Boolean.FALSE);
				return null;
			}
		}

		// Create proxy here if we have a custom TargetSource.
		// Suppresses unnecessary default instantiation of the target bean:
		// The TargetSource will handle target instances in a custom fashion.
		TargetSource targetSource = getCustomTargetSource(beanClass, beanName);
		if (targetSource != null) {
			if (StringUtils.hasLength(beanName)) {
				this.targetSourcedBeans.add(beanName);
			}
			Object[] specificInterceptors = getAdvicesAndAdvisorsForBean(beanClass, beanName, targetSource);
			Object proxy = createProxy(beanClass, beanName, specificInterceptors, targetSource);
			this.proxyTypes.put(cacheKey, proxy.getClass());
			return proxy;
		}

		return null;
	}

	@Override
	public boolean postProcessAfterInstantiation(Object bean, String beanName) {
		return true;
	}

    //后两个是BeanPostProcessor的接口方法

    @Override
	public Object postProcessBeforeInitialization(Object bean, String beanName) {
		return bean;
	}

	/**
	 * Create a proxy with the configured interceptors if the bean is
	 * identified as one to proxy by the subclass.
	 * @see #getAdvicesAndAdvisorsForBean
	 */
	@Override
	public Object postProcessAfterInitialization(@Nullable Object bean, String beanName) throws BeansException {
		if (bean != null) {
			Object cacheKey = getCacheKey(bean.getClass(), beanName);
			if (!this.earlyProxyReferences.contains(cacheKey)) {
				return wrapIfNecessary(bean, beanName, cacheKey);
			}
		}
		return bean;
	}

除此之外，我们还需要关注`BeanFactoryAware`接口的实现方法：

    //BeanFactoryAware接口的实现
    @Override
	public void setBeanFactory(BeanFactory beanFactory) {
		this.beanFactory = beanFactory;
	}

这里对于这些接口方法的分析我们到后面结合实际代码进行分析，现在只需为这些为止打上断点即可。

### AbstractAdvisorAutoProxyCreator

继续向上分析，找到`BeanFactoryAware`和`InstantiationAwareBeanPostProcessor`的接口实现，并为其打上断点。

    //接口的实现
    @Override
	public void setBeanFactory(BeanFactory beanFactory) {
		super.setBeanFactory(beanFactory);
		if (!(beanFactory instanceof ConfigurableListableBeanFactory)) {
			throw new IllegalArgumentException(
					"AdvisorAutoProxyCreator requires a ConfigurableListableBeanFactory: " + beanFactory);
		}
		initBeanFactory((ConfigurableListableBeanFactory) beanFactory);
	}

### 创建ioc容器

我们已经为我们关心的方法都打上了断点，现在对例子进行调试，首先是创建ioc容器的部分。

1. AnnotationConfigApplicationContext()构造器中创建ioc容器。
    
        public AnnotationConfigApplicationContext(Class<?>... annotatedClasses) {
            //初始化一些基础组件
            this();
            //注册配置类作为bean
            //我们调用AnnotationConfigApplicationContext context = new AnnotationConfigApplicationContext(MathConfiguration.class);方法
            //传入的参数是MathConfiguration.class
            //这里就注册并分析MathConfiguration配置类
            register(annotatedClasses);
            
            //刷新容器，这里包括了创建单例bean等很多操作，
            //断点优先指向这里，
            //因为我们之前使用的是@Import的一个用法来注册bean，
            //这里就用于注册这个bean
            refresh();
        }

2. 进入AnnotationConfigApplicationContext(AbstractApplicationContext).refresh() 方法，查看其内部操作，即刷新容器内部操作，这里有很多的操作，但是我们关心的只有中文标注的：


        @Override
        public void refresh() throws BeansException, IllegalStateException {
            synchronized (this.startupShutdownMonitor) {
                // Prepare this context for refreshing.
                prepareRefresh();

                // Tell the subclass to refresh the internal bean factory.
                ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();

                // Prepare the bean factory for use in this context.
                prepareBeanFactory(beanFactory);

                try {
                    // Allows post-processing of the bean factory in context subclasses.
                    postProcessBeanFactory(beanFactory);

                    // Invoke factory processors registered as beans in the context.
                    invokeBeanFactoryPostProcessors(beanFactory);

                    // Register bean processors that intercept bean creation.
                    //由于AnnotationAwareAspectJAutoProxyCreator实现了BeanPostProcessor接口，所以我们关注这里
                    ，其余的先不用管，这里是为
                    //BeanPostProcessor进行注册

                    registerBeanPostProcessors(beanFactory);

                    // Initialize message source for this context.
                    initMessageSource();

                    // Initialize event multicaster for this context.
                    initApplicationEventMulticaster();

                    // Initialize other special beans in specific context subclasses.
                    onRefresh();

                    // Check for listener beans and register them.
                    registerListeners();

                    // Instantiate all remaining (non-lazy-init) singletons.
                    finishBeanFactoryInitialization(beanFactory);

                    // Last step: publish corresponding event.
                    finishRefresh();
                }

                catch (BeansException ex) {
                    if (logger.isWarnEnabled()) {
                        logger.warn("Exception encountered during context initialization - " +
                                "cancelling refresh attempt: " + ex);
                    }

                    // Destroy already created singletons to avoid dangling resources.
                    destroyBeans();

                    // Reset 'active' flag.
                    cancelRefresh(ex);

                    // Propagate exception to caller.
                    throw ex;
                }

                finally {
                    // Reset common introspection caches in Spring's core, since we
                    // might not ever need metadata for singleton beans anymore...
                    resetCommonCaches();
                }
            }
        }

3. 继续进入AnnotationConfigApplicationContext(AbstractApplicationContext).registerBeanPostProcessors(ConfigurableListableBeanFactory)方法，查看注册BeanPostProcessor类型bean的操作

        /**
        * Instantiate and invoke all registered BeanPostProcessor beans,
        * respecting explicit order if given.
        * <p>Must be called before any instantiation of application beans.
        */

        //这个方法的作用是实例化并且执行所有被注册的BeanPostProcessor bean，如果给定了顺序的话，执行时要保证顺序执行，一定要在应用程序bean实例化之前调用
        //它委托给PostProcessorRegistrationDelegate类来做这项操作
        protected void registerBeanPostProcessors(ConfigurableListableBeanFactory beanFactory) {
            PostProcessorRegistrationDelegate.registerBeanPostProcessors(beanFactory, this);
        }

4. 我们查看PostProcessorRegistrationDelegate.registerBeanPostProcessors(ConfigurableListableBeanFactory, AbstractApplicationContext)这个方法，这里就是注册BeanPostProcessor的真正逻辑

        public static void registerBeanPostProcessors(
                ConfigurableListableBeanFactory beanFactory, AbstractApplicationContext applicationContext) {
            //获取所有的BeanPostProcessor类型的bean的bean名称
            //注意在这里，我们在上面AspectJAutoProxyRegistrar中想要注册的bean id为
            //org.springframework.aop.config.internalAutoProxyCreator的bean
            //已经要被注册了，名字在这个数组里
            String[] postProcessorNames = beanFactory.getBeanNamesForType(BeanPostProcessor.class, true, false);

            // Register BeanPostProcessorChecker that logs an info message when
            // a bean is created during BeanPostProcessor instantiation, i.e. when
            // a bean is not eligible for getting processed by all BeanPostProcessors.
            int beanProcessorTargetCount = beanFactory.getBeanPostProcessorCount() + 1 + postProcessorNames.length;
            beanFactory.addBeanPostProcessor(new BeanPostProcessorChecker(beanFactory, beanProcessorTargetCount));

            //这里根据每个BeanPostProcessor实现的表示优先级的接口进行分类，分为三类：
            //   - 实现了PriorityOrdered接口
            //   - 实现了Ordered接口
            //   - 没有实现任何接口

            // Separate between BeanPostProcessors that implement PriorityOrdered,
            // Ordered, and the rest.
            List<BeanPostProcessor> priorityOrderedPostProcessors = new ArrayList<>();
            List<BeanPostProcessor> internalPostProcessors = new ArrayList<>();
            List<String> orderedPostProcessorNames = new ArrayList<>();
            List<String> nonOrderedPostProcessorNames = new ArrayList<>();
            for (String ppName : postProcessorNames) {
                if (beanFactory.isTypeMatch(ppName, PriorityOrdered.class)) {
                    BeanPostProcessor pp = beanFactory.getBean(ppName, BeanPostProcessor.class);
                    priorityOrderedPostProcessors.add(pp);
                    if (pp instanceof MergedBeanDefinitionPostProcessor) {
                        internalPostProcessors.add(pp);
                    }
                }
                else if (beanFactory.isTypeMatch(ppName, Ordered.class)) {
                    orderedPostProcessorNames.add(ppName);
                }
                else {
                    nonOrderedPostProcessorNames.add(ppName);
                }
            }
            
            // First, register the BeanPostProcessors that implement PriorityOrdered.
            
            //这里分别对上述三类BeanPostProcessor进行排序并且注册
            //顺序与刚才描述的顺序相同
            sortPostProcessors(priorityOrderedPostProcessors, beanFactory);
            registerBeanPostProcessors(beanFactory, priorityOrderedPostProcessors);

            // Next, register the BeanPostProcessors that implement Ordered.
            //之前我们知道ProxyProcessorSupport实现了Ordered接口
            //这里对于id为
            //org.springframework.aop.config.internalAutoProxyCreator
            //的bean的处理逻辑在此。
            List<BeanPostProcessor> orderedPostProcessors = new ArrayList<>();
            for (String ppName : orderedPostProcessorNames) {
                //获取对应id的BeanPostProcessor类型的bean（其实获取不到会创建再获取）
                //断点进入这里

                BeanPostProcessor pp = beanFactory.getBean(ppName, BeanPostProcessor.class);
                //将这个bean添加到结果数组里
                orderedPostProcessors.add(pp);
                //这里不熟悉，暂且不管，也用不到
                if (pp instanceof MergedBeanDefinitionPostProcessor) {
                    internalPostProcessors.add(pp);
                }
            }
            //这里通过Ordered接口为被实现了Oredered接口的BeanPostProcessor类型的bean进行排序
            sortPostProcessors(orderedPostProcessors, beanFactory);
            //将排序后的一整个链进行注册
            registerBeanPostProcessors(beanFactory, orderedPostProcessors);

            // Now, register all regular BeanPostProcessors.
            List<BeanPostProcessor> nonOrderedPostProcessors = new ArrayList<>();
            for (String ppName : nonOrderedPostProcessorNames) {
                BeanPostProcessor pp = beanFactory.getBean(ppName, BeanPostProcessor.class);
                nonOrderedPostProcessors.add(pp);
                if (pp instanceof MergedBeanDefinitionPostProcessor) {
                    internalPostProcessors.add(pp);
                }
            }
            registerBeanPostProcessors(beanFactory, nonOrderedPostProcessors);

            // Finally, re-register all internal BeanPostProcessors.
            sortPostProcessors(internalPostProcessors, beanFactory);
            registerBeanPostProcessors(beanFactory, internalPostProcessors);

            // Re-register post-processor for detecting inner beans as ApplicationListeners,
            // moving it to the end of the processor chain (for picking up proxies etc).
            beanFactory.addBeanPostProcessor(new ApplicationListenerDetector(applicationContext));
        }

5. 继续就是获取和创建BeanPostProcessor类型的bean

    DefaultListableBeanFactory(AbstractBeanFactory).getBean(String, Class<T>)就是获取id为String，类型为Class<T>的bean，方法如下：

        @Override
        public <T> T getBean(String name, @Nullable Class<T> requiredType) throws BeansException {
            return doGetBean(name, requiredType, null, false);
        }

    不管怎样调用的是doGetBean(X,XX,XXX,XXXX)方法，那我们跳入，事实证明你掉进了一个大坑，但是不得不分析这段代码，我们这里只分析我们真正用到的：

        /**
        * Return an instance, which may be shared or independent, of the specified bean.
        * @param name the name of the bean to retrieve
        * @param requiredType the required type of the bean to retrieve
        * @param args arguments to use when creating a bean instance using explicit arguments
        * (only applied when creating a new instance as opposed to retrieving an existing one)
        * @param typeCheckOnly whether the instance is obtained for a type check,
        * not for actual use
        * @return an instance of the bean
        * @throws BeansException if the bean could not be created
        */
        @SuppressWarnings("unchecked")
        protected <T> T doGetBean(final String name, @Nullable final Class<T> requiredType,
                @Nullable final Object[] args, boolean typeCheckOnly) throws BeansException {
            
            //就是对bean id的一个转化
            final String beanName = transformedBeanName(name);
            Object bean;

            // Eagerly check singleton cache for manually registered singletons.
            //注释说了，它是从单例cache中直接拿被注册的单例，这里拿到的肯定为null，直接进else咯

            Object sharedInstance = getSingleton(beanName);
            if (sharedInstance != null && args == null) {
                //其实这里拿到了如果不是重复创建的，就直接把bean返回出去了，但是这里和我们这次的分析没啥关系
                if (logger.isDebugEnabled()) {
                    if (isSingletonCurrentlyInCreation(beanName)) {
                        logger.debug("Returning eagerly cached instance of singleton bean '" + beanName +
                                "' that is not fully initialized yet - a consequence of a circular reference");
                    }
                    else {
                        logger.debug("Returning cached instance of singleton bean '" + beanName + "'");
                    }
                }
                bean = getObjectForBeanInstance(sharedInstance, name, beanName, null);
            }

            else {
                // Fail if we're already creating this bean instance:
                // We're assumably within a circular reference.
                //这里就是分析有没有一个多实例的bean，它正在创建呢，你又创建了一次。
                if (isPrototypeCurrentlyInCreation(beanName)) {
                    throw new BeanCurrentlyInCreationException(beanName);
                }

                // Check if bean definition exists in this factory.
                //注释说检测这个bean定义是否存在于bean工厂
                //如果这个bean定义不在当前的bean工厂，就去查它父类bean工厂里有没有，如果有的话，就调用父类创建
                BeanFactory parentBeanFactory = getParentBeanFactory();
                if (parentBeanFactory != null && !containsBeanDefinition(beanName)) {
                    // Not found -> check parent.
                    String nameToLookup = originalBeanName(name);
                    if (parentBeanFactory instanceof AbstractBeanFactory) {
                        return ((AbstractBeanFactory) parentBeanFactory).doGetBean(
                                nameToLookup, requiredType, args, typeCheckOnly);
                    }
                    else if (args != null) {
                        // Delegation to parent with explicit args.
                        return (T) parentBeanFactory.getBean(nameToLookup, args);
                    }
                    else {
                        // No args -> delegate to standard getBean method.
                        return parentBeanFactory.getBean(nameToLookup, requiredType);
                    }
                }

                if (!typeCheckOnly) {
                    markBeanAsCreated(beanName);
                }
                //经过一系列检查，我们到了创建bean的逻辑
                try {
                    //获取bean定义，并使用MergedBeanDefinition处理，这里对我们这个例子没有影响
                    final RootBeanDefinition mbd = getMergedLocalBeanDefinition(beanName);
                    checkMergedBeanDefinition(mbd, beanName, args);

                    // Guarantee initialization of beans that the current bean depends on.
                    String[] dependsOn = mbd.getDependsOn();
                    if (dependsOn != null) {
                        for (String dep : dependsOn) {
                            //检测下是否有循环依赖
                            if (isDependent(beanName, dep)) {
                                throw new BeanCreationException(mbd.getResourceDescription(), beanName,
                                        "Circular depends-on relationship between '" + beanName + "' and '" + dep + "'");
                            }
                            registerDependentBean(dep, beanName);
                            try {
                                getBean(dep);
                            }
                            catch (NoSuchBeanDefinitionException ex) {
                                throw new BeanCreationException(mbd.getResourceDescription(), beanName,
                                        "'" + beanName + "' depends on missing bean '" + dep + "'", ex);
                            }
                        }
                    }

                    // Create bean instance.
                    if (mbd.isSingleton()) {
                        //如果是单例的话，调用getSingleton()方法获取一个单例
                        sharedInstance = getSingleton(beanName, () -> {
                            try {
                                //然而真正创建bean实例的逻辑在这
                                //不过经过这个我们知道Bean是由Bean工厂通过Bean定义创建来的
                                //其实bean id也是从bean定义里来的
                                return createBean(beanName, mbd, args);
                            }
                            catch (BeansException ex) {
                                // Explicitly remove instance from singleton cache: It might have been put there
                                // eagerly by the creation process, to allow for circular reference resolution.
                                // Also remove any beans that received a temporary reference to the bean.
                                destroySingleton(beanName);
                                throw ex;
                            }
                        });
                        bean = getObjectForBeanInstance(sharedInstance, name, beanName, mbd);
                    }

                    else if (mbd.isPrototype()) {
                        // It's a prototype -> create a new instance.
                        Object prototypeInstance = null;
                        try {
                            beforePrototypeCreation(beanName);
                            prototypeInstance = createBean(beanName, mbd, args);
                        }
                        finally {
                            afterPrototypeCreation(beanName);
                        }
                        bean = getObjectForBeanInstance(prototypeInstance, name, beanName, mbd);
                    }

                    else {
                        String scopeName = mbd.getScope();
                        final Scope scope = this.scopes.get(scopeName);
                        if (scope == null) {
                            throw new IllegalStateException("No Scope registered for scope name '" + scopeName + "'");
                        }
                        try {
                            Object scopedInstance = scope.get(beanName, () -> {
                                beforePrototypeCreation(beanName);
                                try {
                                    return createBean(beanName, mbd, args);
                                }
                                finally {
                                    afterPrototypeCreation(beanName);
                                }
                            });
                            bean = getObjectForBeanInstance(scopedInstance, name, beanName, mbd);
                        }
                        catch (IllegalStateException ex) {
                            throw new BeanCreationException(beanName,
                                    "Scope '" + scopeName + "' is not active for the current thread; consider " +
                                    "defining a scoped proxy for this bean if you intend to refer to it from a singleton",
                                    ex);
                        }
                    }
                }
                catch (BeansException ex) {
                    cleanupAfterBeanCreationFailure(beanName);
                    throw ex;
                }
            }

            // Check if required type matches the type of the actual bean instance.
            if (requiredType != null && !requiredType.isInstance(bean)) {
                try {
                    T convertedBean = getTypeConverter().convertIfNecessary(bean, requiredType);
                    if (convertedBean == null) {
                        throw new BeanNotOfRequiredTypeException(name, requiredType, bean.getClass());
                    }
                    return convertedBean;
                }
                catch (TypeMismatchException ex) {
                    if (logger.isDebugEnabled()) {
                        logger.debug("Failed to convert bean '" + name + "' to required type '" +
                                ClassUtils.getQualifiedName(requiredType) + "'", ex);
                    }
                    throw new BeanNotOfRequiredTypeException(name, requiredType, bean.getClass());
                }
            }
            return (T) bean;
        }

    继续调用getSingleton()方法：

        public Object getSingleton(String beanName, ObjectFactory<?> singletonFactory) {
            Assert.notNull(beanName, "Bean name must not be null");
            synchronized (this.singletonObjects) {
                Object singletonObject = this.singletonObjects.get(beanName);
                //如果能从singletonObjects中拿出来就返回
                //它就类似一个缓存
                if (singletonObject == null) {
                    //如果拿不到就做一些检测，例如是否正在创建等等。然后检测后，如果检测通过就创建一个bean。
                    if (this.singletonsCurrentlyInDestruction) {
                        throw new BeanCreationNotAllowedException(beanName,
                                "Singleton bean creation not allowed while singletons of this factory are in destruction " +
                                "(Do not request a bean from a BeanFactory in a destroy method implementation!)");
                    }
                    if (logger.isDebugEnabled()) {
                        logger.debug("Creating shared instance of singleton bean '" + beanName + "'");
                    }
                    beforeSingletonCreation(beanName);
                    boolean newSingleton = false;
                    boolean recordSuppressedExceptions = (this.suppressedExceptions == null);
                    if (recordSuppressedExceptions) {
                        this.suppressedExceptions = new LinkedHashSet<>();
                    }
                    try {
                        //创建一个单例
                        //这里调用的就是我们刚才通过lamda表达式传入的函数
                        singletonObject = singletonFactory.getObject();
                        newSingleton = true;
                    }
                    catch (IllegalStateException ex) {
                        // Has the singleton object implicitly appeared in the meantime ->
                        // if yes, proceed with it since the exception indicates that state.
                        singletonObject = this.singletonObjects.get(beanName);
                        if (singletonObject == null) {
                            throw ex;
                        }
                    }
                    catch (BeanCreationException ex) {
                        if (recordSuppressedExceptions) {
                            for (Exception suppressedException : this.suppressedExceptions) {
                                ex.addRelatedCause(suppressedException);
                            }
                        }
                        throw ex;
                    }
                    finally {
                        if (recordSuppressedExceptions) {
                            this.suppressedExceptions = null;
                        }
                        afterSingletonCreation(beanName);
                    }
                    if (newSingleton) {
                        addSingleton(beanName, singletonObject);
                    }
                }
                return singletonObject;
            }
        }

    接着调用createBean():

        /**
        * Central method of this class: creates a bean instance,
        * populates the bean instance, applies post-processors, etc.
        * @see #doCreateBean
        */
        @Override
        protected Object createBean(String beanName, RootBeanDefinition mbd, @Nullable Object[] args)
                throws BeanCreationException {

            if (logger.isDebugEnabled()) {
                logger.debug("Creating instance of bean '" + beanName + "'");
            }
            RootBeanDefinition mbdToUse = mbd;

            // Make sure bean class is actually resolved at this point, and
            // clone the bean definition in case of a dynamically resolved Class
            // which cannot be stored in the shared merged bean definition.
            //这里就是保证Bean definition不会出什么问题
            Class<?> resolvedClass = resolveBeanClass(mbd, beanName);
            if (resolvedClass != null && !mbd.hasBeanClass() && mbd.getBeanClassName() != null) {
                mbdToUse = new RootBeanDefinition(mbd);
                mbdToUse.setBeanClass(resolvedClass);
            }

            // Prepare method overrides.
            try {
                mbdToUse.prepareMethodOverrides();
            }
            catch (BeanDefinitionValidationException ex) {
                throw new BeanDefinitionStoreException(mbdToUse.getResourceDescription(),
                        beanName, "Validation of method overrides failed", ex);
            }

            try {
                // Give BeanPostProcessors a chance to return a proxy instead of the target bean instance.
                //这里可以使用InstantiationAwareBeanPostProcessors返回一个代理，如果有的话
                //这样就不用执行一般的创建步骤了
                Object bean = resolveBeforeInstantiation(beanName, mbdToUse);
                if (bean != null) {
                    return bean;
                }
            }
            catch (Throwable ex) {
                throw new BeanCreationException(mbdToUse.getResourceDescription(), beanName,
                        "BeanPostProcessor before instantiation of bean failed", ex);
            }

            try {
                //创建实例
                Object beanInstance = doCreateBean(beanName, mbdToUse, args);
                if (logger.isDebugEnabled()) {
                    logger.debug("Finished creating instance of bean '" + beanName + "'");
                }
                return beanInstance;
            }
            catch (BeanCreationException | ImplicitlyAppearedSingletonException ex) {
                // A previously detected exception with proper bean creation context already,
                // or illegal singleton state to be communicated up to DefaultSingletonBeanRegistry.
                throw ex;
            }
            catch (Throwable ex) {
                throw new BeanCreationException(
                        mbdToUse.getResourceDescription(), beanName, "Unexpected exception during bean creation", ex);
            }
        }

    查看doCreateBean()方法：

        protected Object doCreateBean(final String beanName, final RootBeanDefinition mbd, final @Nullable Object[] args)
                throws BeanCreationException {

            // Instantiate the bean.
            BeanWrapper instanceWrapper = null;
            //实例化bean
            if (mbd.isSingleton()) {
                instanceWrapper = this.factoryBeanInstanceCache.remove(beanName);
            }
            if (instanceWrapper == null) {
                //创建一个空实例，啥都没有那种
                instanceWrapper = createBeanInstance(beanName, mbd, args);
            }
            final Object bean = instanceWrapper.getWrappedInstance();
            Class<?> beanType = instanceWrapper.getWrappedClass();
            if (beanType != NullBean.class) {
                mbd.resolvedTargetType = beanType;
            }

            // Allow post-processors to modify the merged bean definition.
            //使用MergedBeanDefinitionPostProcessor处理
            synchronized (mbd.postProcessingLock) {
                if (!mbd.postProcessed) {
                    try {
                        applyMergedBeanDefinitionPostProcessors(mbd, beanType, beanName);
                    }
                    catch (Throwable ex) {
                        throw new BeanCreationException(mbd.getResourceDescription(), beanName,
                                "Post-processing of merged bean definition failed", ex);
                    }
                    mbd.postProcessed = true;
                }
            }

            // Eagerly cache singletons to be able to resolve circular references
            // even when triggered by lifecycle interfaces like BeanFactoryAware.
            boolean earlySingletonExposure = (mbd.isSingleton() && this.allowCircularReferences &&
                    isSingletonCurrentlyInCreation(beanName));
            if (earlySingletonExposure) {
                if (logger.isDebugEnabled()) {
                    logger.debug("Eagerly caching bean '" + beanName +
                            "' to allow for resolving potential circular references");
                }
                addSingletonFactory(beanName, () -> getEarlyBeanReference(beanName, mbd, bean));
            }

            // Initialize the bean instance.
            Object exposedObject = bean;
            try {
                //初始化基本参数
                populateBean(beanName, mbd, instanceWrapper);
                //调用初始化方法
                exposedObject = initializeBean(beanName, exposedObject, mbd);
            }
            catch (Throwable ex) {
                if (ex instanceof BeanCreationException && beanName.equals(((BeanCreationException) ex).getBeanName())) {
                    throw (BeanCreationException) ex;
                }
                else {
                    throw new BeanCreationException(
                            mbd.getResourceDescription(), beanName, "Initialization of bean failed", ex);
                }
            }

            if (earlySingletonExposure) {
                Object earlySingletonReference = getSingleton(beanName, false);
                if (earlySingletonReference != null) {
                    if (exposedObject == bean) {
                        exposedObject = earlySingletonReference;
                    }
                    else if (!this.allowRawInjectionDespiteWrapping && hasDependentBean(beanName)) {
                        String[] dependentBeans = getDependentBeans(beanName);
                        Set<String> actualDependentBeans = new LinkedHashSet<>(dependentBeans.length);
                        for (String dependentBean : dependentBeans) {
                            if (!removeSingletonIfCreatedForTypeCheckOnly(dependentBean)) {
                                actualDependentBeans.add(dependentBean);
                            }
                        }
                        if (!actualDependentBeans.isEmpty()) {
                            throw new BeanCurrentlyInCreationException(beanName,
                                    "Bean with name '" + beanName + "' has been injected into other beans [" +
                                    StringUtils.collectionToCommaDelimitedString(actualDependentBeans) +
                                    "] in its raw version as part of a circular reference, but has eventually been " +
                                    "wrapped. This means that said other beans do not use the final version of the " +
                                    "bean. This is often the result of over-eager type matching - consider using " +
                                    "'getBeanNamesOfType' with the 'allowEagerInit' flag turned off, for example.");
                        }
                    }
                }
            }

            // Register bean as disposable.
            try {
                registerDisposableBeanIfNecessary(beanName, bean, mbd);
            }
            catch (BeanDefinitionValidationException ex) {
                throw new BeanCreationException(
                        mbd.getResourceDescription(), beanName, "Invalid destruction signature", ex);
            }

            return exposedObject;
        }

    调用初始化方法：

        protected Object initializeBean(final String beanName, final Object bean, @Nullable RootBeanDefinition mbd) {
            if (System.getSecurityManager() != null) {
                AccessController.doPrivileged((PrivilegedAction<Object>) () -> {
                    invokeAwareMethods(beanName, bean);
                    return null;
                }, getAccessControlContext());
            }
            else {
                //执行XXXAware接口方法
                invokeAwareMethods(beanName, bean);
            }

            Object wrappedBean = bean;
            if (mbd == null || !mbd.isSynthetic()) {
                //调用各个BeanPostProcessor中的postProcessorsBeforeInitialization方法
                wrappedBean = applyBeanPostProcessorsBeforeInitialization(wrappedBean, beanName);
            }

            try {
                //执行初始化方法
                invokeInitMethods(beanName, wrappedBean, mbd);
            }
            catch (Throwable ex) {
                throw new BeanCreationException(
                        (mbd != null ? mbd.getResourceDescription() : null),
                        beanName, "Invocation of init method failed", ex);
            }
            if (mbd == null || !mbd.isSynthetic()) {
                 //调用各个BeanPostProcessor中的postProcessorsAfterInitialization方法
                wrappedBean = applyBeanPostProcessorsAfterInitialization(wrappedBean, beanName);
            }

            return wrappedBean;
        }

    这个invokeAwareMethods()方法就很友善了，调用逻辑极其简单，实现了XXXAware()就调用对应的set方法：

        private void invokeAwareMethods(final String beanName, final Object bean) {
            if (bean instanceof Aware) {
                if (bean instanceof BeanNameAware) {
                    ((BeanNameAware) bean).setBeanName(beanName);
                }
                if (bean instanceof BeanClassLoaderAware) {
                    ClassLoader bcl = getBeanClassLoader();
                    if (bcl != null) {
                        ((BeanClassLoaderAware) bean).setBeanClassLoader(bcl);
                    }
                }
                if (bean instanceof BeanFactoryAware) {
                    ((BeanFactoryAware) bean).setBeanFactory(AbstractAutowireCapableBeanFactory.this);
                }
            }
        }

### 调用 AnnotationAwareAspectJAutoProxyCreator的setBeanFactory()方法

由于AnnotationAwareAspectJAutoProxyCreator实现了BeanFactoryAware接口，因此就调用了它，
但是由于AnnotationAwareAspectJAutoProxyCreator没有setBeanFactory()方法，就要调用其父类AbstractAdvisorAutoProxyCreator的setBeanFactory()方法:

    @Override
    public void setBeanFactory(BeanFactory beanFactory) {
        super.setBeanFactory(beanFactory);
        //查看这行代码你发现
        //这就意味着你必须把@EnableAspectJAutoProxy加到配置类上
        if (!(beanFactory instanceof ConfigurableListableBeanFactory)) {
            throw new IllegalArgumentException(
                    "AdvisorAutoProxyCreator requires a ConfigurableListableBeanFactory: " + beanFactory);
        }
        initBeanFactory((ConfigurableListableBeanFactory) beanFactory);
    }

他调用了父类的setBeanFactory()方法，父类的方法就很简单：

    @Override
    public void setBeanFactory(BeanFactory beanFactory) {
        this.beanFactory = beanFactory;
    }

然后该方法调用了initBeanFactory()方法，然后我们发现它的子类AnnotationAwareAspectJAutoProxyCreator具有这个方法：

    @Override
	protected void initBeanFactory(ConfigurableListableBeanFactory beanFactory) {
		super.initBeanFactory(beanFactory);
		if (this.aspectJAdvisorFactory == null) {
			this.aspectJAdvisorFactory = new ReflectiveAspectJAdvisorFactory(beanFactory);
		}
		this.aspectJAdvisorsBuilder =
				new BeanFactoryAspectJAdvisorsBuilderAdapter(beanFactory, this.aspectJAdvisorFactory);
	}

然后又调用了父类的initBeanFactory()方法：

    protected void initBeanFactory(ConfigurableListableBeanFactory beanFactory) {
		this.advisorRetrievalHelper = new BeanFactoryAdvisorRetrievalHelperAdapter(beanFactory);
	}

这两个函数为其初始化了3个参数：`aspectJAdvisorFactory`、`aspectJAdvisorBuilder`、`advisorRetrievalHelper`

然后剩下的代码就暂时先不管它。这里我们总结一下，经历了上述那么多步骤，我们`成功注册了一个类型为AnnotationAwareAspectJAutoProxyCreator的bean`，并且`为这个bean注入了DefaultListableBeanFactory的工厂bean`，并且`初始化了aspectJAdvisorFactory、aspectJAdvisorBuilder、advisorRetrievalHelper这三个参数`。

这里分析下上述三个参数到底是什么：

1. `aspectJAdvisorFactory`:`ReflectiveAspectJAdvisorFactory`类型，它的注解告诉我们这个类是`用来创建基于AspectJ 5注解语法的Spring aop advisor，并且在调用对应的通知方法时使用的是反射`。
2. `aspectJAdvisorBuilder`:`BeanFactoryAspectJAdvisorsBuilderAdapter`类型，它的注解只告诉我们它是`BeanFactoryAspectJAdvisorsBuilderAdapter`的子类，被`AnnotationAwareAspectJAutoProxyCreator`所使用，查看`BeanFactoryAspectJAdvisorsBuilderAdapter`注解，我们发现`它用来检索BeanFactory中被@AspectJ修饰了的bean，并基于他们创建Spring Advisors。`
3. `advisorRetrievalHelper`:`BeanFactoryAdvisorRetrievalHelperAdapter`类型，查看注释同样没有什么用处，只能查看父类`BeanFactoryAdvisorRetrievalHelper`注释，父类注释说`它用来帮助检索BeanFactory中的标准Spring Advisors`。

### 创建包含切点的类的bean

我们已经为容器中添加进入了`AnnotationAwareAutoProxyCreator`这个`BeanPostProcessor`，现在我们可以创建需要切面的`MathCalculator bean`，查看这个`AnnotationAwareAutoProxyCreator`到底做了什么？

创建其他的bean的操作是在AnnotationConfigApplicationContext类的refresh()方法中做的，本例中代码如下：

	public AnnotationConfigApplicationContext(Class<?>... annotatedClasses) {
		this();
		register(annotatedClasses);
        //初始化bean，其实这里上面已经说过了
		refresh();
	}

    public void refresh() throws BeansException, IllegalStateException {
		synchronized (this.startupShutdownMonitor) {
			// Prepare this context for refreshing.
			prepareRefresh();

			// Tell the subclass to refresh the internal bean factory.
			ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();

			// Prepare the bean factory for use in this context.
			prepareBeanFactory(beanFactory);

			try {
				// Allows post-processing of the bean factory in context subclasses.
				postProcessBeanFactory(beanFactory);

				// Invoke factory processors registered as beans in the context.
				invokeBeanFactoryPostProcessors(beanFactory);

				// Register bean processors that intercept bean creation.

                            //这些就是刚才说过的创建并注册各种BeanPostProcessor类型的bean,
                            //包括刚才注册的类型为AnnotationAwareAutoProxyCreator
				registerBeanPostProcessors(beanFactory);

				// Initialize message source for this context.
				initMessageSource();

				// Initialize event multicaster for this context.
				initApplicationEventMulticaster();

				// Initialize other special beans in specific context subclasses.
				onRefresh();

				// Check for listener beans and register them.
				registerListeners();

				// Instantiate all remaining (non-lazy-init) singletons.

                            //剩下的bean都在这里进行实例化，这里就是本次分析的重点
				finishBeanFactoryInitialization(beanFactory);

				// Last step: publish corresponding event.
				finishRefresh();
			}

			catch (BeansException ex) {
				if (logger.isWarnEnabled()) {
					logger.warn("Exception encountered during context initialization - " +
							"cancelling refresh attempt: " + ex);
				}

				// Destroy already created singletons to avoid dangling resources.
				destroyBeans();

				// Reset 'active' flag.
				cancelRefresh(ex);

				// Propagate exception to caller.
				throw ex;
			}

			finally {
				// Reset common introspection caches in Spring's core, since we
				// might not ever need metadata for singleton beans anymore...
				resetCommonCaches();
			}
		}
	}

    protected void finishBeanFactoryInitialization(ConfigurableListableBeanFactory beanFactory) {
		// Initialize conversion service for this context.
		if (beanFactory.containsBean(CONVERSION_SERVICE_BEAN_NAME) &&
				beanFactory.isTypeMatch(CONVERSION_SERVICE_BEAN_NAME, ConversionService.class)) {
			beanFactory.setConversionService(
					beanFactory.getBean(CONVERSION_SERVICE_BEAN_NAME, ConversionService.class));
		}

		// Register a default embedded value resolver if no bean post-processor
		// (such as a PropertyPlaceholderConfigurer bean) registered any before:
		// at this point, primarily for resolution in annotation attribute values.
		if (!beanFactory.hasEmbeddedValueResolver()) {
			beanFactory.addEmbeddedValueResolver(strVal -> getEnvironment().resolvePlaceholders(strVal));
		}

		// Initialize LoadTimeWeaverAware beans early to allow for registering their transformers early.
		String[] weaverAwareNames = beanFactory.getBeanNamesForType(LoadTimeWeaverAware.class, false, false);
		for (String weaverAwareName : weaverAwareNames) {
			getBean(weaverAwareName);
		}

		// Stop using the temporary ClassLoader for type matching.
		beanFactory.setTempClassLoader(null);

		// Allow for caching all bean definition metadata, not expecting further changes.
		beanFactory.freezeConfiguration();

		// Instantiate all remaining (non-lazy-init) singletons.
        这里实例化所有剩下的非懒加载的单例bean
		beanFactory.preInstantiateSingletons();
	}

这里的代码就比较容易读懂，我们直接进入beanFactory.preInstantiateSingletons()的具体逻辑：

    @Override
	public void preInstantiateSingletons() throws BeansException {
		if (this.logger.isDebugEnabled()) {
			this.logger.debug("Pre-instantiating singletons in " + this);
		}

		// Iterate over a copy to allow for init methods which in turn register new bean definitions.
		// While this may not be part of the regular factory bootstrap, it does otherwise work fine.
            //创建一个存储bean定义的list的副本
            //虽然前面已经有了冻结配置让bean定义不再动态变化，这样做更保证了bean定义不在变化的事实。
		List<String> beanNames = new ArrayList<>(this.beanDefinitionNames);
		// Trigger initialization of all non-lazy singleton beans...
        //挨个遍历beanName，然后创建
		for (String beanName : beanNames) {
			RootBeanDefinition bd = getMergedLocalBeanDefinition(beanName);
			if (!bd.isAbstract() && bd.isSingleton() && !bd.isLazyInit()) {
                //非抽象的非懒加载的单例进行实例化
				if (isFactoryBean(beanName)) {
                    //如果是工厂bean
                    //首先创建一个工厂bean实例
                    //注意这里的FACTORY_BEAN_PREFIX代表的是&
                    //这也就是为啥工厂bean的id是以&开头的
					Object bean = getBean(FACTORY_BEAN_PREFIX + beanName);
					if (bean instanceof FactoryBean) {
						final FactoryBean<?> factory = (FactoryBean<?>) bean;
						boolean isEagerInit;
						if (System.getSecurityManager() != null && factory instanceof SmartFactoryBean) {
							isEagerInit = AccessController.doPrivileged((PrivilegedAction<Boolean>)
											((SmartFactoryBean<?>) factory)::isEagerInit,
									getAccessControlContext());
						}
						else {
							isEagerInit = (factory instanceof SmartFactoryBean &&
									((SmartFactoryBean<?>) factory).isEagerInit());
						}
						if (isEagerInit) {
                            //然后创建工厂bean指定的实例
							getBean(beanName);
						}
					}
				}
				else {
                    //创建实例
					getBean(beanName);
				}
			}
		}

		// Trigger post-initialization callback for all applicable beans...
		for (String beanName : beanNames) {
			Object singletonInstance = getSingleton(beanName);
			if (singletonInstance instanceof SmartInitializingSingleton) {
				final SmartInitializingSingleton smartSingleton = (SmartInitializingSingleton) singletonInstance;
				if (System.getSecurityManager() != null) {
					AccessController.doPrivileged((PrivilegedAction<Object>) () -> {
						smartSingleton.afterSingletonsInstantiated();
						return null;
					}, getAccessControlContext());
				}
				else {
					smartSingleton.afterSingletonsInstantiated();
				}
			}
		}
	}

接下来就是调用bean工厂的getBean()方法了：

    @Override
	public Object getBean(String name) throws BeansException {
		return doGetBean(name, null, null, false);
	}

这里接下来的步骤我们就很熟悉了，这里我们只挑重点的说：

经过上一部分的分析我们知道AnnotationAwareAutoProxyCreator实现了SmartInstantiationAwareBeanPostProcessor接口，该接口具体处理逻辑的方法有4个：

    //创建bean实例前调用，返回结果是该bean的实例结果
    //如果能通过该方法得到bean实例，就不去进行常规的创建操作
	@Nullable
	default Object postProcessBeforeInstantiation(Class<?> beanClass, String beanName) throws BeansException {
		return null;
	}
    //创建bean实例之后，填充基本属性之前，用于更改bean状态(例如field注入)
    default boolean postProcessAfterInstantiation(Object bean, String beanName) throws BeansException {
		return true;
	}
    //创建bean之后实例化之前调用
    //常规方式创建bean并且初始化实例之前调用
    @Nullable
	default Object postProcessBeforeInitialization(Object bean, String beanName) throws BeansException {
		return bean;
	}
    //创建实例后初始化实例之后调用，这里有两个调用的地方：
    //1. postProcessBeforeInstantiation之后调用
    //2. 常规创建bean并且初始化实例后调用
    @Nullable
	default Object postProcessAfterInitialization(Object bean, String beanName) throws BeansException {
		return bean;
	}

由于是MathCalculator类需要切面，因此我们直接看在创建MathCalculator这个bean时的上述4个方法。创建bean之前优先调用：

    @Override
	public Object postProcessBeforeInstantiation(Class<?> beanClass, String beanName) throws BeansException {
        //获取cache中的cacheKey
		Object cacheKey = getCacheKey(beanClass, beanName);
        //如果这个bean不应该添加代理返回null。
		if (!StringUtils.hasLength(beanName) || !this.targetSourcedBeans.contains(beanName)) {
			if (this.advisedBeans.containsKey(cacheKey)) {
				return null;
			}
			if (isInfrastructureClass(beanClass) || shouldSkip(beanClass, beanName)) {
				this.advisedBeans.put(cacheKey, Boolean.FALSE);
				return null;
			}
		}

		// Create proxy here if we have a custom TargetSource.
		// Suppresses unnecessary default instantiation of the target bean:
		// The TargetSource will handle target instances in a custom fashion.
        //通过指定的Factory直接创建bean
        //当然这里是null
		TargetSource targetSource = getCustomTargetSource(beanClass, beanName);
		if (targetSource != null) {
			if (StringUtils.hasLength(beanName)) {
				this.targetSourcedBeans.add(beanName);
			}
            //为其创建代理
			Object[] specificInterceptors = getAdvicesAndAdvisorsForBean(beanClass, beanName, targetSource);
			Object proxy = createProxy(beanClass, beanName, specificInterceptors, targetSource);
			this.proxyTypes.put(cacheKey, proxy.getClass());
			return proxy;
		}

		return null;
	}

创建实例后调用postProcessBeforeInitialization()方法，明显这个方法是来打酱油的：

    @Override
	public Object postProcessBeforeInitialization(Object bean, String beanName) {
		return bean;
	}

初始化之后调用postProcessAfterInitialization()方法：

    @Override
	public Object postProcessAfterInitialization(@Nullable Object bean, String beanName) throws BeansException {
		if (bean != null) {
            //从缓存中获取cacheKey
			Object cacheKey = getCacheKey(bean.getClass(), beanName);
			if (!this.earlyProxyReferences.contains(cacheKey)) {
                //这里时真正的创建代理的逻辑
				return wrapIfNecessary(bean, beanName, cacheKey);
			}
		}
		return bean;
	}

下面分析wrapIfNecessary()方法的源码：

    /**
	 * Wrap the given bean if necessary, i.e. if it is eligible for being proxied.
	 * @param bean the raw bean instance
	 * @param beanName the name of the bean
	 * @param cacheKey the cache key for metadata access
	 * @return a proxy wrapping the bean, or the raw bean instance as-is
	 */
     //如果这个bean具备被代理的资格，那么就创建代理并包装它
	protected Object wrapIfNecessary(Object bean, String beanName, Object cacheKey) {
        //如果targetSourcedBeans有这个bean的那么就直接返回
		if (StringUtils.hasLength(beanName) && this.targetSourcedBeans.contains(beanName)) {
			return bean;
		}
		if (Boolean.FALSE.equals(this.advisedBeans.get(cacheKey))) {
			return bean;
		}
        //isInfrastructureClass()函数用于返回给定的bean类是否表示永远不应该代理的基础结构类。
        //shouldSkip()用来判断是否该bean应该被这个post-processor自动代理
        //反正基础结构类和切面类都不会被允许被自动代理
        //所以不能有切面的切面
		if (isInfrastructureClass(bean.getClass()) || shouldSkip(bean.getClass(), beanName)) {
			this.advisedBeans.put(cacheKey, Boolean.FALSE);
			return bean;
		}

		// Create proxy if we have advice.
		// 创建Proxy
        // 首先根据bean类型和bean名称找到所有可用的Advisor
        Object[] specificInterceptors = getAdvicesAndAdvisorsForBean(bean.getClass(), beanName, null);
        //如果获取到的Advisor数组不为空
		if (specificInterceptors != DO_NOT_PROXY) {
            //表示这个cacheKey的bean被创建代理
			this.advisedBeans.put(cacheKey, Boolean.TRUE);
            //为对应bean创建代理
			Object proxy = createProxy(
					bean.getClass(), beanName, specificInterceptors, new SingletonTargetSource(bean));
            //将对应代理进行存储
			this.proxyTypes.put(cacheKey, proxy.getClass());
            return proxy;
		}
        //否则标识该类没有被创建代理
		this.advisedBeans.put(cacheKey, Boolean.FALSE);
		return bean;
	}

我们优先考察一下isInfrastructureClass()方法：

    //AnnotationAwareAspectJAutoProxyCreator实现
    @Override
	protected boolean isInfrastructureClass(Class<?> beanClass) {
		// Previously we setProxyTargetClass(true) in the constructor, but that has too
		// broad an impact. Instead we now override isInfrastructureClass to avoid proxying
		// aspects. I'm not entirely happy with that as there is no good reason not
		// to advise aspects, except that it causes advice invocation to go through a
		// proxy, and if the aspect implements e.g the Ordered interface it will be
		// proxied by that interface and fail at runtime as the advice method is not
		// defined on the interface. We could potentially relax the restriction about
		// not advising aspects in the future.
            //调用父类方法判断该类是否是基础结构类并且判断该类是否是切面类
		return (super.isInfrastructureClass(beanClass) ||
				(this.aspectJAdvisorFactory != null && this.aspectJAdvisorFactory.isAspect(beanClass)));
	}
    //默认实现
    //用于返回给定的bean类是否表示永远不应该代理的基础结构类。
    //默认实现认为Advices, Advisors,AopInfrastructureBeans也是基础结构类
    protected boolean isInfrastructureClass(Class<?> beanClass) {
        //判定该类是不是Advice、Advisor、Ponitcut、AopInfrastructureBean的子类
		boolean retVal = Advice.class.isAssignableFrom(beanClass) ||
				Pointcut.class.isAssignableFrom(beanClass) ||
				Advisor.class.isAssignableFrom(beanClass) ||
				AopInfrastructureBean.class.isAssignableFrom(beanClass);
		if (retVal && logger.isTraceEnabled()) {
			logger.trace("Did not attempt to auto-proxy infrastructure class [" + beanClass.getName() + "]");
		}
		return retVal;
	}
然后是canSkip()方法，这个方法的调用关系比较复杂：

    @Override
	protected boolean shouldSkip(Class<?> beanClass, String beanName) {
		// TODO: Consider optimization by caching the list of the aspect names
        //获取候选的Advisors
		List<Advisor> candidateAdvisors = findCandidateAdvisors();
		for (Advisor advisor : candidateAdvisors) {
            //如果候选的Advisors的切面名称和bean名称相同则跳过
            //反正你是切面我就不代理你
			if (advisor instanceof AspectJPointcutAdvisor &&
					((AspectJPointcutAdvisor) advisor).getAspectName().equals(beanName)) {
				return true;
			}
		}
        //调用父类方法
		return super.shouldSkip(beanClass, beanName);
	}

    //父类方法
    //父类方法默认返回false
    protected boolean shouldSkip(Class<?> beanClass, String beanName) {
		return false;
	}

这里有个重要的方法`findCandidateAdvisors()`。这里我们可以分析下这个方法，由于上述方法是在`AspectJAwareAdvisorAutoProxyCreator`中调用的，这里调用`findCandidateAdvisors()`方法，我们先查看它的子类有没有该方法的实现，首先我们发现`AnnotationAwareAspectJAutoProxyCreator`类中有一个实现：

    @Override
	protected List<Advisor> findCandidateAdvisors() {
		// Add all the Spring advisors found according to superclass rules.
		List<Advisor> advisors = super.findCandidateAdvisors();
		// Build Advisors for all AspectJ aspects in the bean factory.
		if (this.aspectJAdvisorsBuilder != null) {
            //这里就会注册并创建所有的Advisor
			advisors.addAll(this.aspectJAdvisorsBuilder.buildAspectJAdvisors());
		}
		return advisors;
	}

首先是通过父类获得候选的`Advisors`，但是除了这个还创建了每个`Advisor`。这里就使用了注册`AnnotationAwareAspectJAutoProxyCreator这个bean`时初始化的一个对象：`aspectJAdvisorsBuilder`。这里我们继续查看其父类的`findCandidateAdvisors()`方法。这里跟踪到其父类`AbstractAdvisorAutoProxyCreator`的`findCandidateAdvisors()`方法。

    protected List<Advisor> findCandidateAdvisors() {
		Assert.state(this.advisorRetrievalHelper != null, "No BeanFactoryAdvisorRetrievalHelper available");
        //使用advisorRetrievalHelper来获取所有候选的advisor
		return this.advisorRetrievalHelper.findAdvisorBeans();
	}

这里就又使用了我们初始化的另一个对象`advisorRetrievalHelper`，具体代码就不分析了，如果感兴趣，自己debug进去看看也是可以的（贼多，我就不看了啊）。

然后这里还有两个盲区，首先

    Object[] specificInterceptors = getAdvicesAndAdvisorsForBean(bean.getClass(), beanName, null);

这个方法到底是怎么找到的可用的Advisor呢？

其次，到底创建代理的过程是怎样的呢？

我们先解决第一个问题，跟踪`getAdvicesAndAdvisorsForBean()`方法，我们在`AbstractAutoProxyCreator`的子类和其本身中都没有找到它，只有在其父类`AbstractAdvisorAutoProxyCreator`中找到了，其代码为：

    @Override
	@Nullable
	protected Object[] getAdvicesAndAdvisorsForBean(
			Class<?> beanClass, String beanName, @Nullable TargetSource targetSource) {
    
		//这里就是调用了findEligibleAdvisors方法获取了所有可用的Advisor，然后将其转化为数组
        List<Advisor> advisors = findEligibleAdvisors(beanClass, beanName);
		if (advisors.isEmpty()) {
			return DO_NOT_PROXY;
		}
		return advisors.toArray();
	}

我们继续寻找findEligibleAdvisors()方法，这个方法还在这个类里：

    /**
	 * Find all eligible Advisors for auto-proxying this class.
	 * @param beanClass the clazz to find advisors for
	 * @param beanName the name of the currently proxied bean
	 * @return the empty List, not {@code null},
	 * if there are no pointcuts or interceptors
	 * @see #findCandidateAdvisors
	 * @see #sortAdvisors
	 * @see #extendAdvisors
	 */
	protected List<Advisor> findEligibleAdvisors(Class<?> beanClass, String beanName) {
        //这个方法先获取到所有的候选Advisor
		List<Advisor> candidateAdvisors = findCandidateAdvisors();
        //然后根据bean类型和bean名称进行筛选
		List<Advisor> eligibleAdvisors = findAdvisorsThatCanApply(candidateAdvisors, beanClass, beanName);
        //这里是一个后处理方法，可以子类选择实现
		extendAdvisors(eligibleAdvisors);
		if (!eligibleAdvisors.isEmpty()) {
            //对所有的Advisor进行排序
			eligibleAdvisors = sortAdvisors(eligibleAdvisors);
		}
		return eligibleAdvisors;
	}

这样我们就知道getAdvicesAndAdvisorsForBean()到底是咋做的了，至于findAdvisorsThatCanApply()方法，我们留作以后研究。

下面是第二个问题：Proxy的创建逻辑：

    Object proxy = createProxy(
					bean.getClass(), beanName, specificInterceptors, new SingletonTargetSource(bean));

debug进入该方法：

    /**
	 * Create an AOP proxy for the given bean.
	 * @param beanClass the class of the bean
	 * @param beanName the name of the bean
	 * @param specificInterceptors the set of interceptors that is
	 * specific to this bean (may be empty, but not null)
	 * @param targetSource the TargetSource for the proxy,
	 * already pre-configured to access the bean
	 * @return the AOP proxy for the bean
	 * @see #buildAdvisors
	 */
	protected Object createProxy(Class<?> beanClass, @Nullable String beanName,
			@Nullable Object[] specificInterceptors, TargetSource targetSource) {
                    
		if (this.beanFactory instanceof ConfigurableListableBeanFactory) {
			AutoProxyUtils.exposeTargetClass((ConfigurableListableBeanFactory) this.beanFactory, beanName, beanClass);
		}
            //创建一个proxyFactory
		ProxyFactory proxyFactory = new ProxyFactory();
            //拷贝配置信息
            proxyFactory.copyFrom(this);

		if (!proxyFactory.isProxyTargetClass()) {
			if (shouldProxyTargetClass(beanClass, beanName)) {
				proxyFactory.setProxyTargetClass(true);
			}
			else {
                            //检查给定bean类的接口，并将其应用于{@link ProxyFactory}（如果适用）。
				evaluateProxyInterfaces(beanClass, proxyFactory);
			}
		}
            //包装下传入的Advisor并添加部分新的
		Advisor[] advisors = buildAdvisors(beanName, specificInterceptors);
		proxyFactory.addAdvisors(advisors);
		proxyFactory.setTargetSource(targetSource);
        //自定义ProxyFactory
		customizeProxyFactory(proxyFactory);

		proxyFactory.setFrozen(this.freezeProxy);
		if (advisorsPreFiltered()) {
			proxyFactory.setPreFiltered(true);
		}
        //创建proxy 
		return proxyFactory.getProxy(getProxyClassLoader());
	}

默认是调用DefaultAopProxyFactory的createApoProxy()方法创建Proxy：

	@Override
	public AopProxy createAopProxy(AdvisedSupport config) throws AopConfigException {
        //根据配置进行选择
        //@EnableAspectJAutoProxy注解不是有属性指定用啥创建代理么
		if (config.isOptimize() || config.isProxyTargetClass() || hasNoUserSuppliedProxyInterfaces(config)) {
			Class<?> targetClass = config.getTargetClass();
			if (targetClass == null) {
				throw new AopConfigException("TargetSource cannot determine target class: " +
						"Either an interface or a target is required for proxy creation.");
			}
            //如果传入的类是接口或者代理类就使用jdk动态代理，否则用cglib
			if (targetClass.isInterface() || Proxy.isProxyClass(targetClass)) {
				return new JdkDynamicAopProxy(config);
			}
			return new ObjenesisCglibAopProxy(config);
		}
		else {
			return new JdkDynamicAopProxy(config);
		}
	}


根据上面的方法，我们知道，我们实际上注册的并不是我们原来的bean了，而是一个被代理包装了的bean。

### AOP使用流程

cglib代理有个神奇的地方，就是调用了对应的切点方法后，会调用代理的`intercept()`方法，当我们调用了`MathCalculator的div(int,int)`方法，就进到这里了。

    //proxy 是被代理的对象
    //method 是执行的方法
    //args 是执行方法的参数数组
    //methodProxy 用来执行未被拦截的原方法

    @Override
	@Nullable
	public Object intercept(Object proxy, Method method, Object[] args, MethodProxy methodProxy) throws Throwable {
		//做一些属性的初始化
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
                    //获取通知方法有关的调用链
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
            //根据调用链进行方法调用，并返回返回值
			retVal = processReturnType(proxy, target, method, retVal);
			return retVal;
		}
		finally {
            //进行释放资源和设置缓存操作
			if (target != null && !targetSource.isStatic()) {
				targetSource.releaseTarget(target);
			}
			if (setProxyContext) {
				// Restore old proxy.
				AopContext.setCurrentProxy(oldProxy);
			}
		}
	}

获取调用链逻辑如下：

    /**
	 * Determine a list of {@link org.aopalliance.intercept.MethodInterceptor} objects
	 * for the given method, based on this configuration.
	 * @param method the proxied method
	 * @param targetClass the target class
	 * @return List of MethodInterceptors (may also include InterceptorAndDynamicMethodMatchers)
	 */
	public List<Object> getInterceptorsAndDynamicInterceptionAdvice(Method method, @Nullable Class<?> targetClass) {
		MethodCacheKey cacheKey = new MethodCacheKey(method);
		//从缓存中获取调用连
        List<Object> cached = this.methodCache.get(cacheKey);
		if (cached == null) {
            //缓存中没有的话调用下面方法获得调用链
			cached = this.advisorChainFactory.getInterceptorsAndDynamicInterceptionAdvice(
					this, method, targetClass);
			this.methodCache.put(cacheKey, cached);
		}
		return cached;
	}

具体获得调用链的逻辑：

    @Override
	public List<Object> getInterceptorsAndDynamicInterceptionAdvice(
			Advised config, Method method, @Nullable Class<?> targetClass) {

		// This is somewhat tricky... We have to process introductions first,
		// but we need to preserve order in the ultimate list.
		//先从配置中获取基本信息
            //有几个Advisor
            //他们作用的类是什么
            //并且拿出注册器
        List<Object> interceptorList = new ArrayList<>(config.getAdvisors().length);
		Class<?> actualClass = (targetClass != null ? targetClass : method.getDeclaringClass());
		boolean hasIntroductions = hasMatchingIntroductions(config, actualClass);
		AdvisorAdapterRegistry registry = GlobalAdvisorAdapterRegistry.getInstance();
            //遍历每个Advisor
            //如果Advisor是切点Advisor并且它已经被注册，其中记录的方法、目标类等数据都与注册的相匹配
            //就将其放入到调用链中
		for (Advisor advisor : config.getAdvisors()) {
			if (advisor instanceof PointcutAdvisor) {
				// Add it conditionally.
				PointcutAdvisor pointcutAdvisor = (PointcutAdvisor) advisor;
				if (config.isPreFiltered() || pointcutAdvisor.getPointcut().getClassFilter().matches(actualClass)) {
					MethodInterceptor[] interceptors = registry.getInterceptors(advisor);
					MethodMatcher mm = pointcutAdvisor.getPointcut().getMethodMatcher();
					if (MethodMatchers.matches(mm, method, actualClass, hasIntroductions)) {
						if (mm.isRuntime()) {
							// Creating a new object instance in the getInterceptors() method
							// isn't a problem as we normally cache created chains.
							for (MethodInterceptor interceptor : interceptors) {
								interceptorList.add(new InterceptorAndDynamicMethodMatcher(interceptor, mm));
							}
						}
						else {
							interceptorList.addAll(Arrays.asList(interceptors));
						}
					}
				}
			}
			else if (advisor instanceof IntroductionAdvisor) {
				IntroductionAdvisor ia = (IntroductionAdvisor) advisor;
				if (config.isPreFiltered() || ia.getClassFilter().matches(actualClass)) {
					Interceptor[] interceptors = registry.getInterceptors(advisor);
					interceptorList.addAll(Arrays.asList(interceptors));
				}
			}
			else {
				Interceptor[] interceptors = registry.getInterceptors(advisor);
				interceptorList.addAll(Arrays.asList(interceptors));
			}
		}

		return interceptorList;
	}

到这里，调用链也就出来了，那么调用调用链的逻辑，即：

    retVal = new CglibMethodInvocation(proxy, target, method, args, targetClass, chain, methodProxy).proceed();

首先创建一个`CglibMethodInvocation`类型的对象，这个类我也不太清楚是干啥的，毕竟对cglib不太了解，但是看名字应该是方法实现：

    public CglibMethodInvocation(Object proxy, @Nullable Object target, Method method,
				Object[] arguments, @Nullable Class<?> targetClass,
				List<Object> interceptorsAndDynamicMethodMatchers, MethodProxy methodProxy) {

			super(proxy, target, method, arguments, targetClass, interceptorsAndDynamicMethodMatchers);
			this.methodProxy = methodProxy;
			this.publicMethod = Modifier.isPublic(method.getModifiers());
		}

然后调用其的`proceed()`方法：

    @Override
	@Nullable
	public Object proceed() throws Throwable {
		//	We start with an index of -1 and increment early.
		//  this.interceptorsAndDynamicMethodMatchers这个对象里存储了所有的要被处理的Advisor方法，这里以Interceptor的形式显现
        //  如果该容器中的需要处理的通知方法个数与this.currentInterceptorIndex时，那么就调用切点方法。
        //this.currentInterceptorIndex的初始值为-1。
        if (this.currentInterceptorIndex == this.interceptorsAndDynamicMethodMatchers.size() - 1) {
			return invokeJoinpoint();
		}
        //否则获取到容器中保存的通知方法
        //注意++this.currentInterceptorIndex这里使用的是前++
		Object interceptorOrInterceptionAdvice =
				this.interceptorsAndDynamicMethodMatchers.get(++this.currentInterceptorIndex);
		if (interceptorOrInterceptionAdvice instanceof InterceptorAndDynamicMethodMatcher) {
			// Evaluate dynamic method matcher here: static part will already have
			// been evaluated and found to match.
			InterceptorAndDynamicMethodMatcher dm =
					(InterceptorAndDynamicMethodMatcher) interceptorOrInterceptionAdvice;
			if (dm.methodMatcher.matches(this.method, this.targetClass, this.arguments)) {
				return dm.interceptor.invoke(this);
			}
			else {
				// Dynamic matching failed.
				// Skip this interceptor and invoke the next in the chain.
				return proceed();
			}
		}
		else {
			// It's an interceptor, so we just invoke it: The pointcut will have
			// been evaluated statically before this object was constructed.
            //在这里调用上述Inteceptor的invoke()方法，注意这里传入了this参数
			return ((MethodInterceptor) interceptorOrInterceptionAdvice).invoke(this);
		}
	}

不同的`MethoodInterceptor的invoke(MethodInvocation mi)方法不同，但是都是按照如下形式的`：

    @Override
	public Object invoke(MethodInvocation mi) throws Throwable {
		//前置操作
		try {
			return mi.proceed();
            //后置操作
		}
        catche(XXXException e){
            //抛出异常后的操作
        }
		finally {
            //清理操作
		}
	}

考察本例中`五个通知方法的MethodInterceptor的invoke()`方法:

1. ExposeInvocationInterceptor:

        @Override
        public Object invoke(MethodInvocation mi) throws Throwable {
            MethodInvocation oldInvocation = invocation.get();
            invocation.set(mi);
            try {
                return mi.proceed();
            }
            finally {
                invocation.set(oldInvocation);
            }
        }

2. AspectJAfterThrowingAdvice:

        @Override
        public Object invoke(MethodInvocation mi) throws Throwable {
            try {
                return mi.proceed();
            }
            catch (Throwable ex) {
                if (shouldInvokeOnThrowing(ex)) {
                    invokeAdviceMethod(getJoinPointMatch(), null, ex);
                }
                throw ex;
            }
        }

3. AfterReturningAdviceInterceptor

        @Override
        public Object invoke(MethodInvocation mi) throws Throwable {
            Object retVal = mi.proceed();
            this.advice.afterReturning(retVal, mi.getMethod(), mi.getArguments(), mi.getThis());
            return retVal;
        }

4. AspectJAfterAdvice

        @Override
        public Object invoke(MethodInvocation mi) throws Throwable {
            try {
                return mi.proceed();
            }
            finally {
                invokeAdviceMethod(getJoinPointMatch(), null, null);
            }
        }

5. MethodBeforeAdviceInterceptor：

        @Override
        public Object invoke(MethodInvocation mi) throws Throwable {
            this.advice.before(mi.getMethod(), mi.getArguments(), mi.getThis());
            return mi.proceed();
        }

每个方法名字就暴露出其对应的通知方法，我们可以看到，`MethodBeforeAdviceInterceptor`中的

    this.advice.before(mi.getMethod(), mi.getArguments(), mi.getThis());

这行代码在`mi.proceed()`方法之前调用，其余的Interceptor都是在这行代码之后有逻辑，这就验证了为何这是前置通知。由于我们这个切面有4个通知，默认被加了一个，总共变为5个通知，即`this.interceptorsAndDynamicMethodMatchers`的长度默认为5，但是由于`每次取出MethodInterceptor时都对this.currentInterceptorIndex进行了++操作`，因此顺序执行了上述几个Interceptor后，到执行MethodBeforeAdviceInterceptor的invoke方法时，

    this.currentInterceptorIndex == this.interceptorsAndDynamicMethodMatchers.size() - 1

条件已经满足了，所以它的`mi.proceed()`方法就是执行切点方法。然后其他的MethodInteceptor中的invoke()方法也比较好理解，这里就不详述了。

至此，基于注解的Spring-aop原理分析完毕。

## findEligibleAdvisors方法解析

该方法源码如下：

    protected List<Advisor> findEligibleAdvisors(Class<?> beanClass, String beanName) {
		List<Advisor> candidateAdvisors = findCandidateAdvisors();
		List<Advisor> eligibleAdvisors = findAdvisorsThatCanApply(candidateAdvisors, beanClass, beanName);
		extendAdvisors(eligibleAdvisors);
		if (!eligibleAdvisors.isEmpty()) {
			eligibleAdvisors = sortAdvisors(eligibleAdvisors);
		}
		return eligibleAdvisors;
	}

方法逻辑如下：
1. 使用findCandidateAdvisors函数获取所有候选的通知器
2. 通过findAdvisorsThatCanApply方法从候选的通知器中选择出可以应用于该bean的通知器。
3. 进行Advisors的扩展，这个方法是空方法，可以使用子类进行扩展
4. 对Advisors进行排序。
5. 返回结果。

### findCandidateAdvisors函数获取候选Advisors

    protected List<Advisor> findCandidateAdvisors() {
		// Add all the Spring advisors found according to superclass rules.
		List<Advisor> advisors = super.findCandidateAdvisors();
		// Build Advisors for all AspectJ aspects in the bean factory.
		if (this.aspectJAdvisorsBuilder != null) {
			advisors.addAll(this.aspectJAdvisorsBuilder.buildAspectJAdvisors());
		}
		return advisors;
	}

这个方法是由AnnotationAwareAspectJAutoProxyCreator类实现的，首先调用父类的findCandidateAdvisors方法：

    protected List<Advisor> findCandidateAdvisors() {
		Assert.state(this.advisorRetrievalHelper != null, "No BeanFactoryAdvisorRetrievalHelper available");
		return this.advisorRetrievalHelper.findAdvisorBeans();
	}

这里是将查找任务委托给advisorRetrievalHelper进行处理，考虑其findAdvisorBeans方法：

    public List<Advisor> findAdvisorBeans() {
		// Determine list of advisor bean names, if not cached already.
		String[] advisorNames = null;
		synchronized (this) {
			advisorNames = this.cachedAdvisorBeanNames;
			if (advisorNames == null) {
				// Do not initialize FactoryBeans here: We need to leave all regular beans
				// uninitialized to let the auto-proxy creator apply to them!
				advisorNames = BeanFactoryUtils.beanNamesForTypeIncludingAncestors(
						this.beanFactory, Advisor.class, true, false);
				this.cachedAdvisorBeanNames = advisorNames;
			}
		}
		if (advisorNames.length == 0) {
			return new LinkedList<>();
		}

		List<Advisor> advisors = new LinkedList<>();
		for (String name : advisorNames) {
			if (isEligibleBean(name)) {
				if (this.beanFactory.isCurrentlyInCreation(name)) {
					if (logger.isDebugEnabled()) {
						logger.debug("Skipping currently created advisor '" + name + "'");
					}
				}
				else {
					try {
						advisors.add(this.beanFactory.getBean(name, Advisor.class));
					}
					catch (BeanCreationException ex) {
						Throwable rootCause = ex.getMostSpecificCause();
						if (rootCause instanceof BeanCurrentlyInCreationException) {
							BeanCreationException bce = (BeanCreationException) rootCause;
							String bceBeanName = bce.getBeanName();
							if (bceBeanName != null && this.beanFactory.isCurrentlyInCreation(bceBeanName)) {
								if (logger.isDebugEnabled()) {
									logger.debug("Skipping advisor '" + name +
											"' with dependency on currently created bean: " + ex.getMessage());
								}
								// Ignore: indicates a reference back to the bean we're trying to advise.
								// We want to find advisors other than the currently created bean itself.
								continue;
							}
						}
						throw ex;
					}
				}
			}
		}
		return advisors;
	}

这里是保留父类的获取配置文件中定义的增强。接下来我们主要讨论以下代码：

    if (this.aspectJAdvisorsBuilder != null) {
		advisors.addAll(this.aspectJAdvisorsBuilder.buildAspectJAdvisors());
	}

这里又将寻找Advisor的逻辑委托给aspectJAdvisorsBuilder对象，调用的是buildAspectJAdvisors方法：

    public List<Advisor> buildAspectJAdvisors() {
		List<String> aspectNames = this.aspectBeanNames;

		if (aspectNames == null) {
			synchronized (this) {
				aspectNames = this.aspectBeanNames;
				if (aspectNames == null) {
					List<Advisor> advisors = new LinkedList<>();
					aspectNames = new LinkedList<>();
                    //获取所有的beanName
					String[] beanNames = BeanFactoryUtils.beanNamesForTypeIncludingAncestors(
							this.beanFactory, Object.class, true, false);
                    //循环找出所有beanName所对应的增强方法
					for (String beanName : beanNames) {
                        //不合法的bean则略过，由子类定义规则，默认返回true
						if (!isEligibleBean(beanName)) {
							continue;
						}
						// We must be careful not to instantiate beans eagerly as in this case they
						// would be cached by the Spring container but would not have been weaved.
						Class<?> beanType = this.beanFactory.getType(beanName);
						if (beanType == null) {
							continue;
						}
                        //如果存在Aspect注解
						if (this.advisorFactory.isAspect(beanType)) {
							aspectNames.add(beanName);
							AspectMetadata amd = new AspectMetadata(beanType, beanName);
							if (amd.getAjType().getPerClause().getKind() == PerClauseKind.SINGLETON) {
								MetadataAwareAspectInstanceFactory factory =
										new BeanFactoryAspectInstanceFactory(this.beanFactory, beanName);
                                //解析标记AspectJ注解中的增强方法
								List<Advisor> classAdvisors = this.advisorFactory.getAdvisors(factory);
								if (this.beanFactory.isSingleton(beanName)) {
									this.advisorsCache.put(beanName, classAdvisors);
								}
								else {
									this.aspectFactoryCache.put(beanName, factory);
								}
								advisors.addAll(classAdvisors);
							}
							else {
								// Per target or per this.
								if (this.beanFactory.isSingleton(beanName)) {
									throw new IllegalArgumentException("Bean with name '" + beanName +
											"' is a singleton, but aspect instantiation model is not singleton");
								}
								MetadataAwareAspectInstanceFactory factory =
										new PrototypeAspectInstanceFactory(this.beanFactory, beanName);
								this.aspectFactoryCache.put(beanName, factory);
								advisors.addAll(this.advisorFactory.getAdvisors(factory));
							}
						}
					}
					this.aspectBeanNames = aspectNames;
					return advisors;
				}
			}
		}

		if (aspectNames.isEmpty()) {
			return Collections.emptyList();
		}
        //记录在缓存中
		List<Advisor> advisors = new LinkedList<>();
		for (String aspectName : aspectNames) {
			List<Advisor> cachedAdvisors = this.advisorsCache.get(aspectName);
			if (cachedAdvisors != null) {
				advisors.addAll(cachedAdvisors);
			}
			else {
				MetadataAwareAspectInstanceFactory factory = this.aspectFactoryCache.get(aspectName);
				advisors.addAll(this.advisorFactory.getAdvisors(factory));
			}
		}
		return advisors;
	}

至此我们完成了Advisor的提取，在上面的步骤中最为重要，也最为繁琐的就是增强器的获取。这一功能委托给getAdvisors方法实现，该方法源码如下：

    public List<Advisor> getAdvisors(MetadataAwareAspectInstanceFactory aspectInstanceFactory) {
        //获取标记为AspectJ的类
		Class<?> aspectClass = aspectInstanceFactory.getAspectMetadata().getAspectClass();
        //获取标记为AspectJ的name
		String aspectName = aspectInstanceFactory.getAspectMetadata().getAspectName();
        //验证
		validate(aspectClass);

		// We need to wrap the MetadataAwareAspectInstanceFactory with a decorator
		// so that it will only instantiate once.
		MetadataAwareAspectInstanceFactory lazySingletonAspectInstanceFactory =
				new LazySingletonAspectInstanceFactoryDecorator(aspectInstanceFactory);

		List<Advisor> advisors = new LinkedList<>();
        //这里获取切面中定义的所有方法
        //然后根据每个方法获取对应的Advisor
		for (Method method : getAdvisorMethods(aspectClass)) {
			Advisor advisor = getAdvisor(method, lazySingletonAspectInstanceFactory, advisors.size(), aspectName);
			if (advisor != null) {
				advisors.add(advisor);
			}
		}

		// If it's a per target aspect, emit the dummy instantiating aspect.
        //如果寻找的增强器不为空而且又配置了增强延迟初始化，那么就需要在首位加入同步实例增强器
		if (!advisors.isEmpty() && lazySingletonAspectInstanceFactory.getAspectMetadata().isLazilyInstantiated()) {
			Advisor instantiationAdvisor = new SyntheticInstantiationAdvisor(lazySingletonAspectInstanceFactory);
			advisors.add(0, instantiationAdvisor);
		}

		// Find introduction fields.
        // 获取DeclareParents注解，DeclareParents注解主要用于引介增强的注解形式的实现，而其实现方式与普通增强很类似，只不过使用DeclareParentsAdvisors对功能进行封装
		for (Field field : aspectClass.getDeclaredFields()) {
			Advisor advisor = getDeclareParentsAdvisor(field);
			if (advisor != null) {
				advisors.add(advisor);
			}
		}

		return advisors;
	}

这里获取真正获取Advisor类型对象的方法就是getAdvisor方法：

    public Advisor getAdvisor(Method candidateAdviceMethod, MetadataAwareAspectInstanceFactory aspectInstanceFactory,
			int declarationOrderInAspect, String aspectName) {

		validate(aspectInstanceFactory.getAspectMetadata().getAspectClass());
        //切点信息获取
		AspectJExpressionPointcut expressionPointcut = getPointcut(
				candidateAdviceMethod, aspectInstanceFactory.getAspectMetadata().getAspectClass());
		if (expressionPointcut == null) {
			return null;
		}
        //根据切点生成增强器
		return new InstantiationModelAwarePointcutAdvisorImpl(expressionPointcut, candidateAdviceMethod,
				this, aspectInstanceFactory, declarationOrderInAspect, aspectName);
	}

逻辑如下：
1. 先通过getPointcut方法获取切点信息
2. 根据切点生成增强其

#### getPointcut方法获取切点

所谓获取切点信息就是指定注解的表达式信息的获取，如@Before("test()")；

    private AspectJExpressionPointcut getPointcut(Method candidateAdviceMethod, Class<?> candidateAspectClass) {
        //获取方法上的注解
		AspectJAnnotation<?> aspectJAnnotation =
				AbstractAspectJAdvisorFactory.findAspectJAnnotationOnMethod(candidateAdviceMethod);
		if (aspectJAnnotation == null) {
			return null;
		}
        //使用AspectJExpressionPointcut实例封装获取的信息
		AspectJExpressionPointcut ajexp =
				new AspectJExpressionPointcut(candidateAspectClass, new String[0], new Class<?>[0]);
        //提取得到的注解中获得的表达式
        //@Before("pointCut()")中的pointCut()
		ajexp.setExpression(aspectJAnnotation.getPointcutExpression());
		if (this.beanFactory != null) {
			ajexp.setBeanFactory(this.beanFactory);
		}
		return ajexp;
	}

#### 根据切点信息生成增强

所有的增强都由Advisor的实现类InstantiationModelAwarePointcutAdvisorImpl统一封装。

    public InstantiationModelAwarePointcutAdvisorImpl(AspectJExpressionPointcut declaredPointcut,
			Method aspectJAdviceMethod, AspectJAdvisorFactory aspectJAdvisorFactory,
			MetadataAwareAspectInstanceFactory aspectInstanceFactory, int declarationOrder, String aspectName) {

		this.declaredPointcut = declaredPointcut;
		this.declaringClass = aspectJAdviceMethod.getDeclaringClass();
		this.methodName = aspectJAdviceMethod.getName();
		this.parameterTypes = aspectJAdviceMethod.getParameterTypes();
		this.aspectJAdviceMethod = aspectJAdviceMethod;
		this.aspectJAdvisorFactory = aspectJAdvisorFactory;
		this.aspectInstanceFactory = aspectInstanceFactory;
		this.declarationOrder = declarationOrder;
		this.aspectName = aspectName;

		if (aspectInstanceFactory.getAspectMetadata().isLazilyInstantiated()) {
			// Static part of the pointcut is a lazy type.
			Pointcut preInstantiationPointcut = Pointcuts.union(
					aspectInstanceFactory.getAspectMetadata().getPerClausePointcut(), this.declaredPointcut);

			// Make it dynamic: must mutate from pre-instantiation to post-instantiation state.
			// If it's not a dynamic pointcut, it may be optimized out
			// by the Spring AOP infrastructure after the first evaluation.
			this.pointcut = new PerTargetInstantiationModelPointcut(
					this.declaredPointcut, preInstantiationPointcut, aspectInstanceFactory);
			this.lazy = true;
		}
		else {
			// A singleton aspect.
			this.pointcut = this.declaredPointcut;
			this.lazy = false;
			this.instantiatedAdvice = instantiateAdvice(this.declaredPointcut);
		}
	}

封装过程中只是简单地将信息封装在类的实例中，所有的信息单纯地赋值，在实例初始化工程中还完成了对于增强器的初始化，对于增强器的初始化时通过instantiateAdvice函数进行实现的。

    private Advice instantiateAdvice(AspectJExpressionPointcut pointcut) {
		Advice advice = this.aspectJAdvisorFactory.getAdvice(this.aspectJAdviceMethod, pointcut,
				this.aspectInstanceFactory, this.declarationOrder, this.aspectName);
		return (advice != null ? advice : EMPTY_ADVICE);
	}

    public Advice getAdvice(Method candidateAdviceMethod, AspectJExpressionPointcut expressionPointcut,
			MetadataAwareAspectInstanceFactory aspectInstanceFactory, int declarationOrder, String aspectName) {

		Class<?> candidateAspectClass = aspectInstanceFactory.getAspectMetadata().getAspectClass();
		validate(candidateAspectClass);

		AspectJAnnotation<?> aspectJAnnotation =
				AbstractAspectJAdvisorFactory.findAspectJAnnotationOnMethod(candidateAdviceMethod);
		if (aspectJAnnotation == null) {
			return null;
		}

		// If we get here, we know we have an AspectJ method.
		// Check that it's an AspectJ-annotated class
		if (!isAspect(candidateAspectClass)) {
			throw new AopConfigException("Advice must be declared inside an aspect type: " +
					"Offending method '" + candidateAdviceMethod + "' in class [" +
					candidateAspectClass.getName() + "]");
		}

		if (logger.isDebugEnabled()) {
			logger.debug("Found AspectJ method: " + candidateAdviceMethod);
		}

		AbstractAspectJAdvice springAdvice;
        //根据不同的注解类型封装不同的增强器
		switch (aspectJAnnotation.getAnnotationType()) {
			case AtBefore:
				springAdvice = new AspectJMethodBeforeAdvice(
						candidateAdviceMethod, expressionPointcut, aspectInstanceFactory);
				break;
			case AtAfter:
				springAdvice = new AspectJAfterAdvice(
						candidateAdviceMethod, expressionPointcut, aspectInstanceFactory);
				break;
			case AtAfterReturning:
				springAdvice = new AspectJAfterReturningAdvice(
						candidateAdviceMethod, expressionPointcut, aspectInstanceFactory);
				AfterReturning afterReturningAnnotation = (AfterReturning) aspectJAnnotation.getAnnotation();
				if (StringUtils.hasText(afterReturningAnnotation.returning())) {
					springAdvice.setReturningName(afterReturningAnnotation.returning());
				}
				break;
			case AtAfterThrowing:
				springAdvice = new AspectJAfterThrowingAdvice(
						candidateAdviceMethod, expressionPointcut, aspectInstanceFactory);
				AfterThrowing afterThrowingAnnotation = (AfterThrowing) aspectJAnnotation.getAnnotation();
				if (StringUtils.hasText(afterThrowingAnnotation.throwing())) {
					springAdvice.setThrowingName(afterThrowingAnnotation.throwing());
				}
				break;
			case AtAround:
				springAdvice = new AspectJAroundAdvice(
						candidateAdviceMethod, expressionPointcut, aspectInstanceFactory);
				break;
			case AtPointcut:
				if (logger.isDebugEnabled()) {
					logger.debug("Processing pointcut '" + candidateAdviceMethod.getName() + "'");
				}
				return null;
			default:
				throw new UnsupportedOperationException(
						"Unsupported advice type on method: " + candidateAdviceMethod);
		}

		// Now to configure the advice...
		springAdvice.setAspectName(aspectName);
		springAdvice.setDeclarationOrder(declarationOrder);
		String[] argNames = this.parameterNameDiscoverer.getParameterNames(candidateAdviceMethod);
		if (argNames != null) {
			springAdvice.setArgumentNamesFromStringArray(argNames);
		}
		springAdvice.calculateArgumentBindings();

		return springAdvice;
	}

上述函数逻辑如下：
1. 获取切点的对应注解
2. 根据注解创建Advice
3. 将Advice进行属性设置
4. 返回

### 寻找匹配的增强器

前面的函数中已经完成了所有增强器的解析，但是对于所有增强器来讲，并不一定都适用于当前bean，还要挑取出适合的增强器，也就是满足我们配置的通配符的增强器，具体实现逻辑在findAdvisorsThatCanApply中：

    protected List<Advisor> findAdvisorsThatCanApply(
			List<Advisor> candidateAdvisors, Class<?> beanClass, String beanName) {

		ProxyCreationContext.setCurrentProxiedBeanName(beanName);
		try {
			return AopUtils.findAdvisorsThatCanApply(candidateAdvisors, beanClass);
		}
		finally {
			ProxyCreationContext.setCurrentProxiedBeanName(null);
		}
	}

继续看AopUtils.findAdvisorsThatCanApply方法：

    public static List<Advisor> findAdvisorsThatCanApply(List<Advisor> candidateAdvisors, Class<?> clazz) {
		if (candidateAdvisors.isEmpty()) {
			return candidateAdvisors;
		}
		List<Advisor> eligibleAdvisors = new LinkedList<>();
        //首先处理引介增强
		for (Advisor candidate : candidateAdvisors) {
			if (candidate instanceof IntroductionAdvisor && canApply(candidate, clazz)) {
				eligibleAdvisors.add(candidate);
			}
		}
		boolean hasIntroductions = !eligibleAdvisors.isEmpty();
		for (Advisor candidate : candidateAdvisors) {
            // 引介增强已经处理
			if (candidate instanceof IntroductionAdvisor) {
				// already processed
				continue;
			}
            //对于普通bean的处理
			if (canApply(candidate, clazz, hasIntroductions)) {
				eligibleAdvisors.add(candidate);
			}
		}
		return eligibleAdvisors;
	}

查看源码我们发现实际的匹配规则是在canApply方法中，方法源码如下：

    public static boolean canApply(Advisor advisor, Class<?> targetClass, boolean hasIntroductions) {
		if (advisor instanceof IntroductionAdvisor) {
			return ((IntroductionAdvisor) advisor).getClassFilter().matches(targetClass);
		}
		else if (advisor instanceof PointcutAdvisor) {
			PointcutAdvisor pca = (PointcutAdvisor) advisor;
			return canApply(pca.getPointcut(), targetClass, hasIntroductions);
		}
		else {
			// It doesn't have a pointcut so we assume it applies.
			return true;
		}
	}

    public static boolean canApply(Pointcut pc, Class<?> targetClass, boolean hasIntroductions) {
		Assert.notNull(pc, "Pointcut must not be null");
		if (!pc.getClassFilter().matches(targetClass)) {
			return false;
		}

		MethodMatcher methodMatcher = pc.getMethodMatcher();
		if (methodMatcher == MethodMatcher.TRUE) {
			// No need to iterate the methods if we're matching any method anyway...
			return true;
		}

		IntroductionAwareMethodMatcher introductionAwareMethodMatcher = null;
		if (methodMatcher instanceof IntroductionAwareMethodMatcher) {
			introductionAwareMethodMatcher = (IntroductionAwareMethodMatcher) methodMatcher;
		}

		Set<Class<?>> classes = new LinkedHashSet<>();
		if (!Proxy.isProxyClass(targetClass)) {
			classes.add(ClassUtils.getUserClass(targetClass));
		}
		classes.addAll(ClassUtils.getAllInterfacesForClassAsSet(targetClass));

		for (Class<?> clazz : classes) {
			Method[] methods = ReflectionUtils.getAllDeclaredMethods(clazz);
			for (Method method : methods) {
				if (introductionAwareMethodMatcher != null ?
						introductionAwareMethodMatcher.matches(method, targetClass, hasIntroductions) :
						methodMatcher.matches(method, targetClass)) {
					return true;
				}
			}
		}

		return false;
	}

该方法逻辑如下：
1. 调用Pointcut接口的getClassFilter方法获取ClassFilter，然后调用其matches方法进行类型匹配，匹配错误返回false
2. 调用Pointcut接口的getMethodMatcher方法获取MethodMatcher对象，然后调用其matches方法进行方法、类型匹配，众多方法中，只要有一个匹配成功则返回true，否则返回false。

## 创建代理

获取了所有的增强器就可以进行代理的创建。

    protected Object createProxy(Class<?> beanClass, @Nullable String beanName,
			@Nullable Object[] specificInterceptors, TargetSource targetSource) {

		if (this.beanFactory instanceof ConfigurableListableBeanFactory) {
			AutoProxyUtils.exposeTargetClass((ConfigurableListableBeanFactory) this.beanFactory, beanName, beanClass);
		}

		ProxyFactory proxyFactory = new ProxyFactory();
		// 获取当前类中相关属性
        proxyFactory.copyFrom(this);
        // 决定对于给定的bean是否应该使用targetClass而不是他的接口代理
        // 检查proxyTargetClass设置以及preserveTargetClass属性
		if (!proxyFactory.isProxyTargetClass()) {
			if (shouldProxyTargetClass(beanClass, beanName)) {
				proxyFactory.setProxyTargetClass(true);
			}
			else {
                //添加代理接口
				evaluateProxyInterfaces(beanClass, proxyFactory);
			}
		}

		Advisor[] advisors = buildAdvisors(beanName, specificInterceptors);
        //加入增强器
		proxyFactory.addAdvisors(advisors);
        //设置要代理的类
		proxyFactory.setTargetSource(targetSource);
		//定制代理
        customizeProxyFactory(proxyFactory);
        // 用来控制代理工厂被配置之后，是否还允许修改通知
        // 缺省值为false（即在代理被配置之后，不允许修改代理的配置）
		proxyFactory.setFrozen(this.freezeProxy);
		if (advisorsPreFiltered()) {
			proxyFactory.setPreFiltered(true);
		}

		return proxyFactory.getProxy(getProxyClassLoader());
	}

对于代理类的创建及处理，Spring委托给了ProxyFactory去处理，而此函数主要是对ProxyFactory的初始化操作，进而对真正的创建代理做准备，初始化操作如下:
1. 获取当前类的属性
2. 添加代理接口
3. 封装Advisor并加入到ProxyFactory中
4. 设置要代理的类
5. 当然在Spring中还为子类提供了定制的函数customizeProxyFactory，子类可以在此函数中进行对ProxyFactory的进一步封装。
6. 进行获取代理的操作。

### buildAdvisors

该函数用于将拦截器封装为增强器，源码如下：

    protected Advisor[] buildAdvisors(@Nullable String beanName, @Nullable Object[] specificInterceptors) {
		// Handle prototypes correctly...
		Advisor[] commonInterceptors = resolveInterceptorNames();

		//加入拦截器
		List<Object> allInterceptors = new ArrayList<>();
		if (specificInterceptors != null) {
			allInterceptors.addAll(Arrays.asList(specificInterceptors));
			if (commonInterceptors.length > 0) {
				if (this.applyCommonInterceptorsFirst) {
					allInterceptors.addAll(0, Arrays.asList(commonInterceptors));
				}
				else {
					allInterceptors.addAll(Arrays.asList(commonInterceptors));
				}
			}
		}
		if (logger.isDebugEnabled()) {
			int nrOfCommonInterceptors = commonInterceptors.length;
			int nrOfSpecificInterceptors = (specificInterceptors != null ? specificInterceptors.length : 0);
			logger.debug("Creating implicit proxy for bean '" + beanName + "' with " + nrOfCommonInterceptors +
					" common interceptors and " + nrOfSpecificInterceptors + " specific interceptors");
		}

		Advisor[] advisors = new Advisor[allInterceptors.size()];
		for (int i = 0; i < allInterceptors.size(); i++) {
			//拦截器进行封装转化为Advisor
			advisors[i] = this.advisorAdapterRegistry.wrap(allInterceptors.get(i));
		}
		return advisors;
	}

    public Advisor wrap(Object adviceObject) throws UnknownAdviceTypeException {
		// 如果要封装的对象本身就是Advisor类型的那么无需再过多处理
		if (adviceObject instanceof Advisor) {
			return (Advisor) adviceObject;
		}
		// 因为此封装方法支队Advisor和Advice两种数据类型有效，如果不是将无法封装
		if (!(adviceObject instanceof Advice)) {
			throw new UnknownAdviceTypeException(adviceObject);
		}

		Advice advice = (Advice) adviceObject;
		if (advice instanceof MethodInterceptor) {
			// So well-known it doesn't even need an adapter.
			return new DefaultPointcutAdvisor(advice);
		}
		//如果存在Advisor的适配器那么也同样需要封装
		for (AdvisorAdapter adapter : this.adapters) {
			// Check that it is supported.
			if (adapter.supportsAdvice(advice)) {
				return new DefaultPointcutAdvisor(advice);
			}
		}
		throw new UnknownAdviceTypeException(advice);
	}

### 创建代理

#### 创建代理

	public Object getProxy(@Nullable ClassLoader classLoader) {
		return createAopProxy().getProxy(classLoader);
	}

	protected final synchronized AopProxy createAopProxy() {
		if (!this.active) {
			activate();
		}
		return getAopProxyFactory().createAopProxy(this);
	}

	@Override
	public AopProxy createAopProxy(AdvisedSupport config) throws AopConfigException {
		if (config.isOptimize() || config.isProxyTargetClass() || hasNoUserSuppliedProxyInterfaces(config)) {
			Class<?> targetClass = config.getTargetClass();
			if (targetClass == null) {
				throw new AopConfigException("TargetSource cannot determine target class: " +
						"Either an interface or a target is required for proxy creation.");
			}
			if (targetClass.isInterface() || Proxy.isProxyClass(targetClass)) {
				return new JdkDynamicAopProxy(config);
			}
			return new ObjenesisCglibAopProxy(config);
		}
		else {
			return new JdkDynamicAopProxy(config);
		}
	}

这段代码负责完成代理的创建，那么是如何创建代理，我们查看创建代理过程中的几个条件：

1. optimize:用来控制通过CGLIB创建的代理是否使用激进的优化策略。除非完全了解AOP代理如何处理优化，否则不推荐用户使用这个设置。目前这个属性仅用于CGLIB代理，对于JDK动态代理（缺省代理）无效。
2. proxyTargetClass:这个属性为true时，目标类本身被代理而不是目标类的接口。如果这个属性被设置为true，CGLIB代理将被创建。
3. hasNoUserSuppliedProxyInterfaces:是否存在代理接口。

- 如果目标对象实现了接口，默认情况下会采用JDK动态代理实现AOP。
- 如果目标对象实现了接口，可以强制使用CGLIB实现AOP。
- 如果目标对象没有实现接口，必须采用CGLIB库，Spring会自动再JDK动态代理和CGLIB之间的转换。

JDK动态代理和CGLIB字节码生成的区别：
- JDK动态代理只能对实现了接口的类生成代理，而不能针对类
- CGLIB时针对列实现代理，主要是对指定的类生成一个子类，覆盖其中的方法，因为时继承，所以该类或方法最好不要声明成final。

#### 获取代理

获取代理在ObjenesisCglibAopProxy和JdkDynamicAopProxy的getProxy(ClassLoader)方法中完成，我们分别查看这两个类：

1. ObjenesisCglibAopProxy

   该类的getProxy方法是交给其父类CglibAopProxy实现的，源码如下：

		public Object getProxy(@Nullable ClassLoader classLoader) {
			if (logger.isDebugEnabled()) {
				logger.debug("Creating CGLIB proxy: target source is " + this.advised.getTargetSource());
			}

			try {
				Class<?> rootClass = this.advised.getTargetClass();
				Assert.state(rootClass != null, "Target class must be available for creating a CGLIB proxy");

				Class<?> proxySuperClass = rootClass;
				if (ClassUtils.isCglibProxyClass(rootClass)) {
					proxySuperClass = rootClass.getSuperclass();
					Class<?>[] additionalInterfaces = rootClass.getInterfaces();
					for (Class<?> additionalInterface : additionalInterfaces) {
						this.advised.addInterface(additionalInterface);
					}
				}

				// Validate the class, writing log messages as necessary.
				validateClassIfNecessary(proxySuperClass, classLoader);

				// Configure CGLIB Enhancer...
				Enhancer enhancer = createEnhancer();
				if (classLoader != null) {
					enhancer.setClassLoader(classLoader);
					if (classLoader instanceof SmartClassLoader &&
							((SmartClassLoader) classLoader).isClassReloadable(proxySuperClass)) {
						enhancer.setUseCache(false);
					}
				}
				enhancer.setSuperclass(proxySuperClass);
				enhancer.setInterfaces(AopProxyUtils.completeProxiedInterfaces(this.advised));
				enhancer.setNamingPolicy(SpringNamingPolicy.INSTANCE);
				enhancer.setStrategy(new ClassLoaderAwareUndeclaredThrowableStrategy(classLoader));
				//设置拦截器
				Callback[] callbacks = getCallbacks(rootClass);
				Class<?>[] types = new Class<?>[callbacks.length];
				for (int x = 0; x < types.length; x++) {
					types[x] = callbacks[x].getClass();
				}
				// fixedInterceptorMap only populated at this point, after getCallbacks call above
				enhancer.setCallbackFilter(new ProxyCallbackFilter(
						this.advised.getConfigurationOnlyCopy(), this.fixedInterceptorMap, this.fixedInterceptorOffset));
				enhancer.setCallbackTypes(types);

				// Generate the proxy class and create a proxy instance.
				return createProxyClassAndInstance(enhancer, callbacks);
			}
			catch (CodeGenerationException | IllegalArgumentException ex) {
				throw new AopConfigException("Could not generate CGLIB subclass of " + this.advised.getTargetClass() +
						": Common causes of this problem include using a final class or a non-visible class",
						ex);
			}
			catch (Throwable ex) {
				// TargetSource.getTarget() failed
				throw new AopConfigException("Unexpected AOP exception", ex);
			}
		}

		private Callback[] getCallbacks(Class<?> rootClass) throws Exception {
			// Parameters used for optimization choices...
			boolean exposeProxy = this.advised.isExposeProxy();
			boolean isFrozen = this.advised.isFrozen();
			boolean isStatic = this.advised.getTargetSource().isStatic();

			// Choose an "aop" interceptor (used for AOP calls).
			// 将拦截器封装在DynamicAdvisedInterceptor中
			Callback aopInterceptor = new DynamicAdvisedInterceptor(this.advised);

			// Choose a "straight to target" interceptor. (used for calls that are
			// unadvised but can return this). May be required to expose the proxy.
			Callback targetInterceptor;
			if (exposeProxy) {
				targetInterceptor = isStatic ?
						new StaticUnadvisedExposedInterceptor(this.advised.getTargetSource().getTarget()) :
						new DynamicUnadvisedExposedInterceptor(this.advised.getTargetSource());
			}
			else {
				targetInterceptor = isStatic ?
						new StaticUnadvisedInterceptor(this.advised.getTargetSource().getTarget()) :
						new DynamicUnadvisedInterceptor(this.advised.getTargetSource());
			}

			// Choose a "direct to target" dispatcher (used for
			// unadvised calls to static targets that cannot return this).
			Callback targetDispatcher = isStatic ?
					new StaticDispatcher(this.advised.getTargetSource().getTarget()) : new SerializableNoOp();

			Callback[] mainCallbacks = new Callback[] {
					aopInterceptor,  // for normal advice
					targetInterceptor,  // invoke target without considering advice, if optimized
					new SerializableNoOp(),  // no override for methods mapped to this
					targetDispatcher, this.advisedDispatcher,
					new EqualsInterceptor(this.advised),
					new HashCodeInterceptor(this.advised)
			};

			Callback[] callbacks;

			// If the target is a static one and the advice chain is frozen,
			// then we can make some optimizations by sending the AOP calls
			// direct to the target using the fixed chain for that method.
			if (isStatic && isFrozen) {
				Method[] methods = rootClass.getMethods();
				Callback[] fixedCallbacks = new Callback[methods.length];
				this.fixedInterceptorMap = new HashMap<>(methods.length);

				// TODO: small memory optimization here (can skip creation for methods with no advice)
				for (int x = 0; x < methods.length; x++) {
					List<Object> chain = this.advised.getInterceptorsAndDynamicInterceptionAdvice(methods[x], rootClass);
					fixedCallbacks[x] = new FixedChainStaticTargetInterceptor(
							chain, this.advised.getTargetSource().getTarget(), this.advised.getTargetClass());
					this.fixedInterceptorMap.put(methods[x].toString(), x);
				}

				// Now copy both the callbacks from mainCallbacks
				// and fixedCallbacks into the callbacks array.
				callbacks = new Callback[mainCallbacks.length + fixedCallbacks.length];
				System.arraycopy(mainCallbacks, 0, callbacks, 0, mainCallbacks.length);
				System.arraycopy(fixedCallbacks, 0, callbacks, mainCallbacks.length, fixedCallbacks.length);
				this.fixedInterceptorOffset = mainCallbacks.length;
			}
			else {
				callbacks = mainCallbacks;
			}
			return callbacks;
		}


2. JdkDynamicAopProxy

	public Object getProxy(@Nullable ClassLoader classLoader) {
		if (logger.isDebugEnabled()) {
			logger.debug("Creating JDK dynamic proxy: target source is " + this.advised.getTargetSource());
		}
		Class<?>[] proxiedInterfaces = AopProxyUtils.completeProxiedInterfaces(this.advised, true);
		findDefinedEqualsAndHashCodeMethods(proxiedInterfaces);
		return Proxy.newProxyInstance(classLoader, proxiedInterfaces, this);
	}
