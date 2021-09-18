## 1. 什么是循环依赖？

简单说就是A以来B，B依赖C，C依赖A，形成了一个圈。

Spring如何知道存在循环依赖呢？通过 `isPrototypeCurrentlyInCreation` 方法判断，核心思想就是正在创建列表中有没有这个Bean，有说明存在循环依赖。

```
/** Names of beans that are currently in creation. */
private final ThreadLocal<Object> prototypesCurrentlyInCreation = new NamedThreadLocal<>("Prototype beans currently in creation");

protected boolean isPrototypeCurrentlyInCreation(String beanName) {
	Object curVal = this.prototypesCurrentlyInCreation.get();
	return (curVal != null &&
			(curVal.equals(beanName) || (curVal instanceof Set && ((Set<?>) curVal).contains(beanName))));
}
```

## 2. spring的三级缓存

名称 | 存储属性 | 存储内容 | 作用
---|---|---|---
一级缓存 | singletonObjects | 用于保存实例化、注入、初始化完成的bean实例 |
二级缓存 | earlySingletonObjects | 用于保存刚实例化完成的bean实例 | 
三级缓存 | singletonFactories | 用于保存bean创建工厂 | 以便于后面扩展有机会创建代理对象。

```
public class DefaultSingletonBeanRegistry extends SimpleAliasRegistry implements SingletonBeanRegistry {
    //1级缓存 用于存放 已经属性赋值 初始化后的 单列BEAN
    private final Map<String, Object> singletonObjects = new ConcurrentHashMap<>(256);
    //2级缓存 用于存在已经实例化，还未做代理属性赋值操作的 单例
    private final Map<String, Object> earlySingletonObjects = new HashMap<>(16);
    //3级缓存 存储单例BEAN的工厂
    private final Map<String, ObjectFactory<?>> singletonFactories = new HashMap<>(16);
    //已经注册的单例池里的beanName
    private final Set<String> registeredSingletons = new LinkedHashSet<>(256);
    //正在创建中的beanName
    private final Set<String> singletonsCurrentlyInCreation =
            Collections.newSetFromMap(new ConcurrentHashMap<>(16));
    //缓存查找bean  如果1级没有，从2级获取,也没有,从3级创建放入2级
    protected Object getSingleton(String beanName, boolean allowEarlyReference) {
        Object singletonObject = this.singletonObjects.get(beanName); //1级
        if (singletonObject == null && isSingletonCurrentlyInCreation(beanName)) {
            synchronized (this.singletonObjects) {
                singletonObject = this.earlySingletonObjects.get(beanName); //2级
                if (singletonObject == null && allowEarlyReference) {
                    //3级缓存  在doCreateBean中创建了bean的实例后，封装ObjectFactory放入缓存的
                    ObjectFactory<?> singletonFactory = this.singletonFactories.get(beanName);
                    if (singletonFactory != null) {
                        //创建未赋值的bean
                        singletonObject = singletonFactory.getObject();
                        //放入到二级缓存
                        this.earlySingletonObjects.put(beanName, singletonObject);
                        //从三级缓存删除
                        this.singletonFactories.remove(beanName);
                    }
                }
            }
        }
        return singletonObject;
    }   
}
```

