---
title: Spring源码分析之事务管理
date: 2016-05-08 20:44:33
tags: Spring
---

# Spring源码分析之事务管理 #
## Spring声明式事务处理 ##
### 事务处理基本过程 ###
Spring进行声明式事务处理，主要是通过Ioc容器和Spring的TransactionProxyFactory对事务管理进行配置实现的。

- 首先读取和处理在Ioc容器中配置的事务处理属性，转化为Spring事务处理中的数据结构TransactionAttribute，主要是配置TransactionAttributeSourceAdvisor，通过TransactionAttributeSourceAdvisor通知器来完成对事务处理属性值的处理，读入和转化为TransactionAttribute。
- Spring事务处理模块完成统一的事务处理过程，主要是处理事务配置属性和线程绑定完成事务处理的过程，通过TransactionInfo和TransactionStatus类在事务处理过程中记录和传递相关执行场景。
- Spring通过委托PlatformTransactionManager进行底层的事务操作，具体由子类来实现。

### 事务处理拦截器配置 ###

```xml
<bean id="baseTransactionProxy" class="org.springframework.transaction.interceptor.TransactionProxyFactoryBean"
 abstract="true">
    <property name="transactionManager" ref="transactionManager"/>
    <property name="transactionAttributes">
      <props>
        <prop key="insert">PROPAGATION_REQUIRED</prop>
        <prop key="update">PROPAGATION_REQUIRED</prop>
        <prop key="">PROPAGATION_REQUIRED,readOnly</prop>
      </props>
    </property>
</bean>

<bean id="myProxy" parent="baseTransactionProxy">
    <property name="target" ref="myTarget"/>
</bean>

<bean id="yourProxy" parent="baseTransactionProxy">
    <property name="target" ref="yourTarget"/>
</bean>
```

