SpringIoC控制反转



## 控制反转

控制反转（Inversion of Control）是一种思想，将控制权转移给外部实体，自身只需要关注自己的核心职责。

IoC 认为“控制一切”是繁琐，低效且危险的，转而通过“委托”和“协作”，利用更高级的抽象（容器、事件循环）来管理复杂性，从而让开发者能够专注于真正有价值的业务逻辑。

- JavaScript的回调函数是一种控制反转，比如使用Ajax调用接口时注册一个回调函数，不需要关心具体的执行逻辑，只需要关系自己的职责（ajax调用成功之后的业务逻辑），控制权在浏览器或者V8引擎手中。

- 模板方法设计模式是一种控制反转，子类只需要关注自己的覆写方法，控制权在父类手中。

- Spring容器是一种控制反转，每个Java类只需要关注自己的业务逻辑，而不需要去管理自己依赖的其他类。

可以看到 IoC 是一个极其宽泛和抽象的设计原则，任何转移控制权的场景都算广义的控制反转。

**Spring的IoC容器只是一种具体应用，它将 IoC 原则具体化为：管理 Java 对象（Beans）的创建、生命周期和依赖关系（Dependency Injection）的控制权，从应用程序代码反转给了 Spring 容器。**

在我们谈论Spring的控制反转时，实际上在讨论这种具体化的IoC。Spring IoC 容器负责 **对象创建**、**生命周期管理**和**依赖注入** 这三项核心任务。依赖注入 (DI) 是这三项任务中，最直接地体现“控制反转”原则的那一项。它明确地**将“获取依赖”的控制权从对象自身反转给了容器**。对象创建和声明周期管理一定程度上也是为依赖注入而服务的。因此，我们说“DI 是 IoC 的实现”，是因为 DI 是 IoC 原则在“依赖管理”这个关键领域最具体、最有效的落地方式。



## 配置Bean（XML方式）

XML 配置文件是配置 Bean 的主要方式之一，相较于注解方式具有以下优势：

**配置信息（XML 文件）与业务代码完全解耦**，实现了配置与实现的物理分离。理论上，无需修改 Java 代码即可调整对象的依赖关系——例如，类 A 依赖于接口 B，原本注入的是实现类 B1，只需修改 XML 配置即可将其切换为实现类 B2，从而实现依赖的动态变更。此外，该方式更符合 **Spring 框架“非侵入式设计”的核心理念**：业务代码中无需引入任何 Spring 特定的类或接口（而注解方式则需在代码中使用 Spring 相关注解，具有一定侵入性）。

```java
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xsi:schemaLocation="http://www.springframework.org/schema/beans
        https://www.springframework.org/schema/beans/spring-beans.xsd">

    <bean id="userService" class="com.example.service.UserService">
        <!-- setter依赖注入 --- name： 对象的属性名 --- ref: 引用另一个 Bean 的 id -->
        <property name="userRepository" ref="userRepository"/>
        <!-- 构造方法依赖注入（注意：XML配置不支持字段注入形式） -->
		<constructor-arg ref="beanTwo"/>
    </bean>
            
	<!-- autowire：Spring 会根据属性类型自动查找并注入匹配的 Bean -->
	<bean id="orderService" class="com.example.service.OrderService" autowire="byType"/>
        
    <!-- scope：代表Bean的Scope，这里是prototype原型 -->
	<bean id="userSession" class="com.example.model.UserSession" scope="prototype"/>
        
	<!-- 引入其他XML配置 -->
	<import resource="database-config.xml"/>

</beans>
```

bean元素的class属性定义都应该是可实例化的具体类，如果是接口的话会报错，Spring在依赖注入的时候进行类型检查，将符合接口类型的Bean进行注入，实现了依赖倒置原则。



## 配置Bean（注解方式）

随着 Java 5 引入注解，Spring开始支持注解配置Bean简化配置。Spring 2.5 引入了注解和自动扫描机制。

- **自动扫描配置**（直接配置应用业务的所有Bean，不需要写很多的XML配置了）

核心注解：@ComponentScan（启用自动扫描），@Autowired（依赖注入），@Component 及其衍生注解 (@Service, @Repository, @Controller)

- **JavaConfig**（Spring 3.0 引入，完全取代XML配置，将第三方库的Bean或者复杂创建过程的Bean纳入配置）

核心注解（注解名字跟XML元素值类似）：@Configuration（标记配置类，类似于一个XML配置文件）@Bean（在配置类上面使用，方法返回值被注册为一个Bean，类似于XML的bean元素）







END