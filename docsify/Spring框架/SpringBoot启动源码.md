Spring Boot 启动流程



## SpringApplication#run 方法

```java
/**
	 * Run the Spring application, creating and refreshing a new
	 * {@link ApplicationContext}.
	 * @param args the application arguments (usually passed from a Java main method)
	 * @return a running {@link ApplicationContext}
	 * github.com/spring-projects/spring-boot/blob/v2.7.18/spring-boot-project/spring-boot/src/main/java/org/springframework/boot/SpringApplication.java
	 */
public ConfigurableApplicationContext run(String... args) {
    // 记录启动时间
    long startTime = System.nanoTime();
    
    // 创建一个启动时临时的Context容器 为一些ApplicationListener, ApplicationContextInitializer组件提供环境 还有管理配置等功能
    DefaultBootstrapContext bootstrapContext = createBootstrapContext();
    
    // 定义方法返回值
    ConfigurableApplicationContext context = null;
    
    // 配置java.awt.headless 用于无图形环境的服务器环境
    configureHeadlessProperty();
    
    // SpringApplication Events 事件监听器
    SpringApplicationRunListeners listeners = getRunListeners(args);
    // 发布事件(ApplicationStartingEvent)
    listeners.starting(bootstrapContext, this.mainApplicationClass);
    
    try {
        // 解析命令行参数
        ApplicationArguments applicationArguments = new DefaultApplicationArguments(args);
        
        // 准备Environment 包括发布事件(ApplicationEnvironmentPreparedEvent)
        ConfigurableEnvironment environment = prepareEnvironment(listeners, bootstrapContext, applicationArguments);
        
        // 配置spring.beaninfo.ignore 忽略BeanInfo类 优化IoC容器的性能
        configureIgnoreBeanInfo(environment);
        
        // 打印Banner
        Banner printedBanner = printBanner(environment);
        
        // 根据WebApplicationType创建上下文 Servlet环境创建AnnotationConfigServletWebServerApplicationContext
        // 继承自ServletWebServerApplicationContext 重写了onRefresh方法用于刷新上下文的时候启动WebServer
        context = createApplicationContext();
        
        // 注册ApplicationStartup 用于应用程序启动追踪 详见SpringBoot文档
        context.setApplicationStartup(this.applicationStartup);
        
        // 准备应用上下文
        prepareContext(bootstrapContext, context, environment, listeners, applicationArguments, printedBanner);
        
        // 刷新上下文(启动IoC容器, 启动WebServer)
        refreshContext(context);
        
        // 刷新上下文后的回调(在SpringApplication类中是一个protected空方法)
        afterRefresh(context, applicationArguments);
        
        // 计算启动耗时, 输出日志(INFO: Started Application in xxx seconds (JVM running for xxx))
        Duration timeTakenToStartup = Duration.ofNanos(System.nanoTime() - startTime);
        if (this.logStartupInfo) {
            new StartupInfoLogger(this.mainApplicationClass).logStarted(getApplicationLog(), timeTakenToStartup);
        }
        
        // 发布事件(ApplicationStartedEvent)
        listeners.started(context, timeTakenToStartup);
        
        // 执行ApplicationRunner和CommandLineRunner
        callRunners(context, applicationArguments);
    }
    catch (Throwable ex) {
        // 处理启动失败的异常 包括发布事件(ApplicationFailedEvent)
        handleRunFailure(context, ex, listeners);
        throw new IllegalStateException(ex);
    }
    
    try {
        // 计算应用启动事件
        Duration timeTakenToReady = Duration.ofNanos(System.nanoTime() - startTime);
        // 发布事件(ApplicationReadyEvent)
        listeners.ready(context, timeTakenToReady);
    }
    catch (Throwable ex) {
        // 处理启动失败的异常
        handleRunFailure(context, ex, null);
        throw new IllegalStateException(ex);
    }
    return context;
}
```





## SpringApplication#prepareContext 方法

```java
private void prepareContext(DefaultBootstrapContext bootstrapContext, ConfigurableApplicationContext context,
			ConfigurableEnvironment environment, SpringApplicationRunListeners listeners,
			ApplicationArguments applicationArguments, Banner printedBanner) {
    
    // 配置上一步设置好的ConfigurableEnvironment
    context.setEnvironment(environment);
    
    // 注册特殊的Bean, BeanNameGenerator. 设置ResourceLoader和ConversionService
    postProcessApplicationContext(context);
    
    // 应用初始化器(ApplicationContextInitializer)
    applyInitializers(context);
    
    // 发布事件(ApplicationContextInitializedEvent)
    listeners.contextPrepared(context);
    
    // 关闭引导上下文(BootstrapContext)
    bootstrapContext.close(context);
    
    // 打印日志(Starting Application using Java ...)(active profile)
    if (this.logStartupInfo) {
        logStartupInfo(context.getParent() == null);
        logStartupProfileInfo(context);
    }

    // 获取BeanFactory用于下面注册特殊的Bean
    ConfigurableListableBeanFactory beanFactory = context.getBeanFactory();
    
    // 注册特殊的Bean, springApplicationArguments(命令行参数), springBootBanner(Banner)
    beanFactory.registerSingleton("springApplicationArguments", applicationArguments);
    if (printedBanner != null) {
        beanFactory.registerSingleton("springBootBanner", printedBanner);
    }
    
    // 设置是否允许循环依赖和同名覆盖
    if (beanFactory instanceof AbstractAutowireCapableBeanFactory) {
        ((AbstractAutowireCapableBeanFactory) beanFactory).setAllowCircularReferences(this.allowCircularReferences);
        if (beanFactory instanceof DefaultListableBeanFactory) {
            ((DefaultListableBeanFactory) beanFactory)
            .setAllowBeanDefinitionOverriding(this.allowBeanDefinitionOverriding);
        }
    }
    
    // 根据配置设置全局懒加载
    if (this.lazyInitialization) {
        context.addBeanFactoryPostProcessor(new LazyInitializationBeanFactoryPostProcessor());
    }
    
    // 保证Environment中PropertySource的加载顺序
    context.addBeanFactoryPostProcessor(new PropertySourceOrderingBeanFactoryPostProcessor(context));
    
    // 加载sources(主配置类,@SpringBootApplication注解所在的类, 和setSources方法显式设置的其他配置类)
    Set<Object> sources = getAllSources();
    Assert.notEmpty(sources, "Sources must not be empty");
    load(context, sources.toArray(new Object[0]));
    
    // 发布事件
    listeners.contextLoaded(context);
}
```





