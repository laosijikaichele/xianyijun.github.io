---
title: Spring Aop 源码分析
date: 2016-04-29 17:11:22
tags: Spring
---

---

# Spring Aop 源码分析 #

---

## **AOP概述**

### **什么是AOP**
    
AOP是面向切面编程的简称。可以把aop看成是一种新的模块化机制，它主要是用来对一些分散在类、对象、方法的公共行为进行分离。将公共行为与业务逻辑进行分离，独立出来进行管理，业务逻辑中不再有针对特定行为的方法调用，例如：事务管理，日志记录等。业务逻辑跟公共行为可以通过切面来封装和维护，把aop看成是对目标的对象的环绕增强。

- Aspect切面
切面是指对待增强的目标对象的增强应用。它是切点和通知的结合。定义了在什么时候，在什么地方去完成怎么样的功能。 
- Advice通知
在AOP中，advice是指环绕切面增强的而注入的切面行为。它定义了切面是什么和什么时候执行。
Spring aspect提高了五种类型的advice：
    - BeforeAdvice ：beforeAdvice出现在被通知的方法执行之前。
    - AfterAdvice：afterAdvice出现在被通知的方法执行完成之后。（无论是正常完成还是异常退出都会执行）
    - After-returning ：After-returning出现在被通知的方法正常执行完成之后。
    - After-throwing ：After-throwing出现在被通知的方法执行期间异常退出之后。
    - AroundAdvice：AroundAdvice环绕被通知的方法。提供了在被通知方法执行前后完成切面行为。

- JoinPoint连接点
在应用程序执行的过程中，能够被插入切面的点，例如方法调用，字段的设置及读写，这些点被称作JoinPoint连接点，在Spring中，只支持方法层次的JoinPoint。你可以在这些JointPoint执行的时候织入新的代码，为应用程序添加新的行为。
- PointCut切点
Point定义了切面的where，它会匹配advice要织入的一个或者多个连接点。一般使用类的全限定名和方法来指定切点或者通过正则表达式来模式匹配切点。
- Advisor通知器
Advisor把Advisor和PointCut结合起来，可以定义应用程序使用哪个advice并在哪个PointCut使用它。
        
### **动态代理**
    
Spring AOP的实现中，使用的核心技术是动态代理。
    
#### **代理模式** 
在代理模式中，通常会设计一个真实对象既目标对象RealSubject和一个接口与目标对象一致的代理对象Proxy。实现了与接口相同的方法。当对目标对象进行方法调用的时候，会被代理对象拦截，改变其运行轨迹。在客户调用Proxy代理对象的方法的时候，会在目标对象执行方法的前后调用一系列的处理，再执行目标对象的方法。这系列的处理对于目标对象来说都是透明的，目标对象对这些处理毫不知情。
   
接口类

```java
public interface Subject {
   public void request();
}
```
目标对象类

```java
public class RealSubject implements Subject {
	@Override
	public void request() {
		System.out.println("this is the real subject");
	}
}
```
静态代理类
```java
public class SubjectProxy implements Subject {
    private Subject target;
    
    public SubjectProxy(Subject subject) {
      this.target = subject;
    }

    public void request() {
        before();
        target.request();
        after();
    }
    
    public void before() {
        System.out.println("this is before the request");
    }
    
    public void after() {
        System.out.println("this is after the request");
    }
}
```

JDK动态代理类

```java
public class SubjectProxyHandler implements InvocationHandler {
    private Object target;
    public Object bind(Object target) {
        this.target = target;
        return Proxy.newProxyInstance(target.getClass().getClassLoader(), target.getClass().getInterfaces(), this);
    }
    
    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        before();
        Object result = method.invoke(target, args);
        after();
        return result;
    }
    
    private void before() {
        System.out.println("this is the jdk proxy before");
    }
    
    private void after() {
        System.out.println("this is the jdk proxy after");
    }
}
```
客户端
```java
public class ProxyClient {
	public static void main(String[] args) {
		Subject subject = new RealSubject();
		SubjectProxy proxy = new SubjectProxy(subject);
		ProxyClient client = new ProxyClient();
		client.invoke(proxy);
		System.out.println("=============================");
		client.invoke(subject);
		System.out.println("===========================");
		SubjectProxyHandler proxyHandler =new SubjectProxyHandler();
		Subject subjectProxy =(Subject) proxyHandler.bind(subject);
		client.invoke(subjectProxy);
	}
	private void invoke(Subject subject) {
		subject.request();
	}
}
```
## Spring AOP实现 ##