在进行声明式事务处理的时候，我们需要配置TransactionProxyFactoryBean，需要注入TransactionManager和TransactionAttributes。
在
由于TransactionProxyFactoryBean实现了InitializingBean接口，在容器初始化initializeBean的时候会调用afterPropertiesSet方法，afterPropertiesSet会实例化ProxyFactory，然后为ProxyFactory设置Advisor、调用createMainInterceptor根据PointCut设置通知器DefaultPointcutAdvisor或者TransactionAttributeSourceAdvisor、目标对象，最终返回Proxy代理对象。Proxy代理对象在调用代理方法，会调用相应的TransactionInterceptor拦截器，根据TransactionAttribute配置的事务属性进行配置。
```java
if (this.target instanceof InitializingBean) {
	((InitializingBean) this.target).afterPropertiesSet();
}
```
```java
public void afterPropertiesSet() {
	if (this.target == null) {
		throw new IllegalArgumentException("Property 'target' is required");
	}
	if (this.target instanceof String) {
		throw new IllegalArgumentException("'target' needs to be a bean reference, not a bean name as value");
	}
	if (this.proxyClassLoader == null) {
		this.proxyClassLoader = ClassUtils.getDefaultClassLoader();
	}

	ProxyFactory proxyFactory = new ProxyFactory();

	if (this.preInterceptors != null) {
		for (Object interceptor : this.preInterceptors) {
			proxyFactory.addAdvisor(this.advisorAdapterRegistry.wrap(interceptor));
		}
	}

	// Add the main interceptor (typically an Advisor).
	proxyFactory.addAdvisor(this.advisorAdapterRegistry.wrap(createMainInterceptor()));

	if (this.postInterceptors != null) {
		for (Object interceptor : this.postInterceptors) {
			proxyFactory.addAdvisor(this.advisorAdapterRegistry.wrap(interceptor));
		}
	}

	proxyFactory.copyFrom(this);

	TargetSource targetSource = createTargetSource(this.target);
	proxyFactory.setTargetSource(targetSource);

	if (this.proxyInterfaces != null) {
		proxyFactory.setInterfaces(this.proxyInterfaces);
	}
	else if (!isProxyTargetClass()) {
		// Rely on AOP infrastructure to tell us what interfaces to proxy.
		proxyFactory.setInterfaces(
				ClassUtils.getAllInterfacesForClass(targetSource.getTargetClass(), this.proxyClassLoader));
	}

	postProcessProxyFactory(proxyFactory);

	this.proxy = proxyFactory.getProxy(this.proxyClassLoader);
}
protected Object createMainInterceptor() {
	this.transactionInterceptor.afterPropertiesSet();
	if (this.pointcut != null) {
		return new DefaultPointcutAdvisor(this.pointcut, this.transactionInterceptor);
	}
	else {
		// Rely on default pointcut.
		return new TransactionAttributeSourceAdvisor(this.transactionInterceptor);
	}
}
```
### 事务处理配置读入 ###
主要是通过TransactionAttributeSourceAdvisor对事务属性配置进行读入到TransactionAttributeSource中。
```java
private TransactionInterceptor transactionInterceptor;
private final TransactionAttributeSourcePointcut pointcut = new TransactionAttributeSourcePointcut() {
	@Override
	protected TransactionAttributeSource getTransactionAttributeSource() {
		return (transactionInterceptor != null ? transactionInterceptor.getTransactionAttributeSource() : null);
	}
};
```
在读入事务属性配置中，我们可以根据配置来进行目标对象的方法调用进行拦截，在TransactionAttributeSourcePointcut的matches会对目标方法是不是一个配置好的并且需要进行事务处理的方法调用进行判断。
matchs方法主要是根据方法名在nameMap中查找对应的事务处理属性值，如果不等于null,说明调用方法跟事务方法一致，直接返回TransactionAttribute，如果等于null的话，即是说明没有事务方法直接对应调用方法，可能是调用方法不是事务方法，或者需要进行命名模式进行匹配(通配符)，首先遍历nameMap,对每一个方法名，使用PatternMatchUtils.simpleMatch(mappedName, methodName)方法进行命名模式匹配，获取事务配置属性。nameMap是在TransactionAttributeSource设置的时候构建的。
```java
public boolean matches(Method method, Class<?> targetClass) {
	if (TransactionalProxy.class.isAssignableFrom(targetClass)) {
		return false;
	}
	TransactionAttributeSource tas = getTransactionAttributeSource();
	return (tas == null || tas.getTransactionAttribute(method, targetClass) != null);
}
```
```java
public TransactionAttribute getTransactionAttribute(Method method, Class<?> targetClass) {
	if (!ClassUtils.isUserLevelMethod(method)) {
		return null;
	}
	String methodName = method.getName();
	TransactionAttribute attr = this.nameMap.get(methodName);

	if (attr == null) {
		// Look for most specific name match.
		String bestNameMatch = null;
		for (String mappedName : this.nameMap.keySet()) {
			if (isMatch(methodName, mappedName) &&
					(bestNameMatch == null || bestNameMatch.length() <= mappedName.length())) {
				attr = this.nameMap.get(mappedName);
				bestNameMatch = mappedName;
			}
		}
	}

	return attr;
}
```
```java
protected boolean isMatch(String methodName, String mappedName) {
	return PatternMatchUtils.simpleMatch(mappedName, methodName);
}
```
TransactionAttributeSource是在创建TransactionInterceptor的时候注入的，在TransactionAspectSupport中定义的setTransactionAttributes中设置。NameMatchTransactionAttributeSource是TransactionAttributeSource的具体实现，NameMatchTransactionAttributeSource在对事务属性transactionAttributes进行设置的时候，会从事务属性中获取事务方法名和配置属性，然后把它们作为键值对添加到nameMap中。
```java
public void setTransactionAttributes(Properties transactionAttributes) {
	NameMatchTransactionAttributeSource tas = new NameMatchTransactionAttributeSource();
	tas.setProperties(transactionAttributes);
	this.transactionAttributeSource = tas;
}
```
```java
public void setProperties(Properties transactionAttributes) {
	TransactionAttributeEditor tae = new TransactionAttributeEditor();
	Enumeration<?> propNames = transactionAttributes.propertyNames();
	while (propNames.hasMoreElements()) {
		String methodName = (String) propNames.nextElement();
		String value = transactionAttributes.getProperty(methodName);
		tae.setAsText(value);
		TransactionAttribute attr = (TransactionAttribute) tae.getValue();
		addTransactionalMethod(methodName, attr);
	}
}
```
TransactionAttribute封装了事务处理的配置，对TransactionInterceptor对目标方法添加事务处理作准备。
### 事务处理拦截器的实现 ###
在完成事务属性的配置之后，当我们对目标对象进行方法调用的时候，调用的是一个Proxy代理对象，在对目标对象的方法调用时，不会直接调用TransactionProxyFactoryBean设置的target对象，会被配置的拦截器拦截。
事务处理拦截器TransactionInterceptor中的invoke方法是Proxy代理对象的回调方法，在调用Proxy对象的代理方法时会触发这个回调。
具体的实现是在invokeWithinTransaction中实现的，首先根据methodName和targetClass获取事务配置属性，然后根据配置的事务处理器，PlatformTransactionManager,通过事务处理器来管理事务的创建、挂起、提交、回滚操作，事务的状态通过TransactionInfo进行抽象，根据TransactionInfo来决定是否需要创建新的事务，然后根据拦截器链进行处理，处理完之后对TransactionInfo的信息进行更新，最后进行事务提交。
```java
public Object invoke(final MethodInvocation invocation) throws Throwable {
	// Work out the target class: may be {@code null}.
	// The TransactionAttributeSource should be passed the target class
	// as well as the method, which may be from an interface.
	Class<?> targetClass = (invocation.getThis() != null ? AopUtils.getTargetClass(invocation.getThis()) : null);

	// Adapt to TransactionAspectSupport's invokeWithinTransaction...
	return invokeWithinTransaction(invocation.getMethod(), targetClass, new InvocationCallback() {
		@Override
		public Object proceedWithInvocation() throws Throwable {
			return invocation.proceed();
		}
	});
}
```
```java
protected Object invokeWithinTransaction(Method method, Class<?> targetClass, final InvocationCallback invocation)
		throws Throwable {
		//根据method和targetClass获取TransactionAttribute
	final TransactionAttribute txAttr = getTransactionAttributeSource().getTransactionAttribute(method, targetClass);
	//根据TransactionAttribute获取PlatformTransactionManager
	final PlatformTransactionManager tm = determineTransactionManager(txAttr);
	
	final String joinpointIdentification = methodIdentification(method, targetClass);
    //如果不是CallbackPreferringPlatformTransactionManager就不需要使用回调函数实现事务的创建和视角。
	if (txAttr == null || !(tm instanceof CallbackPreferringPlatformTransactionManager)) {
	//创建事务，同时把创建事务过程中得到的信息方法TransactionInfo中。
		TransactionInfo txInfo = createTransactionIfNecessary(tm, txAttr, joinpointIdentification);
		Object retVal = null;
		try {
		//沿着拦截器链进行调用。
			retVal = invocation.proceedWithInvocation();
		}
		catch (Throwable ex) {
		//如果抛出异常，根据事务配置属性进行回滚或者提交
			completeTransactionAfterThrowing(txInfo, ex);
			throw ex;
		}
		finally {
		//把与线程绑定的TransactionInfo设置为oldTransactionInfo
			cleanupTransactionInfo(txInfo);
		}
		//通过事务处理器对事务进行提交
		commitTransactionAfterReturning(txInfo);
		return retVal;
	}
	else {
		try {
		//通过回调使用 事务处理器
			Object result = ((CallbackPreferringPlatformTransactionManager) tm).execute(txAttr,
					new TransactionCallback<Object>() {
						@Override
						public Object doInTransaction(TransactionStatus status) {
							TransactionInfo txInfo = prepareTransactionInfo(tm, txAttr, joinpointIdentification, status);
							try {
								return invocation.proceedWithInvocation();
							}
							catch (Throwable ex) {
							//异常导致事务回滚
								if (txAttr.rollbackOn(ex)) {
									if (ex instanceof RuntimeException) {
										throw (RuntimeException) ex;
									}
									else {
										throw new ThrowableHolderException(ex);
									}
								}
								else {
									return new ThrowableHolder(ex);
								}
							}
							finally {
							//事务提交
								cleanupTransactionInfo(txInfo);
							}
						}
					});
			if (result instanceof ThrowableHolder) {
				throw ((ThrowableHolder) result).getThrowable();
			}
			else {
				return result;
			}
		}
		catch (ThrowableHolderException ex) {
			throw ex.getCause();
		}
	}
}

```
## Spring声明式事务实现 ##
### 事务创建 ###
事务的创建是在TransactionAspectSupport的createTransactionIfNecessary方法中实现的，首先通过PlatformTransactionManager的getTransaction获取去Transaction事务对象,getTransaction在AbstractPlatformTransactionManager提供了模板，具体由子类来实现，然后返回TransactionStatus来记录当前的事务状态，然后把TransactionStatus设置到TransactionInfo中，最后TransactionInfo与当前线程绑定在一起。
事务的创建主要分为两种，新事务的创建和已有事务下的事务创建。
新事务的创建需要根据配置的事务属性来进行创建，具体的事务对象由具体的事务处理器来创建，然后把事务对象保存在TransactionStatus中，然后对线程的LocalThread变量进行绑定。
已有事务下的事务创建，需要根据事务配置的传播属性对现有事务的处理，在handleExistingTransaction进行调用。

