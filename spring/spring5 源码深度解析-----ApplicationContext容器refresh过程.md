
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


# BeanFactory的后处理

BeanFactory作为spring中容器功能的基础，用于存放所有已经加载的bean，为例保证程序上的高可扩展性，spring针对BeanFactory做了大量的扩展，比如我们熟悉的PostProcessor就是在这里实现的。接下来我们就深入分析下BeanFactory后处理

激活注册的BeanFactoryPostProcessor

在正式介绍BeanFactoryPostProcessor的后处理前我们先简单的了解下其用法，BeanFactoryPostProcessor接口跟BeanPostProcessor类似，都可以对bean的定义（配置元数据）进行处理，也就是说spring IoC容器允许BeanFactoryPostProcessor在容器实际实例化任何其他的bean之前读取配置元数据，并可能修改他。也可以配置多个BeanFactoryPostProcessor，可以通过order属性来控制BeanFactoryPostProcessor的执行顺序（此属性必须当BeanFactoryPostProcessor实现了Ordered的接口时才可以赊账，因此在实现BeanFactoryPostProcessor时应该考虑实现Ordered接口）。

如果想改变世界的bean实例（例如从配置元数据创建的对象），那最好使用BeanPostProcessor。同样的BeanFactoryPostProcessor的作用域范围是容器级别的，它只是和你所使用的容器有关。如果你在容器中定义了一个BeanFactoryPostProcessor，它仅仅对此容器中的bean进行后置处理。BeanFactoryPostProcessor不会对定义在另一个容器中的bean进行后置处理，即使这两个容器都在同一层次上。在spring中存在对于BeanFactoryPostProcessor的典型应用，如PropertyPlaceholderConfigurer。

BeanFactoryPostProcessor的典型应用：PropertyPlaceholderConfigurer

有时候我们在阅读spring的配置文件中的Bean的描述时，会遇到类似如下情况：

```xml
<bean id="user" class="com.yhl.myspring.demo.applicationcontext.User">
    <property name="name" value="${user.name}"/>
    <property name="birthday" value="${user.birthday"/>
</bean>
```

这其中出现了变量：{user.name}、{user.birthday}，这是spring的分散配置，可以在另外的配置文件中为user.name、user.birthday指定值，例如在bean.properties文件中定义

```xml
user.name = xiaoming
user.birthday = 2019-04-19
```

当访问名为user的bean时，其name属性就会被字符串xiaoming替换，那spring框架是怎么知道存在这样的配置文件呢，这个就是PropertyPlaceholderConfigurer，需要在配置文件中添加一下代码：

```xml
<bean id="userHandler" class="org.springframework.beans.factory.config.PropertyPlaceholderConfigurer">
    <property name="locations">
        <list>
            <value>classpath:bean.properties</value>
        </list>
    </property>
</bean>
```

在这个bean中指定了配置文件的位置。其实还是有个问题，这个userHandler只不过是spring框架管理的一个bean，并没有被别的bean或者对象引用，spring的beanFactory是怎么知道这个需要从这个bean中获取配置信息呢？我们看下PropertyPlaceholderConfigurer这个类的层次结构，如下图： 

