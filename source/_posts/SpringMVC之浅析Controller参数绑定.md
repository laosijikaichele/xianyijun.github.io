---
title: SpringMVC之浅析Controller参数绑定
date: 2016-05-30 16:06:59
tags: Spring
---

# **SpringMVC之浅析Controller参数绑定** #
写这篇博文主要是前几天在进行项目接口对接的时候，由于我们的项目基于restful风格的，全站ajax，后端只需要接收json参数，进行业务逻辑处理之后返回json数据就可以了，但是我后端是springmvc利用@RquestBody和@RequestParam来进行参数接收的，无法直接接收json参数完成变量映射，现在只能通过map来接收参数了。

## **URL到框架的映射** ##
在分析htpp请求参数绑定前，先对SpringMVC的url映射进行分析。
### **DispatcherServlet** ###
DispatcherServlet是SpringMVC中的核心类，当一个http请求服务器的时候，会被DispatcherServlet接收，然后DispatcherServlet会根据request找到对应controller,然后委托给controller进行处理，controller主要完成数据模型的创建和业务逻辑的处理，然后把填充完数据的模型委托给DispatcherServlet进行渲染，然后根据对应数据模板视图进行结合，向response进行输出响应。
![DispatcherServlet](http://7xrl91.com1.z0.glb.clouddn.com/DispatcherServlet.png)
#### **DispatcherServlet初始化** ####
- init-param配置文件读取

```java
public final void init() throws ServletException {
	try {
		PropertyValues pvs = new ServletConfigPropertyValues(getServletConfig(), this.requiredProperties);
		BeanWrapper bw = PropertyAccessorFactory.forBeanPropertyAccess(this);
		ResourceLoader resourceLoader = new ServletContextResourceLoader(getServletContext());
		bw.registerCustomEditor(Resource.class, new ResourceEditor(resourceLoader, getEnvironment()));
		initBeanWrapper(bw);
		bw.setPropertyValues(pvs, true);
	}
	catch (BeansException ex) {
		throw ex;
	}
	initServletBean();
}
```

DispatcherServle的初始化是在HttpServletBean／init方法中实现的，init方法主要注册了资源文件编辑器，让Servlet的init-param配置元素可以通过classpath来指定Spring框架的bean文件路径。

```xml
<servlet>
    <servlet-name>DispatcherServlet</servlet-name>
    <servlet-class>org.springframework.web.servlet.DispatcherServlet</servlet-class>
    <init-param>
        <param-name>contextConfigLocation</param-name>
        <param-value>classpath:spring/spring-servlet.xml</param-value>
    </init-param>
    <load-on-startup>1</load-on-startup>
</servlet>
<servlet-mapping>
    <servlet-name>DispatcherServlet</servlet-name>
    <url-pattern>/</url-pattern>
</servlet-mapping>
```

通过获取contextConfigLocation的init-param元素，将其转换为classpath路径下的资源文件，为Spring框架提供配置信息和设置DispatcherServlet的contextConfigLocation值。

- 容器上下文建立

DispatcherServlet的上下文创建是在initServletBean方法中完成，是一个模板方法，具体实现由子类实现(FrameworkServlet).

```java
protected final void initServletBean() throws ServletException {
	long startTime = System.currentTimeMillis();
	try {
		this.webApplicationContext = initWebApplicationContext();
		initFrameworkServlet();
	}
	catch (ServletException ex) {
		throw ex;
	}
	catch (RuntimeException ex) {
		throw ex;
	}
}
```

具体的上下文环境创建是在initWebApplicationContext中完成。

```java
protected WebApplicationContext initWebApplicationContext() {
	WebApplicationContext rootContext =
			WebApplicationContextUtils.getWebApplicationContext(getServletContext());
	WebApplicationContext wac = null;

	if (this.webApplicationContext != null) {
		wac = this.webApplicationContext;
		if (wac instanceof ConfigurableWebApplicationContext) {
			ConfigurableWebApplicationContext cwac = (ConfigurableWebApplicationContext) wac;
			if (!cwac.isActive()) {
				if (cwac.getParent() == null) {
					cwac.setParent(rootContext);
				}
				configureAndRefreshWebApplicationContext(cwac);
			}
		}
	}
	if (wac == null) {
		wac = findWebApplicationContext();
	}
	if (wac == null) {
		wac = createWebApplicationContext(rootContext);
	}

	if (!this.refreshEventReceived) {
		onRefresh(wac);
	}

	if (this.publishContext) {
		String attrName = getServletContextAttributeName();
		getServletContext().setAttribute(attrName, wac);
		}
	}
	return wac;
}
```

initWebApplicationContext方法首先获取在ContextLoaderListener初始化并注册到ServletContext的rootContext,然后对webApplicationContext进行判断，如果是不为空的话，就说明这个servlet是通过编程式注册到servletContext中的(servlet3.0中的servletContext.addServlet)的，上下文也是通过编程式创建的，如果上下文还没有初始化的话，就将rootContext作为上下文的父上下文，然后进行初始化。否则直接使用。

```java
protected WebApplicationContext findWebApplicationContext() {
	String attrName = getContextAttribute();
	if (attrName == null) {
		return null;
	}
	WebApplicationContext wac =
			WebApplicationContextUtils.getWebApplicationContext(getServletContext(), attrName);
	if (wac == null) {
		throw new IllegalStateException("No WebApplicationContext found: initializer not registered?");
	}
	return wac;
}
```

如果上下文不是通过编程式创建的，就通过contextAttribute属性为键，在ServletContext查找上下文，如果上下文不为空的话，说明上下文在其他地方进行初始化并注册到contextAttribute下。
如果没有编程式创建上下文或者在servletContext查找不到上下文，就调用createWebApplicationContext创建一个以rootContext为父上下文的上下文。

```java
protected WebApplicationContext createWebApplicationContext(ApplicationContext parent) {
	Class<?> contextClass = getContextClass();
	if (!ConfigurableWebApplicationContext.class.isAssignableFrom(contextClass)) {
		throw new ApplicationContextException(
				"Fatal initialization error in servlet with name '" + getServletName() +
				"': custom WebApplicationContext class [" + contextClass.getName() +
				"] is not of type ConfigurableWebApplicationContext");
	}
	ConfigurableWebApplicationContext wac =
			(ConfigurableWebApplicationContext) BeanUtils.instantiateClass(contextClass);
	wac.setEnvironment(getEnvironment());
	wac.setParent(parent);
	wac.setConfigLocation(getContextConfigLocation());
	configureAndRefreshWebApplicationContext(wac);
	return wac;
}
```

最后通过调用onRefresh方法(在DispatcherServlet)进行回调实现默认类的初始化。

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

#### **DispatcherServlet请求转发** ####
在完成初始化之后，DispatcherServlet只需要等待请求的到来进行接收和转发就可以了。
DispatcherServlet只要是在父类FrameworkServlet对doGet、doPost、doPut、doDelete进行重载，都委托给processRequest方法来进行处理。
processRequest主要是将请求的Locale对象和属性与线程进行绑定，在执行完doService之后，在跟线程进行解绑，最后请求处理结束之后，发布ServletRequestHandledEvent事件，可以通过注册监听器来监听事件。

```java
protected final void processRequest(HttpServletRequest request, HttpServletResponse response)
			throws ServletException, IOException {

	long startTime = System.currentTimeMillis();
	Throwable failureCause = null;

	LocaleContext previousLocaleContext = LocaleContextHolder.getLocaleContext();
	LocaleContext localeContext = buildLocaleContext(request);

	RequestAttributes previousAttributes = RequestContextHolder.getRequestAttributes();
	ServletRequestAttributes requestAttributes = buildRequestAttributes(request, response, previousAttributes);

	WebAsyncManager asyncManager = WebAsyncUtils.getAsyncManager(request);
	asyncManager.registerCallableInterceptor(FrameworkServlet.class.getName(), new RequestBindingInterceptor());

	initContextHolders(request, localeContext, requestAttributes);

	try {
		doService(request, response);
	}
	catch (ServletException ex) {
		failureCause = ex;
		throw ex;
	}
	catch (IOException ex) {
		failureCause = ex;
		throw ex;
	}
	catch (Throwable ex) {
		failureCause = ex;
		throw new NestedServletException("Request processing failed", ex);
	}
	finally {
		resetContextHolders(request, previousLocaleContext, previousAttributes);
		if (requestAttributes != null) {
			requestAttributes.requestCompleted();
		}
		publishRequestHandledEvent(request, response, startTime, failureCause);
	}
}
```

请求处理主要在doService进行，具体实现是在子类(DispatcherServlet)中实现。doService首先将之前初始化的对象设置到request中，供下一步进行处理，最后调用doDispatcher方法进行转发。

```java
protected void doService(HttpServletRequest request, HttpServletResponse response) throws Exception {
	Map<String, Object> attributesSnapshot = null;
	if (WebUtils.isIncludeRequest(request)) {
		attributesSnapshot = new HashMap<String, Object>();
		Enumeration<?> attrNames = request.getAttributeNames();
		while (attrNames.hasMoreElements()) {
			String attrName = (String) attrNames.nextElement();
			if (this.cleanupAfterInclude || attrName.startsWith("org.springframework.web.servlet")) {
				attributesSnapshot.put(attrName, request.getAttribute(attrName));
			}
		}
	}
	request.setAttribute(WEB_APPLICATION_CONTEXT_ATTRIBUTE, getWebApplicationContext());
	request.setAttribute(LOCALE_RESOLVER_ATTRIBUTE, this.localeResolver);
	request.setAttribute(THEME_RESOLVER_ATTRIBUTE, this.themeResolver);
	request.setAttribute(THEME_SOURCE_ATTRIBUTE, getThemeSource());

	FlashMap inputFlashMap = this.flashMapManager.retrieveAndUpdate(request, response);
	if (inputFlashMap != null) {
		request.setAttribute(INPUT_FLASH_MAP_ATTRIBUTE, Collections.unmodifiableMap(inputFlashMap));
	}
	request.setAttribute(OUTPUT_FLASH_MAP_ATTRIBUTE, new FlashMap());
	request.setAttribute(FLASH_MAP_MANAGER_ATTRIBUTE, this.flashMapManager);

	try {
		doDispatch(request, response);
	}
	finally {
		if (!WebAsyncUtils.getAsyncManager(request).isConcurrentHandlingStarted()) {
			if (attributesSnapshot != null) {
				restoreAttributesAfterInclude(request, attributesSnapshot);
			}
		}
	}
}
```

doDispatch方法主要是通过调用getHandler方法获取HandlerExecutionChain，然后通过HandlerExecutionChain获取HandlerAdapter来处理HandlerExecutionChain获取渲染的视图对象。

```java
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
			mappedHandler = getHandler(processedRequest);
			if (mappedHandler == null || mappedHandler.getHandler() == null) {
				noHandlerFound(processedRequest, response);
				return;
			}

			HandlerAdapter ha = getHandlerAdapter(mappedHandler.getHandler());

			String method = request.getMethod();
			boolean isGet = "GET".equals(method);
			if (isGet || "HEAD".equals(method)) {
				long lastModified = ha.getLastModified(request, mappedHandler.getHandler());
				if (new ServletWebRequest(request, response).checkNotModified(lastModified) && isGet) {
					return;
				}
			}
			if (!mappedHandler.applyPreHandle(processedRequest, response)) {
				return;
			}
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
		processDispatchResult(processedRequest, response, mappedHandler, mv, dispatchException);
	}
	catch (Exception ex) {
		triggerAfterCompletion(processedRequest, response, mappedHandler, ex);
	}
	catch (Error err) {
		triggerAfterCompletionWithError(processedRequest, response, mappedHandler, err);
	}
	finally {
		if (asyncManager.isConcurrentHandlingStarted()) {
			if (mappedHandler != null) {
				mappedHandler.applyAfterConcurrentHandlingStarted(processedRequest, response);
			}
		}
		else {
			if (multipartRequestParsed) {
				cleanupMultipart(processedRequest);
			}
		}
	}
}
```

## **http请求参数绑定** ##
在上面的分析，我们可以知道在HandlerAdapter调用handle方法处理请求。

- **AbstractHandlerMethodAdapter/handle** 
handler方法委托给handleInternal调用，是一个钩子函数，具体实现由子类实现。

```java
public final ModelAndView handle(HttpServletRequest request, HttpServletResponse response, Object handler)
		throws Exception {

	return handleInternal(request, response, (HandlerMethod) handler);
}
```

- **RequestMappingHandlerAdapter/handleInternal** 
handleInternal主要是对session和浏览器缓存控制，委托invokeHandlerMethod进行处理，synchronizeOnSession默认为false,如果为true的话，需要在同步代码块执行invokeHandlerMethod，串行化处理会话请求。

```java
protected ModelAndView handleInternal(HttpServletRequest request,
		HttpServletResponse response, HandlerMethod handlerMethod) throws Exception {

	checkRequest(request);

	if (getSessionAttributesHandler(handlerMethod).hasSessionAttributes()) {
		applyCacheSeconds(response, this.cacheSecondsForSessionAttributeHandlers);
	}
	else {
		prepareResponse(response);
	}
	if (this.synchronizeOnSession) {
		HttpSession session = request.getSession(false);
		if (session != null) {
			Object mutex = WebUtils.getSessionMutex(session);
			synchronized (mutex) {
				return invokeHandlerMethod(request, response, handlerMethod);
			}
		}
	}
	return invokeHandlerMethod(request, response, handlerMethod);
}
```

-  **RequestMappingHandlerAdapter/invokeHandlerMethod**
invokeHandlerMethod首先创建mavContainer存储ModelAndView,然后将request中所有的attribute设置到mavContainer中，然后调用modelFactory.initModel将所有的@ModeAndView方法执行一遍,把调用方法的结果放到mavContainer中，调用ServletInvocableHandlerMethod的invokeAndHandle方法处理，最后返回ModeAndView

```java
protected ModelAndView invokeHandlerMethod(HttpServletRequest request,
		HttpServletResponse response, HandlerMethod handlerMethod) throws Exception {

	ServletWebRequest webRequest = new ServletWebRequest(request, response);

	WebDataBinderFactory binderFactory = getDataBinderFactory(handlerMethod);
	ModelFactory modelFactory = getModelFactory(handlerMethod, binderFactory);

	ServletInvocableHandlerMethod invocableMethod = createInvocableHandlerMethod(handlerMethod);
	invocableMethod.setHandlerMethodArgumentResolvers(this.argumentResolvers);
	invocableMethod.setHandlerMethodReturnValueHandlers(this.returnValueHandlers);
	invocableMethod.setDataBinderFactory(binderFactory);
	invocableMethod.setParameterNameDiscoverer(this.parameterNameDiscoverer);

	ModelAndViewContainer mavContainer = new ModelAndViewContainer();
	mavContainer.addAllAttributes(RequestContextUtils.getInputFlashMap(request));
	modelFactory.initModel(webRequest, mavContainer, invocableMethod);
	mavContainer.setIgnoreDefaultModelOnRedirect(this.ignoreDefaultModelOnRedirect);

	AsyncWebRequest asyncWebRequest = WebAsyncUtils.createAsyncWebRequest(request, response);
	asyncWebRequest.setTimeout(this.asyncRequestTimeout);

	WebAsyncManager asyncManager = WebAsyncUtils.getAsyncManager(request);
	asyncManager.setTaskExecutor(this.taskExecutor);
	asyncManager.setAsyncWebRequest(asyncWebRequest);
	asyncManager.registerCallableInterceptors(this.callableInterceptors);
	asyncManager.registerDeferredResultInterceptors(this.deferredResultInterceptors);

	if (asyncManager.hasConcurrentResult()) {
		Object result = asyncManager.getConcurrentResult();
		mavContainer = (ModelAndViewContainer) asyncManager.getConcurrentResultContext()[0];
		asyncManager.clearConcurrentResult();
		invocableMethod = invocableMethod.wrapConcurrentResult(result);
	}

	invocableMethod.invokeAndHandle(webRequest, mavContainer);
	if (asyncManager.isConcurrentHandlingStarted()) {
		return null;
	}

	return getModelAndView(mavContainer, modelFactory, webRequest);
}
```

- **InvocableHandlerMethod/invokeForRequest** 
传入对应参数调用doInvoke进行处理。

```java
public Object invokeForRequest(NativeWebRequest request, ModelAndViewContainer mavContainer,
		Object... providedArgs) throws Exception {

	Object[] args = getMethodArgumentValues(request, mavContainer, providedArgs);
	Object returnValue = doInvoke(args);
	return returnValue;
}
```
- **InvocableHandlerMethod/doInvoke**
通过反射执行对应的Method。

```java
protected Object doInvoke(Object... args) throws Exception {
	ReflectionUtils.makeAccessible(getBridgedMethod());
	try {
		return getBridgedMethod().invoke(getBean(), args);
	}
	catch (IllegalArgumentException ex) {
		assertTargetBean(getBridgedMethod(), getBean(), args);
		throw new IllegalStateException(getInvocationErrorMessage(ex.getMessage(), args), ex);
	}
	catch (InvocationTargetException ex) {
		Throwable targetException = ex.getTargetException();
		if (targetException instanceof RuntimeException) {
			throw (RuntimeException) targetException;
		}
		else if (targetException instanceof Error) {
			throw (Error) targetException;
		}
		else if (targetException instanceof Exception) {
			throw (Exception) targetException;
		}
		else {
			String msg = getInvocationErrorMessage("Failed to invoke controller method", args);
			throw new IllegalStateException(msg, targetException);
		}
	}
}
```
- **InvocableHandlerMethod/getMethodArgumentValues** 
通过获取argumentResolvers对parameter进行解析处理

```java
private Object[] getMethodArgumentValues(NativeWebRequest request, ModelAndViewContainer mavContainer,
		Object... providedArgs) throws Exception {

	MethodParameter[] parameters = getMethodParameters();
	Object[] args = new Object[parameters.length];
	for (int i = 0; i < parameters.length; i++) {
		MethodParameter parameter = parameters[i];
		parameter.initParameterNameDiscovery(this.parameterNameDiscoverer);
		GenericTypeResolver.resolveParameterType(parameter, getBean().getClass());
		args[i] = resolveProvidedArgument(parameter, providedArgs);
		if (args[i] != null) {
			continue;
		}
		if (this.argumentResolvers.supportsParameter(parameter)) {
			try {
				args[i] = this.argumentResolvers.resolveArgument(
						parameter, mavContainer, request, this.dataBinderFactory);
				continue;
			}
			catch (Exception ex) {
				throw ex;
			}
		}
		if (args[i] == null) {
			String msg = getArgumentResolutionErrorMessage("No suitable resolver for argument", i);
			throw new IllegalStateException(msg);
		}
	}
	return args;
}
```

- **HandlerMethodArgumentResolverComposite/resolveArgument** 
迭代所有注册的HandlerMethodArgumentResolver并调用解析参数。

```java
public Object resolveArgument(MethodParameter parameter, ModelAndViewContainer mavContainer,
		NativeWebRequest webRequest, WebDataBinderFactory binderFactory) throws Exception {

	HandlerMethodArgumentResolver resolver = getArgumentResolver(parameter);
	if (resolver == null) {
		throw new IllegalArgumentException("Unknown parameter type [" + parameter.getParameterType().getName() + "]");
	}
	return resolver.resolveArgument(parameter, mavContainer, webRequest, binderFactory);
}
```

- **ServletInvocableHandlerMethod/invokeAndHandle** 
执行调用方法并对通过HandlerMethodReturnValueHandler对返回值进行处理

```java
public void invokeAndHandle(ServletWebRequest webRequest,
		ModelAndViewContainer mavContainer, Object... providedArgs) throws Exception {

	Object returnValue = invokeForRequest(webRequest, mavContainer, providedArgs);
	setResponseStatus(webRequest);

	if (returnValue == null) {
		if (isRequestNotModified(webRequest) || hasResponseStatus() || mavContainer.isRequestHandled()) {
			mavContainer.setRequestHandled(true);
			return;
		}
	}
	else if (StringUtils.hasText(this.responseReason)) {
		mavContainer.setRequestHandled(true);
		return;
	}

	mavContainer.setRequestHandled(false);
	try {
		this.returnValueHandlers.handleReturnValue(
				returnValue, getReturnValueType(returnValue), mavContainer, webRequest);
	}
	catch (Exception ex) {
		if (logger.isTraceEnabled()) {
			logger.trace(getReturnValueHandlingErrorMessage("Error handling return value", returnValue), ex);
		}
		throw ex;
	}
}
```

- **HandlerMethodReturnValueHandlerComposite/handleReturnValue** 
根据returnValue和returnType查找对应的HandlerMethodReturnValueHandler进行处理

```java
public void handleReturnValue(Object returnValue, MethodParameter returnType,
		ModelAndViewContainer mavContainer, NativeWebRequest webRequest) throws Exception {

	HandlerMethodReturnValueHandler handler = selectHandler(returnValue, returnType);
	handler.handleReturnValue(returnValue, returnType, mavContainer, webRequest);
}
```

### **RequestResponseBodyMethodProcessor** ###
![RequestResponseBodyMethodProcessor](http://7xrl91.com1.z0.glb.clouddn.com/RequestResponseBodyMethodProcessor.png)
RequestResponseBodyMethodProcessor实现了HandlerMethodArgumentResolver和HandlerMethodReturnValueHandler接口，通过HttpMessageConverter对request和response的body进行读写操作来将请求报文绑定处理方法的@RequestBody注解的参数和对@ResponseBody注解的返回值进行处理。

#### **resolveArgument** ####
![resolveArgument](http://7xrl91.com1.z0.glb.clouddn.com/Selection_030.png)
- resolveArgument

```java
public Object resolveArgument(MethodParameter parameter, ModelAndViewContainer mavContainer,
		NativeWebRequest webRequest, WebDataBinderFactory binderFactory) throws Exception {

	Object arg = readWithMessageConverters(webRequest, parameter, parameter.getGenericParameterType());
	String name = Conventions.getVariableNameForParameter(parameter);

	WebDataBinder binder = binderFactory.createBinder(webRequest, arg, name);
	if (arg != null) {
		validateIfApplicable(binder, parameter);
		if (binder.getBindingResult().hasErrors() && isBindExceptionRequired(binder, parameter)) {
			throw new MethodArgumentNotValidException(parameter, binder.getBindingResult());
		}
	}
	mavContainer.addAttribute(BindingResult.MODEL_KEY_PREFIX + name, binder.getBindingResult());

	return arg;
}
```

- readWithMessageConverters

```java
protected <T> Object readWithMessageConverters(NativeWebRequest webRequest, MethodParameter methodParam,
		Type paramType) throws IOException, HttpMediaTypeNotSupportedException, HttpMessageNotReadableException {

	HttpServletRequest servletRequest = webRequest.getNativeRequest(HttpServletRequest.class);
	ServletServerHttpRequest inputMessage = new ServletServerHttpRequest(servletRequest);

	Object arg = readWithMessageConverters(inputMessage, methodParam, paramType);
	if (arg == null) {
		if (methodParam.getParameterAnnotation(RequestBody.class).required()) {
			throw new HttpMessageNotReadableException("Required request body is missing: " +
					methodParam.getMethod().toGenericString());
		}
	}
	return arg;
}
```

readWithMessageConverters

```java
protected <T> Object readWithMessageConverters(HttpInputMessage inputMessage, MethodParameter param,
			Type targetType) throws IOException, HttpMediaTypeNotSupportedException, HttpMessageNotReadableException {

	MediaType contentType;
	boolean noContentType = false;
	try {
		contentType = inputMessage.getHeaders().getContentType();
	}
	catch (InvalidMediaTypeException ex) {
		throw new HttpMediaTypeNotSupportedException(ex.getMessage());
	}
	if (contentType == null) {
		noContentType = true;
		contentType = MediaType.APPLICATION_OCTET_STREAM;
	}

	Class<?> contextClass = (param != null ? param.getContainingClass() : null);
	Class<T> targetClass = (targetType instanceof Class<?> ? (Class<T>) targetType : null);
	if (targetClass == null) {
		ResolvableType resolvableType = (param != null ?
				ResolvableType.forMethodParameter(param) : ResolvableType.forType(targetType));
		targetClass = (Class<T>) resolvableType.resolve();
	}

	HttpMethod httpMethod = ((HttpRequest) inputMessage).getMethod();
	Object body = NO_VALUE;

	try {
		inputMessage = new EmptyBodyCheckingHttpInputMessage(inputMessage);

		for (HttpMessageConverter<?> converter : this.messageConverters) {
			Class<HttpMessageConverter<?>> converterType = (Class<HttpMessageConverter<?>>) converter.getClass();
			if (converter instanceof GenericHttpMessageConverter) {
				GenericHttpMessageConverter<?> genericConverter = (GenericHttpMessageConverter<?>) converter;
				if (genericConverter.canRead(targetType, contextClass, contentType)) {
					if (inputMessage.getBody() != null) {
						inputMessage = getAdvice().beforeBodyRead(inputMessage, param, targetType, converterType);
						body = genericConverter.read(targetType, contextClass, inputMessage);
						body = getAdvice().afterBodyRead(body, inputMessage, param, targetType, converterType);
					}
					else {
						body = null;
						body = getAdvice().handleEmptyBody(body, inputMessage, param, targetType, converterType);
					}
					break;
				}
			}
			else if (targetClass != null) {
				if (converter.canRead(targetClass, contentType)) {
					if (inputMessage.getBody() != null) {
						inputMessage = getAdvice().beforeBodyRead(inputMessage, param, targetType, converterType);
						body = ((HttpMessageConverter<T>) converter).read(targetClass, inputMessage);
						body = getAdvice().afterBodyRead(body, inputMessage, param, targetType, converterType);
					}
					else {
						body = null;
						body = getAdvice().handleEmptyBody(body, inputMessage, param, targetType, converterType);
					}
					break;
				}
			}
		}
	}
	catch (IOException ex) {
		throw new HttpMessageNotReadableException("Could not read document: " + ex.getMessage(), ex);
	}

	if (body == NO_VALUE) {
		if (!SUPPORTED_METHODS.contains(httpMethod)
				|| (noContentType && inputMessage.getBody() == null)) {
			return null;
		}
		throw new HttpMediaTypeNotSupportedException(contentType, this.allSupportedMediaTypes);
	}

	return body;
}
```

#### **handleReturnValue** ####
![handleReturnValue](http://7xrl91.com1.z0.glb.clouddn.com/Selection_029.png)

- handleReturnValue

```java
public void handleReturnValue(Object returnValue, MethodParameter returnType,
		ModelAndViewContainer mavContainer, NativeWebRequest webRequest)
		throws IOException, HttpMediaTypeNotAcceptableException, HttpMessageNotWritableException {
	mavContainer.setRequestHandled(true);
	writeWithMessageConverters(returnValue, returnType, webRequest);
}
```

- writeWithMessageConverters

```java
protected <T> void writeWithMessageConverters(T returnValue, MethodParameter returnType, NativeWebRequest webRequest)
		throws IOException, HttpMediaTypeNotAcceptableException, HttpMessageNotWritableException {

	ServletServerHttpRequest inputMessage = createInputMessage(webRequest);
	ServletServerHttpResponse outputMessage = createOutputMessage(webRequest);
	writeWithMessageConverters(returnValue, returnType, inputMessage, outputMessage);
}
```

- writeWithMessageConverters

```java
protected <T> void writeWithMessageConverters(T returnValue, MethodParameter returnType,
			ServletServerHttpRequest inputMessage, ServletServerHttpResponse outputMessage)
			throws IOException, HttpMediaTypeNotAcceptableException, HttpMessageNotWritableException {

	Class<?> returnValueClass = getReturnValueType(returnValue, returnType);
	Type returnValueType = getGenericType(returnType);
	HttpServletRequest servletRequest = inputMessage.getServletRequest();
	List<MediaType> requestedMediaTypes = getAcceptableMediaTypes(servletRequest);
	List<MediaType> producibleMediaTypes = getProducibleMediaTypes(servletRequest, returnValueClass, returnValueType);

	if (returnValue != null && producibleMediaTypes.isEmpty()) {
		throw new IllegalArgumentException("No converter found for return value of type: " + returnValueClass);
	}

	Set<MediaType> compatibleMediaTypes = new LinkedHashSet<MediaType>();
	for (MediaType requestedType : requestedMediaTypes) {
		for (MediaType producibleType : producibleMediaTypes) {
			if (requestedType.isCompatibleWith(producibleType)) {
				compatibleMediaTypes.add(getMostSpecificMediaType(requestedType, producibleType));
			}
		}
	}
	if (compatibleMediaTypes.isEmpty()) {
		if (returnValue != null) {
			throw new HttpMediaTypeNotAcceptableException(producibleMediaTypes);
		}
		return;
	}

	List<MediaType> mediaTypes = new ArrayList<MediaType>(compatibleMediaTypes);
	MediaType.sortBySpecificityAndQuality(mediaTypes);

	MediaType selectedMediaType = null;
	for (MediaType mediaType : mediaTypes) {
		if (mediaType.isConcrete()) {
			selectedMediaType = mediaType;
			break;
		}
		else if (mediaType.equals(MediaType.ALL) || mediaType.equals(MEDIA_TYPE_APPLICATION)) {
			selectedMediaType = MediaType.APPLICATION_OCTET_STREAM;
			break;
		}
	}

	if (selectedMediaType != null) {
		selectedMediaType = selectedMediaType.removeQualityValue();
		for (HttpMessageConverter<?> messageConverter : this.messageConverters) {
			if (messageConverter instanceof GenericHttpMessageConverter) {
				if (((GenericHttpMessageConverter<T>) messageConverter).canWrite(returnValueType,
						returnValueClass, selectedMediaType)) {
					returnValue = (T) getAdvice().beforeBodyWrite(returnValue, returnType, selectedMediaType,
							(Class<? extends HttpMessageConverter<?>>) messageConverter.getClass(),
							inputMessage, outputMessage);
					if (returnValue != null) {
						addContentDispositionHeader(inputMessage, outputMessage);
						((GenericHttpMessageConverter<T>) messageConverter).write(returnValue,
								returnValueType, selectedMediaType, outputMessage);
					}
					return;
				}
			}
			else if (messageConverter.canWrite(returnValueClass, selectedMediaType)) {
				returnValue = (T) getAdvice().beforeBodyWrite(returnValue, returnType, selectedMediaType,
						(Class<? extends HttpMessageConverter<?>>) messageConverter.getClass(),
						inputMessage, outputMessage);
				if (returnValue != null) {
					addContentDispositionHeader(inputMessage, outputMessage);
					((HttpMessageConverter<T>) messageConverter).write(returnValue,
							selectedMediaType, outputMessage);
				}
				return;
			}
		}
	}

	if (returnValue != null) {
		throw new HttpMediaTypeNotAcceptableException(this.allSupportedMediaTypes);
	}
}
```