createTransactionIfNecessary
```java
protected TransactionInfo createTransactionIfNecessary(
		PlatformTransactionManager tm, TransactionAttribute txAttr, final String joinpointIdentification) {
	if (txAttr != null && txAttr.getName() == null) {
		txAttr = new DelegatingTransactionAttribute(txAttr) {
			@Override
			public String getName() {
				return joinpointIdentification;
			}
		};
	}
	TransactionStatus status = null;
	if (txAttr != null) {
		if (tm != null) {
			status = tm.getTransaction(txAttr);
		}
		else {
			if (logger.isDebugEnabled()) {
				logger.debug("Skipping transactional joinpoint [" + joinpointIdentification +
						"] because no transaction manager has been configured");
			}
		}
	}
	return prepareTransactionInfo(tm, txAttr, joinpointIdentification, status);
}
```
getTransaction
```java
public final TransactionStatus getTransaction(TransactionDefinition definition) throws TransactionException {
	//具体由子类来实现
	Object transaction = doGetTransaction();
	boolean debugEnabled = logger.isDebugEnabled();
    //如果没有设置事务属性，使用默认的事务属性
	if (definition == null) {
		// Use defaults if no transaction definition given.
		definition = new DefaultTransactionDefinition();
	}
    //检查当前啊线程是否已经存在事务，如果已存在事务，根据事务属性定义的事务传播属性来处理事务的产生。
	if (isExistingTransaction(transaction)) {
		return handleExistingTransaction(definition, transaction, debugEnabled);
	}
    //检查事务配置的timeout是否合理
	if (definition.getTimeout() < TransactionDefinition.TIMEOUT_DEFAULT) {
		throw new InvalidTimeoutException("Invalid transaction timeout", definition.getTimeout());
	}
    //如果没有事务的话，根据事务属性来处理事务的创建
	if (definition.getPropagationBehavior() == TransactionDefinition.PROPAGATION_MANDATORY) {
		throw new IllegalTransactionStateException(
				"No existing transaction found for transaction marked with propagation 'mandatory'");
	}
	else if (definition.getPropagationBehavior() == TransactionDefinition.PROPAGATION_REQUIRED ||
			definition.getPropagationBehavior() == TransactionDefinition.PROPAGATION_REQUIRES_NEW ||
			definition.getPropagationBehavior() == TransactionDefinition.PROPAGATION_NESTED) {
		SuspendedResourcesHolder suspendedResources = suspend(null);
		if (debugEnabled) {
			logger.debug("Creating new transaction with name [" + definition.getName() + "]: " + definition);
		}
		try {
			boolean newSynchronization = (getTransactionSynchronization() != SYNCHRONIZATION_NEVER);
			DefaultTransactionStatus status = newTransactionStatus(
					definition, transaction, true, newSynchronization, debugEnabled, suspendedResources);
			//创建事务的调用，由具体事务处理器来完成。
			doBegin(transaction, definition);
			prepareSynchronization(status, definition);
			return status;
		}
		catch (RuntimeException ex) {
			resume(null, suspendedResources);
			throw ex;
		}
		catch (Error err) {
			resume(null, suspendedResources);
			throw err;
		}
	}
	else {
        //如果TransactionStatus没有Transaction对象，因此prepareTransactionStatus方法中传递transaction的参数为null
		if (definition.getIsolationLevel() != TransactionDefinition.ISOLATION_DEFAULT && logger.isWarnEnabled()) {
			logger.warn("Custom isolation level specified but no actual transaction initiated; " +
					"isolation level will effectively be ignored: " + definition);
		}
		boolean newSynchronization = (getTransactionSynchronization() == SYNCHRONIZATION_ALWAYS);
		return prepareTransactionStatus(definition, null, true, newSynchronization, debugEnabled, null);
	}
}
```
handleExistingTransaction
```java
private TransactionStatus handleExistingTransaction(
	TransactionDefinition definition, Object transaction, boolean debugEnabled)
	throws TransactionException {
    //如果当前线程已有事务存在，当前事务配置的事务属性是TransactionDefinition.PROPAGATION_NEVER，那么就抛出异常
    if (definition.getPropagationBehavior() == TransactionDefinition.PROPAGATION_NEVER) {
    	throw new IllegalTransactionStateException(
    			"Existing transaction found for transaction marked with propagation 'never'");
    }
    //如果配置的是TransactionDefinition.PROPAGATION_NOT_SUPPORTE且已有事务存在，需要把当前事务挂起
    if (definition.getPropagationBehavior() == TransactionDefinition.PROPAGATION_NOT_SUPPORTED) {
    	if (debugEnabled) {
    		logger.debug("Suspending current transaction");
    	}
    	Object suspendedResources = suspend(transaction);
    	boolean newSynchronization = (getTransactionSynchronization() == SYNCHRONIZATION_ALWAYS);
    	//transaction为null,newTransaction为false。
    	return prepareTransactionStatus(
    			definition, null, false, newSynchronization, debugEnabled, suspendedResources);
    }
    //如果TransactionDefinition.PROPAGATION_REQUIRES_NEW的话，将当前事务挂起，创建新的事务调用
    if (definition.getPropagationBehavior() == TransactionDefinition.PROPAGATION_REQUIRES_NEW) {
    	if (debugEnabled) {
    		logger.debug("Suspending current transaction, creating new transaction with name [" +
    				definition.getName() + "]");
    	}
    	SuspendedResourcesHolder suspendedResources = suspend(transaction);
    	try {
    		boolean newSynchronization = (getTransactionSynchronization() != SYNCHRONIZATION_NEVER);
    		//传入的transaction不为null和newTransaction为true
    		DefaultTransactionStatus status = newTransactionStatus(
    				definition, transaction, true, newSynchronization, debugEnabled, suspendedResources);
    		doBegin(transaction, definition);
    		prepareSynchronization(status, definition);
    		return status;
    	}
    	catch (RuntimeException beginEx) {
    		resumeAfterBeginException(transaction, suspendedResources, beginEx);
    		throw beginEx;
    	}
    	catch (Error beginErr) {
    		resumeAfterBeginException(transaction, suspendedResources, beginErr);
    		throw beginErr;
    	}
    }
    //如果是TransactionDefinition.PROPAGATION_NESTED
    if (definition.getPropagationBehavior() == TransactionDefinition.PROPAGATION_NESTED) {
    	if (!isNestedTransactionAllowed()) {
    		throw new NestedTransactionNotSupportedException(
    				"Transaction manager does not allow nested transactions by default - " +
    				"specify 'nestedTransactionAllowed' property with value 'true'");
    	}
    	if (debugEnabled) {
    		logger.debug("Creating nested transaction with name [" + definition.getName() + "]");
    	}
    	//使用savePoint来实现嵌套事务，默认返回true
    	if (useSavepointForNestedTransaction()) {
    	    //传入transaction不为null和newTransaction为false 
    		DefaultTransactionStatus status =
    				prepareTransactionStatus(definition, transaction, false, false, debugEnabled, null);
    		//创建并设置savePoint，为当前connection创建jdbc3.0 savepoint，生成唯一的name
    		status.createAndHoldSavepoint();
    		return status;
    	}
    	else {
    		boolean newSynchronization = (getTransactionSynchronization() != SYNCHRONIZATION_NEVER);
    		DefaultTransactionStatus status = newTransactionStatus(
    				definition, transaction, true, newSynchronization, debugEnabled, null);
    		doBegin(transaction, definition);
    		prepareSynchronization(status, definition);
    		return status;
    	}
}
    if (debugEnabled) {
    	logger.debug("Participating in existing transaction");
    }
    if (isValidateExistingTransaction()) {
    	if (definition.getIsolationLevel() != TransactionDefinition.ISOLATION_DEFAULT) {
    		Integer currentIsolationLevel = TransactionSynchronizationManager.getCurrentTransactionIsolationLevel();
    		if (currentIsolationLevel == null || currentIsolationLevel != definition.getIsolationLevel()) {
    			Constants isoConstants = DefaultTransactionDefinition.constants;
    			throw new IllegalTransactionStateException("Participating transaction with definition [" +
    					definition + "] specifies isolation level which is incompatible with existing transaction: " +
    					(currentIsolationLevel != null ?
    							isoConstants.toCode(currentIsolationLevel, DefaultTransactionDefinition.PREFIX_ISOLATION) :
    							"(unknown)"));
    		}
    	}
    	if (!definition.isReadOnly()) {
    		if (TransactionSynchronizationManager.isCurrentTransactionReadOnly()) {
    			throw new IllegalTransactionStateException("Participating transaction with definition [" +
    					definition + "] is not marked as read-only but existing transaction is");
    		}
    	}
    }
    boolean newSynchronization = (getTransactionSynchronization() != SYNCHRONIZATION_NEVER);
    return prepareTransactionStatus(definition, transaction, false, newSynchronization, debugEnabled, null);
}
```
prepareTransactionInfo
```java
protected TransactionInfo prepareTransactionInfo(PlatformTransactionManager tm,
		TransactionAttribute txAttr, String joinpointIdentification, TransactionStatus status) {

	TransactionInfo txInfo = new TransactionInfo(tm, txAttr, joinpointIdentification);
	if (txAttr != null) {
		// We need a transaction for this method
		if (logger.isTraceEnabled()) {
			logger.trace("Getting transaction for [" + txInfo.getJoinpointIdentification() + "]");
		}
		// The transaction manager will flag an error if an incompatible tx already exists
		txInfo.newTransactionStatus(status);
	}
	else {
		if (logger.isTraceEnabled())
			logger.trace("Don't need to create transaction for [" + joinpointIdentification +
					"]: This method isn't transactional.");
	}
	//将TransactionInfo与线程绑定。
	txInfo.bindToThread();
	return txInfo;
}
```
newTransactionStatus
```java
protected DefaultTransactionStatus newTransactionStatus(
		TransactionDefinition definition, Object transaction, boolean newTransaction,
		boolean newSynchronization, boolean debug, Object suspendedResources) {
    //如果是新事务的话，需要把事务属性放到当前线程中，通过TransactionSynchronizationManager的ThreadLocal执行持有。
	boolean actualNewSynchronization = newSynchronization &&
			!TransactionSynchronizationManager.isSynchronizationActive();
	return new DefaultTransactionStatus(
			transaction, newTransaction, actualNewSynchronization,
			definition.isReadOnly(), debug, suspendedResources);
}
```

