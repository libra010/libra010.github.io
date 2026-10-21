Spring 动态数据源



## AbstractRoutingDataSource

Spring框架的数据源是DataSource类型的Bean，如果要实现动态数据源，需要提供一个特殊的DataSource来作为Bean依赖注入。

Spring提供了AbstractRoutingDataSource这个抽象类作为动态数据源，实现了DataSource接口，可以作为DataSource的Bean用来依赖注入。提供了一个determineCurrentLookupKey方法用来返回数据源。

```java
/**
 * 动态数据源
 * 
 * @author ruoyi
 */
public class DynamicDataSource extends AbstractRoutingDataSource
{
    public DynamicDataSource(DataSource defaultTargetDataSource, Map<Object, Object> targetDataSources)
    {
        // 设置默认数据源
        super.setDefaultTargetDataSource(defaultTargetDataSource);
        // 设置数据源Map
        super.setTargetDataSources(targetDataSources);
        // 必须调用，触发父类初始化代码，将targetDataSources进行解析转化
        super.afterPropertiesSet();
    }

    @Override
    protected Object determineCurrentLookupKey()
    {
        // 返回数据源，作为一个Key值从targetDataSources从查找数据源
        return DynamicDataSourceContextHolder.getDataSourceType();
    }
}

// 将其作为DataSource的Primary Bean来注册
@Bean(name = "dynamicDataSource")
@Primary
public DynamicDataSource dataSource(DataSource masterDataSource)
{
    Map<Object, Object> targetDataSources = new HashMap<>();
    targetDataSources.put(DataSourceType.MASTER.name(), masterDataSource);
    setDataSource(targetDataSources, DataSourceType.SLAVE.name(), "slaveDataSource");
    return new DynamicDataSource(masterDataSource, targetDataSources);
}
```



## DynamicDataSourceContextHolder

上述代码中使用了DynamicDataSourceContextHolder#getDataSourceType代码来获取数据源（其实是一个注册在targetDataSources中的Key值）。

下面来实现DynamicDataSourceContextHolder这个类，为了实现线程安全，当前数据源Key使用ThreadLocal静态变量存储（具体看注释）。

```java
/**
 * 数据源切换处理
 * 
 * @author ruoyi
 */
public class DynamicDataSourceContextHolder
{
    public static final Logger log = LoggerFactory.getLogger(DynamicDataSourceContextHolder.class);

    /**
     * 使用ThreadLocal维护变量，ThreadLocal为每个使用该变量的线程提供独立的变量副本，
     * 所以每一个线程都可以独立地改变自己的副本，而不会影响其它线程所对应的副本。
     */
    private static final ThreadLocal<String> CONTEXT_HOLDER = new ThreadLocal<>();

    /**
     * 设置数据源的变量
     */
    public static void setDataSourceType(String dsType)
    {
        log.info("切换到{}数据源", dsType);
        CONTEXT_HOLDER.set(dsType);
    }

    /**
     * 获得数据源的变量
     */
    public static String getDataSourceType()
    {
        return CONTEXT_HOLDER.get();
    }

    /**
     * 清空数据源变量
     */
    public static void clearDataSourceType()
    {
        CONTEXT_HOLDER.remove();
    }
}
```





## DataSourceAspect

在业务系统中推荐使用AOP切面来进行数据源的动态切换，可以统一对ThreadLocal进行remove处理，如果要自己直接使用DynamicDataSourceContextHolder来切换数据源，要注意remove掉ThreadLocal。

在这里我们定义一个DataSourceAspect切面类，DataSource自定义注解用来定义切入点。

注意：Spring声明式事务也是通过AOP来实现的，默认事务的Order是一个很大的数字，所以我们可以定义DataSourceAspect的Order注解值为1，来让他先于事务AOP执行。不定义Order的AOP具有最低优先级。