## 3. 循环依赖场景
![image](https://raw.githubusercontent.com/future94/java-technology/master/spring/spring/images/ff86d100cc169247e6a.png)

### 3.1 单例的setter注入

```
@Service
publicclass TestService1 {

    @Autowired
    private TestService2 testService2;

    public void test1() {
    }
}
```

```
@Service
publicclass TestService2 {

    @Autowired
    private TestService1 testService1;

    public void test2() {
    }
}
```

这是一个经典的循环依赖，但是它能正常运行，得益于spring的内部机制，让我们根本无法感知它有问题，因为spring默默帮我们解决了。那么他是怎么解决的呢？

![image](https://raw.githubusercontent.com/future94/java-technology/master/spring/spring/images/06fd33d36ac3bfd99f015f.png)

### 二级缓存的作用

```
@Service
publicclass TestService1 {

    @Autowired
    private TestService2 testService2;
    @Autowired
    private TestService3 testService3;

    public void test1() {
    }
}
```

```
@Service
publicclass TestService2 {

    @Autowired
    private TestService1 testService1;

    public void test2() {
    }
}
```

```
@Service
publicclass TestService3 {

    @Autowired
    private TestService1 testService1;

    public void test3() {
    }
}
```

TestService2和TestService3同时依赖了TestService1，而三级缓存中TestService1存的是 `ObjectFactory`，所以每次创建出不一样的对象，那么这样肯定是不对的，这时候引入二级缓存，当二级缓存中存在，直接取出即可，保证多个地方引入都是相同的单例对象。如下图：

![image](https://raw.githubusercontent.com/future94/java-technology/master/spring/spring/images/158838a4aaedfe0f7a6fdab.png)

### 三级缓存的作用

第三级缓存中为什么要添加ObjectFactory对象，直接保存实例对象不行吗？

肯定不行，因为Spring好多功能都是通过代理类来实现的（如AOP），三级缓存存储ObjectFactory和其要加强的aop处理，如果需要aop增强的bean遇到了循环依赖，则使用该缓存中的aop处理代理增强bean。实际上不用三级缓存也可以解决循环依赖，但是如果有需要aop增强的bean时，就要在初始化bean的时候对bean做增强了，这违背了Spring在结合AOP跟Bean的生命周期的设计！

总结就是：<font color="red">在循环依赖的环境下保证bean初始化完成之后才生成代理，而不是实例化之后就生成代理，保证了bean的生命周期。</font>

**源码验证**：
```
protected Object getEarlyBeanReference(String beanName, RootBeanDefinition mbd, Object bean) {
    Object exposedObject = bean;
    if (!mbd.isSynthetic() && hasInstantiationAwareBeanPostProcessors()) {
        for (BeanPostProcessor bp : getBeanPostProcessors()) {
            if (bp instanceof SmartInstantiationAwareBeanPostProcessor) {
                SmartInstantiationAwareBeanPostProcessor ibp = (SmartInstantiationAwareBeanPostProcessor) bp;
                exposedObject = ibp.getEarlyBeanReference(exposedObject, beanName);
            }
        }
    }
    return exposedObject;
}
```

它实际上就是调用了后置处理器的 `getEarlyBeanReference`，而真正实现了这个方法的后置处理器只有一个，就是通过 `@EnableAspectJAutoProxy` 注解导入的 `AnnotationAwareAspectJAutoProxyCreator`。也就是说如果在不考虑AOP的情况下，上面的代码等价于：

```
protected Object getEarlyBeanReference(String beanName, RootBeanDefinition mbd, Object bean) {
    Object exposedObject = bean;
    return exposedObject;
}
```
也就是说这个工厂啥都没干，直接将实例化阶段创建的对象返回了！所以说在不考虑AOP的情况下三级缓存有用嘛？讲道理，真的没什么用


如果在开启AOP的情况下，那么就是调用到 `AnnotationAwareAspectJAutoProxyCreator` 的父类的 `AbstractAutoProxyCreator` 的 `getEarlyBeanReference` 方法，对应的源码如下：
```
public Object getEarlyBeanReference(Object bean, String beanName) {
    Object cacheKey = getCacheKey(bean.getClass(), beanName);
    this.earlyProxyReferences.put(cacheKey, bean);
    // 如果需要代理，返回一个代理对象，不需要代理，直接返回当前传入的这个bean对象
    return wrapIfNecessary(bean, beanName, cacheKey);
}
```

A依赖B，B依赖A，我们对A进行了AOP代理的话，那么此时getEarlyBeanReference将返回一个代理后的对象，而不是实例化阶段创建的对象，这样就意味着B中注入的A将是一个代理对象而不是A的实例化阶段创建后的对象。

在使用了三级缓存的情况下为A创建代理的时机是在B中需要注入A的时候，而不使用三级缓存的话在A实例化后就需要马上为A创建代理然后放入到二级缓存中去。差别就是在哪里创建代理。如果不用三级缓存，使用二级缓存，违背了Spring在结合AOP跟Bean的生命周期的设计！Spring结合AOP跟Bean的生命周期本身就是通过AnnotationAwareAspectJAutoProxyCreator这个后置处理器来完成的，在这个后置处理的postProcessAfterInitialization方法中对初始化后的Bean完成AOP代理。如果出现了循环依赖，那没有办法，只有给Bean先创建代理，但是没有出现循环依赖的情况下，设计之初就是让Bean在生命周期的最后一步完成代理而不是在实例化后就立马完成代理。

### 3.2 多例的setter注入

```
@Scope(ConfigurableBeanFactory.SCOPE_PROTOTYPE)
@Service
publicclass TestService1 {

    @Autowired
    private TestService2 testService2;

    public void test1() {
    }
}
```
```
@Scope(ConfigurableBeanFactory.SCOPE_PROTOTYPE)
@Service
publicclass TestService2 {

    @Autowired
    private TestService1 testService1;

    public void test2() {
    }
}
```
这种情况不会影响启动，其实在 `AbstractApplicationContext` 类的 `refresh` 方法中告诉了我们答案，它会调用 `finishBeanFactoryInitialization` 方法，该方法的作用是为了spring容器启动的时候提前初始化一些bean。该方法的内部又调用了 `preInstantiateSingletons` 方法，只有单利的Bean才会被初始化。

![image](https://raw.githubusercontent.com/future94/java-technology/master/spring/spring/images/755906540e544724904264.png)

如果有一个单例的类，因为多例的类出现循环，那么无法解决，因为它没有用缓存，每次都会生成一个新对象。

### 3.3 构造器注入

```
@Service
publicclass TestService1 {

    public TestService1(TestService2 testService2) {
    }
}
```

```
@Service
publicclass TestService2 {

    public TestService2(TestService1 testService1) {
    }
}
```

因为创建实例之后才会放入三级缓存，所以这时候出现循环依赖，依赖的实例还没有加入到缓存之中，肯定找不到，所以出现循环依赖但Spring无法解决。如下图：

![image](https://raw.githubusercontent.com/future94/java-technology/master/spring/spring/images/d11690d5476070ae8609751e.png)

### 3.4 单例的代理对象setter注入

比如@Async注解的场景，会通过AOP自动生成代理对象。这种情况Spring也无法解决循环依赖.

```
@Service
publicclass TestService1 {

    @Autowired
    private TestService2 testService2;

    @Async
    public void test1() {
    }
}
```

```
@Service
publicclass TestService2 {

    @Autowired
    private TestService1 testService1;

    public void test2() {
    }
}
```

testService2创建实例后发现依赖testService1，我们从三级缓存获取到testService1实例并放入二级缓存中，这时候我们成功的创建了testService2实例，但是testService2有@Async注解，这时候我们要对testService1进行代理，这时候发现二级缓存中与原始对象不一致，所以跑出异常，如下：

![image](https://raw.githubusercontent.com/future94/java-technology/master/spring/spring/images/7fdbe6427b2d7fd5c70ce66.png)

Spring源码如下：

![image](https://raw.githubusercontent.com/future94/java-technology/master/spring/spring/images/756248a94801d31761f8d2519.png)

### 神奇现象

我们将TestService1改名为TestService6，其他什么都不变，这时候循环依赖被解决了。

```
@Service
publicclass TestService6 {

    @Autowired
    private TestService2 testService2;

    @Async
    public void test1() {
    }
}
```

这就要从spring的bean加载顺序说起了，默认情况下，spring是按照文件完整路径递归查找的，按路径+文件名排序，排在前面的先加载。所以TestService1比TestService2先加载，而改了文件名称之后，TestService2比TestService6先加载。

因为先加载TestService2的时候，要生成TestService6，TestService6创建完毕生成TestService2的时候，二级缓存中的值为空所以不会进行校验，流程如下：

![image](https://raw.githubusercontent.com/future94/java-technology/master/spring/spring/images/b90aaf1e6b1dba8524967fac20.png)

### 3.5 DependsOn循环依赖

还有一种有些特殊的场景，比如我们需要在实例化Bean A之前，先实例化Bean B，这个时候就可以使用@DependsOn注解。

```
@DependsOn(value = "testService2")
@Service
publicclass TestService1 {

    @Autowired
    private TestService2 testService2;

    public void test1() {
    }
}
```

```
@DependsOn(value = "testService1")
@Service
publicclass TestService2 {

    @Autowired
    private TestService1 testService1;

    public void test2() {
    }
}
```

本来可以解决掉循环依赖，但是加了 `@DependsOn` 反而不行，因为启动的时候检查dependsOn的实例有没有循环依赖，如果有循环依赖则抛异常。源码如下：

![image](https://raw.githubusercontent.com/future94/java-technology/master/spring/spring/images/8aa3686d5fbaae9cc5d980c4f8.png)


## 4. 循环依赖的解决

### 4.1 生成代理对象产生的循环依赖

- 使用@Lazy注解，延迟加载
- 使用@DependsOn注解，指定加载先后关系
- 修改文件名称，改变循环依赖类的加载顺序

### 4.2 使用@DependsOn产生的循环依赖

这类循环依赖问题要找到@DependsOn注解循环依赖的地方，迫使它不循环依赖就可以解决问题。

### 4.3 多例循环依赖
这类循环依赖问题可以通过把bean改成单例的解决。

### 4.4 构造器循环依赖
这类循环依赖问题可以通过使用@Lazy注解解决。


## 5. 解决循环依赖核心源码整体流程分析

入口文件为`org.springframework.beans.factory.support.AbstractBeanFactory#doGetBean`，只展示了核心代码。

### 5.1 获取单例的Bean

如下，这个方法首先会调用 getSingleton 获取 bean，如果可以获取到(一级缓存中存在)，就会直接返回，否则会执行创建 bean 的流程


```
protected <T> T doGetBean(
		String name, @Nullable Class<T> requiredType, @Nullable Object[] args, boolean typeCheckOnly)
		throws BeansException {
    # 获取单例的Bean，多例的无法解决。
	Object sharedInstance = getSingleton(beanName);
}
```

### 5.2 从缓存中获取

这个方法内部会调用getSingleton(beanName, true)获取 bean，注意第二个参数是true，这个表示是否可以获取早期的 bean，这个参数为 true，会尝试从三级缓存singletonFactories中获取 bean，然后将三级缓存中获取到的 bean 丢到二级缓存中。

```
public Object getSingleton(String beanName) {
    return getSingleton(beanName, true);
}

protected Object getSingleton(String beanName, boolean allowEarlyReference) {
    //从第1级缓存中获取bean
    Object singletonObject = this.singletonObjects.get(beanName);
    //第1级中没有,且当前beanName在创建列表中
    if (singletonObject == null && isSingletonCurrentlyInCreation(beanName)) {
        synchronized (this.singletonObjects) {
            //从第2级缓存汇总获取bean
            singletonObject = this.earlySingletonObjects.get(beanName);
            //第2级缓存中没有 && allowEarlyReference为true，也就是说2级缓存中没有找到bean且beanName在当前创建列表中的时候，才会继续想下走。
            if (singletonObject == null && allowEarlyReference) {
                //从第3级缓存中获取bean
                ObjectFactory<?> singletonFactory = this.singletonFactories.get(beanName);
                //第3级中有获取到了
                if (singletonFactory != null) {
                    //3级缓存汇总放的是ObjectFactory，所以会调用其getObject方法获取bean
                    singletonObject = singletonFactory.getObject();
                    //将3级缓存中的bean丢到第2级中
                    this.earlySingletonObjects.put(beanName, singletonObject);
                    //将bean从三级缓存中干掉
                    this.singletonFactories.remove(beanName);
                }
            }
        }
    }
    return singletonObject;
}
```

### 5.3 如果缓存中没有，校验，并确定是单例

```
// 获取缓存
Object sharedInstance = getSingleton(beanName);
if (sharedInstance != null && args == null) {
    // 如果有
	bean = getObjectForBeanInstance(sharedInstance, name, beanName, null);
}

else {
	// 如果已经在正在创建列表中，说明已经有人在创建了，则表示可能是循环依赖。
	if (isPrototypeCurrentlyInCreation(beanName)) {
		throw new BeanCurrentlyInCreationException(beanName);
	}

	
	try {
		RootBeanDefinition mbd = getMergedLocalBeanDefinition(beanName);
		checkMergedBeanDefinition(mbd, beanName, args);

		// 是否dependsOn其他，并检查dependsOn的Bean是否存在循环依赖
		String[] dependsOn = mbd.getDependsOn();
		if (dependsOn != null) {
			for (String dep : dependsOn) {
			    // 存在则异常
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

		if (mbd.isSingleton()) {
	        // 如果是单例的，调用createBean创建
			sharedInstance = getSingleton(beanName, () -> {
				try {
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
			// 是否是多例子
		}

		else {
			// 其他情况
		}
	}
	catch (BeansException ex) {
		cleanupAfterBeanCreationFailure(beanName);
		throw ex;
	}
}
```

### 5.4 创建
```
protected Object doCreateBean(String beanName, RootBeanDefinition mbd, @Nullable Object[] args)
		throws BeanCreationException {

	// ①：创建bean实例，通过反射实例化bean，相当于new X()创建bean的实例
	instanceWrapper = createBeanInstance(beanName, mbd, args);
	// bean = 获取刚刚new出来的bean
	Object bean = instanceWrapper.getWrappedInstance();
	// ②：是否需要将早期的bean暴露出去，所谓早期的bean相当于这个bean就是通过new的方式创建了这个对象，但是这个对象还没有填充属性，所以是个半成品
    // 是否需要将早期的bean暴露出去，判断规则（bean是单例 && 是否允许循环依赖 && bean是否在正在创建的列表中）
	boolean earlySingletonExposure = (mbd.isSingleton() && this.allowCircularReferences &&
			isSingletonCurrentlyInCreation(beanName));
	if (earlySingletonExposure) {
		if (logger.isTraceEnabled()) {
			logger.trace("Eagerly caching bean '" + beanName +
					"' to allow for resolving potential circular references");
		}
		//③：调用addSingletonFactory方法，这个方法内部会将其丢到第3级缓存中，getEarlyBeanReference的源码大家可以看一下，内部会调用一些方法获取早期的bean对象，比如可以在这个里面通过aop生成代理对象
		addSingletonFactory(beanName, () -> getEarlyBeanReference(beanName, mbd, bean));
	}

	// 这个变量用来存储最终返回的bean
	Object exposedObject = bean;
	try {
	    //填充属性，这里面会调用setter方法或者通过反射将依赖的bean注入进去
		populateBean(beanName, mbd, instanceWrapper);
		//④：初始化bean，内部会调用BeanPostProcessor的一些方法，对bean进行处理，这里可以对bean进行包装，比如生成代理
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
    //早期的bean是否被暴露出去了
	if (earlySingletonExposure) {
        // 注意第二个参数是false，这个为false的时候，只会从第1和第2级中获取bean，此时第1级中肯定是没有的（只有bean创建完毕之后才会放入1级缓存）
		Object earlySingletonReference = getSingleton(beanName, false);
		// ⑥：如果earlySingletonReference不为空，说明第2级缓存有这个bean，二级缓存中有这个bean，说明了什么？大家回头再去看看上面的分析，看一下什么时候bean会被放入2级缓存?（若 bean存在三级缓存中 && beanName在当前创建列表的时候，此时其他地方调用了getSingleton(beanName, false)方法，那么bean会从三级缓存移到二级缓存）
		if (earlySingletonReference != null) {
		    //⑥：exposedObject==bean，说明bean创建好了之后，后期没有被修改
			if (exposedObject == bean) {
			    //earlySingletonReference是从二级缓存中获取的，二级缓存中的bean来源于三级缓存，三级缓存中可能对bean进行了包装，比如生成了代理对象
                //那么这个地方就需要将 earlySingletonReference 作为最终的bean
				exposedObject = earlySingletonReference;
			}
			else if (!this.allowRawInjectionDespiteWrapping && hasDependentBean(beanName)) {
			    //回头看看上面的代码，刚开始exposedObject=bean，
                // 此时能走到这里，说明exposedObject和bean不一样了，他们不一样了说明了什么？
                // 说明initializeBean内部对bean进行了修改
                // allowRawInjectionDespiteWrapping（默认是false）：是否允许早期暴露出去的bean(earlySingletonReference)和最终的bean不一致
                // hasDependentBean(beanName)：表示有其他bean以利于beanName
                // getDependentBeans(beanName)：获取有哪些bean依赖beanName
				String[] dependentBeans = getDependentBeans(beanName);
				Set<String> actualDependentBeans = new LinkedHashSet<>(dependentBeans.length);
				for (String dependentBean : dependentBeans) {
				    //判断dependentBean是否已经被标记为创建了，就是判断dependentBean是否已经被创建了
					if(!removeSingletonIfCreatedForTypeCheckOnly(dependentBean)) {
						actualDependentBeans.add(dependentBean);
					}
				}
				// 能走到这里，说明早期的bean被别人使用了，而后面程序又将exposedObject做了修改，也就是说早期创建的bean是A，这个A已经被有些地方使用了，但是A通过initializeBean之后可能变成了B，比如B是A的一个代理对象，这个时候就坑了，别人已经用到的A和最终容器中创建完成的A不是同一个A对象了，那么使用过程中就可能存在问题了，比如后期对A做了增强（Aop），而早期别人用到的A并没有被增强
				if (!actualDependentBeans.isEmpty()) {
					throw new BeanCurrentlyInCreationException(beanName,
							"Bean with name '" + beanName + "' has been injected into other beans [" +
							StringUtils.collectionToCommaDelimitedString(actualDependentBeans) +
							"] in its raw version as part of a circular reference, but has eventually been " +
							"wrapped. This means that said other beans do not use the final version of the " +
							"bean. This is often the result of over-eager type matching - consider using " +
							"'getBeanNamesForType' with the 'allowEagerInit' flag turned off, for example.");
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
```

## 6. 问题
1、什么时候 bean 被放入 3 级缓存？

早期的 bean 被放入 3 级缓存

2、什么时候 bean 会被放入 2 级缓存？

当 beanX 还在创建的过程中，此时被加入当前 beanName 创建列表了，但是这个时候 bean 并没有被创建完毕（bean 被丢到一级缓存才算创建完毕），此时 bean 还是个半成品，这个时候其他 bean 需要用到 beanX，此时会从三级缓存中获取到 beanX，beanX 会从三级缓存中丢到 2 级缓存中。

3、什么时候 bean 会被放入 1 级缓存？

bean 实例化完毕，初始化完毕，属性注入完毕，bean 完全组装完毕之后，才会被丢到 1 级缓存。


参考文章：
- https://mp.weixin.qq.com/s/VpCt49_Li35caK5IaQTuNg
- https://juejin.cn/post/6930904292958142478
- https://itsoku.blog.csdn.net/article/details/113977261