### 事务挂起 ###
suspend方法可以对传入的transaction进行挂起，通过委托doSuspend进行实现，具体实现由子类完成。
suspend方法对TransactionSynchronizationManager的事务处理属性进行保存与重置，最后创建SuspendedResourcesHolder返回。
SuspendedResourcesHolder是AbstractPlatformTransactionManager的静态内部类，用来持有被挂起的资源
```java
protected final SuspendedResourcesHolder suspend(Object transaction) throws TransactionException {
    if (TransactionSynchronizationManager.isSynchronizationActive()) {
    	List<TransactionSynchronization> suspendedSynchronizations = doSuspendSynchronization();
    	try {
    		Object suspendedResources = null;
    		if (transaction != null) {
    			suspendedResources = doSuspend(transaction);
    		}
    		//保存和重置ThreadLocal变量
    		String name = TransactionSynchronizationManager.getCurrentTransactionName();
    		TransactionSynchronizationManager.setCurrentTransactionName(null);
    		boolean readOnly = TransactionSynchronizationManager.isCurrentTransactionReadOnly();
    		TransactionSynchronizationManager.setCurrentTransactionReadOnly(false);
    		Integer isolationLevel = TransactionSynchronizationManager.getCurrentTransactionIsolationLevel();
    		TransactionSynchronizationManager.setCurrentTransactionIsolationLevel(null);
    		boolean wasActive = TransactionSynchronizationManager.isActualTransactionActive();
    		TransactionSynchronizationManager.setActualTransactionActive(false);
    		return new SuspendedResourcesHolder(
    				suspendedResources, suspendedSynchronizations, name, readOnly, isolationLevel, wasActive);
    	}
    	catch (RuntimeException ex) {
    		//挂起操作失败，而初始事务还在
    		doResumeSynchronization(suspendedSynchronizations);
    		throw ex;
    	}
    	catch (Error err) {
    		// doSuspend failed - original transaction is still active...
    		doResumeSynchronization(suspendedSynchronizations);
    		throw err;
    	}
    }
    else if (transaction != null) {
    	// Transaction active but no synchronization active.
    	Object suspendedResources = doSuspend(transaction);
    	return new SuspendedResourcesHolder(suspendedResources);
    }
    else {
    	// Neither transaction nor synchronization active.
    	return null;
    }
}
```
### 事务提交 ###
事务提交是commitTransactionAfterReturning为入口方法，它是通过根据传入的TransactionInfo调用事务处理器来完成事务提交的。
commitTransactionAfterReturn
```java
protected void commitTransactionAfterReturning(TransactionInfo txInfo) {
	if (txInfo != null && txInfo.hasTransaction()) {
		if (logger.isTraceEnabled()) {
			logger.trace("Completing transaction for [" + txInfo.getJoinpointIdentification() + "]");
		}
		txInfo.getTransactionManager().commit(txInfo.getTransactionStatus());
	}
}
```
commit
```java
public final void commit(TransactionStatus status) throws TransactionException {
//如果事务已经完成了，则抛出异常
	if (status.isCompleted()) {
		throw new IllegalTransactionStateException(
				"Transaction is already completed - do not call commit or rollback more than once per transaction");
	}
	DefaultTransactionStatus defStatus = (DefaultTransactionStatus) status;
    //如果在事务处理的过程当中，发生了异常，调用回滚
	if (defStatus.isLocalRollbackOnly()) {
		if (defStatus.isDebug()) {
			logger.debug("Transactional code has requested rollback");
		}
		//调用回滚
		processRollback(defStatus);
		return;
	}
	if (!shouldCommitOnGlobalRollbackOnly() && defStatus.isGlobalRollbackOnly()) {
		if (defStatus.isDebug()) {
			logger.debug("Global transaction is marked as rollback-only but transactional code requested commit");
		}
		processRollback(defStatus);
		// Throw UnexpectedRollbackException only at outermost transaction boundary
		// or if explicitly asked to.
		if (status.isNewTransaction() || isFailEarlyOnGlobalRollbackOnly()) {
			throw new UnexpectedRollbackException(
					"Transaction rolled back because it has been marked as rollback-only");
		}
		return;
	}

	processCommit(defStatus);
}
```
processCommit
```java
private void processCommit(DefaultTransactionStatus status) throws TransactionException {
	try {
		boolean beforeCompletionInvoked = false;
		try {
		//抽象方法，由具体子类实现
			prepareForCommit(status);
			triggerBeforeCommit(status);
			triggerBeforeCompletion(status);
			beforeCompletionInvoked = true;
			boolean globalRollbackOnly = false;
			if (status.isNewTransaction() || isFailEarlyOnGlobalRollbackOnly()) {
				globalRollbackOnly = status.isGlobalRollbackOnly();
			}
			//嵌套事务处理
			if (status.hasSavepoint()) {
				if (status.isDebug()) {
					logger.debug("Releasing transaction savepoint");
				}
				status.releaseHeldSavepoint();
			}
			//对当前线程保存的事务状态进行处理，如果是个新事务，就提交事务，由具体子类来实现
			else if (status.isNewTransaction()) {
				if (status.isDebug()) {
					logger.debug("Initiating transaction commit");
				}
				doCommit(status);
			}
			if (globalRollbackOnly) {
				throw new UnexpectedRollbackException(
						"Transaction silently rolled back because it has been marked as rollback-only");
			}
		}
		catch (UnexpectedRollbackException ex) {
			// can only be caused by doCommit
			triggerAfterCompletion(status, TransactionSynchronization.STATUS_ROLLED_BACK);
			throw ex;
		}
		catch (TransactionException ex) {
			// can only be caused by doCommit
			if (isRollbackOnCommitFailure()) {
				doRollbackOnCommitException(status, ex);
			}
			else {
				triggerAfterCompletion(status, TransactionSynchronization.STATUS_UNKNOWN);
			}
			throw ex;
		}
		catch (RuntimeException ex) {
			if (!beforeCompletionInvoked) {
				triggerBeforeCompletion(status);
			}
			doRollbackOnCommitException(status, ex);
			throw ex;
		}
		catch (Error err) {
			if (!beforeCompletionInvoked) {
				triggerBeforeCompletion(status);
			}
			doRollbackOnCommitException(status, err);
			throw err;
		}

		// Trigger afterCommit callbacks, with an exception thrown there
		// propagated to callers but the transaction still considered as committed.
		try {
			triggerAfterCommit(status);
		}
		finally {
			triggerAfterCompletion(status, TransactionSynchronization.STATUS_COMMITTED);
		}

	}
	finally {
		cleanupAfterCompletion(status);
	}
}
```