在Spring AOP模块中，主要是代理对象的生产和配置拦截器的处理

### **建立AopProxy代理实现** ####
![aopproxy](http://7xrl91.com1.z0.glb.clouddn.com/ProxyFactory.png)

Spring通过配置ProxyFactoryBean来完成代理对象的生成。主要有两种生成策略：Jdk的Proxy和GCLIB的两种生成方式。
在ProxyFactory的类继承关系中，ProxyConfig是一个数据基类，主要是为了ProxyFactoryBean、ProxyFactory、AspectJProxyFactory这些子类提供可配置的属性，如proxyTargetClass、optimize、opaque、exposeProxy、frozen等属性。确保所有的子类都有这些属性。AdvisedSupport是一个AOP代理配置manager的基类，它主要封装了AOP对Advice和Advisor的相关操作，这些操作对于所有的代理对象效果都一样的，然而在这个类中却没有具体的AOP代理对象的创建的方法，而是交给它的子类去完成。对于不同的应用，具体的AOP代理对象的生成，由ProxyFactoryBean、AsepectJProxyFactory、ProxyFactory来完成。

#### **配置ProxyFactoryBean** ####

```java
<bean id="helloAdvisor" class="me.kuye.helloAdvisor"/>
<bean id="helloAdviceAop" class="org.springframework.aop.framework.ProxyFactoryBean">
  <property name="proxyInterfaces">
    <value>me.kuye.helloInterface</value>
  </property>
  <property name="target">
    <bean class="me.kuye.helloTarget"/>
  </property>
  <property name="interceptorNames">
    <list>
      <value>helloAdvice</value>
    </list>
  </property>
</bean>
```
- 定义通知器Advisor
- 定义ProxyFactoryBean，配置AOP实现的相关属性，例如：proxyInterfaces、target、interceptorNames等；interceptorsNames设置要定义的通知器，target属性定义AOP通知器中的切面用来增强的目标对象。

#### **ProxyFactoryBean生成AopProxy代理对象** ####
proxyFactoryBean通过getObject()方法来获取对象。getObject需要对target目标对象的增强处理进行封装，为AOP功能的实现提供服务。getObject方法首先调用initializeAdvisorChain方法对通知器链进行初始化，通知器链里面封装了一系列定义的拦截器，拦截器从interceptorNames属性配置中获取，为代理对象的生成作准备，还需要根据singleton和prototype的类型bean类型进行区分。

```java
public Object getObject() throws BeansException {
	initializeAdvisorChain();
	if (isSingleton()) {
		return getSingletonInstance();
	}
	else {
		return newPrototypeInstance();
	}
}
```
在initializeAdvisorChain方法中，根据advisorChainInitialized标志位来判断是否已经初始化了，如果已经初始化过了，就立即返回，否则就继续进行初始化操作，也就是说初始化操作是在第一次通过ProxyFactoryBean去获取代理对象。然后遍历interceptorNames，根据getBean返回对应的通知器，通过addAdvisorOnChainCreation方法对Ioc进行回调加入到拦截器链中。去除异常处理和日志管理下的源码如下。

```java   
private synchronized void initializeAdvisorChain() throws AopConfigException, BeansException {
	if (this.advisorChainInitialized) {
		return;
	}

	if (!ObjectUtils.isEmpty(this.interceptorNames)) {
		for (String name : this.interceptorNames) {
			if (name.endsWith(GLOBAL_SUFFIX)) {
				addGlobalAdvisor((ListableBeanFactory) this.beanFactory,
						name.substring(0, name.length() - GLOBAL_SUFFIX.length()));
			}
			else {
				Object advice;
				if (this.singleton || this.beanFactory.isSingleton(name)) {
					advice = this.beanFactory.getBean(name);
				}
				else {
					advice = new PrototypePlaceholderAdvisor(name);
				}
				addAdvisorOnChainCreation(advice, name);
			}
		}
	}
	this.advisorChainInitialized = true;
}
```
##### **生成Singleton的代理对象** #####
通过getSingletonInstance方法来获取代理对象，代理对象会对target目标对象的调用进行封装，既会对目标对象的方法调用时会被代理对象所拦截。通过createAopProxy方法来获取AopProxy来具体生成代理对象，Spring通过aopProxy将代理对象的生成与其他部分进行分离，AopProxy有两个子类实现：ObjenesisCglibAopProxy和JdkDynamicAopProxy，既通过GCLIB和JDK来生成具体的Proxy代理对象。
createAopProxy是通过基类的AdvisedSupport中的AopProxyFactory来完成的，利用了AdvisedSupport中配置信息来生成具体的代理对象。AopProxyFactory是在ProxyCreatorSupport中创建的，设置为DefaultAopProxyFactory。
对于AopProxy代理对象的生成，是目标对象的类型来决定的。如果目标对象是接口类，那么就使用JDK来生成代理对象，否则就会使用GCLIB来生成目标对象的代理对象。DefaultAopProxyFactory是AopProxy的生产工厂，根据不同的需要来生成这两种AopProxy对象。DefaultAopProxyFactory中并没有具体的生成代理对象，具体的实现层次交由它的子类去实现。

```java
public AopProxy createAopProxy(AdvisedSupport config) throws AopConfigException {
    if (config.isOptimize() || config.isProxyTargetClass() || hasNoUserSuppliedProxyInterfaces(config)) {
	    Class<?> targetClass = config.getTargetClass();

    	if (targetClass.isInterface()) {
		    return new JdkDynamicAopProxy(config);
    	}
    	return new ObjenesisCglibAopProxy(config);
	}
    else {
    	return new JdkDynamicAopProxy(config);
	}
}
``` 
	
##### **生成prototype的代理对象** #####

```java        
private synchronized Object newPrototypeInstance() {
	ProxyCreatorSupport copy = new ProxyCreatorSupport(getAopProxyFactory());
	TargetSource targetSource = freshTargetSource();
	copy.copyConfigurationFrom(this, targetSource, freshAdvisorChain());
	if (this.autodetectInterfaces && getProxiedInterfaces().length == 0 && !isProxyTargetClass()) {
		copy.setInterfaces(
				ClassUtils.getAllInterfacesForClass(targetSource.getTargetClass(), this.proxyClassLoader));
	}
	copy.setFrozen(this.freezeProxy);
	return getProxy(copy.createAopProxy());
}
```        

- Jdk生成代理对象
JdkDynamicAopProxy通过JDK的Proxy来生成代理对象。在生成代理对象之前，首先根据advised成员属性来获取代理对象的代理接口配置，然后使用Proxy.newProxyInstance来获取Proxy代理对象，需要指定目标对象的类加载器，代理接口，以及Proxy回调方法的对象既实现InvocationHandler接口的类本身。实现InvocationHandler接口的类需要实现invoke方法，invoke是Proxy代理对象的回调方法，是AOP编织实现的封装。

```java    
public Object getProxy(ClassLoader classLoader) {

	Class<?>[] proxiedInterfaces = AopProxyUtils.completeProxiedInterfaces(this.advised);
	findDefinedEqualsAndHashCodeMethods(proxiedInterfaces);
	return Proxy.newProxyInstance(classLoader, proxiedInterfaces, this);
	
}
```
- Gclib生成代理对象
ObjenesisCglibAopProxy是CglibAopProxy的子类，Proxy代理对象的生成是在CglibAopProxy的getProxy方法中完成的，在getProxy中主要是Enhancer对象的生成和完成对EnHancer对象的配置。EnHancer生成代理对象是通过对EnHancer自身的CallBack回调的设置，完成了AOP的封装。对EnHancer的CallBack设置是在getCallbacks方法中，通过DynamicAdvisedInterceptor设置完成的。拦截入口是DynamicAdvisedInterceptor的intercept方法。Callback aopInterceptor = new DynamicAdvisedInterceptor(this.advised);

```java    
public Object getProxy(ClassLoader classLoader) {
	Class<?> rootClass = this.advised.getTargetClass();
	Assert.state(rootClass != null, "Target class must be available for creating a CGLIB proxy");

	Class<?> proxySuperClass = rootClass;
	if (ClassUtils.isCglibProxyClass(rootClass)) {
		proxySuperClass = rootClass.getSuperclass();
		Class<?>[] additionalInterfaces = rootClass.getInterfaces();
		for (Class<?> additionalInterface : additionalInterfaces) {
			this.advised.addInterface(additionalInterface);
		}
	}

	validateClassIfNecessary(proxySuperClass);

	Enhancer enhancer = createEnhancer();
	if (classLoader != null) {
		enhancer.setClassLoader(classLoader);
		if (classLoader instanceof SmartClassLoader &&
				((SmartClassLoader) classLoader).isClassReloadable(proxySuperClass)) {
			enhancer.setUseCache(false);
		}
	}
	enhancer.setSuperclass(proxySuperClass);
	enhancer.setInterfaces(AopProxyUtils.completeProxiedInterfaces(this.advised));
	enhancer.setNamingPolicy(SpringNamingPolicy.INSTANCE);
	enhancer.setStrategy(new UndeclaredThrowableStrategy(UndeclaredThrowableException.class));
	Callback[] callbacks = getCallbacks(rootClass);
	Class<?>[] types = new Class<?>[callbacks.length];
	for (int x = 0; x < types.length; x++) {
		types[x] = callbacks[x].getClass();
	}
	enhancer.setCallbackFilter(new ProxyCallbackFilter(
			this.advised.getConfigurationOnlyCopy(), this.fixedInterceptorMap, this.fixedInterceptorOffset));
	enhancer.setCallbackTypes(types);
	return createProxyClassAndInstance(enhancer, callbacks);
}
```    
在通过AopProxy完成了对target目标对象的封装之后，使用ProxyFactoryBean的getObject方法获取的不再是原先的target对象，而是AopProxy的代理对象，在对原target对象的方法调用时，会被AopProxy代理对象所拦截，通过拦截器回调进行环绕增强。
    
### Spring AOP拦截器调用实现 ###

#### **JdkDynamicAopProxy的invoke拦截** ####
invoke是JdkDynamicAopProxy对代理对象的拦截的入口方法。当Proxy代理对象的代理方法被调用的时候，invoke方法会被调用，完成对目标对象方法的增强。invoke方法首先对目标对象有没有实现hashcode和equals进行判断，来返回AopProxy的对应方法。然后调用advised的getInterceptorsAndDynamicInterceptionAdvice方法获取拦截器链，来创建ReflectiveMethodInvocation对象调用proceed来完成拦截过程，它逐个运行拦截器链中的拦截器进行拦截增强，最后执行目标对象的方法。

```java
public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
	MethodInvocation invocation;
	Object oldProxy = null;
	boolean setProxyContext = false;

	TargetSource targetSource = this.advised.targetSource;
	Class<?> targetClass = null;
	Object target = null;

	try {
		if (!this.equalsDefined && AopUtils.isEqualsMethod(method)) {
			return equals(args[0]);
		}
		if (!this.hashCodeDefined && AopUtils.isHashCodeMethod(method)) {
			return hashCode();
		}
		if (!this.advised.opaque && method.getDeclaringClass().isInterface() &&
				method.getDeclaringClass().isAssignableFrom(Advised.class)) {
			return AopUtils.invokeJoinpointUsingReflection(this.advised, method, args);
		}

		Object retVal;

		if (this.advised.exposeProxy) {
			oldProxy = AopContext.setCurrentProxy(proxy);
			setProxyContext = true;
		}

		target = targetSource.getTarget();
		if (target != null) {
			targetClass = target.getClass();
		}
		List<Object> chain = this.advised.getInterceptorsAndDynamicInterceptionAdvice(method, targetClass);
		if (chain.isEmpty()) {
			retVal = AopUtils.invokeJoinpointUsingReflection(target, method, args);
		}
		else {
			invocation = new ReflectiveMethodInvocation(proxy, target, method, args, targetClass, chain);
			retVal = invocation.proceed();
		}

		Class<?> returnType = method.getReturnType();
		if (retVal != null && retVal == target && returnType.isInstance(proxy) &&
				!RawTargetAccess.class.isAssignableFrom(method.getDeclaringClass())) {
			retVal = proxy;
		} else if (retVal == null && returnType != Void.TYPE && returnType.isPrimitive()) {
		}
		return retVal;
	}finally {
		if (target != null && !targetSource.isStatic()) {
			targetSource.releaseTarget(target);
		}
		if (setProxyContext) {
			AopContext.setCurrentProxy(oldProxy);
		}
	}
}
```    	


#### **CglibAopProxy的intercept拦截** ####
CglibAopProxy的调用拦截是在DynamicAdvisedInterceptor的intercept方法中实现的。它是通过构建CglibMethodInvocation对象执行proceed方法来完成拦截器链的调用的。与JdkDynamicAopProxy的invoke拦截类似。

```java
public Object intercept(Object proxy, Method method, Object[] args, MethodProxy methodProxy)throws Throwable {
	Object oldProxy = null;
	boolean setProxyContext = false;
	Class<?> targetClass = null;
	Object target = null;
	try {
		if (this.advised.exposeProxy) {
			// Make invocation available if necessary.
			oldProxy = AopContext.setCurrentProxy(proxy);
			setProxyContext = true;
		}
		target = getTarget();
		if (target != null) {
			targetClass = target.getClass();
		}
		List<Object> chain = this.advised.getInterceptorsAndDynamicInterceptionAdvice(method, targetClass);
		Object retVal;
		if (chain.isEmpty() && Modifier.isPublic(method.getModifiers())) {
			retVal = methodProxy.invoke(target, args);
		}
		else {
			retVal = new CglibMethodInvocation(proxy, target, method, args, targetClass, chain, methodProxy).proceed();
		}
		retVal = processReturnType(proxy, target, method, retVal);
		return retVal;
	}
	finally {
		if (target != null) {
			releaseTarget(target);
		}
		if (setProxyContext) {
			AopContext.setCurrentProxy(oldProxy);
		}
	}
}
```

#### **目标对象方法调用** ####
如果目标对象的方法没有设置拦截器，就会对方法进行直接调用。在JdkDynamicAopProxy中是通过AopUtils.invokeJoinpointUsingReflection(this.advised, method, args)实现的，是通过jdk反射来实现的，首先获取要执行方法的Method对象，然后传入目标对象和方法参数作为invoke方法参数进行调用。method.invoke(target,args)。
在CglibAopProxy中，则是通过调用methodProxy的invoke来执行目标对象的方法调用的。

#### **AOP拦截器链调用** ####

在JdkDynamicAopProxy和CglibAopProxy对拦截器的调用都是通过构建ReflectiveMethodInvocation对象调用proceed方法实现的。在proceed方法中，会逐个调用拦截器的拦截方法，不过在执行拦截器方法之前，首先通过PointCut切点的matches方法对代理方法进行匹配，如果匹配，就会拦截器中获取通知器，并调用通知器的invoke方法进行切面增强。然后继续迭代执行proceed方法，直到执行到拦截器链末尾，才会去执行目标对象的实现方法。

```java
public Object proceed() throws Throwable {
	if (this.currentInterceptorIndex == this.interceptorsAndDynamicMethodMatchers.size() - 1) {
		return invokeJoinpoint();
	}

	Object interceptorOrInterceptionAdvice =
			this.interceptorsAndDynamicMethodMatchers.get(++this.currentInterceptorIndex);
	if (interceptorOrInterceptionAdvice instanceof InterceptorAndDynamicMethodMatcher) {
		InterceptorAndDynamicMethodMatcher dm =
				(InterceptorAndDynamicMethodMatcher) interceptorOrInterceptionAdvice;
		if (dm.methodMatcher.matches(this.method, this.targetClass, this.arguments)) {
			return dm.interceptor.invoke(this);
		}
		else {
			return proceed();
		}
	}
	else {
		return ((MethodInterceptor) interceptorOrInterceptionAdvice).invoke(this);
	}
}
```
#### **配置通知器** ####

在拦截器链调用的过程中，我们可以看到interceptorOrInterceptionAdvice是通过ReflectiveMethodInvocation类中持有的interceptorsAndDynamicMethodMatchers的list集合获取。而interceptorsAndDynamicMethodMatchers是在构建interceptorsAndDynamicMethodMatchers实例的时候通过传入参数来赋于的。我们通过查看JdkDynamicAopProxy的invoke方法中发现，chain是通过调用this.advised.getInterceptorsAndDynamicInterceptionAdvice方法来获取拦截器链。advised是AdvisedSupport的实例，在AdvisedSupport中的getInterceptorsAndDynamicInterceptionAdvice方法中，拦截器链是通过调用advisorChainFactory的getInterceptorsAndDynamicInterceptionAdvice方法来获取的，为了提高获取拦截器链的速率，还添加了缓存。advisorChainFactory可以看出是一个获取advisorChain的工厂类。DefaultAdvisorChainFactory是advisorChainFactory的默认实现。在DefaultAdvisorChainFactory中通过AdvisorAdapterRegistry来完成拦截器的注册。AdvisorAdapterRegistry根据之前在ProxyFactoryBean中的配置来获取对应的拦截器，然后添加到List中，完成注册。

```java
public List<Object> getInterceptorsAndDynamicInterceptionAdvice(Method method, Class<?> targetClass) {
	MethodCacheKey cacheKey = new MethodCacheKey(method);
	List<Object> cached = this.methodCache.get(cacheKey);
	if (cached == null) {
		cached = this.advisorChainFactory.getInterceptorsAndDynamicInterceptionAdvice(
				this, method, targetClass);
		this.methodCache.put(cacheKey, cached);
	}
	return cached;
}
```   	
  

#### **Advice通知的实现** ####

Advice拦截器获取的生成是通过DefaultAdvisorChainFactory类实现的。在工厂类的getInterceptorsAndDynamicInterceptionAdvice的方法中，完成了拦截器的适配与注册。首先，构建了一个GlobalAdvisorAdapterRegistry的单例对象，然后通过传入参数的Advised中去获取在ProxyFactoryBean中配置的interceptorNames来获取Advisor通知器，接着遍历Advisor通知器，通过GlobalAdvisorAdapterRegistry的getInterceptors方法获取拦截器来完成对拦截器的适配和注册过程。在getInterceptors方法中，完成了对拦截器的适配，如果是MethodInterceptor类型的advice，就直接添加到interceptors中，如果是MethodBeforeAdvice、AfterReturningAdvice、throwsAdvice等类型的advice，就得通过在构建DefaultAdvisorAdapterRegistry对象中注册的adapter对advisor进行适配，在添加到Interceptors中。AdvisorAdapter接口定义supportsAdvice和getInterceptor两种方法。supportsAdvice对传入的advice的类型进行判断，返回true或者false；getInterceptor主要是将对应的advice进行包装，返回对应的MethodInterceptor。Spring提供了三种类型的AdvisorAdapter：AfterReturningAdviceAdapter、MethodBeforeAdviceAdapter、ThrowsAdviceAdapter。在MethodInterceptor中，对advice就行了封装，根据不同的advice，在ReflectiveMethodInvocation触发拦截器的invoke方法的时候，触发不同的advice拦截器封装，对目标对象进行增强。

```java
public List<Object> getInterceptorsAndDynamicInterceptionAdvice(
	Advised config, Method method, Class<?> targetClass) {
	List<Object> interceptorList = new ArrayList<Object>(config.getAdvisors().length);
	boolean hasIntroductions = hasMatchingIntroductions(config, targetClass);
	AdvisorAdapterRegistry registry = GlobalAdvisorAdapterRegistry.getInstance();
	for (Advisor advisor : config.getAdvisors()) {
		if (advisor instanceof PointcutAdvisor) {
			PointcutAdvisor pointcutAdvisor = (PointcutAdvisor) advisor;
			if (config.isPreFiltered() || pointcutAdvisor.getPointcut().getClassFilter().matches(targetClass)) {
				MethodInterceptor[] interceptors = registry.getInterceptors(advisor);
				MethodMatcher mm = pointcutAdvisor.getPointcut().getMethodMatcher();
				if (MethodMatchers.matches(mm, method, targetClass, hasIntroductions)) {
					if (mm.isRuntime()) {
						for (MethodInterceptor interceptor : interceptors) {
							interceptorList.add(new InterceptorAndDynamicMethodMatcher(interceptor, mm));
						}
					}
					else {
						interceptorList.addAll(Arrays.asList(interceptors));
					}
				}
			}
		}
		else if (advisor instanceof IntroductionAdvisor) {
			IntroductionAdvisor ia = (IntroductionAdvisor) advisor;
			if (config.isPreFiltered() || ia.getClassFilter().matches(targetClass)) {
				Interceptor[] interceptors = registry.getInterceptors(advisor);
				interceptorList.addAll(Arrays.asList(interceptors));
			}
		}
		else {
			Interceptor[] interceptors = registry.getInterceptors(advisor);
			interceptorList.addAll(Arrays.asList(interceptors));
		}
	}
	return interceptorList;
}

public MethodInterceptor[] getInterceptors(Advisor advisor) throws UnknownAdviceTypeException {
    List<MethodInterceptor> interceptors = new ArrayList<MethodInterceptor>(3);
    Advice advice = advisor.getAdvice();
    if (advice instanceof MethodInterceptor) {
    	interceptors.add((MethodInterceptor) advice);
    }
    for (AdvisorAdapter adapter : this.adapters) {
    	if (adapter.supportsAdvice(advice)) {
    		interceptors.add(adapter.getInterceptor(advisor));
    	}
    }
    if (interceptors.isEmpty()) {
    	throw new UnknownAdviceTypeException(advisor.getAdvice());
    }
    return interceptors.toArray(new MethodInterceptor[interceptors.size()]);
}
```	
    	
#### **ProxyFactory实现AOP** ####
ProxyFactory实现AOP是以getProxy为入口，获取AopProxy代理对象，通过调用基类ProxyCreatorSupport的createProxy方法来生成AopProxy代理对象，其实的就跟ProxyFactoryBean的差不多了。