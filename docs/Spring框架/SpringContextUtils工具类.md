Spring SpringContextUtils



Spring项目中通常会写一个ContextUtils（上下文工具类），用来从Spring容器中获取Bean，一般用于不方便采用依赖注入的地方。



1、采用**ApplicationContextAware**实现，注入ApplicationContext。

```java
// 用Component注解让其作为一个Spring Bean，纳入Spring容器管理
// 实现ApplicationContextAware接口，SpringBean的生命周期会调用接口方法注入ApplicationContext
// ApplicationContext属性和getBean方法都采用static静态实现，用于全局单例
@Component
public class SpringContextUtils implements ApplicationContextAware {
    private static ApplicationContext context;

    @Override
    public void setApplicationContext(ApplicationContext applicationContext) {
        context = applicationContext;
    }

    public static <T> T getBean(Class<T> beanClass) {
        return context.getBean(beanClass);
    }
    
    // ... other getBean methods
}
```



2、采用**BeanFactoryPostProcessor**实现，注入ConfigurableListableBeanFactory。

BeanFactoryPostProcessor容器后置处理器，ConfigurableListableBeanFactory是一个高级子接口，除了访问Bean，还可以更直接地操作BeanFactory，比如注册Bean。（一般获取Bean只使用ApplicationContext就够了，如果需要高级功能，才需要ConfigurableListableBeanFactory）

```java
@Component
public class SpringContextUtils implements BeanFactoryPostProcessor {
    private static ConfigurableListableBeanFactory beanFactory;

    @Override
    public void postProcessBeanFactory(ConfigurableListableBeanFactory factory) {
        beanFactory = factory;
    }

    public static <T> T getBean(Class<T> beanClass) {
        return beanFactory.getBean(beanClass);
    }
}
```







END