## ServletWebServerApplicationContext#createWebServer 方法

```java
// 覆写ApplicationContext的onRefresh方法，在IoC容器启动的时候调用
@Override
protected void onRefresh() {
	super.onRefresh();
	try {
		createWebServer();
	}
	catch (Throwable ex) {
		throw new ApplicationContextException("Unable to start web server", ex);
	}
}

// 启动WebServer的主方法
private void createWebServer() {
	WebServer webServer = this.webServer;
	ServletContext servletContext = getServletContext();
	if (webServer == null && servletContext == null) {
		StartupStep createWebServer = this.getApplicationStartup().start("spring.boot.webserver.create");
		ServletWebServerFactory factory = getWebServerFactory();
		createWebServer.tag("factory", factory.getClass().toString());
        
        // 从工厂方法中获取WebServer，默认的工厂是TomcatServletWebServerFactory
        // 之后会实例化一个TomcatWebServer类, 在构造方法中启动嵌入式Tomcat
		this.webServer = factory.getWebServer(getSelfInitializer());
        
		createWebServer.end();
		getBeanFactory().registerSingleton("webServerGracefulShutdown",
				new WebServerGracefulShutdownLifecycle(this.webServer));
		getBeanFactory().registerSingleton("webServerStartStop",
				new WebServerStartStopLifecycle(this, this.webServer));
	}
	else if (servletContext != null) {
		try {
			getSelfInitializer().onStartup(servletContext);
		}
		catch (ServletException ex) {
			throw new ApplicationContextException("Cannot initialize servlet context", ex);
		}
	}
	initPropertySources();
}
```





## 配置WebServer

1、使用 `web.xml` 配置

配置ContextLoaderListener（ServletContext 监听器）以启动根 IoC 容器（Root WebApplicationContext）。

配置DispatcherServlet（Servlet）。

```xml
<listener>
    <listener-class>org.springframework.web.context.ContextLoaderListener</listener-class>
</listener>
<!-- 指定根上下文的配置文件位置 -->
<context-param>
    <param-name>contextConfigLocation</param-name>
    <param-value>/WEB-INF/applicationContext.xml</param-value>
</context-param>
<servlet>
    <servlet-name>dispatcher</servlet-name>
    <servlet-class>org.springframework.web.servlet.DispatcherServlet</servlet-class>
    <init-param>
        <param-name>contextConfigLocation</param-name>
        <param-value>/WEB-INF/dispatcher-servlet.xml</param-value>
    </init-param>
    <load-on-startup>1</load-on-startup>
</servlet>
<servlet-mapping>
    <servlet-name>dispatcher</servlet-name>
    <url-pattern>/</url-pattern>
</servlet-mapping>
```

2、使用 `WebApplicationInitializer` 配置

Spring 3.1 使用 Servlet 3.0+ 规范中的自动发现机制，提供了**ServletContainerInitializer**类，Servlet启动时会自动扫描这个的实现类并调用其中的方法。可以实现SpringMVC编程式地注册Servlet，Filter等功能，避免了`web.xml` 配置文件的使用。

Spring提供了AbstractAnnotationConfigDispatcherServletInitializer类来更方便的使用这个机制。

```java
public class MyWebAppInitializer extends AbstractAnnotationConfigDispatcherServletInitializer {
    // 指定根配置类 (对应 ContextLoaderListener 加载的上下文)
    @Override
    protected Class<?>[] getRootConfigClasses() {
        return new Class[] { RootConfig.class };
    }
    // 指定 Web 配置类 (对应 DispatcherServlet 加载的上下文)
    @Override
    protected Class<?>[] getServletConfigClasses() {
        return new Class[] { WebConfig.class };
    }
    // 指定 DispatcherServlet 映射
    @Override
    protected String[] getServletMappings() {
        return new String[] { "/" };
    }
}
```

3、SpringBoot自动配置

SpringBoot的ServletWebServerApplicationContext类是一个特殊的ApplicationContext，除了管理Bean外，它还负责Servlet容器的创建与管理。

SpringBoot提供了**ServletContextInitializer**类，模仿了Servlet容器的自动发现机制，在启动Servlet容器的时候，SpringBoot会发现所有的ServletContextInitializer实现类，并应用到嵌入式Servlet容器中。

ServletContextInitializer实现类包括，DispatcherServletRegistrationBean（配置DispatcherServlet），ServletRegistrationBean，FilterRegistrationBean等等。









END