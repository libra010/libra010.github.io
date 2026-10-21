Java ContextHolder



## ContextHolder

Java项目中可以看到一些以ContextHolder结尾的类名，通常体现的是一种**上下文（Context）模式**，它结合了 **ThreadLocal 模式** 和 **单例模式（或静态工具类）** 的思想，用于在同一个线程中传递和共享上下文信息。

`ContextHolder` 类的核心目标是：**在当前线程的执行过程中，方便地访问某个上下文对象（如用户身份、请求信息、租户信息等），而无需层层传递参数。**为此，它通常使用 `ThreadLocal` 来实现线程隔离的上下文存储。

通常在请求处理的入口设置上下文（如 Filter、Interceptor）。在请求结束时务必 clear()，防止内存泄漏。

```java
// 常见结构
public class SecurityContextHolder {
    private static final ThreadLocal<SecurityContext> contextHolder = 
        new ThreadLocal<>();

    public static void setContext(SecurityContext context) {
        contextHolder.set(context);
    }

    public static SecurityContext getContext() {
        return contextHolder.get();
    }

    public static void clearContext() {
        contextHolder.remove();
    }
}
```



Spring 中的实现：

#### RequestContextHolder

RequestContextHolder 是 Spring 框架中用于获取当前RequestContext（请求上下文信息）的工具类。它允许你在应用的不同层次访问到与当前线程绑定的 `HttpServletRequest` 和 `HttpServletResponse` 对象。

Spring 的 `DispatcherServlet`处理请求时会调用`RequestContextFilter`这个Filter，负责将当前的 `ServletRequestAttributes`（包含了 `HttpServletRequest` 和 `HttpServletResponse`）绑定到 `RequestContextHolder` 的 `ThreadLocal` 变量上。

```java
// 使用示例
public class RequestUtil {

    public static HttpServletRequest getCurrentRequest() {

        ServletRequestAttributes attributes =

        	(ServletRequestAttributes) RequestContextHolder.currentRequestAttributes();

        return attributes.getRequest();

    }

    public static HttpServletResponse getCurrentResponse() {

        ServletRequestAttributes attributes =

        	(ServletRequestAttributes) RequestContextHolder.currentRequestAttributes();

        return attributes.getResponse();

    }

}
```



#### SecurityContextHolder

SecurityContextHolder 是Spring Security框架中的一个核心组件，用于管理当前用户的SecurityContext（安全上下文信息），包括认证信息和授权信息。

`SecurityContextHolder` 中存储的核心对象是 `SecurityContext`。此对象包含了一个 `Authentication` 对象，它代表了当前已认证的主体（通常是用户）。











END