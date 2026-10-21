Spring IoC容器 启动流程



## AbstractApplicationContext#refresh 方法

```java
/**
 * refresh即是IoC容器启动的方法，也可以重新调用以重建IoC容器
 * github.com/spring-projects/spring-framework/blob/v5.3.39/spring-context/src/main/java/org/springframework/context/support/AbstractApplicationContext.java
 */
public void refresh() throws BeansException, IllegalStateException {
    // 加锁
	synchronized (this.startupShutdownMonitor) {
		StartupStep contextRefresh = this.applicationStartup.start("spring.context.refresh");

		// 准备工作，记录下容器的启动时间、标记“已启动”状态、处理配置文件中的占位符
		prepareRefresh();

		// 初始化Bean注册中心，这是BeanDefinition的Map
		ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();

		// 设置类加载器和一些BeanPostProcessor
		prepareBeanFactory(beanFactory);

		try {
			// BeanPostProcessor后置处理器扩展点
			postProcessBeanFactory(beanFactory);

			StartupStep beanPostProcess = this.applicationStartup.start("spring.context.beans.post-process");
            // Invoke factory processors registered as beans in the context.
			// 也是关于后置处理器的
			invokeBeanFactoryPostProcessors(beanFactory);
            // Register bean processors that intercept bean creation.
			// 注册 BeanPostProcessor 的实现类
			registerBeanPostProcessors(beanFactory);
			beanPostProcess.end();

            // Initialize message source for this context.
			// 初始化ApplicationContext的MessageSource组件
			initMessageSource();
            
            // Initialize event multicaster for this context.
			// 初始化ApplicationContext的Event事件组件
			initApplicationEventMulticaster();

            // Initialize other special beans in specific context subclasses.
			// on开头的方法一般是模板方法（钩子方法）SpringBoot中用于启动嵌入式WebServer
			onRefresh();

            // Check for listener beans and register them.
			// 注册事件监听器
			registerListeners();

            // Instantiate all remaining (non-lazy-init) singletons.
			// 初始化所有Bean
			finishBeanFactoryInitialization(beanFactory);

			// Last step: publish corresponding event.
            // 发布事件
			finishRefresh();
		}
		catch (BeansException ex) {
			if (logger.isWarnEnabled()) {
				logger.warn("Exception encountered during context initialization - " +
						"cancelling refresh attempt: " + ex);
			}

			// Destroy already created singletons to avoid dangling resources.
            // 销毁Bean
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
			contextRefresh.end();
		}
	}
}
```





















END