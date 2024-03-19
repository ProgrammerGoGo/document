
> [Spring是如何利用“三级缓存”巧妙解决Bean的循环依赖问题的？](https://zhuanlan.zhihu.com/p/162316846)  
> [spring源码系列（一）——spring循环引用](https://blog.csdn.net/java_lyvee/article/details/101793774?spm=1001.2014.3001.5502)  
> [spring循环依赖](https://blog.csdn.net/wang489687009/article/details/84538634?spm=1001.2014.3001.5502)  
> [【Spring源码三千问】哪些循环依赖问题Spring解决不了？](https://blog.csdn.net/wang489687009/article/details/120546430)  
> [【Spring源码三千问】@Lazy原理分析——它为什么可以解决特殊的循环依赖问题？](https://blog.csdn.net/wang489687009/article/details/120577472)


```java
@Configurable
@ComponentScan("com.shadow")
public class Appconfig {
}


@Component
public class X {

	@Autowired
	Y y;

	public X(){
		System.out.println("X create");
	}
}


@Component
public class Y {
	@Autowired
	X x;
	
	public Y(){
		System.out.println("Y create");
	}
}



@Component
public class Z implements ApplicationContextAware {
	@Autowired
	X x;//注入X

    //构造方法
	public Z(){
		System.out.println("Z create");
	}

    //生命周期初始化回调方法
	@PostConstruct
	public void zinit(){
		System.out.println("call z lifecycle init callback");
	}

	//ApplicationContextAware 回调方法
	@Override
	public void setApplicationContext(ApplicationContext ac) {
		System.out.println("call aware callback");
	}
}
```

spring的bean是在 `AbstractApplicationContext#finishBeanFactoryInitialization()` 方法完成的初始化，即循环依赖也在这个方法里面完成的。该方法里面调用了一个非常重要的方法 doGetBean 的方法.

针对上面的源码我做一个简单的总结：首先spring从单例池当中获取x，前面说过获取不到，然后判断是否在正在创建bean的集合当中，前面分析过这个集合现在存在x，和y；所以if成立进入分支；进入分支spring直接从三级缓存中获取x，根据前面的分析三级缓存当中现在什么都没有，故而返回nll；进入下一个if分支，从二级缓存中获取一个ObjectFactory工厂对象；根据前面分析，二级缓存中存在x，故而可以获取到；跟着调用singletonFactory.getObject();拿到一个半成品的x bean对象；然后把这个x对象放到三级缓存，同时把二级缓存中x清除（此时二级缓存中只存在一个y了，而三级缓存中多了一个x）；

问题1、为什么首先是从三级缓存中取呢？主要是为了性能，因为三级缓存中存的是一个x对象，如果能取到则不去二级找了；哪有人会问二级有什么用呢？为什么一开始要存工厂呢？为什么一开始不直接存三级缓存？这里稍微有点复杂，如果直接存到三级缓存，只能存一个对象，假设以前存这个对象的时候这对象的状态为xa，但是我们这里y要注入的x为xc状态，那么则无法满足；但是如果存一个工厂，工厂根据情况产生任意xa或者xb或者xc等等情况；比如说aop的情况下x注入y，y也注入x；而y中注入的x需要加代理（aop），但是加代理的逻辑在注入属性之后，也就是x的生命周期周到注入属性的时候x还不是一个代理对象，那么这个时候把x存起来，然后注入y，获取、创建y，y注入x，获取x；拿出来的x是一个没有代理的对象；但是如果存的是个工厂就不一样；首先把一个能产生x的工厂存起来，然后注入y，注入y的时候获取、创建y，y注入x，获取x，先从三级缓存获取，为null，然后从二级缓存拿到一个工厂，调用工厂的getObject()；spring在getObject方法中判断这个时候x被aop配置了故而需要返回一个代理的x出来注入给y。当然有的读者会问你不是前面说过getObject会返回一个当前状态的xbean嘛？我说这个的前提是不去计较getObject的具体源码，因为这块东西比较复杂，需要去了解spring的后置处理器功能，这里先不讨论，总之getObject会根据情况返回一个x，但是这个x是什么状态，spring会自己根据情况返回；

问题2、为什么要从二级缓存remove？因为如果存在比较复杂的循环依赖可以提高性能；比如x，y，z相互循环依赖，那么第一次y注入x的时候从二级缓存通过工厂返回了一个x，放到了三级缓存，而第二次z注入x的时候便不需要再通过工厂去获得x对象了。因为if分支里面首先是访问三级缓存；至于remove则是为了gc吧；


Spring 默认为我们解决了常规场景下的循环依赖问题。但是有些特殊的场景下的循环依赖，Spring 默认是没有解决的。
主要的场景如下：

* prototype 类型的循环依赖
* constructor 注入的循环依赖
* @Async 类型的 Bean 的循环依赖

这些特殊的场景，我们都可以通过 @Lazy 来解决。  
@Lazy 的本质就是将注入的依赖变成了一个代理对象。使用 @Lazy 时，不会触发依赖 bean 的加载。



# Spring 为什么不用二级缓存来解决循环依赖问题？

> [【Spring源码三千问】为什么要用三级缓存来解决循环依赖问题？二级缓存行不行？一级缓存行不行？](https://blog.csdn.net/wang489687009/article/details/120655156)

Spring 原本的设计是，bean 的创建过程分三个阶段：

1 创建实例 createBeanInstance – 创建出 bean 的原始对象

2 填充依赖 populateBean – 利用反射，使用 BeanWrapper 来设置属性值

3 initializeBean – 执行 bean 创建后的处理，包括 AOP 对象的产生

在没有循环依赖的场景下：第 1,2 步都是 bean 的原始对象，第 3 步 initializeBean 时，才会生成 AOP 代理对象。

循环依赖属于一个特殊的场景，如果在第 3 步 initializeBean 时才去生成 AOP 代理 bean 的话，那么在第 2 步 populateBean 注入循环依赖 bean 时就拿不到 AOP 代理 bean 进行注入。所以，循环依赖打破了 AOP 代理 bean 生成的时机，需要在 populateBean 之前就生成 AOP 代理 bean。而且，生成 AOP 代理需要执行 BeanPostProcessor，而 Spring 原本的设计是在第 3 步 initializeBean 时才去调用 BeanPostProcessor 的。并不是每个 bean 都需要进行这样的处理，所以， Spring 没有直接在 createBeanInstance 之后直接生成 bean 的早期引用，而是将 bean 的原始对象包装成了一个 ObjectFactory 放到了三级缓存 Map<String, Object> earlySingletonObjects。当需要用到 bean 的早期引用的时候，才通过三级缓存 Map<String, ObjectFactory<?>> singletonFactories 来进行获取。

如果只使用二级缓存来解决循环依赖的话，那么每个 bean 的创建流程中都需要插入一个流程——创建 bean 的早期引用放入二级缓存。其实，在真实的开发中，绝大部分的情况下都不涉及到循环依赖，而且 createBeanInstance --> populateBean --> initializeBean 这个流程也更加符合常理。

所以，猜想 Spring 不用二级缓存来解决循环依赖问题，是为了保证处理时清晰明了，bean 的创建就是三个阶段: createBeanInstance --> populateBean --> initializeBean只有碰到 AOP 代理 bean 被循环依赖时的场景，才去特殊处理，提前生成 AOP 代理 bean

# 辟谣：使用二级缓存解决不了 AOP 代理 bean 的循环依赖？

在网上有看到文章分析，说 Spring 必须要使用三级缓存是因为只用二级缓存解决不了 AOP 代理 bean 的循环依赖问题。通过前面的分析，理论上来说，使用二级缓存是可以解决 AOP 代理 bean 的循环依赖的。只是 Spring 没有选择这样去实现。

AOP 代理的产生是在：bean 创建的第三个阶段 initializeBean 的时候，它会处理 @Async、@Schedule 的代理对象 和 @Around 等切入点表达增强的代理对象 AsyncAnnotationBeanPostProcessor —> 处理 @Async AnnotationAwareAspectJAutoProxyCreator -——> 处理 @Around 等 advisor 切入点表达式的 AOP 代理（@Transactional 也归为 advisor 一类，它使用的是内置的 BeanFactoryTransactionAttributeSourceAdvisor）

当 AOP 代理 bean 被循环依赖时，AOP 代理的产生时机就会提前。Spring 会提前通过三级缓存 singletonFactories 来获取到 bean 的早期引用，这个早期引用就是 Spring 容器最终暴露的 bean 的引用。




