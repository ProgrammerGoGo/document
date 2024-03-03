
> [spring5 源码深度解析-----ApplicationContext容器refresh过程](https://www.cnblogs.com/java-chen-hao/p/11579591.html)


refresh方法：

```java
public void refresh() throws BeansException, IllegalStateException {
    synchronized (this.startupShutdownMonitor) {
        //准备刷新的上下文 环境  
        prepareRefresh();
        //初始化BeanFactory，并进行XML文件读取  
        /* 
         * ClassPathXMLApplicationContext包含着BeanFactory所提供的一切特征，在这一步骤中将会复用 
         * BeanFactory中的配置文件读取解析及其他功能，这一步之后，ClassPathXmlApplicationContext 
         * 实际上就已经包含了BeanFactory所提供的功能，也就是可以进行Bean的提取等基础操作了。 
         */  
        ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();
        //对beanFactory进行各种功能填充  
        prepareBeanFactory(beanFactory);
        try {
            //子类覆盖方法做额外处理  
            /* 
             * Spring之所以强大，为世人所推崇，除了它功能上为大家提供了便利外，还有一方面是它的 
             * 完美架构，开放式的架构让使用它的程序员很容易根据业务需要扩展已经存在的功能。这种开放式 
             * 的设计在Spring中随处可见，例如在本例中就提供了一个空的函数实现postProcessBeanFactory来 
             * 方便程序猿在业务上做进一步扩展 
             */ 
            postProcessBeanFactory(beanFactory);
            //激活各种beanFactory处理器  
            invokeBeanFactoryPostProcessors(beanFactory);
            //注册拦截Bean创建的Bean处理器，这里只是注册，真正的调用实在getBean时候 
            registerBeanPostProcessors(beanFactory);
            //为上下文初始化Message源，即不同语言的消息体，国际化处理  
            initMessageSource();
            //初始化应用消息广播器，并放入“applicationEventMulticaster”bean中  
            initApplicationEventMulticaster();
            //留给子类来初始化其它的Bean  
            onRefresh();
            //在所有注册的bean中查找Listener bean，注册到消息广播器中  
            registerListeners();
            //初始化剩下的单实例（非惰性的）  
            finishBeanFactoryInitialization(beanFactory);
            //完成刷新过程，通知生命周期处理器lifecycleProcessor刷新过程，同时发出ContextRefreshEvent通知别人  
            finishRefresh();
        }
        catch (BeansException ex) {
            if (logger.isWarnEnabled()) {
                logger.warn("Exception encountered during context initialization - " +
                        "cancelling refresh attempt: " + ex);
            }
            destroyBeans();
            cancelRefresh(ex);
            throw ex;
        }
        finally {
            resetCommonCaches();
        }
    }
}
```

分析下代码的步骤：

（1）初始化前的准备工作，例如对系统属性或者环境变量进行准备及验证。  
在某种情况下项目的使用需要读取某些系统变量，而这个变量的设置很可能会影响着系统的正确性，那么ClassPathXmlApplicationContext为我们提供的这个准备函数就显得非常必要，他可以在spring启动的时候提前对必须的环境变量进行存在性验证。

（2）初始化BeanFactory，并进行XML文件读取。  
之前提到ClassPathXmlApplicationContext包含着对BeanFactory所提供的一切特征，那么这一步中将会复用BeanFactory中的配置文件读取解析其他功能，这一步之后ClassPathXmlApplicationContext实际上就已经包含了BeanFactory所提供的功能，也就是可以进行Bean的提取等基本操作了。

（3）对BeanFactory进行各种功能填充  
@Qualifier和@Autowired应该是大家非常熟悉的注解了，那么这两个注解正是在这一步骤中增加支持的。

（4）子类覆盖方法做额外处理。  
spring之所以强大，为世人所推崇，除了它功能上为大家提供了遍历外，还有一方面是它完美的架构，开放式的架构让使用它的程序员很容易根据业务需要扩展已经存在的功能。这种开放式的设计在spring中随处可见，例如本利中就提供了一个空的函数实现postProcessBeanFactory来方便程序员在业务上做进一步的扩展。

（5）激活各种BeanFactory处理器  

（6）注册拦截bean创建的bean处理器，这里只是注册，真正的调用是在getBean时候

（7）为上下文初始化Message源，及对不同语言的小西天进行国际化处理

（8）初始化应用消息广播器，并放入“applicationEventMulticaster”bean中

（9）留给子类来初始化其他的bean

（10）在所有注册的bean中查找listener bean，注册到消息广播器中

（11）初始化剩下的单实例（非惰性的）

（12）完成刷新过程，通知生命周期处理器lifecycleProcessor刷新过程，同时发出ContextRefreshEvent通知别人。


# prepareRefresh 刷新上下文的准备工作

```java
/**
 * 准备刷新上下文环境，设置它的启动日期和活动标志，以及执行任何属性源的初始化。
 * Prepare this context for refreshing, setting its startup date and
 * active flag as well as performing any initialization of property sources.
 */
protected void prepareRefresh() {
    this.startupDate = System.currentTimeMillis();
    this.closed.set(false);
    this.active.set(true);

    // 在上下文环境中初始化任何占位符属性源。(空的方法,留给子类覆盖)
    initPropertySources();

    // 验证需要的属性文件是否都已放入环境中
    getEnvironment().validateRequiredProperties();

    // 允许收集早期的应用程序事件，一旦有了多播器，就可以发布……
    this.earlyApplicationEvents = new LinkedHashSet<>();
}
```

# obtainFreshBeanFactory->读取xml并初始化BeanFactory

obtainFreshBeanFactory方法从字面理解是获取beanFactory。ApplicationContext是对BeanFactory的扩展，在其基础上添加了大量的基础应用，obtainFreshBeanFactory正是实现beanFactory的地方，经过这个函数后ApplicationContext就有了BeanFactory的全部功能。我们看下此方法的代码：

```java
protected ConfigurableListableBeanFactory obtainFreshBeanFactory() {
    //初始化BeanFactory,并进行XML文件读取，并将得到的BeanFactory记录在当前实体的属性中  
    refreshBeanFactory();
    //返回当前实体的beanFactory属性 
    ConfigurableListableBeanFactory beanFactory = getBeanFactory();
    if (logger.isDebugEnabled()) {
        logger.debug("Bean factory for " + getDisplayName() + ": " + beanFactory);
    }
    return beanFactory;
}
```

继续深入到refreshBeanFactory方法中，方法的实现是在AbstractRefreshableApplicationContext中：

```java
@Override
protected final void refreshBeanFactory() throws BeansException {
    if (hasBeanFactory()) {
        destroyBeans();
        closeBeanFactory();
    }
    try {
        //创建DefaultListableBeanFactory  
        /* 
         * 以前我们分析BeanFactory的时候，不知道是否还有印象，声明方式为：BeanFactory bf =  
         * new XmlBeanFactory("beanFactoryTest.xml")，其中的XmlBeanFactory继承自DefaulltListableBeanFactory; 
         * 并提供了XmlBeanDefinitionReader类型的reader属性，也就是说DefaultListableBeanFactory是容器的基础。必须 
         * 首先要实例化。 
         */  
        DefaultListableBeanFactory beanFactory = createBeanFactory();
        //为了序列化指定id,如果需要的话，让这个BeanFactory从id反序列化到BeanFactory对象  
        beanFactory.setSerializationId(getId());
        //定制beanFactory，设置相关属性，包括是否允许覆盖同名称的不同定义的对象以及循环依赖以及设置  
        //@Autowired和Qualifier注解解析器QualifierAnnotationAutowireCandidateResolver  
        customizeBeanFactory(beanFactory);
        //加载BeanDefiniton  
        loadBeanDefinitions(beanFactory);
        synchronized (this.beanFactoryMonitor) {
            //使用全局变量记录BeanFactory实例。  
            //因为DefaultListableBeanFactory类型的变量beanFactory是函数内部的局部变量，  
            //所以要使用全局变量记录解析结果  
            this.beanFactory = beanFactory;
        }
    }
    catch (IOException ex) {
        throw new ApplicationContextException("I/O error parsing bean definition source for " + getDisplayName(), ex);
    }
}
```

加载BeanDefinition

在第一步中提到了将ClassPathXmlApplicationContext与XMLBeanFactory创建的对比，除了初始化DefaultListableBeanFactory外，还需要XmlBeanDefinitionReader来读取XML，那么在loadBeanDefinitions方法中首先要做的就是初始化XmlBeanDefinitonReader，我们跟着到loadBeanDefinitions(beanFactory)方法体中，我们看到的是在AbstractXmlApplicationContext中实现的，具体代码如下：

```java
@Override
protected void loadBeanDefinitions(DefaultListableBeanFactory beanFactory) throws BeansException, IOException {
    // Create a new XmlBeanDefinitionReader for the given BeanFactory.
    XmlBeanDefinitionReader beanDefinitionReader = new XmlBeanDefinitionReader(beanFactory);

    // Configure the bean definition reader with this context's
    // resource loading environment.
    beanDefinitionReader.setEnvironment(this.getEnvironment());
    beanDefinitionReader.setResourceLoader(this);
    beanDefinitionReader.setEntityResolver(new ResourceEntityResolver(this));

    // Allow a subclass to provide custom initialization of the reader,
    // then proceed with actually loading the bean definitions.
    initBeanDefinitionReader(beanDefinitionReader);
    loadBeanDefinitions(beanDefinitionReader);
}
```

在初始化了DefaultListableBeanFactory和XmlBeanDefinitionReader后，就可以进行配置文件的读取了。继续进入到loadBeanDefinitions(beanDefinitionReader)方法体中，代码如下：

```java
protected void loadBeanDefinitions(XmlBeanDefinitionReader reader) throws BeansException, IOException {
    Resource[] configResources = getConfigResources();
    if (configResources != null) {
        reader.loadBeanDefinitions(configResources);
    }
    String[] configLocations = getConfigLocations();
    if (configLocations != null) {
        reader.loadBeanDefinitions(configLocations);
    }
}
```

因为在XmlBeanDefinitionReader中已经将之前初始化的DefaultListableBeanFactory注册进去了，所以XmlBeanDefinitionReader所读取的BeanDefinitionHolder都会注册到DefinitionListableBeanFactory中，也就是经过这个步骤，DefaultListableBeanFactory的变量beanFactory已经包含了所有解析好的配置。

# 功能扩展

如上图所示prepareBeanFactory(beanFactory)就是在功能上扩展的方法，而在进入这个方法前spring已经完成了对配置的解析，接下来我们详细分析下次函数，进入方法体：

```java
protected void prepareBeanFactory(ConfigurableListableBeanFactory beanFactory) {
    // Tell the internal bean factory to use the context's class loader etc.
    //设置beanFactory的classLoader为当前context的classloader  
    beanFactory.setBeanClassLoader(getClassLoader());
    //设置beanFactory的表达式语言处理器，Spring3增加了表达式语言的支持，  
    //默认可以使用#{bean.xxx}的形式来调用相关属性值  
    beanFactory.setBeanExpressionResolver(new StandardBeanExpressionResolver(beanFactory.getBeanClassLoader()));
    //为beanFactory增加了一个的propertyEditor，这个主要是对bean的属性等设置管理的一个工具
    beanFactory.addPropertyEditorRegistrar(new ResourceEditorRegistrar(this, getEnvironment()));

    // Configure the bean factory with context callbacks.
    beanFactory.addBeanPostProcessor(new ApplicationContextAwareProcessor(this));
    //设置了几个忽略自动装配的接口
    beanFactory.ignoreDependencyInterface(EnvironmentAware.class);
    beanFactory.ignoreDependencyInterface(EmbeddedValueResolverAware.class);
    beanFactory.ignoreDependencyInterface(ResourceLoaderAware.class);
    beanFactory.ignoreDependencyInterface(ApplicationEventPublisherAware.class);
    beanFactory.ignoreDependencyInterface(MessageSourceAware.class);
    beanFactory.ignoreDependencyInterface(ApplicationContextAware.class);

    // BeanFactory interface not registered as resolvable type in a plain factory.
    // MessageSource registered (and found for autowiring) as a bean.
    //设置了几个自动装配的特殊规则
    beanFactory.registerResolvableDependency(BeanFactory.class, beanFactory);
    beanFactory.registerResolvableDependency(ResourceLoader.class, this);
    beanFactory.registerResolvableDependency(ApplicationEventPublisher.class, this);
    beanFactory.registerResolvableDependency(ApplicationContext.class, this);

    // Register early post-processor for detecting inner beans as ApplicationListeners.
    beanFactory.addBeanPostProcessor(new ApplicationListenerDetector(this));

    // Detect a LoadTimeWeaver and prepare for weaving, if found.
    //增加对AspectJ的支持 
    if (beanFactory.containsBean(LOAD_TIME_WEAVER_BEAN_NAME)) {
        beanFactory.addBeanPostProcessor(new LoadTimeWeaverAwareProcessor(beanFactory));
        // Set a temporary ClassLoader for type matching.
        beanFactory.setTempClassLoader(new ContextTypeMatchClassLoader(beanFactory.getBeanClassLoader()));
    }

    // Register default environment beans.
    //添加默认的系统环境bean  
    if (!beanFactory.containsLocalBean(ENVIRONMENT_BEAN_NAME)) {
        beanFactory.registerSingleton(ENVIRONMENT_BEAN_NAME, getEnvironment());
    }
    if (!beanFactory.containsLocalBean(SYSTEM_PROPERTIES_BEAN_NAME)) {
        beanFactory.registerSingleton(SYSTEM_PROPERTIES_BEAN_NAME, getEnvironment().getSystemProperties());
    }
    if (!beanFactory.containsLocalBean(SYSTEM_ENVIRONMENT_BEAN_NAME)) {
        beanFactory.registerSingleton(SYSTEM_ENVIRONMENT_BEAN_NAME, getEnvironment().getSystemEnvironment());
    }
}
```

详细分析下代码发现上面函数主要是在以下方法进行了扩展：  

（1）对SPEL语言的支持

（2）增加对属性编辑器的支持

（3）增加对一些内置类的支持，如EnvironmentAware、MessageSourceAware的注入

（4）设置了依赖功能可忽略的接口

（5）注册一些固定依赖的属性

（6）增加了AspectJ的支持

（7）将相关环境变量及属性以单例模式注册 
