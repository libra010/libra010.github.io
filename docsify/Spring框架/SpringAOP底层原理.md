Spring AOP 实现



## 如何启用AOP功能

Spring框架中使用 `@EnableAspectJAutoProxy` 来启用AOP功能，Spring Boot 则通过 AopAutoConfiguration 自动配置类来启用，其底层也是通过该注册来启用AOP功能。

`@EnableAspectJAutoProxy` 注解通过 `@Import({AspectJAutoProxyRegistrar.class})` 引入了 AspectJAutoProxyRegistrar 类，它会向 BeanDefinitionRegistry 中注册一个名为 `org.springframework.aop.config.internalAutoProxyCreator` 的 BeanDefinition。这个 Bean 的类型是 `AnnotationAwareAspectJAutoProxyCreator`。

```java
class AspectJAutoProxyRegistrar implements ImportBeanDefinitionRegistrar {
    public void registerBeanDefinitions(AnnotationMetadata importingClassMetadata, BeanDefinitionRegistry registry) {
        // 注册BeanDefinition(org.springframework.aop.config.internalAutoProxyCreator)
        AopConfigUtils.registerAspectJAnnotationAutoProxyCreatorIfNecessary(registry);
        AnnotationAttributes enableAspectJAutoProxy = AnnotationConfigUtils.attributesFor(importingClassMetadata, EnableAspectJAutoProxy.class);
        // 根据注解设置BeanDefinition的属性
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
```



## AbstractAutoProxyCreator
先看一下`AnnotationAwareAspectJAutoProxyCreator`的继承关系，可以发现它最终是一个 `BeanPostProcessor` 类型。这样就能猜到AOP的代理的原理是通过这个后置处理器更换了原始的Bean，这个动作发生在IoC容器的启动过程中。

```
BeanPostProcessor (Spring容器提供的一个Bean后置处理器)
          ^
          |
InstantiationAwareBeanPostProcessor
SmartInstantiationAwareBeanPostProcessor
          ^
          |
AbstractAutoProxyCreator (BeanPostProcessor接口的实现在这个抽象类里面)
          ^
          |
AbstractAdvisorAutoProxyCreator
AspectJAwareAdvisorAutoProxyCreator
          ^
          |
AnnotationAwareAspectJAutoProxyCreator (Spring启用AOP引入的Bean)
```

同时也能找到 BeanPostProcessor 接口的实现在 AbstractAutoProxyCreator 抽象类中。

```java
// BeanPostProcessor::postProcessAfterInitialization 方法的实现
public Object postProcessAfterInitialization(@Nullable Object bean, String beanName) {
    if (bean != null) {
        Object cacheKey = this.getCacheKey(bean.getClass(), beanName);
        if (this.earlyProxyReferences.remove(cacheKey) != bean) {
            // 调用这个方法返回代理Bean
            return this.wrapIfNecessary(bean, beanName, cacheKey);
        }
    }

    return bean;
}

protected Object wrapIfNecessary(Object bean, String beanName, Object cacheKey) {
    if (StringUtils.hasLength(beanName) && this.targetSourcedBeans.contains(beanName)) {
		return bean;
    } else if (Boolean.FALSE.equals(this.advisedBeans.get(cacheKey))) {
		return bean;
    } else if (!this.isInfrastructureClass(bean.getClass()) && !this.shouldSkip(bean.getClass(), beanName)) { // 判断基础设施或者覆写跳过功能
        // 核心方法：查找所有匹配的拦截器 (Advisors)
        // 该方法定义在AbstractAdvisorAutoProxyCreator类
		Object[] specificInterceptors = this.getAdvicesAndAdvisorsForBean(bean.getClass(), beanName, (TargetSource)null);
        // 返回不为null, 说明需要代理
		if (specificInterceptors != DO_NOT_PROXY) {
            // 保存结果
			this.advisedBeans.put(cacheKey, Boolean.TRUE);
            // 核心方法：创建代理对象
			Object proxy = this.createProxy(bean.getClass(), beanName, specificInterceptors, new SingletonTargetSource(bean));
            // 保存结果
			this.proxyTypes.put(cacheKey, proxy.getClass());
            // 返回代理对象
			return proxy;
		} else {
			this.advisedBeans.put(cacheKey, Boolean.FALSE);
            return bean;
        }
	} else {
		this.advisedBeans.put(cacheKey, Boolean.FALSE);
		return bean;
	}
}
```

