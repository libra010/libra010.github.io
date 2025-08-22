Spring IoC Environment



`Environment` 接口自 Spring 3.1 版本引入，旨在统一抽象 Spring 中的 profiles 和 properties 机制，为应用程序的环境配置提供一致的访问方式。在 Spring IoC 容器初始化过程中，`Environment` 会作为默认的 Bean 被自动注册，供容器内的其他组件使用。



## 介绍

**外部化配置**是 Spring 框架提供的一种机制，用于将配置信息（如数据库连接参数、服务端口、功能开关等）从应用程序代码中解耦，提升应用的可配置性和环境适应性。

核心概念：

- **属性来源（Property Sources）**：配置属性的来源，支持多种格式，包括属性文件（如 `.properties` 和 `.yml`）、环境变量、JVM 系统属性、命令行参数等。

- **环境抽象（Environment Abstraction）** ：通过 `Environment` 接口对配置属性的访问进行统一抽象，屏蔽底层不同配置源的差异。

简单来说，Spring将各种配置项统一包装成PropertySources对象，Environment背后管理着一个`PropertySource`的有序集合来统一访问这些配置。



## 配置源（Property Sources）

- **标准属性源（StandardEnvironment）**：包括两个 PropertySource 对象，其一是 JVM 系统属性集合（通过System::getProperties获取），其二是环境变量（通过System::getenv获取）

- **自定义属性源（PropertySource）**：添加自定义属性源有两种方式，其一是编程式（实现 `PropertySource` 接口，并通过编程方式将其注册到 `Environment` 中），其二是声明式（使用 `@PropertySource` 注解从外部配置文件加载属性，当前仅支持properties格式）



## Spring Boot 配置源

**优先级从低到高排序**（即后面的配置源会覆盖前面的配置源）

1、默认属性（通过 SpringApplication#setDefaultProperties 指定）

2、自定义PropertySource注解类（只有IoC容器刷新时才会添加）

3、application.properties 配置文件

4、application-{profile}.properties 配置文件

5、标准数据源（操作系统环境变量，JVM系统属性）

6、命令行参数（使用诸如 java -Dspring.profiles.active=test 的启动参数）



## 配置的使用

注意，配置的来源总是从Environment中获取。使用可以有多种方式。

- 通过 `@Value` 注解来注入。

- 直接从 `Environment` 中获取配置。

- Spring Boot 可以使用 `@ConfigurationProperties` 注解将一系列属性绑定到 Java Bean。



## Profiles

Spring 通过 **Profiles** 机制支持环境隔离，用于区分不同的运行环境，例如开发环境（`dev`）、测试环境（`test`）、生产环境（`prod`）等。根据不同激活的 Profile，Spring 可以动态注册不同的 Bean，实现环境相关的配置与行为切换。

通过设置 `spring.profiles.active` 属性来指定当前激活的 Profile，该配置由 `Environment` 管理，**因此 Profile 机制属于 Environment 抽象的一部分。**



**Spring Boot 对 Profile 的增强支持**：其一，在 application.properties 文件内通过分隔符定义多个profile。其二，独立的 application-{profile}.properties 文件。



## 附录：Spring Resource

Spring 框架中的 Resource 接口定义了一种统一的方式来访问**底层资源**。它屏蔽了不同底层资源（文件系统、类路径、URL、相对路径等）的类型差异。

核心方法是通过 `Resource::getInputStream` 来获取输入流进行处理。

与 Environment 的协作关系：@PropertySource 机制在加载配置文件时，底层依赖 Resource 来获取文件流。







END