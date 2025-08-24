SpringMVC



## 原理介绍

Spring MVC 提供了一个 `DispatcherServlet` 作为中央Servlet处理所有请求。

它用一系列用于处理请求的核心组件来处理请求（这里列出三个比较重要的Bean）：

| Bean 类型                | 实现类                            | 说明                                                         |
| ------------------------ | --------------------------------- | ------------------------------------------------------------ |
| HandlerMapping           | RequestMappingHandlerMapping      | 处理 @RequestMapping 注解，将 URL 映射到控制器方法           |
| HandlerAdapter           | RequestMappingHandlerAdapter      | 调用控制器方法，处理调用方法中的方法参数，注解，返回值等等各类问题。 |
| HandlerExceptionResolver | ExceptionHandlerExceptionResolver | 处理调用控制器方法中出现的异常问题。                         |

DispatcherServlet 在 initStrategies 方法中从IoC容器中获取所有的核心组件并注册。

```java
protected void initStrategies(ApplicationContext context) {
    initMultipartResolver(context);
    initLocaleResolver(context);
    initThemeResolver(context);
    initHandlerMappings(context);
    initHandlerAdapters(context);
    initHandlerExceptionResolvers(context);
    initRequestToViewNameTranslator(context);
    initViewResolvers(context);
    initFlashMapManager(context);
}
```





## MVC自定义配置

1. @EnableWebMvc 注解会引入 DelegatingWebMvcConfiguration 类

这个类的主要作用就是收集所有的 WebMvcConfigurer 实现，所以我们可以 WebMvcConfigurer 实现这个接口来自定义一些配置。

2. DelegatingWebMvcConfiguration 类实现了 WebMvcConfigurationSupport 类。

这个类是SpringMVC的核心配置类，里面有一系列的 @Bean 注解的方法，主要作用是通过这些方法向IoC容器中注册上述HandlerMapping这些核心组件。在注册核心组件的时候会调用之前收集到 WebMvcConfigurer 一系列自定义配置进行组装这些核心组件。比如实现 addInterceptors 添加拦截器方法添加的拦截器会添加到HandlerMapping这个核心组件中去。

3. 最后就是上述 DispatcherServlet 启动时会通过 initStrategies 方法中从IoC容器中获取所有的核心组件并注册。

所以自定义 WebMvcConfigurer 就是用于定制HandlerMapping这些核心组件。

注意：在Spring Boot中不要使用 @EnableWebMvc 注解，因为Spring Boot已经自动配置了，如果使用这个注解的话就会使自动配置失效。

PS：如果想去自己实现一个类似HandlerMapping这样的核心组件也是可以的，具体方法略。





## DispatcherServlet 处理请求的流程

既然本质是一个Servlet，那请求处理入口肯定是 doGet 或 doPost 方法。

```text
FrameworkServlet#doGet/doPost --->
FrameworkServlet#processRequest --->
DispatcherServlet#doService --->
DispatcherServlet#doDispatch
```

上述是调用链，下面直接看 doDispatch 方法。

```java
/**
	 * Process the actual dispatching to the handler. 
	 * <p>The handler will be obtained by applying the servlet's HandlerMappings in order.
	 * The HandlerAdapter will be obtained by querying the servlet's installed HandlerAdapters
	 * to find the first that supports the handler class.
	 * <p>All HTTP methods are handled by this method. It's up to HandlerAdapters or handlers
	 * themselves to decide which methods are acceptable.
	 * @param request current HTTP request
	 * @param response current HTTP response
	 * @throws Exception in case of any kind of processing failure
	 * github.com/spring-projects/spring-framework/blob/v5.3.39/spring-webmvc/src/main/java/org/springframework/web/servlet/DispatcherServlet.java
	 */
	@SuppressWarnings("deprecation")
	// 方法参数是Servlet标准
	protected void doDispatch(HttpServletRequest request, HttpServletResponse response) throws Exception {
		HttpServletRequest processedRequest = request;
		HandlerExecutionChain mappedHandler = null;
		boolean multipartRequestParsed = false;

		WebAsyncManager asyncManager = WebAsyncUtils.getAsyncManager(request);

		try {
			ModelAndView mv = null;
			Exception dispatchException = null;

			try {
                // 1. 检查是否为文件上传请求，如果是，包装成 MultipartHttpServletRequest
				processedRequest = checkMultipart(request);
				multipartRequestParsed = (processedRequest != request);

                // 2. 根据 request 找到对应的处理器（Controller 方法）
            	//    调用 HandlerMapping.getHandler()
				// Determine handler for the current request.
				mappedHandler = getHandler(processedRequest);
				if (mappedHandler == null) {
					noHandlerFound(processedRequest, response);
					return;
				}

                // 3. 根据 handler 找到对应的适配器（HandlerAdapter）
            	//    调用 HandlerAdapter.supports() 和 getHandlerAdapter()
				// Determine handler adapter for the current request.
				HandlerAdapter ha = getHandlerAdapter(mappedHandler.getHandler());

				// Process last-modified header, if supported by the handler.
				String method = request.getMethod();
				boolean isGet = HttpMethod.GET.matches(method);
				if (isGet || HttpMethod.HEAD.matches(method)) {
					long lastModified = ha.getLastModified(request, mappedHandler.getHandler());
					if (new ServletWebRequest(request, response).checkNotModified(lastModified) && isGet) {
						return;
					}
				}

                // 4. 拦截器 preHandle
				if (!mappedHandler.applyPreHandle(processedRequest, response)) {
					return;
				}

                // 5. 真正调用 Controller 方法！调用 HandlerAdapter.handle()
				// Actually invoke the handler.
				mv = ha.handle(processedRequest, response, mappedHandler.getHandler());

				if (asyncManager.isConcurrentHandlingStarted()) {
					return;
				}

                // 6. 如果返回值没设置 view name，尝试默认视图名
				applyDefaultViewName(processedRequest, mv);
                // 7. 拦截器 postHandle
				mappedHandler.applyPostHandle(processedRequest, response, mv);
			}
			catch (Exception ex) {
				dispatchException = ex;
			}
			catch (Throwable err) {
				// 8. 处理异常（调用 @ExceptionHandler、HandlerExceptionResolver）
				dispatchException = new NestedServletException("Handler dispatch failed", err);
			}
            // 处理最终结果 返回异常 or 渲染视图 or 直接返回json
			processDispatchResult(processedRequest, response, mappedHandler, mv, dispatchException);
		}
		catch (Exception ex) {
			triggerAfterCompletion(processedRequest, response, mappedHandler, ex);
		}
		catch (Throwable err) {
			triggerAfterCompletion(processedRequest, response, mappedHandler,
					new NestedServletException("Handler processing failed", err));
		}
		finally {
			if (asyncManager.isConcurrentHandlingStarted()) {
				// Instead of postHandle and afterCompletion
				if (mappedHandler != null) {
					mappedHandler.applyAfterConcurrentHandlingStarted(processedRequest, response);
				}
			}
			else {
				// Clean up any resources used by a multipart request.
				if (multipartRequestParsed) {
					cleanupMultipart(processedRequest);
				}
			}
		}
	}
```







END