创建代理对象的createProxy方法

```java
protected Object createProxy(
      Class<?> beanClass, String beanName, Object[] specificInterceptors, TargetSource targetSource) {
    // ......
	ProxyFactory proxyFactory = new ProxyFactory();
	proxyFactory.copyFrom(this);
    // ......
    // 该类主要创建了一个ProxyFactory工厂类 用于创建代理对象
    return proxyFactory.getProxy(getProxyClassLoader());
}
```

ProxyFactory类

```java
public Object getProxy(@Nullable ClassLoader classLoader) {
    // TODO
    return this.createAopProxy().getProxy(classLoader);
}
```





## 编程式AOP

Spring AOP 文档提供了 Spring AOP API，可以让我们编程式的使用AOP功能，这种功能使得AOP能够不依赖于IoC容器进行使用，同时可以提供更高的精细度，当然也可以用 `@Bean` 或者 `@Component` 将其注册到IoC容器使用，IoC和AOP这两者是完全正交的概念。

通过上述的AOP源码学习，可以了解到 ProxyFactory 这个类，它的主要作用是创建代理，可以不依赖 ApplicationContext 的情况下进行使用。

```java
// 一个普通的 Java 类，不在 Spring 容器中
public class MyBusinessService {
    public void doBusinessOperation() {
        System.out.println("执行核心业务逻辑...");
    }
}
// 创建目标实例
MyBusinessService targetObject = new MyBusinessService();
```

**定义切面逻辑 (Advice/Advisor)**：创建实现 Spring AOP `Advice` 接口（如 `MethodBeforeAdvice`, `AfterReturningAdvice`, `ThrowsAdvice`, `MethodInterceptor`）的类或匿名内部类。

```java
import org.aopalliance.intercept.MethodInterceptor;
import org.aopalliance.intercept.MethodInvocation;

// 实现一个环绕通知
MethodInterceptor timingAdvice = new MethodInterceptor() {
    @Override
    public Object invoke(MethodInvocation invocation) throws Throwable {
        long startTime = System.currentTimeMillis();
        System.out.println("方法调用开始: " + invocation.getMethod().getName());

        try {
            // 执行目标方法
            Object result = invocation.proceed();
            return result;
        } finally {
            long duration = System.currentTimeMillis() - startTime;
            System.out.println("方法调用结束: " + invocation.getMethod().getName() + ", 耗时: " + duration + "ms");
        }
    }
};
```

**使用 `ProxyFactory` 创建代理**：

```java
// 创建 ProxyFactory
ProxyFactory proxyFactory = new ProxyFactory();
// 设置目标对象
proxyFactory.setTarget(targetObject);
// 添加通知
proxyFactory.addAdvice(timingAdvice);

// 可选：如果需要强制使用 CGLIB 代理（即使目标有接口）
// proxyFactory.setProxyTargetClass(true);

// 创建代理对象
// 注意：如果 MyBusinessService 实现了接口，getProxy() 返回的是该接口的代理。
// 如果没有接口或强制使用 CGLIB，则返回 MyBusinessService 的子类代理。
MyBusinessService proxy = (MyBusinessService) proxyFactory.getProxy();
```

**使用代理对象**：现在，你应该使用返回的 `proxy` 对象，而不是原始的 `targetObject`。对 `proxy` 方法的调用会触发 AOP 逻辑。

```java
// 调用代理方法
proxy.doBusinessOperation();
// 输出:
// 方法调用开始: doBusinessOperation
// 执行核心业务逻辑...
// 方法调用结束: doBusinessOperation, 耗时: Xms
```





END