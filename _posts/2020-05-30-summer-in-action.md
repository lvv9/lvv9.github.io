# Spring实战

## Spring配置
- Java配置@Configuration
- XML配置（可以使用Spring Tool Suite）

## Bean声明
- @Component注解类声明（类似的包括@Repository、@Service、@Controller）
- Bean声明，包括Java配置的@Bean和XML配置的\<bean\>

## Bean发现
- 自动发现，包括Java配置的@ComponentScan和XML配置的\<context:component-scan\>，能发现@Component注解的组件
- @Bean或\<bean\>声明时发现

## Bean装配
- 自动装配@Autowired，会自动装配扫描或者声明时发现的Bean（类似的包括@Resource）
- Bean声明时作为参数自动装配，会自动装配扫描或者声明时发现的Bean

## XML配置
- c-命名空间对应\<constructor-arg\>构造器注入，p-命名空间对应\<property\>属性注入
  
## 混合配置
- Java与Java配置混合————@Import
- XML配置混入Java配置————@ImportResource
- XML与XML配置混合————\<import\>
- Java配置混入XML配置————\<Bean\>

## Profile定义
- Java配置使用@Profile注解配置类或方法
- XML配置使用\<beans\> profile属性

## Profile激活
- 分为设定值和默认值，都没有时仅创建没有定义在Profile中的Bean

## 条件化Bean发现
- @Conditional，与@Bean、@Component一起使用（或许只能用在Java配置上，XML配置表达能力较弱但可以通过其它方式实现？）
- 需要指定实现了Condition接口的xxx.class
- 在Spring 4中，Profile是用Conditional实现的

## 多Bean装配歧义
- 首选@Primary、\<bean\> primary属性，可以与@Component、@Bean、\<Bean\>一起使用，仅能设置一个
- @Qualifier限定，在Bean发现时与@Bean、@Component一起使用创建自定义@Qualifier限定符，在Bean装配时与@Autowired限定要装配的Bean
- 一个Bean只能有一个@Qualifier，可以使用多个自定义限定符注解代替多个自定义@Qualifier限定符

## Bean作用域
- @Scope、\<bean\> scope属性，value指定是单例还是其它
- 当注入到单例的Bean时，需要关注proxyMode、\<aop:scope-proxy\>

## IoC
控制反转，即对象的管理（Aka 控制），由对象控制（在对象的代码中硬编码）变为由框架、IoC容器根据一定的策略（单例等）控制（Aka 反转了）。

## Bean生命周期
按顺序用的比较多的有Constructor、setter、afterPropertiesSet()、init-method。

## AOP
- 面向切面编程，Spring AOP基于动态代理运行时增强，有JDK Proxy（接口）、CGLib（类）。
- 另外有AspectJ基于字节码操作。
- @Transactional是一种切面。

## MVC
前后端分离，Spring (Rest)Controller成为后端（非框架部分）的入口，View基本由前端实现，Controller返回业务数据Model。

## Filter vs. Interceptor
- Filter是javax.servlet的，HandlerInterceptor是org.springframework.web.servlet的
- Filter是定义在servlet的，理所当然是在servlet request的生命周期内作用，在Spring的应用里，是在DispatcherServlet接收请求前。
- 对于HandlerInterceptor，在DispatcherServlet处理逻辑的调用链中，会落到
```text
	protected void doDispatch(HttpServletRequest request, HttpServletResponse response) throws Exception {
		HttpServletRequest processedRequest = request;
		HandlerExecutionChain mappedHandler = null;
		boolean multipartRequestParsed = false;

		WebAsyncManager asyncManager = WebAsyncUtils.getAsyncManager(request);

		try {
			ModelAndView mv = null;
			Exception dispatchException = null;

			try {
				processedRequest = checkMultipart(request);
				multipartRequestParsed = (processedRequest != request);

				// Determine handler for the current request.
				mappedHandler = getHandler(processedRequest);
				if (mappedHandler == null) {
					noHandlerFound(processedRequest, response);
					return;
				}

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

				if (!mappedHandler.applyPreHandle(processedRequest, response)) {
					return;
				}

				// Actually invoke the handler.
				mv = ha.handle(processedRequest, response, mappedHandler.getHandler());

				if (asyncManager.isConcurrentHandlingStarted()) {
					return;
				}

				applyDefaultViewName(processedRequest, mv);
				mappedHandler.applyPostHandle(processedRequest, response, mv);
			}
			catch (Exception ex) {
				dispatchException = ex;
			}
			catch (Throwable err) {
				// As of 4.3, we're processing Errors thrown from handler methods as well,
				// making them available for @ExceptionHandler methods and other scenarios.
				dispatchException = new NestedServletException("Handler dispatch failed", err);
			}
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
在请求被handle前后，会触发HandlerInterceptor的preHandle、postHandle、afterCompletion。
- 即从调用链的角度看，Interceptor更接近Controller。

以上的拦截器是spring web的拦截器，在spring aop中，还有一种拦截器，它是实现spring bean增强的基础。