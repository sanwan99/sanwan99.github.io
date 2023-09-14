---
title: 3、Spring之Bean生命周期源码解析
date: '2023-09-11 11:36:08'
updated: '2023-09-14 11:06:48'
permalink: /post/3-bean-s-life-cycle-source-code-analysis-7npda.html
comments: true
toc: true
---
# 3、Spring之Bean生命周期源码解析

* 📄 [AutowiredAnnotationBeanPostProcessor](siyuan://blocks/20230911140713-1rluqq2)
* 📄 [BeanFactoryAware](siyuan://blocks/20230911140830-7u0vdhc)
* 📄 [BeanPostProcessor](siyuan://blocks/20230911140601-p5qtko8)
* 📄 [CachingMetadataReaderFactory](siyuan://blocks/20230911122212-o8zt0a3)
* 📄 [ClassUtil.getDefaultClassLoader()](siyuan://blocks/20230911140434-6xdbsg0)
* 📄 [InstantiationAwareBeanPostProcessor](siyuan://blocks/20230911140620-iy84usd)
* 📄 [MergedBeanDefinitionPostProcessor](siyuan://blocks/20230911140639-tyn9qtn)
* 📄 [bean的生命周期流程图](siyuan://blocks/20230911121138-2eaf2t1)
* 📑 [createBean](siyuan://blocks/20230911135649-rs5o5z0)

  * 📄 [getMergedLocalBeanDefinition](siyuan://blocks/20230911135756-cwyddnu)
* 📑 [doGetBean](siyuan://blocks/20230911134815-xou1kee)

  * 📄 [getObjectForBeanInstance](siyuan://blocks/20230911135431-um67trz)
* 📑 [scan](siyuan://blocks/20230911120051-plzieuy)

  * 📑 [doscan](siyuan://blocks/20230911120152-mjfs7b0)

    * 📑 [findCandidateComponents](siyuan://blocks/20230911120352-zg04a6d)

      * 📄 [isCandidateComponent](siyuan://blocks/20230911120514-3sm0taz)
* 📑 [扫描的过程](siyuan://blocks/20230911121652-vdt1sg7)

  * 📄 [metadataReader](siyuan://blocks/20230911121756-u35t4jk)
* 📄 [父子BeanDefinition](siyuan://blocks/20230911134123-nc5int4)

‍

bean的生命周期流程图

## Bean的生成过程

### 1. 生成BeanDefinition

Spring启动的时候会进行扫描，会先调用`org.springframework.context.annotation.ClassPathScanningCandidateComponentProvider#scanCandidateComponents(String basePackage)`
扫描某个包路径，并得到BeanDefinition的Set集合。

**关于Spring启动流程，后续会单独的课详细讲，这里先讲一下Spring扫描的底层实现：**

扫描的过程

#### **代码入口：**scan

值得注意的是，CachingMetadataReaderFactory解析某个.class文件得到MetadataReader对象是利用的**ASM**技术，并没有加载这个类到JVM。并且，最终得到的ScannedGenericBeanDefinition对象，**beanClass属性存储的是当前类的名字，而不是class对象**。（beanClass属性的类型是Object，它即可以存储类的名字，也可以存储class对象）

最后，上面是说的通过扫描得到BeanDefinition对象，我们还可以通过直接定义BeanDefinition，或解析spring.xml文件的<bean/>，或者@Bean注解得到BeanDefinition对象。（后续课程会分析@Bean注解是怎么生成BeanDefinition的）。

### 2. 合并BeanDefinition

通过扫描得到所有BeanDefinition之后，就可以根据BeanDefinition创建Bean对象了，但是在Spring中支持父子BeanDefinition，和Java父子类类似，但是完全不是一回事。

父子BeanDefinition

‍

#### 代码入口：doGetBean

名称转换  
判断是否为beanFactory  通过&符号  
如果是  有&的话直接去单例池拿bean。没则去缓存池拿（调用getObject） 或者走创建

getObjectForBeanInstance  
如果不是 则去单例池拿 没有就走创建  
合并BeanDefinition  
拿到要创建的bean所依赖的bean名称  
判断是否循环依赖  
再调用doGetBean方法，其实就是又调用了自己 容易理解

getMergedLocalBeanDefinition  
到此为止合并了beanDefition  
最后才是走到createBean

### 3. 加载类

BeanDefinition合并之后，就可以去创建Bean对象了，而创建Bean就必须实例化对象，而实例化就必须先加载当前BeanDefinition所对应的class，在AbstractAutowireCapableBeanFactory类的createBean()方法中，一开始就会调用：

```java
Class<?> resolvedClass = resolveBeanClass(mbd, beanName);
```

这行代码就是去加载类，该方法是这么实现的：

```java
if (mbd.hasBeanClass()) {
	return mbd.getBeanClass();
}
if (System.getSecurityManager() != null) {
	return AccessController.doPrivileged((PrivilegedExceptionAction<Class<?>>) () ->
		doResolveBeanClass(mbd, typesToMatch), getAccessControlContext());
	}
else {
	return doResolveBeanClass(mbd, typesToMatch);
}
```

```java
public boolean hasBeanClass() {
	return (this.beanClass instanceof Class);
}
```

如果beanClass属性的类型是Class，那么就直接返回，如果不是，则会根据类名进行加载（doResolveBeanClass方法所做的事情）

会利用BeanFactory所设置的类加载器来加载类，如果没有设置，则默认使用ClassUtil.getDefaultClassLoader()所返回的类加载器来加载。

### 4. 实例化前

可以看出，BeanPostProcessor只有初始化前和初始化后两个方法。实例化是子类InstantiationAwareBeanPostProcessor自己的方法，后置处理是MergedBeanDefinitionPostProcessor的方法

AutowiredAnnotationBeanPostProcessor实现了InstantiationAwareBeanPostProcessor，MergedBeanDefinitionPostProcessor,BeanFactoryAware

当前BeanDefinition对应的类成功加载后，就可以实例化对象了，但是...

在Spring中，实例化对象之前，Spring提供了一个扩展点，允许用户来控制是否在某个或某些Bean实例化之前做一些启动动作。这个扩展点叫**InstantiationAwareBeanPostProcessor.postProcessBeforeInstantiation()**。比如：

```java
@Component
public class ZhouyuBeanPostProcessor implements InstantiationAwareBeanPostProcessor {

	@Override
	public Object postProcessBeforeInstantiation(Class<?> beanClass, String beanName) throws BeansException {
		if ("userService".equals(beanName)) {
			System.out.println("实例化前");
		}
		return null;
	}
}
```

如上代码会导致，在userService这个Bean实例化前，会进行打印。

值得注意的是，postProcessBeforeInstantiation()是有返回值的，如果这么实现：

```java
@Component
public class ZhouyuBeanPostProcessor implements InstantiationAwareBeanPostProcessor {

	@Override
	public Object postProcessBeforeInstantiation(Class<?> beanClass, String beanName) throws BeansException {
		if ("userService".equals(beanName)) {
			System.out.println("实例化前");
			return new UserService();
		}
		return null;
	}
}
```

userService这个Bean，在实例化前会直接返回一个由我们所定义的UserService对象。如果是这样，表示不需要Spring来实例化了，并且后续的Spring依赖注入也不会进行了，会跳过一些步骤，直接执行初始化后这一步。

‍

### 5. 实例化

在这个步骤中就会根据BeanDefinition去创建一个对象了。

### 5.1 Supplier创建对象

首先判断BeanDefinition中是否设置了Supplier，如果设置了则调用Supplier的get()得到对象。

得直接使用BeanDefinition对象来设置Supplier，比如：

```java
AbstractBeanDefinition beanDefinition = BeanDefinitionBuilder.genericBeanDefinition().getBeanDefinition();
beanDefinition.setInstanceSupplier(new Supplier<Object>() {
	@Override
	public Object get() {
		return new UserService();
	}
});
context.registerBeanDefinition("userService", beanDefinition);
```

### 5.2 工厂方法创建对象

如果没有设置Supplier，则检查BeanDefinition中是否设置了factoryMethod，也就是工厂方法，有两种方式可以设置factoryMethod，比如：

方式一：

```java
<bean id="userService" class="com.zhouyu.service.UserService" factory-method="createUserService" />
```

对应的UserService类为：

```java
public class UserService {

	public static UserService createUserService() {
		System.out.println("执行createUserService()");
		UserService userService = new UserService();
		return userService;
	}

	public void test() {
		System.out.println("test");
	}

}
```

方式二：

```java
<bean id="commonService" class="com.zhouyu.service.CommonService"/>
<bean id="userService1" factory-bean="commonService" factory-method="createUserService" />
```

对应的CommonService的类为：

```java
public class CommonService {

	public UserService createUserService() {
		return new UserService();
	}
}
```

Spring发现当前BeanDefinition方法设置了工厂方法后，就会区分这两种方式，然后调用工厂方法得到对象。

值得注意的是，我们通过@Bean所定义的BeanDefinition，是存在factoryMethod和factoryBean的，也就是和上面的方式二非常类似，@Bean所注解的方法就是factoryMethod，AppConfig对象就是factoryBean。如果@Bean所所注解的方法是static的，那么对应的就是方式一。

### 5.3 推断构造方法

第一节已经讲过一遍大概原理了，后面有一节课单独分析源码实现。推断完构造方法后，就会使用构造方法来进行实例化了。

额外的，在推断构造方法逻辑中除开会去选择构造方法以及查找入参对象意外，会还判断是否在对应的类中是否存在使用**@Lookup注解**了方法。如果存在则把该方法封装为LookupOverride对象并添加到BeanDefinition中。

在实例化时，如果判断出来当前BeanDefinition中没有LookupOverride，那就直接用构造方法反射得到一个实例对象。如果存在LookupOverride对象，也就是类中存在@Lookup注解了的方法，那就会生成一个代理对象。

@Lookup注解就是**方法注入**，使用demo如下：

```java
@Component
public class UserService {

	private OrderService orderService;

	public void test() {
		OrderService orderService = createOrderService();
		System.out.println(orderService);
	}

	@Lookup("orderService")
	public OrderService createOrderService() {
		return null;
	}

}
```

### 6. BeanDefinition的后置处理

Bean对象实例化出来之后，接下来就应该给对象的属性赋值了。在真正给属性赋值之前，Spring又提供了一个扩展点MergedBeanDefinitionPostProcessor.**postProcessMergedBeanDefinition()**，可以对此时的BeanDefinition进行加工，比如：

```java
@Override
	public void postProcessMergedBeanDefinition(RootBeanDefinition beanDefinition, Class<?> beanType, String beanName) {
		InjectionMetadata metadata = findAutowiringMetadata(beanName, beanType, null);
		metadata.checkConfigMembers(beanDefinition);
	}
```

```java
@Component
public class ZhouyuMergedBeanDefinitionPostProcessor implements MergedBeanDefinitionPostProcessor {

	@Override
	public void postProcessMergedBeanDefinition(RootBeanDefinition beanDefinition, Class<?> beanType, String beanName) {
		if ("userService".equals(beanName)) {
			beanDefinition.getPropertyValues().add("orderService", new OrderService());
		}
	}
}
```

在Spring源码中，AutowiredAnnotationBeanPostProcessor就是一个MergedBeanDefinitionPostProcessor，它的postProcessMergedBeanDefinition()中会去查找注入点，并缓存在AutowiredAnnotationBeanPostProcessor对象的一个Map中（injectionMetadataCache）。

### 7. 实例化后

在处理完BeanDefinition后，Spring又设计了一个扩展点：InstantiationAwareBeanPostProcessor**.postProcessAfterInstantiation()**，比如：

```java
@Component
public class ZhouyuInstantiationAwareBeanPostProcessor implements InstantiationAwareBeanPostProcessor {

	@Override
	public boolean postProcessAfterInstantiation(Object bean, String beanName) throws BeansException {

		if ("userService".equals(beanName)) {
			UserService userService = (UserService) bean;
			userService.test();
		}

		return true;
	}
}
```

上述代码就是对userService所实例化出来的对象进行处理。

这个扩展点，在Spring源码中基本没有怎么使用。

### 8. 自动注入

这里的自动注入指的是Spring的自动注入，后续依赖注入课程中单独讲

### 9. 处理属性

这个步骤中，就会处理@Autowired、@Resource、@Value等注解，也是通过InstantiationAwareBeanPostProcessor**.postProcessProperties()**扩展点来实现的，比如我们甚至可以实现一个自己的自动注入功能，比如：

```java
	@Override
	public PropertyValues postProcessProperties(PropertyValues pvs, Object bean, String beanName) {
		// 找注入点（所有被@Autowired注解了的Field或Method）
		InjectionMetadata metadata = findAutowiringMetadata(beanName, bean.getClass(), pvs);
		try {
			metadata.inject(bean, beanName, pvs);
		}
		catch (BeanCreationException ex) {
			throw ex;
		}
		catch (Throwable ex) {
			throw new BeanCreationException(beanName, "Injection of autowired dependencies failed", ex);
		}
		return pvs;
	}
```

```java
@Component
public class ZhouyuInstantiationAwareBeanPostProcessor implements InstantiationAwareBeanPostProcessor {

	@Override
	public PropertyValues postProcessProperties(PropertyValues pvs, Object bean, String beanName) throws BeansException {
		if ("userService".equals(beanName)) {
			for (Field field : bean.getClass().getFields()) {
				if (field.isAnnotationPresent(ZhouyuInject.class)) {
					field.setAccessible(true);
					try {
						field.set(bean, "123");
					} catch (IllegalAccessException e) {
						e.printStackTrace();
					}
				}
			}
		}

		return pvs;
	}
}
```

关于@Autowired、@Resource、@Value的底层源码，会在后续的依赖注入课程中详解。

### 10. 执行Aware

完成了属性赋值之后，Spring会执行一些回调，包括：

1. BeanNameAware：回传beanName给bean对象。
2. BeanClassLoaderAware：回传classLoader给bean对象。
3. BeanFactoryAware：回传beanFactory给对象。

### 11. 初始化前

初始化前，也是Spring提供的一个扩展点：BeanPostProcessor**.postProcessBeforeInitialization()**，比如

```java
@Component
public class ZhouyuBeanPostProcessor implements BeanPostProcessor {

	@Override
	public Object postProcessBeforeInitialization(Object bean, String beanName) throws BeansException {
		if ("userService".equals(beanName)) {
			System.out.println("初始化前");
		}

		return bean;
	}
}
```

利用初始化前，可以对进行了依赖注入的Bean进行处理。

在Spring源码中：

1. InitDestroyAnnotationBeanPostProcessor会在初始化前这个步骤中执行@PostConstruct的方法，
2. ApplicationContextAwareProcessor会在初始化前这个步骤中进行其他Aware的回调：
   1. EnvironmentAware：回传环境变量
   2. EmbeddedValueResolverAware：回传占位符解析器
   3. ResourceLoaderAware：回传资源加载器
   4. ApplicationEventPublisherAware：回传事件发布器
   5. MessageSourceAware：回传国际化资源
   6. ApplicationStartupAware：回传应用其他监听对象，可忽略
   7. ApplicationContextAware：回传Spring容器ApplicationContext

### 12. 初始化

1. 查看当前Bean对象是否实现了InitializingBean接口，如果实现了就调用其afterPropertiesSet()方法
2. 执行BeanDefinition中指定的初始化方法

‍

### 13. 初始化后

这是Bean创建生命周期中的最后一个步骤，也是Spring提供的一个扩展点：**BeanPostProcessor.postProcessAfterInitialization()**，比如：

```java
@Component
public class ZhouyuBeanPostProcessor implements BeanPostProcessor {

	@Override
	public Object postProcessAfterInitialization(Object bean, String beanName) throws BeansException {
		if ("userService".equals(beanName)) {
			System.out.println("初始化后");
		}

		return bean;
	}
}
```

可以在这个步骤中，对Bean最终进行处理，Spring中的**AOP就是基于初始化后实现**的，**初始化后返回的对象才是最终的Bean对象**。

### 总结BeanPostProcessor

1. InstantiationAwareBeanPostProcessor.postProcessBeforeInstantiation()
2. 实例化
3. MergedBeanDefinitionPostProcessor.postProcessMergedBeanDefinition()
4. InstantiationAwareBeanPostProcessor.postProcessAfterInstantiation()
5. 自动注入
6. InstantiationAwareBeanPostProcessor.postProcessProperties()
7. Aware对象
8. BeanPostProcessor.postProcessBeforeInitialization()
9. 初始化
10. BeanPostProcessor.postProcessAfterInitialization()

## Bean的销毁过程

Bean销毁是发送在Spring容器关闭过程中的。

在Spring容器关闭时，比如：

```java
AnnotationConfigApplicationContext context = new AnnotationConfigApplicationContext(AppConfig.class);
UserService userService = (UserService) context.getBean("userService");
userService.test();

// 容器关闭
context.close();
```

在Bean创建过程中，在最后（初始化之后），有一个步骤会去判断当前创建的Bean是不是DisposableBean：

1. 当前Bean是否实现了DisposableBean接口
2. 或者，当前Bean是否实现了AutoCloseable接口
3. BeanDefinition中是否指定了destroyMethod
4. 调用DestructionAwareBeanPostProcessor.requiresDestruction(bean)进行判断
   1. ApplicationListenerDetector中直接使得ApplicationListener是DisposableBean
   2. InitDestroyAnnotationBeanPostProcessor中使得拥有@PreDestroy注解了的方法就是DisposableBean
5. 把符合上述任意一个条件的Bean适配成DisposableBeanAdapter对象，并存入disposableBeans中（一个LinkedHashMap）

在Spring容器关闭过程时：

1. 首先发布ContextClosedEvent事件
2. 调用lifecycleProcessor的onCloese()方法
3. 销毁单例Bean
   1. 遍历disposableBeans
      1. 把每个disposableBean从单例池中移除
      2. 调用disposableBean的destroy()
      3. 如果这个disposableBean还被其他Bean依赖了，那么也得销毁其他Bean
      4. 如果这个disposableBean还包含了inner beans，将这些Bean从单例池中移除掉 (inner bean参考[https://docs.spring.io/spring-framework/docs/current/spring-framework-reference/core.html#beans-inner-beans](https://docs.spring.io/spring-framework/docs/current/spring-framework-reference/core.html#beans-inner-beans))
   2. 清空manualSingletonNames，是一个Set，存的是用户手动注册的单例Bean的beanName
   3. 清空allBeanNamesByType，是一个Map，key是bean类型，value是该类型所有的beanName数组
   4. 清空singletonBeanNamesByType，和allBeanNamesByType类似，只不过只存了单例Bean

这里涉及到一个设计模式：**适配器模式**

在销毁时，Spring会找出实现了DisposableBean接口的Bean。

但是我们在定义一个Bean时，如果这个Bean实现了DisposableBean接口，或者实现了AutoCloseable接口，或者在BeanDefinition中指定了destroyMethodName，那么这个Bean都属于“DisposableBean”，这些Bean在容器关闭时都要调用相应的销毁方法。

所以，这里就需要进行适配，将实现了DisposableBean接口、或者AutoCloseable接口等适配成实现了DisposableBean接口，所以就用到了DisposableBeanAdapter。

会把实现了AutoCloseable接口的类封装成DisposableBeanAdapter，而DisposableBeanAdapter实现了DisposableBean接口。

‍
