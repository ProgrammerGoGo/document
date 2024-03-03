
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