```java
/**
 * 多数据源处理
 * 
 * @author ruoyi
 */
@Aspect
@Order(1)
@Component
public class DataSourceAspect
{
    protected Logger logger = LoggerFactory.getLogger(getClass());

    @Pointcut("@annotation(com.ruoyi.common.annotation.DataSource)"
            + "|| @within(com.ruoyi.common.annotation.DataSource)")
    public void dsPointCut()
    {

    }

    @Around("dsPointCut()")
    public Object around(ProceedingJoinPoint point) throws Throwable
    {
        DataSource dataSource = getDataSource(point);

        if (StringUtils.isNotNull(dataSource))
        {
            DynamicDataSourceContextHolder.setDataSourceType(dataSource.value().name());
        }

        try
        {
            return point.proceed();
        }
        finally
        {
            // 销毁数据源 在执行方法之后
            DynamicDataSourceContextHolder.clearDataSourceType();
        }
    }

    /**
     * 获取需要切换的数据源
     */
    public DataSource getDataSource(ProceedingJoinPoint point)
    {
        MethodSignature signature = (MethodSignature) point.getSignature();
        DataSource dataSource = AnnotationUtils.findAnnotation(signature.getMethod(), DataSource.class);
        if (Objects.nonNull(dataSource))
        {
            return dataSource;
        }

        return AnnotationUtils.findAnnotation(signature.getDeclaringType(), DataSource.class);
    }
}
```





## 事务管理

Spring的AbstractRoutingDataSource仅支持动态切换数据源，本身不支持多数据源的事务管理。

可以使用 Sprin g的 JTA 事务管理器来支持，JtaTransactionManager类。（默认的事务管理器是DataSourceTransactionManager）

```java
@Bean
public PlatformTransactionManager transactionManager() {
    return new JtaTransactionManager();
}
```





## 动态添加数据源

可以修改DynamicDataSource类，使其能够在应用运行期间接收新的数据源并将其注册到路由机制中。

```java
public class DynamicDataSource extends AbstractRoutingDataSource {

    private final Map<Object, Object> targetDataSources = new HashMap<>();

    public DynamicDataSource(DataSource defaultTargetDataSource) {
        super.setDefaultTargetDataSource(defaultTargetDataSource);
        super.setTargetDataSources(targetDataSources);
        super.afterPropertiesSet();
    }

    /**
     * 动态添加数据源
     * @param key 数据源标识符
     * @param dataSource 数据源实例
     */
    public void addDataSource(Object key, DataSource dataSource) {
        synchronized (this) {
            targetDataSources.put(key, dataSource);
        	// 重新初始化以确保新数据源被识别，这个操作需要加锁保证线程安全
        	super.afterPropertiesSet();
        }
    }

    @Override
    protected Object determineCurrentLookupKey() {
        return DynamicDataSourceContextHolder.getDataSourceType();
    }
}
```

另外需要提供一个服务类来封装对DynamicDataSource的操作。

```java
@Service
public class DataSourceManagementService {

    @Autowired
    private DynamicDataSource dynamicDataSource;

    public void addDataSource(String dataSourceKey, String url, String username, String password) {
        // 创建新的 DruidDataSource 实例
        DruidDataSource dataSource = new DruidDataSource();
        dataSource.setUrl(url);
        dataSource.setUsername(username);
        dataSource.setPassword(password);
        // 可以根据需要设置更多属性

        // 将新数据源添加到动态数据源中
        dynamicDataSource.addDataSource(dataSourceKey, dataSource);
    }

    // 可以添加其他方法如 removeDataSource, updateDataSource 等
}
```



## 框架支持

如果需要轻量级的动态数据源，使用AbstractRoutingDataSource足矣。

Apache ShardingSphere，它支持分库分表、复杂路由、分布式事务，是一个全面的框架。

MyBatis-Plus 多数据源插件，基于AbstractRoutingDataSource的轻量级框架，集成Seata分布式事务和懒加载数据源的功能。









END