### 事务回滚 ###
事务回滚是以rollback为入口，委托processRollback进行实现的。
rollback
```java
public final void rollback(TransactionStatus status) throws TransactionException {
	if (status.isCompleted()) {
		throw new IllegalTransactionStateException(
				"Transaction is already completed - do not call commit or rollback more than once per transaction");
	}

	DefaultTransactionStatus defStatus = (DefaultTransactionStatus) status;
	processRollback(defStatus);
}
```
processRollback
```java
private void processRollback(DefaultTransactionStatus status) {
	try {
		try {
		    //触发器调用
			triggerBeforeCompletion(status);
			//如果是嵌套事务
			if (status.hasSavepoint()) {
				if (status.isDebug()) {
					logger.debug("Rolling back transaction to savepoint");
				}
				status.rollbackToHeldSavepoint();
			}
			//对调用方法中新建事务的回滚处理
			else if (status.isNewTransaction()) {
				if (status.isDebug()) {
					logger.debug("Initiating transaction rollback");
				}
				doRollback(status);
			}
			//对事务调用方法中没有新建事务的回滚处理
			else if (status.hasTransaction()) {
				if (status.isLocalRollbackOnly() || isGlobalRollbackOnParticipationFailure()) {
					if (status.isDebug()) {
						logger.debug("Participating transaction failed - marking existing transaction as rollback-only");
					}
					doSetRollbackOnly(status);
				}
				//由线程中的前一个事务来处理回滚
				else {
					if (status.isDebug()) {
						logger.debug("Participating transaction failed - letting transaction originator decide on rollback");
					}
				}
			}
			else {
				logger.debug("Should roll back transaction but cannot - no transaction available");
			}
		}
		catch (RuntimeException ex) {
			triggerAfterCompletion(status, TransactionSynchronization.STATUS_UNKNOWN);
			throw ex;
		}
		catch (Error err) {
			triggerAfterCompletion(status, TransactionSynchronization.STATUS_UNKNOWN);
			throw err;
		}
		triggerAfterCompletion(status, TransactionSynchronization.STATUS_ROLLED_BACK);
	}
	finally {
		cleanupAfterCompletion(status);
	}
}
```
## Spiring事务处理器实现
![TransactionManager](http://7xrl91.com1.z0.glb.clouddn.com/transactionManager.png)

PlatformTransactionManager是事务处理器的接口，提供了getTransaction、commit、rollback等方法，AbstractPlatformTransactionManager是PlatformTransactionManager的基类，为方法的实现提供了模板实现，封装了事务处理中的通用模块，对与具体的事务管理器，只需要处理和设置具体数据源就可以了。
### DataSourceTransactionManager ###
DataSourceTransactionManager是javax.sql.DataSource的事务管理器实现。

- doGetTransaction
```java
protected Object doGetTransaction() {
    //承载当前事务的必要信息
	DataSourceTransactionObject txObject = new DataSourceTransactionObject();
	//当前数据源是否支持嵌套事务、savePoint
	txObject.setSavepointAllowed(isNestedTransactionAllowed());
	ConnectionHolder conHolder =
			(ConnectionHolder) TransactionSynchronizationManager.getResource(this.dataSource);
	txObject.setConnectionHolder(conHolder, false);
	return txObject;
}
```
- doBegin
```java
protected void doBegin(Object transaction, TransactionDefinition definition) {
	DataSourceTransactionObject txObject = (DataSourceTransactionObject) transaction;
	Connection con = null;
	try {
		if (txObject.getConnectionHolder() == null ||
				txObject.getConnectionHolder().isSynchronizedWithTransaction()) {
			//获取线程
			Connection newCon = this.dataSource.getConnection();
			if (logger.isDebugEnabled()) {
				logger.debug("Acquired Connection [" + newCon + "] for JDBC transaction");
			}
			txObject.setConnectionHolder(new ConnectionHolder(newCon), true);
		}
        //设置事务同步
		txObject.getConnectionHolder().setSynchronizedWithTransaction(true);
		con = txObject.getConnectionHolder().getConnection();
        //事务隔离级别
		Integer previousIsolationLevel = DataSourceUtils.prepareConnectionForTransaction(con, definition);
		txObject.setPreviousIsolationLevel(previousIsolationLevel);
		//是否需要重置autoCommit和设置autoCommit为false
		if (con.getAutoCommit()) {
			txObject.setMustRestoreAutoCommit(true);
			if (logger.isDebugEnabled()) {
				logger.debug("Switching JDBC Connection [" + con + "] to manual commit");
			}
			con.setAutoCommit(false);
		}
		txObject.getConnectionHolder().setTransactionActive(true);
        //获取事务配置属性的超时时间
		int timeout = determineTimeout(definition);
		if (timeout != TransactionDefinition.TIMEOUT_DEFAULT) {
			txObject.getConnectionHolder().setTimeoutInSeconds(timeout);
		}
		//是不是新创建的ConnectionHolder
		if (txObject.isNewConnectionHolder()) {
			TransactionSynchronizationManager.bindResource(getDataSource(), txObject.getConnectionHolder());
		}
	}

	catch (Throwable ex) {
		if (txObject.isNewConnectionHolder()) {
			DataSourceUtils.releaseConnection(con, this.dataSource);
			txObject.setConnectionHolder(null, false);
		}
		throw new CannotCreateTransactionException("Could not open JDBC Connection for transaction", ex);
	}
}
```
- 	isExistingTransaction
```java
//判断当前调用方法是否存在事务
protected boolean isExistingTransaction(Object transaction) {
	DataSourceTransactionObject txObject = (DataSourceTransactionObject) transaction;
	return (txObject.getConnectionHolder() != null && txObject.getConnectionHolder().isTransactionActive());
}
```
- doSuspend
```java
//挂起事务，解绑对应事务资源
protected Object doSuspend(Object transaction) {
	DataSourceTransactionObject txObject = (DataSourceTransactionObject) transaction;
	txObject.setConnectionHolder(null);
	ConnectionHolder conHolder = (ConnectionHolder)
			TransactionSynchronizationManager.unbindResource(this.dataSource);
	return conHolder;
}
```
- doResume
```java 
//重新开始，恢复挂起的事务，绑定资源
protected void doResume(Object transaction, Object suspendedResources) {
	ConnectionHolder conHolder = (ConnectionHolder) suspendedResources;
	TransactionSynchronizationManager.bindResource(this.dataSource, conHolder);
}
```
- doCommit
```java
@Override
//事务提交
protected void doCommit(DefaultTransactionStatus status) {
	DataSourceTransactionObject txObject = (DataSourceTransactionObject) status.getTransaction();
	Connection con = txObject.getConnectionHolder().getConnection();
	if (status.isDebug()) {
		logger.debug("Committing JDBC transaction on Connection [" + con + "]");
	}
	try {
		con.commit();
	}
	catch (SQLException ex) {
		throw new TransactionSystemException("Could not commit JDBC transaction", ex);
	}
}
```
- doRollback
```java
@Override
//事务回滚
protected void doRollback(DefaultTransactionStatus status) {
	DataSourceTransactionObject txObject = (DataSourceTransactionObject) status.getTransaction();
	Connection con = txObject.getConnectionHolder().getConnection();
	if (status.isDebug()) {
		logger.debug("Rolling back JDBC transaction on Connection [" + con + "]");
	}
	try {
		con.rollback();
	}
	catch (SQLException ex) {
		throw new TransactionSystemException("Could not roll back JDBC transaction", ex);
	}
}
```
- doSetRollbackOnly
```java
@Override
protected void doSetRollbackOnly(DefaultTransactionStatus status) {
	DataSourceTransactionObject txObject = (DataSourceTransactionObject) status.getTransaction();
	if (status.isDebug()) {
		logger.debug("Setting JDBC transaction [" + txObject.getConnectionHolder().getConnection() +
				"] rollback-only");
	}
	txObject.setRollbackOnly();
}
```
- doCleanupAfterCompletion
```java
@Override
//重置事务信息和释放资源
protected void doCleanupAfterCompletion(Object transaction) {
	DataSourceTransactionObject txObject = (DataSourceTransactionObject) transaction;

	// Remove the connection holder from the thread, if exposed.
	if (txObject.isNewConnectionHolder()) {
		TransactionSynchronizationManager.unbindResource(this.dataSource);
	}

	// Reset connection.
	Connection con = txObject.getConnectionHolder().getConnection();
	try {
		if (txObject.isMustRestoreAutoCommit()) {
			con.setAutoCommit(true);
		}
		DataSourceUtils.resetConnectionAfterTransaction(con, txObject.getPreviousIsolationLevel());
	}
	catch (Throwable ex) {
		logger.debug("Could not reset JDBC Connection after transaction", ex);
	}

	if (txObject.isNewConnectionHolder()) {
		if (logger.isDebugEnabled()) {
			logger.debug("Releasing JDBC Connection [" + con + "] after transaction");
		}
		DataSourceUtils.releaseConnection(con, this.dataSource);
	}

	txObject.getConnectionHolder().clear();
}
```