![image](https://github.com/ProgrammerGoGo/document/assets/98639494/cb40ce87-81f1-4f05-974e-71992b42a34c)

从上图中我们可以看到PropertyPlaceholderConfigurer间接的继承了BeanFactoryPostProcessor接口，这是一个很特别的接口，当spring加载任何实现了这个接口的bean的配置时，都会在bean工厂载入所有bean的配置之后执行postProcessBeanFactory方法。在PropertyResourceConfigurer类中实现了postProcessBeanFactory方法，在方法中先后调用了mergeProperties、convertProperties、processProperties这三个方法，分别得到配置，将得到的配置转换为合适的类型，最后将配置内容告知BeanFactory。

正是通过实现BeanFactoryPostProcessor接口，BeanFactory会在实例化任何bean之前获得配置信息，从而能够正确的解析bean描述文件中的变量引用。

```java
public void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) throws BeansException {
    try {
        Properties mergedProps = this.mergeProperties();
        this.convertProperties(mergedProps);
        this.processProperties(beanFactory, mergedProps);
    } catch (IOException var3) {
        throw new BeanInitializationException("Could not load properties", var3);
    }
}
```

自定义BeanFactoryPostProcessor

编写实现了BeanFactoryPostProcessor接口的MyBeanFactoryPostProcessor的容器后处理器，如下代码：

```java
public class MyBeanFactoryPostProcessor implements BeanFactoryPostProcessor {
    public void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) throws BeansException {
        System.out.println("对容器进行后处理。。。。");
    }
}
```

然后在配置文件中注册这个bean，如下：

```mxl
<bean id="myPost" class="com.yhl.myspring.demo.applicationcontext.MyBeanFactoryPostProcessor"></bean>
```

最后编写测试代码：

```java
public class Test {
    public static void main(String[] args) {
        ApplicationContext context = new ClassPathXmlApplicationContext("applicationContext.xml");

        User user = (User)context.getBean("user");
        System.out.println(user.getName());

    }
}
```

# 激活BeanFactoryPostProcessor

在了解BeanFactoryPostProcessor的用法后我们便可以深入的研究BeanFactoryPostProcessor的调用过程了，其实是在方法invokeBeanFactoryPostProcessors(beanFactory)中实现的，进入到方法内部：

```java
public static void invokeBeanFactoryPostProcessors(ConfigurableListableBeanFactory beanFactory, List<BeanFactoryPostProcessor> beanFactoryPostProcessors) {

    // Invoke BeanDefinitionRegistryPostProcessors first, if any.
    // 1、首先调用BeanDefinitionRegistryPostProcessors
    Set<String> processedBeans = new HashSet<>();

    // beanFactory是BeanDefinitionRegistry类型
    if (beanFactory instanceof BeanDefinitionRegistry) {
        BeanDefinitionRegistry registry = (BeanDefinitionRegistry) beanFactory;
        // 定义BeanFactoryPostProcessor
        List<BeanFactoryPostProcessor> regularPostProcessors = new ArrayList<>();
        // 定义BeanDefinitionRegistryPostProcessor集合
        List<BeanDefinitionRegistryPostProcessor> registryProcessors = new ArrayList<>();

        // 循环手动注册的beanFactoryPostProcessors
        for (BeanFactoryPostProcessor postProcessor : beanFactoryPostProcessors) {
            // 如果是BeanDefinitionRegistryPostProcessor的实例话,则调用其postProcessBeanDefinitionRegistry方法,对bean进行注册操作
            if (postProcessor instanceof BeanDefinitionRegistryPostProcessor) {
                // 如果是BeanDefinitionRegistryPostProcessor类型,则直接调用其postProcessBeanDefinitionRegistry
                BeanDefinitionRegistryPostProcessor registryProcessor = (BeanDefinitionRegistryPostProcessor) postProcessor;
                registryProcessor.postProcessBeanDefinitionRegistry(registry);
                registryProcessors.add(registryProcessor);
            }
            // 否则则将其当做普通的BeanFactoryPostProcessor处理,直接加入regularPostProcessors集合,以备后续处理
            else {
                regularPostProcessors.add(postProcessor);
            }
        }

        // Do not initialize FactoryBeans here: We need to leave all regular beans
        // uninitialized to let the bean factory post-processors apply to them!
        // Separate between BeanDefinitionRegistryPostProcessors that implement
        // PriorityOrdered, Ordered, and the rest.
        List<BeanDefinitionRegistryPostProcessor> currentRegistryProcessors = new ArrayList<>();

        // First, invoke the BeanDefinitionRegistryPostProcessors that implement PriorityOrdered.
        // 首先调用实现了PriorityOrdered(有限排序接口)的BeanDefinitionRegistryPostProcessors
        String[] postProcessorNames = beanFactory.getBeanNamesForType(BeanDefinitionRegistryPostProcessor.class, true, false);
        for (String ppName : postProcessorNames) {
            if (beanFactory.isTypeMatch(ppName, PriorityOrdered.class)) {
                currentRegistryProcessors.add(beanFactory.getBean(ppName, BeanDefinitionRegistryPostProcessor.class));
                processedBeans.add(ppName);
            }
        }
        // 排序
        sortPostProcessors(currentRegistryProcessors, beanFactory);
        // 加入registryProcessors集合
        registryProcessors.addAll(currentRegistryProcessors);
        // 调用所有实现了PriorityOrdered的的BeanDefinitionRegistryPostProcessors的postProcessBeanDefinitionRegistry方法,注册bean
        invokeBeanDefinitionRegistryPostProcessors(currentRegistryProcessors, registry);
        // 清空currentRegistryProcessors,以备下次使用
        currentRegistryProcessors.clear();

        // Next, invoke the BeanDefinitionRegistryPostProcessors that implement Ordered.
        // 其次,调用实现了Ordered(普通排序接口)的BeanDefinitionRegistryPostProcessors
        postProcessorNames = beanFactory.getBeanNamesForType(BeanDefinitionRegistryPostProcessor.class, true, false);
        for (String ppName : postProcessorNames) {
            if (!processedBeans.contains(ppName) && beanFactory.isTypeMatch(ppName, Ordered.class)) {
                currentRegistryProcessors.add(beanFactory.getBean(ppName, BeanDefinitionRegistryPostProcessor.class));
                processedBeans.add(ppName);
            }
        }
        // 排序
        sortPostProcessors(currentRegistryProcessors, beanFactory);
        // 加入registryProcessors集合
        registryProcessors.addAll(currentRegistryProcessors);
        // 调用所有实现了PriorityOrdered的的BeanDefinitionRegistryPostProcessors的postProcessBeanDefinitionRegistry方法,注册bean
        invokeBeanDefinitionRegistryPostProcessors(currentRegistryProcessors, registry);
        // 清空currentRegistryProcessors,以备下次使用
        currentRegistryProcessors.clear();

        // Finally, invoke all other BeanDefinitionRegistryPostProcessors until no further ones appear.
        // 最后,调用其他的BeanDefinitionRegistryPostProcessors
        boolean reiterate = true;
        while (reiterate) {
            reiterate = false;
            postProcessorNames = beanFactory.getBeanNamesForType(BeanDefinitionRegistryPostProcessor.class, true, false);
            for (String ppName : postProcessorNames) {
                if (!processedBeans.contains(ppName)) {
                    currentRegistryProcessors.add(beanFactory.getBean(ppName, BeanDefinitionRegistryPostProcessor.class));
                    processedBeans.add(ppName);
                    reiterate = true;
                }
            }
            // 排序
            sortPostProcessors(currentRegistryProcessors, beanFactory);
            // 加入registryProcessors集合
            registryProcessors.addAll(currentRegistryProcessors);
            // 调用其他的BeanDefinitionRegistryPostProcessors的postProcessBeanDefinitionRegistry方法,注册bean
            invokeBeanDefinitionRegistryPostProcessors(currentRegistryProcessors, registry);
            // 清空currentRegistryProcessors,以备下次使用
            currentRegistryProcessors.clear();
        }

        // Now, invoke the postProcessBeanFactory callback of all processors handled so far.
        // 调用所有BeanDefinitionRegistryPostProcessor(包括手动注册和通过配置文件注册)
        // 和BeanFactoryPostProcessor(只有手动注册)的回调函数-->postProcessBeanFactory
        invokeBeanFactoryPostProcessors(registryProcessors, beanFactory);
        invokeBeanFactoryPostProcessors(regularPostProcessors, beanFactory);
    }

    // 2、如果不是BeanDefinitionRegistry的实例,那么直接调用其回调函数即可-->postProcessBeanFactory
    else {
        // Invoke factory processors registered with the context instance.
        invokeBeanFactoryPostProcessors(beanFactoryPostProcessors, beanFactory);
    }

    // Do not initialize FactoryBeans here: We need to leave all regular beans
    // uninitialized to let the bean factory post-processors apply to them!
    // 3、上面的代码已经处理完了所有的BeanDefinitionRegistryPostProcessors和手动注册的BeanFactoryPostProcessor
    // 接下来要处理通过配置文件注册的BeanFactoryPostProcessor
    // 首先获取所有的BeanFactoryPostProcessor(注意:这里获取的集合会包含BeanDefinitionRegistryPostProcessors)
    String[] postProcessorNames = beanFactory.getBeanNamesForType(BeanFactoryPostProcessor.class, true, false);

    // Separate between BeanFactoryPostProcessors that implement PriorityOrdered, Ordered, and the rest.
    // 这里,将实现了PriorityOrdered,Ordered的处理器和其他的处理器区分开来,分别进行处理
    // PriorityOrdered有序处理器
    List<BeanFactoryPostProcessor> priorityOrderedPostProcessors = new ArrayList<>();
    // Ordered有序处理器
    List<String> orderedPostProcessorNames = new ArrayList<>();
    // 无序处理器
    List<String> nonOrderedPostProcessorNames = new ArrayList<>();
    for (String ppName : postProcessorNames) {
        // 判断processedBeans是否包含当前处理器(processedBeans中的处理器已经被处理过);如果包含,则不做任何处理
        if (processedBeans.contains(ppName)) {
            // skip - already processed in first phase above
        }
        else if (beanFactory.isTypeMatch(ppName, PriorityOrdered.class)) {
            // 加入到PriorityOrdered有序处理器集合
            priorityOrderedPostProcessors.add(beanFactory.getBean(ppName, BeanFactoryPostProcessor.class));
        }
        else if (beanFactory.isTypeMatch(ppName, Ordered.class)) {
            // 加入到Ordered有序处理器集合
            orderedPostProcessorNames.add(ppName);
        }
        else {
            // 加入到无序处理器集合
            nonOrderedPostProcessorNames.add(ppName);
        }
    }

    // First, invoke the BeanFactoryPostProcessors that implement PriorityOrdered.
    // 首先调用实现了PriorityOrdered接口的处理器
    sortPostProcessors(priorityOrderedPostProcessors, beanFactory);
    invokeBeanFactoryPostProcessors(priorityOrderedPostProcessors, beanFactory);

    // Next, invoke the BeanFactoryPostProcessors that implement Ordered.
    // 其次,调用实现了Ordered接口的处理器
    List<BeanFactoryPostProcessor> orderedPostProcessors = new ArrayList<>();
    for (String postProcessorName : orderedPostProcessorNames) {
        orderedPostProcessors.add(beanFactory.getBean(postProcessorName, BeanFactoryPostProcessor.class));
    }
    sortPostProcessors(orderedPostProcessors, beanFactory);
    invokeBeanFactoryPostProcessors(orderedPostProcessors, beanFactory);

    // Finally, invoke all other BeanFactoryPostProcessors.
    // 最后,调用无序处理器
    List<BeanFactoryPostProcessor> nonOrderedPostProcessors = new ArrayList<>();
    for (String postProcessorName : nonOrderedPostProcessorNames) {
        nonOrderedPostProcessors.add(beanFactory.getBean(postProcessorName, BeanFactoryPostProcessor.class));
    }
    invokeBeanFactoryPostProcessors(nonOrderedPostProcessors, beanFactory);

    // Clear cached merged bean definitions since the post-processors might have
    // modified the original metadata, e.g. replacing placeholders in values...
    // 清理元数据
    beanFactory.clearMetadataCache();
}
```









