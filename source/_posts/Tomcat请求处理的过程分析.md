---
title: Tomcat请求处理的过程分析
date: 2016-05-11 20:51:51
tags: Tomcat
---
[toc]
# **Tomcat请求处理的过程分析** #
## **Tomcat启动** ##
Tomcat启动是以Bootstrap的main方法为入口类。
在加载Boostrap的时候，首先会执行Bootstrap的内部静态代码块，会对userDir、homeFile、baseFile属性进行初始化与设置。
main方法首先daemon进行非空判断，daemon是Bootstrap类型的成员变量，初始值为null，所有会创建一个Bootstrap对象，调用init方法，完成之后设置为daemon。
```java
public static void main(String args[]) {
    if (daemon == null) {
        // Don't set daemon until init() has completed
        Bootstrap bootstrap = new Bootstrap();
        try {
            bootstrap.init();
        } catch (Throwable t) {
            handleThrowable(t);
            t.printStackTrace();
            return;
        }
        daemon = bootstrap;
    } else {
        Thread.currentThread().setContextClassLoader(daemon.catalinaLoader);
    }
    try {
        String command = "start";
        if (args.length > 0) {
            command = args[args.length - 1];
        }
        if (command.equals("startd")) {
            args[args.length - 1] = "start";
            daemon.load(args);
            daemon.start();
        } else if (command.equals("stopd")) {
            args[args.length - 1] = "stop";
            daemon.stop();
        } else if (command.equals("start")) {
            daemon.setAwait(true);
            daemon.load(args);
            daemon.start();
        } else if (command.equals("stop")) {
            daemon.stopServer(args);
        } else if (command.equals("configtest")) {
            daemon.load(args);
            if (null==daemon.getServer()) {
                System.exit(1);
            }
            System.exit(0);
        } else {
            log.warn("Bootstrap: command \"" + command + "\" does not exist.");
        }
    } catch (Throwable t) {
        if (t instanceof InvocationTargetException &&
                t.getCause() != null) {
            t = t.getCause();
        }
        handleThrowable(t);
        t.printStackTrace();
        System.exit(1);
    }
}
```
## **Tomcat初始化** ##
在init方法中设置catalina.base和catalina.home系统属性进行设置，然后创建和初始化commonClassLoader、catalinaClassLoader和shareClassLoader类加载器，然后通过反射创建一个org.apache.catalina.startup.Catalina类型的实例，设置为catalinaDomain，然后调用setParentClassLoader方法将shareClassLoader作为Catalina的父类加载器。
```
 public void init() throws Exception {
    initClassLoaders();
    Thread.currentThread().setContextClassLoader(catalinaLoader);
    SecurityClassLoad.securityClassLoad(catalinaLoader);
    // Load our startup class and call its process() method
    if (log.isDebugEnabled())
        log.debug("Loading startup class");
    Class<?> startupClass =
        catalinaLoader.loadClass
        ("org.apache.catalina.startup.Catalina");
    Object startupInstance = startupClass.newInstance();
    if (log.isDebugEnabled())
        log.debug("Setting startup class properties");
    String methodName = "setParentClassLoader";
    Class<?> paramTypes[] = new Class[1];
    paramTypes[0] = Class.forName("java.lang.ClassLoader");
    Object paramValues[] = new Object[1];
    paramValues[0] = sharedLoader;
    Method method =startupInstance.getClass().getMethod(methodName, paramTypes);
    method.invoke(startupInstance, paramValues);
    catalinaDaemon = startupInstance;
}
```

### **解析web.xml和初始化Server** ###
然后根据用户传入的参数进行判断，执行对应方法，默认为start。
我们启动tomcat的话，执行的分支肯定是start。
首先调用Bootstrap的load方法，load主要是通过反射去调用之前设置catalinaDaemon的load方法和start方法。
Catalina的load方法主要是创建一个Digester对tomcat的server.xml配置文件进行解析和对相关catainer容器对象进行创建，
然后对Server的部分属性进行设置和调用Server的init方法进行调用。Server的默认实现为StandardServer。

```java
 public void load() {
    long t1 = System.nanoTime();
    initDirs();
    initNaming();
    Digester digester = createStartDigester();
    InputSource inputSource = null;
    InputStream inputStream = null;
    File file = null;
    try {
        try {
            file = configFile();
            inputStream = new FileInputStream(file);
            inputSource = new InputSource(file.toURI().toURL().toString());
        } catch (Exception e) {
            if (log.isDebugEnabled()) {
                log.debug(sm.getString("catalina.configFail", file), e);
            }
        }
        if (inputStream == null) {
            try {
                inputStream = getClass().getClassLoader()
                    .getResourceAsStream(getConfigFile());
                inputSource = new InputSource
                    (getClass().getClassLoader()
                     .getResource(getConfigFile()).toString());
            } catch (Exception e) {
                if (log.isDebugEnabled()) {
                    log.debug(sm.getString("catalina.configFail",
                            getConfigFile()), e);
                }
            }
        }

        if (inputStream == null) {
            try {
                inputStream = getClass().getClassLoader()
                        .getResourceAsStream("server-embed.xml");
                inputSource = new InputSource
                (getClass().getClassLoader()
                        .getResource("server-embed.xml").toString());
            } catch (Exception e) {
                if (log.isDebugEnabled()) {
                    log.debug(sm.getString("catalina.configFail",
                            "server-embed.xml"), e);
                }
            }
        }


        if (inputStream == null || inputSource == null) {
            if  (file == null) {
                log.warn(sm.getString("catalina.configFail",
                        getConfigFile() + "] or [server-embed.xml]"));
            } else {
                log.warn(sm.getString("catalina.configFail",
                        file.getAbsolutePath()));
                if (file.exists() && !file.canRead()) {
                    log.warn("Permissions incorrect, read permission is not allowed on the file.");
                }
            }
            return;
        }

        try {
            inputSource.setByteStream(inputStream);
            digester.push(this);
            digester.parse(inputSource);
        } catch (SAXParseException spe) {
            log.warn("Catalina.start using " + getConfigFile() + ": " +
                    spe.getMessage());
            return;
        } catch (Exception e) {
            log.warn("Catalina.start using " + getConfigFile() + ": " , e);
            return;
        }
    } finally {
        if (inputStream != null) {
            try {
                inputStream.close();
            } catch (IOException e) {
            }
        }
    }

    getServer().setCatalina(this);
    getServer().setCatalinaHome(Bootstrap.getCatalinaHomeFile());
    getServer().setCatalinaBase(Bootstrap.getCatalinaBaseFile());

    initStreams();

    try {
        getServer().init();
    } catch (LifecycleException e) {
        if (Boolean.getBoolean("org.apache.catalina.startup.EXIT_ON_INIT_FAILURE")) {
            throw new java.lang.Error(e);
        } else {
            log.error("Catalina.start", e);
        }
    }

    long t2 = System.nanoTime();
    if(log.isInfoEnabled()) {
        log.info("Initialization processed in " + ((t2 - t1) / 1000000) + " ms");
    }
}
```
StandardServer的init方法是在LifecycleBase中实现的。init、start、stop、destory方法是在Lifecycle接口中定义的，LifecycleBase抽象类通过模板模式实现了对应的方法和进行管理，提供initInternal、startInternal、stopInternal、destroyInternal等钩子方法让子类来实现。
```java
public final synchronized void init() throws LifecycleException {
    if (!state.equals(LifecycleState.NEW)) {
        invalidTransition(Lifecycle.BEFORE_INIT_EVENT);
    }
    setStateInternal(LifecycleState.INITIALIZING, null, false);

    try {
        initInternal();
    } catch (Throwable t) {
        ExceptionUtils.handleThrowable(t);
        setStateInternal(LifecycleState.FAILED, null, false);
        throw new LifecycleException(
                sm.getString("lifecycleBase.initFail",toString()), t);
    }

    setStateInternal(LifecycleState.INITIALIZED, null, false);
}
```
init方法主要是对LifecycleState进行判断与设置，并调用对应子类的initInternal方法。
StandardServer的initInternal方法主要是注册全局StringCache、MBeanFactory、namingResource，然后调用CommonClassLoader和shareClassLoader来加载extension的jar包，最后循环调用子容器service(默认为StandardService)的init方法。
```java   
protected void initInternal() throws LifecycleException {

    super.initInternal();

    onameStringCache = register(new StringCache(), "type=StringCache");

    MBeanFactory factory = new MBeanFactory();
    factory.setContainer(this);
    onameMBeanFactory = register(factory, "type=MBeanFactory");

    globalNamingResources.init();

    if (getCatalina() != null) {
        ClassLoader cl = getCatalina().getParentClassLoader();
        while (cl != null && cl != ClassLoader.getSystemClassLoader()) {
            if (cl instanceof URLClassLoader) {
                URL[] urls = ((URLClassLoader) cl).getURLs();
                for (URL url : urls) {
                    if (url.getProtocol().equals("file")) {
                        try {
                            File f = new File (url.toURI());
                            if (f.isFile() &&
                                    f.getName().endsWith(".jar")) {
                                ExtensionValidator.addSystemResource(f);
                            }
                        } catch (URISyntaxException e) {
                        } catch (IOException e) {
                        }
                    }
                }
            }
            cl = cl.getParent();
        }
    }
    for (int i = 0; i < services.length; i++) {
        services[i].init();
    }
}
```    
### **初始化Service** ###
StandardService的initInternal方法,主要是初始化Executor、container(Engine)、mapperListenner和对connectors进行循环初始化。
```java
protected void initInternal() throws LifecycleException {

    super.initInternal();

    if (container != null) {
        container.init();
    }

    for (Executor executor : findExecutors()) {
        if (executor instanceof JmxEnabled) {
            ((JmxEnabled) executor).setDomain(getDomain());
        }
        executor.init();
    }

    mapperListener.init();

    synchronized (connectorsLock) {
        for (Connector connector : connectors) {
            try {
                connector.init();
            } catch (Exception e) {
                String message = sm.getString(
                        "standardService.connector.initFailed", connector);
                log.error(message, e);

                if (Boolean.getBoolean("org.apache.catalina.startup.EXIT_ON_INIT_FAILURE"))
                    throw new LifecycleException(message);
            }
        }
    }
}
```
### **初始化Engine** ###
Engine的initInternal主要对确保至少存在一个realm，默认创建NullRealm。
```java
protected void initInternal() throws LifecycleException {
	getRealm();
	super.initInternal();
}
```
```java
public Realm getRealm() {
	Realm configured = super.getRealm();
	// If no set realm has been called - default to NullRealm
	// This can be overridden at engine, context and host level
	if (configured == null) {
		configured = new NullRealm();
		this.setRealm(configured);
	}
	return configured;
}
```
### **初始化Connector** ###
Connector的initInternal方法主要是设置parseBodyMethod的默认值（POST）、初始化CoyoteAdapter和protocolHandler
```java
protected void initInternal() throws LifecycleException {
	super.initInternal();
	adapter = new CoyoteAdapter(this);
	protocolHandler.setAdapter(adapter);
	if (null == parseBodyMethodsSet) {
		setParseBodyMethods(getParseBodyMethods());
	}
	if (protocolHandler.isAprRequired() && !AprLifecycleListener.isAprAvailable()) {
		throw new LifecycleException(
				sm.getString("coyoteConnector.protocolHandlerNoApr", getProtocolHandlerClassName()));
	}
	try {
		protocolHandler.init();
	} catch (Exception e) {
		throw new LifecycleException(sm.getString("coyoteConnector.protocolHandlerInitializationFailed"), e);
	}
}
```
ProtocolHandler的init是在AbstractProtocol实现，主要调用EndPoint的init方法
```java
public void init() throws Exception {
    if (getLog().isInfoEnabled())
        getLog().info(sm.getString("abstractProtocolHandler.init",getName()));

    if (oname == null) {
        // Component not pre-registered so register it
        oname = createObjectName();
        if (oname != null) {
            Registry.getRegistry(null, null).registerComponent(this, oname, null);
        }
    }

    if (this.domain != null) {
        try {
            tpOname = new ObjectName(domain + ":" +"type=ThreadPool,name=" + getName());
            Registry.getRegistry(null, null).registerComponent(endpoint,tpOname, null);
        } catch (Exception e) {
            getLog().error(sm.getString(
                    "abstractProtocolHandler.mbeanRegistrationFailed",
                    tpOname, getName()), e);
        }
        rgOname=new ObjectName(domain +
                ":type=GlobalRequestProcessor,name=" + getName());
        Registry.getRegistry(null, null).registerComponent(
                getHandler().getGlobal(), rgOname, null );
    }

    String endpointName = getName();
    endpoint.setName(endpointName.substring(1, endpointName.length()-1));

    try {
        endpoint.init();
    } catch (Exception ex) {
        getLog().error(sm.getString("abstractProtocolHandler.initError",
                getName()), ex);
        throw ex;
    }
}
```
EndPoint的init方法主要是根据bingOnInit变量是否调用bind方法，bind主要是由子类来实习的。
```java
public final void init() throws Exception {
    testServerCipherSuitesOrderSupport();
    if (bindOnInit) {
        bind();
        bindState = BindState.BOUND_ON_INIT;
    }
}
```    
---
### **启动Catalina** ###
在对所有的组件进行init完成之后，接下来就是调用Catalina的start方法进行启动了。跟init差不多，都是父组件在完成自身的start方法之后，循环对子组件进行start。
Catalina的start方法主要是对StandardServer的start方法进行调用和注册shutdownHook。在Bootstrap中调用setAwait利用反射将daemon的await值设置为true。await将调用main方法的线程进行阻塞，防止main方法退出，JVM会在所有非Daemon(守护)线程都退出之后，杀死所有Daemon线程，退出程序。
await方法主要是创建一个监听指定端口的ServerSocket，当有请求过来的时候，会对特定的SHUTDOWN字符串进行比对，如果相等的话，就会退出循环，方法返回。main线程就会退出，程序也就结束了。
shutDownHook是一个关闭钩子，可以通过传入一个Thread的实例调用addShutdownHook方法，来指定当程序关闭之前需要进行什么动作或者采取措施，一般是用来释放资源的。
```
 public void start() {

    if (getServer() == null) {
        load();
    }

    if (getServer() == null) {
        return;
    }

    long t1 = System.nanoTime();

    try {
        getServer().start();
    } catch (LifecycleException e) {
        log.fatal(sm.getString("catalina.serverStartFail"), e);
        try {
            getServer().destroy();
        } catch (LifecycleException e1) {
            log.debug("destroy() failed for failed Server ", e1);
        }
        return;
    }

    long t2 = System.nanoTime();
    if (useShutdownHook) {
        if (shutdownHook == null) {
            shutdownHook = new CatalinaShutdownHook();
        }
        Runtime.getRuntime().addShutdownHook(shutdownHook);
        LogManager logManager = LogManager.getLogManager();
        if (logManager instanceof ClassLoaderLogManager) {
            ((ClassLoaderLogManager) logManager).setUseShutdownHook(
                    false);
        }
    }

    if (await) {
        await();
        stop();
    }
}
```
### **启动Server** ###
StandardServer的startInternal方法主要是对globalNamingResources（J2EE标准的JNDI）进行启动和子容器service进行循环调用start方法。
```java
protected void startInternal() throws LifecycleException {

    fireLifecycleEvent(CONFIGURE_START_EVENT, null);
    setState(LifecycleState.STARTING);

    globalNamingResources.start();

    // Start our defined Services
    synchronized (servicesLock) {
        for (int i = 0; i < services.length; i++) {
            services[i].start();
        }
    }
}
```
### **启动Service** ###
Standard的startInternal方法主要对Executor、Connector等嵌套组件和container(Engine)进行start方法调用。
```java
protected void startInternal() throws LifecycleException {
    setState(LifecycleState.STARTING);
    if (container != null) {
        synchronized (container) {
            container.start();
        }
    }
    synchronized (executors) {
        for (Executor executor: executors) {
            executor.start();
        }
    }
    mapperListener.start();
    synchronized (connectorsLock) {
        for (Connector connector: connectors) {
            try {
                if (connector.getState() != LifecycleState.FAILED) {
                    connector.start();
                }
            } catch (Exception e) {
                log.error(sm.getString(
                        "standardService.connector.startFailed",
                        connector), e);
            }
        }
    }
}
```
### **启动Listener** ###
MapperListener的startInternal主要注册host，在注册host的方法中会递归注册对应context和wrapper。
```java
public void startInternal() throws LifecycleException {

	setState(LifecycleState.STARTING);
	Engine engine = (Engine) service.getContainer();
	if (engine == null) {
		return;
	}

	findDefaultHost();

	addListeners(engine);

	Container[] conHosts = engine.findChildren();
	for (Container conHost : conHosts) {
		Host host = (Host) conHost;
		if (!LifecycleState.NEW.equals(host.getState())) {
			registerHost(host);
		}
	}
}
```
### **启动Connector** ###
Connector主要是启动protocolHandler，通过启动protocolHandler来委托调用启动endPoint。
```java
protected void startInternal() throws LifecycleException {
	if (getPort() < 0) {
		throw new LifecycleException(sm.getString("coyoteConnector.invalidPort", Integer.valueOf(getPort())));
	}
	setState(LifecycleState.STARTING);
	try {
		protocolHandler.start();
	} catch (Exception e) {
		String errPrefix = "";
		if (this.service != null) {
			errPrefix += "service.getName(): \"" + this.service.getName() + "\"; ";
		}
		throw new LifecycleException(errPrefix + +sm.getString("coyoteConnector.protocolHandlerStartFailed"),e);
	}
}
```
#### **启动EndPoint** ####
```java
public void start() throws Exception {
    try {
        endpoint.start();
    } catch (Exception ex) {
        throw ex;
    }
}
```
EndPoint的startInternal是个抽象方法，主要是由子类来实现。有AprEndpoint、JIoEndpoint、Nio2Endpoint、NioEndpoint四种，我用的是Tomcat8，主要使用的是NioEndPoint的，接下来主要是以NioEndPoint分析为主。
startInternal主要是启动worker线程池（Executor主要是用来执行SocketProcessor线程）、Poller线程(根据处理器数量来创建一定数目的轮询器)和Acceptor(接收者线程)。不过如果是JIOEndPoint的话，还会启动Async Timeout thread.
```java
 public void startInternal() throws Exception {

    if (!running) {
        running = true;
        paused = false;

        processorCache = new SynchronizedStack<>(SynchronizedStack.DEFAULT_SIZE,
                socketProperties.getProcessorCache());
        eventCache = new SynchronizedStack<>(SynchronizedStack.DEFAULT_SIZE,
                        socketProperties.getEventCache());
        nioChannels = new SynchronizedStack<>(SynchronizedStack.DEFAULT_SIZE,
                socketProperties.getBufferPool());
        if ( getExecutor() == null ) {
            createExecutor();
        }
        initializeConnectionLatch();
        pollers = new Poller[getPollerThreadCount()];
        for (int i=0; i<pollers.length; i++) {
            pollers[i] = new Poller();
            Thread pollerThread = new Thread(pollers[i], getName() + "-ClientPoller-"+i);
            pollerThread.setPriority(threadPriority);
            pollerThread.setDaemon(true);
            pollerThread.start();
        }
        
        startAcceptorThreads();
    }
}
```
```java
protected final void startAcceptorThreads() {
    int count = getAcceptorThreadCount();
    acceptors = new Acceptor[count];

    for (int i = 0; i < count; i++) {
        acceptors[i] = createAcceptor();
        String threadName = getName() + "-Acceptor-" + i;
        acceptors[i].setThreadName(threadName);
        Thread t = new Thread(acceptors[i], threadName);
        t.setPriority(getAcceptorThreadPriority());
        t.setDaemon(getDaemon());
        t.start();
    }
}
```
## **Socket对象转为内部Request** ##
在tomcat启动之后，我们可以看到Tomcat是通过NioEndPoint的Acceptor来接收Socket链接，交由对应Processor来处理。
Acceptor的run中会一直循环直到接收到关闭命令为止，如果已经达到最大连接数通过调用countUpOrAwaitConnection方法让线程等待，当有请求进行连接的时候，会获取对应的Socket，然后调用setSocketOptions方法把socket添加到轮询器Poller中。
### **Processor** ###
```java
public void run() {
        int errorDelay = 0;
        while (running) {
            while (paused && running) {
                state = AcceptorState.PAUSED;
                try {
                    Thread.sleep(50);
                } catch (InterruptedException e) {
                    // Ignore
                }
            }

            if (!running) {
                break;
            }
            state = AcceptorState.RUNNING;

            try {
                countUpOrAwaitConnection();
                SocketChannel socket = null;
                try {
                    socket = serverSock.accept();
                } catch (IOException ioe) {
                    countDownConnection();
                    errorDelay = handleExceptionWithDelay(errorDelay);
                    throw ioe;
                }
                errorDelay = 0;

                if (running && !paused) {
                    if (!setSocketOptions(socket)) {
                        countDownConnection();
                        closeSocket(socket);
                    }
                } else {
                    countDownConnection();
                    closeSocket(socket);
                }
            } catch (SocketTimeoutException sx) {
            } catch (IOException x) {
                if (running) {
                    log.error(sm.getString("endpoint.accept.fail"), x);
                }
            } catch (OutOfMemoryError oom) {
                try {
                    oomParachuteData = null;
                    releaseCaches();
                }catch ( Throwable oomt ) {
                    try {
                        try {
                            System.err.println(oomParachuteMsg);
                            oomt.printStackTrace();
                        }catch (Throwable letsHopeWeDontGetHere){
                            ExceptionUtils.handleThrowable(letsHopeWeDontGetHere);
                        }
                    }catch (Throwable letsHopeWeDontGetHere){
                        ExceptionUtils.handleThrowable(letsHopeWeDontGetHere);
                    }
                }
            } catch (Throwable t) {
                ExceptionUtils.handleThrowable(t);
            }
        }
        state = AcceptorState.ENDED;
    }
}
```
setSocketOptions方法主要是对SocketChannel的参数进行设置，然后将SocketChannel注册到Poller中。
首先将SocketChannel设置为not-block，然后根据在server.xml的connector节点中的参数通过socketProperties的setProperties方法对socket进行设置，例如socket接收或者发送的窗口大小、心跳检查等，然后从NioChannels队列中获取NIoChannel对象，NioChannel是SocketChannel的一个封装，主要是用来屏蔽SSL TCP连接和普通连接的差异。
如果NioChannel为null的话，就创建NioChannel(分为SSL和普通连接两种情况)，需要传入SocketChannel和NioBufferHandler，否则的话需要对NioChannel进行reset和与SocketChannel进行关联。
最后通过getPoller0方法轮询当前的Poller数组，取出一个Poller调用register对NioChannel进行注册。
```java
protected boolean setSocketOptions(SocketChannel socket) {
    // Process the connection
    try {
        //disable blocking, APR style, we are gonna be polling it
        socket.configureBlocking(false);
        Socket sock = socket.socket();
        socketProperties.setProperties(sock);

        NioChannel channel = nioChannels.pop();
        if ( channel == null ) {
            if (sslContext != null) {
                SSLEngine engine = createSSLEngine();
                int appbufsize = engine.getSession().getApplicationBufferSize();
                NioBufferHandler bufhandler = new NioBufferHandler(Math.max(appbufsize,socketProperties.getAppReadBufSize()),
                                                                   Math.max(appbufsize,socketProperties.getAppWriteBufSize()),
                                                                   socketProperties.getDirectBuffer());
                channel = new SecureNioChannel(socket, engine, bufhandler, selectorPool);
            } else {
                // normal tcp setup
                NioBufferHandler bufhandler = new NioBufferHandler(socketProperties.getAppReadBufSize(),
                                                                   socketProperties.getAppWriteBufSize(),
                                                                   socketProperties.getDirectBuffer());

                channel = new NioChannel(socket, bufhandler);
            }
        } else {
            channel.setIOChannel(socket);
            if ( channel instanceof SecureNioChannel ) {
                SSLEngine engine = createSSLEngine();
                ((SecureNioChannel)channel).reset(engine);
            } else {
                channel.reset();
            }
        }
        getPoller0().register(channel);
    } catch (Throwable t) {
        ExceptionUtils.handleThrowable(t);
        try {
            log.error("",t);
        } catch (Throwable tt) {
            ExceptionUtils.handleThrowable(t);
        }
        // Tell to close the socket
        return false;
    }
    return true;
}
```
register主要是将NioChannel封装成一个PollerEvent，然后加入到Poller的Events缓存队列中。
首先设置NioChannel的Poller，然后创建KeyAttachment对NioChannel进行封装和设置对应的参数，设置读操作为感兴趣操作。
然后从Poller中的事件缓存中取出一个PollerEvent，如果为null的话，传入NioChannel和KeyAttachment构建一个PollerEvent对象，否则的话对PollerEvent进行重置；最后将PollerEvent添加到Poller对象里的缓存事件队列，如果当前事件队列没有事件，就会唤醒当前阻塞的Selector。
```java
 public void register(final NioChannel socket) {
    socket.setPoller(this);
    KeyAttachment ka = new KeyAttachment(socket);
    ka.setPoller(this);
    ka.setTimeout(getSocketProperties().getSoTimeout());
    ka.setKeepAliveLeft(NioEndpoint.this.getMaxKeepAliveRequests());
    ka.setSecure(isSSLEnabled());
    PollerEvent r = eventCache.pop();
    ka.interestOps(SelectionKey.OP_READ);//this is what OP_REGISTER turns into.
    if ( r==null) r = new PollerEvent(socket,ka,OP_REGISTER);
    else r.reset(socket,ka,OP_REGISTER);
    addEvent(r);
}
```
Acceptor是将接收到的Socket的封装成PollerEvent添加到Poller的事件缓存队列，然后Poller的run方法对事件进行处理。
每一Poller对象都会聚合一个Selector进行处理。
run循环中对Poller的状态进行判断，如果是paused的话，就会无限sleep，直到唤醒；如果是close的话，就会调用Events方法对事件队列的事件逐一取出并执行对应的事件线程，如果事件是PollerEvent的话，会将事件处理完成之后，将事件对象放到事件缓存eventCache中，否则如果事件队列为空的话，直接返回false。
```java
 public boolean events() {
    boolean result = false;

    PollerEvent pe = null;
    while ( (pe = events.poll()) != null ) {
        result = true;
        try {
            pe.run();
            pe.reset();
            if (running && !paused) {
                eventCache.push(pe);
            }
        } catch ( Throwable x ) {
            log.error("",x);
        }
    }

    return result;
}
```
PollerEvent主要是向socket注册和更新感兴趣的事件。
```java
 public void run() {
        if ( interestOps == OP_REGISTER ) {
            try {
                socket.getIOChannel().register(socket.getPoller().getSelector(), SelectionKey.OP_READ, key);
            } catch (Exception x) {
                log.error("", x);
            }
        } else {
            final SelectionKey key = socket.getIOChannel().keyFor(socket.getPoller().getSelector());
            try {
                boolean cancel = false;
                if (key != null) {
                    final KeyAttachment att = (KeyAttachment) key.attachment();
                    if ( att!=null ) {
                        att.access();//to prevent timeout
                        //we are registering the key to start with, reset the fairness counter.
                        int ops = key.interestOps() | interestOps;
                        att.interestOps(ops);
                        key.interestOps(ops);
                    } else {
                        cancel = true;
                    }
                } else {
                    cancel = true;
                }
                if ( cancel ) socket.getPoller().cancelledKey(key,SocketStatus.ERROR);
            }catch (CancelledKeyException ckx) {
                try {
                    socket.getPoller().cancelledKey(key,SocketStatus.DISCONNECT);
                }catch (Exception ignore) {}
            }
        }
    }
```
如果没有close的话，会调用唤醒多路复用器的条件阀值wakeupCounter的getAndSet(-1)方法唤醒Selector，然后调用selector的非阻塞selectNow方法，否则的话调用Selector的select方法查看是否事件发生。
然后获取在Selector的selectionKey方法获取在Selector中已注册的Channel中已经就绪SelectionKey，然后对SelectionKey集合进行迭代处理。
在迭代处理中，首先获取SelectionKey的attachment(KeyAttachment)，然后更新通道最近一次事件发生的时间，调用remove方法将SelectionKey从set中移除，防止多次处理，最后调用processKey方法进行具体处理。最后通过timeout方法对超时进行处理，每执行一个select操作就去参看所有的通道是否超时，如果超时的话，就将对应的通道从Selector中移除。
```java
 public void run() {
    while (true) {
        try {
            while (paused && (!close) ) {
                try {
                    Thread.sleep(100);
                } catch (InterruptedException e) {
                }
            }
            boolean hasEvents = false;
            if (close) {
                events();
                timeout(0, false);
                try {
                    selector.close();
                } catch (IOException ioe) {
                    log.error(sm.getString(
                            "endpoint.nio.selectorCloseFail"), ioe);
                }
                break;
            } else {
                hasEvents = events();
            }
            try {
                if ( !close ) {
                    if (wakeupCounter.getAndSet(-1) > 0) {
                        keyCount = selector.selectNow();
                    } else {
                        keyCount = selector.select(selectorTimeout);
                    }
                    wakeupCounter.set(0);
                }
                if (close) {
                    events();
                    timeout(0, false);
                    try {
                        selector.close();
                    } catch (IOException ioe) {
                        log.error(sm.getString(
                                "endpoint.nio.selectorCloseFail"), ioe);
                    }
                    break;
                }
            } catch (Throwable x) {
                ExceptionUtils.handleThrowable(x);
                log.error("",x);
                continue;
            }
            if ( keyCount == 0 ) hasEvents = (hasEvents | events());

            Iterator<SelectionKey> iterator =
                keyCount > 0 ? selector.selectedKeys().iterator() : null;
            while (iterator != null && iterator.hasNext()) {
                SelectionKey sk = iterator.next();
                KeyAttachment attachment = (KeyAttachment)sk.attachment();
                if (attachment == null) {
                    iterator.remove();
                } else {
                    attachment.access();
                    iterator.remove();
                    processKey(sk, attachment);
                }
            }//while
            timeout(keyCount,hasEvents);
            if ( oomParachute > 0 && oomParachuteData == null ) checkParachute();
        } catch (OutOfMemoryError oom) {
            try {
                oomParachuteData = null;
                releaseCaches();
                log.error("", oom);
            }catch ( Throwable oomt ) {
                try {
                    System.err.println(oomParachuteMsg);
                    oomt.printStackTrace();
                }catch (Throwable letsHopeWeDontGetHere){
                    ExceptionUtils.handleThrowable(letsHopeWeDontGetHere);
                }
            }
        }
    }

    stopLatch.countDown();
}
```

processKey主要用来处理Selector检查到的通道事件。
首先对状态进行判断，如果是close的话，调用cancelledKey进行取消操作；否则对SelectionKey的isValid和KeyAttachment进行检验的话，通过的话，对事件发生的时间进行更新，防止timeout,然后SelectionKey进行读写判断，如果是读或者写事件的话，就对attachment的sendfileData进行非空判断，如果不为空的话，就调用processSendfile方法处理，否则的话就对通道上已经发生的事件进行取消关注，然后调用processSocket进行处理，如果返回false的话，就调用cancelledKey进行取消
```java
 protected boolean processKey(SelectionKey sk, KeyAttachment attachment) {
    boolean result = true;
    try {
        if ( close ) {
            cancelledKey(sk, SocketStatus.STOP);
        } else if ( sk.isValid() && attachment != null ) {
            attachment.access();//make sure we don't time out valid sockets
            if (sk.isReadable() || sk.isWritable() ) {
                if ( attachment.getSendfileData() != null ) {
                    processSendfile(sk,attachment, false);
                } else {
                    if ( isWorkerAvailable() ) {
                        unreg(sk, attachment, sk.readyOps());
                        boolean closeSocket = false;
                        // Read goes before write
                        if (sk.isReadable()) {
                            if (!processSocket(attachment, SocketStatus.OPEN_READ, true)) {
                                closeSocket = true;
                            }
                        }
                        if (!closeSocket && sk.isWritable()) {
                            if (!processSocket(attachment, SocketStatus.OPEN_WRITE, true)) {
                                closeSocket = true;
                            }
                        }
                        if (closeSocket) {
                            cancelledKey(sk,SocketStatus.DISCONNECT);
                        }
                    } else {
                        result = false;
                    }
                }
            }
        } else {
            //invalid key
            cancelledKey(sk, SocketStatus.ERROR);
        }
    } catch ( CancelledKeyException ckx ) {
        cancelledKey(sk, SocketStatus.ERROR);
    } catch (Throwable t) {
        ExceptionUtils.handleThrowable(t);
        log.error("",t);
    }
    return result;
}
```

processSendfile
```java
public boolean processSendfile(SelectionKey sk, KeyAttachment attachment, boolean event) {
    NioChannel sc = null;
    try {
        unreg(sk, attachment, sk.readyOps());
        SendfileData sd = attachment.getSendfileData();

        if (log.isTraceEnabled()) {
            log.trace("Processing send file for: " + sd.fileName);
        }

        //setup the file channel
        if ( sd.fchannel == null ) {
            File f = new File(sd.fileName);
            if ( !f.exists() ) {
                cancelledKey(sk,SocketStatus.ERROR);
                return false;
            }
            @SuppressWarnings("resource") // Closed when channel is closed
            FileInputStream fis = new FileInputStream(f);
            sd.fchannel = fis.getChannel();
        }

        sc = attachment.getSocket();
        WritableByteChannel wc = ((sc instanceof SecureNioChannel)?sc:sc.getIOChannel());

        if (sc.getOutboundRemaining()>0) {
            if (sc.flushOutbound()) {
                attachment.access();
            }
        } else {
            long written = sd.fchannel.transferTo(sd.pos,sd.length,wc);
            if ( written > 0 ) {
                sd.pos += written;
                sd.length -= written;
                attachment.access();
            } else {
                if (sd.fchannel.size() <= sd.pos) {
                    throw new IOException("Sendfile configured to " +
                            "send more data than was available");
                }
            }
        }
        if ( sd.length <= 0 && sc.getOutboundRemaining()<=0) {
            if (log.isDebugEnabled()) {
                log.debug("Send file complete for: "+sd.fileName);
            }
            attachment.setSendfileData(null);
            try {
                sd.fchannel.close();
            } catch (Exception ignore) {
            }
            if ( sd.keepAlive ) {
                    if (log.isDebugEnabled()) {
                        log.debug("Connection is keep alive, registering back for OP_READ");
                    }
                    if (event) {
                        this.add(attachment.getSocket(),SelectionKey.OP_READ);
                    } else {
                        reg(sk,attachment,SelectionKey.OP_READ);
                    }
            } else {
                if (log.isDebugEnabled()) {
                    log.debug("Send file connection is being closed");
                }
                cancelledKey(sk,SocketStatus.STOP);
                return false;
            }
        } else {
            if (log.isDebugEnabled()) {
                log.debug("OP_WRITE for sendfile: " + sd.fileName);
            }
            if (event) {
                add(attachment.getSocket(),SelectionKey.OP_WRITE);
            } else {
                reg(sk,attachment,SelectionKey.OP_WRITE);
            }
        }
    }catch ( IOException x ) {
        if ( log.isDebugEnabled() ) log.debug("Unable to complete sendfile request:", x);
        cancelledKey(sk,SocketStatus.ERROR);
        return false;
    }catch ( Throwable t ) {
        log.error("",t);
        cancelledKey(sk, SocketStatus.ERROR);
        return false;
    }
    return true;
}
```
processSocket主要是从SocketProcessor缓存队列中取出一个SocketProcessor来处理Socket
```java
protected boolean processSocket(KeyAttachment attachment, SocketStatus status, boolean dispatch) {
    try {
        if (attachment == null) {
            return false;
        }
        SocketProcessor sc = processorCache.pop();
        if ( sc == null ) sc = new SocketProcessor(attachment, status);
        else sc.reset(attachment, status);
        Executor executor = getExecutor();
        if (dispatch && executor != null) {
            executor.execute(sc);
        } else {
            sc.run();
        }
    } catch (RejectedExecutionException ree) {
        log.warn(sm.getString("endpoint.executor.fail", attachment.getSocket()), ree);
        return false;
    } catch (Throwable t) {
        ExceptionUtils.handleThrowable(t);
        log.error(sm.getString("endpoint.process.fail"), t);
        return false;
    }
    return true;
}
```
SocketProcessor/run主要是委托给doRun方法调用。
```java
public void run() {
    NioChannel socket = ka.getSocket();
    if (ka.isUpgraded() && SocketStatus.OPEN_WRITE == status) {
        synchronized (ka.getWriteThreadLock()) {
            doRun();
        }
    } else {
        synchronized (socket) {
            doRun();
        }
    }
}
```
doRun方法主要是将KeyAttachment(即Socket)交给对应的Handler来处理和对status进行更新与修改。
```
private void doRun() {
    NioChannel socket = ka.getSocket();
    SelectionKey key = socket.getIOChannel().keyFor(socket.getPoller().getSelector());

    try {
        int handshake = -1;
        try {
            if (key != null) {
                if (socket.isHandshakeComplete() ||
                        status == SocketStatus.STOP) {
                    handshake = 0;
                } else {
                    handshake = socket.handshake(
                            key.isReadable(), key.isWritable());
                    status = SocketStatus.OPEN_READ;
                }
            }
        } catch (IOException x) {
            handshake = -1;
            if (log.isDebugEnabled()) log.debug("Error during SSL handshake",x);
        } catch (CancelledKeyException ckx) {
            handshake = -1;
        }
        if (handshake == 0) {
            SocketState state = SocketState.OPEN;
            if (status == null) {
                state = handler.process(ka, SocketStatus.OPEN_READ);
            } else {
                state = handler.process(ka, status);
            }
            if (state == SocketState.CLOSED) {
                close(socket, key, SocketStatus.ERROR);
            }
        } else if (handshake == -1 ) {
            close(socket, key, SocketStatus.DISCONNECT);
        } else {
            ka.getPoller().add(socket,handshake);
        }
    } catch (CancelledKeyException cx) {
        socket.getPoller().cancelledKey(key, null);
    } catch (OutOfMemoryError oom) {
        try {
            oomParachuteData = null;
            log.error("", oom);
            socket.getPoller().cancelledKey(key,SocketStatus.ERROR);
            releaseCaches();
        } catch (Throwable oomt) {
            try {
                System.err.println(oomParachuteMsg);
                oomt.printStackTrace();
            } catch (Throwable letsHopeWeDontGetHere){
                ExceptionUtils.handleThrowable(letsHopeWeDontGetHere);
            }
        }
    } catch (VirtualMachineError vme) {
        ExceptionUtils.handleThrowable(vme);
    } catch (Throwable t) {
        log.error("", t);
        socket.getPoller().cancelledKey(key,SocketStatus.ERROR);
    } finally {
        ka = null;
        status = null;
        //return to cache
        if (running && !paused) {
            processorCache.push(this);
        }
    }
}
```
process方法是在AbstractConnectionHandler中实现的，它提供了一个模板方法，子类只需要实现相应的钩子函数就可以了。
默认实现为Http11ConnectionHandler实现。
```java
public SocketState process(SocketWrapper<S> wrapper,
            SocketStatus status) {
    if (wrapper == null) {
        return SocketState.CLOSED;
    }

    S socket = wrapper.getSocket();
    if (socket == null) {
        return SocketState.CLOSED;
    }

    Processor<S> processor = connections.get(socket);
    if (status == SocketStatus.DISCONNECT && processor == null) {
        return SocketState.CLOSED;
    }

    wrapper.setAsync(false);
    ContainerThreadMarker.set();

    try {
        if (processor == null) {
            processor = recycledProcessors.pop();
        }
        if (processor == null) {
            processor = createProcessor();
        }

        initSsl(wrapper, processor);

        SocketState state = SocketState.CLOSED;
        Iterator<DispatchType> dispatches = null;
        do {
            if (status == SocketStatus.CLOSE_NOW) {
                processor.errorDispatch();
                state = SocketState.CLOSED;
            } else if (dispatches != null) {
                connections.put(socket, processor);
                DispatchType nextDispatch = dispatches.next();
                if (processor.isUpgrade()) {
                    state = processor.upgradeDispatch(
                            nextDispatch.getSocketStatus());
                } else {
                    state = processor.asyncDispatch(
                            nextDispatch.getSocketStatus());
                }
            } else if (processor.isComet()) {
                state = processor.event(status);
            } else if (processor.isUpgrade()) {
                state = processor.upgradeDispatch(status);
            } else if (status == SocketStatus.DISCONNECT) {
            } else if (processor.isAsync()) {
                state = processor.asyncDispatch(status);
            } else if (state == SocketState.ASYNC_END) {
                state = processor.asyncDispatch(status);
                getProtocol().endpoint.removeWaitingRequest(wrapper);
                if (state == SocketState.OPEN) {
                    state = processor.process(wrapper);
                }
            } else if (status == SocketStatus.OPEN_WRITE) {
                state = SocketState.LONG;
            } else {
                state = processor.process(wrapper);
            }

            if (state != SocketState.CLOSED && processor.isAsync()) {
                state = processor.asyncPostProcess();
            }

            if (state == SocketState.UPGRADING) {
                UpgradeToken upgradeToken = processor.getUpgradeToken();
                HttpUpgradeHandler httpUpgradeHandler = upgradeToken.getHttpUpgradeHandler();
                ByteBuffer leftoverInput = processor.getLeftoverInput();
                release(wrapper, processor, false, false);
                processor = createUpgradeProcessor(
                        wrapper, leftoverInput, upgradeToken);
                wrapper.setUpgraded(true);
                connections.put(socket, processor);
                if (upgradeToken.getInstanceManager() == null) {
                    httpUpgradeHandler.init((WebConnection) processor);
                } else {
                    ClassLoader oldCL = upgradeToken.getContextBind().bind(false, null);
                    try {
                        httpUpgradeHandler.init((WebConnection) processor);
                    } finally {
                        upgradeToken.getContextBind().unbind(false, oldCL);
                    }
                }
            }
            if (getLog().isDebugEnabled()) {
                getLog().debug("Socket: [" + wrapper +
                        "], Status in: [" + status +
                        "], State out: [" + state + "]");
            }
            if (dispatches == null || !dispatches.hasNext()) {
                dispatches = wrapper.getIteratorAndClearDispatches();
            }
        } while (state == SocketState.ASYNC_END ||
                state == SocketState.UPGRADING ||
                dispatches != null && state != SocketState.CLOSED);

        if (state == SocketState.LONG) {
            connections.put(socket, processor);
            longPoll(wrapper, processor);
        } else if (state == SocketState.OPEN) {
            connections.remove(socket);
            release(wrapper, processor, false, true);
        } else if (state == SocketState.SENDFILE) {
            connections.remove(socket);
            release(wrapper, processor, false, false);
        } else if (state == SocketState.UPGRADED) {
            if (status != SocketStatus.OPEN_WRITE) {
                longPoll(wrapper, processor);
            }
        } else {
            connections.remove(socket);
            if (processor.isUpgrade()) {
                UpgradeToken upgradeToken = processor.getUpgradeToken();
                HttpUpgradeHandler httpUpgradeHandler = upgradeToken.getHttpUpgradeHandler();
                InstanceManager instanceManager = upgradeToken.getInstanceManager();
                if (instanceManager == null) {
                    httpUpgradeHandler.destroy();
                } else {
                    ClassLoader oldCL = upgradeToken.getContextBind().bind(false, null);
                    try {
                        httpUpgradeHandler.destroy();
                        instanceManager.destroyInstance(httpUpgradeHandler);
                    } finally {
                        upgradeToken.getContextBind().unbind(false, oldCL);
                    }
                }
            } else {
                release(wrapper, processor, true, false);
            }
        }
        return state;
    } catch(java.net.SocketException e) {
    } catch (java.io.IOException e) {
    }
    catch (Throwable e) {
        ExceptionUtils.handleThrowable(e);
    } finally {
        ContainerThreadMarker.clear();
    }
    connections.remove(socket);
    if (processor !=null && !processor.isUpgrade()) {
        release(wrapper, processor, true, false);
    }
    return SocketState.CLOSED;
}
```
## **Request在容器中的流转** ##
AbstractConnectionHandler的process方法会调用AbstractHttp11Processor的process方法。
process方法首先初始化变量，然后进行解析请求头操作，将请求交给adapter的service去处理，请求会从Connector、Engine、Host、Context、Servlet流转进行处理，最后进行结束请求的处理，设置状态码。
```java
public SocketState process(SocketWrapper<S> socketWrapper)
    throws IOException {
    RequestInfo rp = request.getRequestProcessor();
    rp.setStage(org.apache.coyote.Constants.STAGE_PARSE);

    setSocketWrapper(socketWrapper);
    getInputBuffer().init(socketWrapper, endpoint);
    getOutputBuffer().init(socketWrapper, endpoint);

    keepAlive = true;
    comet = false;
    openSocket = false;
    sendfileInProgress = false;
    readComplete = true;
    if (endpoint.getUsePolling()) {
        keptAlive = false;
    } else {
        keptAlive = socketWrapper.isKeptAlive();
    }

    if (disableKeepAlive()) {
        socketWrapper.setKeepAliveLeft(0);
    }

    while (!getErrorState().isError() && keepAlive && !comet && !isAsync() &&
            upgradeToken == null && !endpoint.isPaused()) {

        try {
            setRequestLineReadTimeout();

            if (!getInputBuffer().parseRequestLine(keptAlive)) {
                if (handleIncompleteRequestLineRead()) {
                    break;
                }
            }

            if (endpoint.isPaused()) {
                response.setStatus(503);
                setErrorState(ErrorState.CLOSE_CLEAN, null);
            } else {
                keptAlive = true;
                request.getMimeHeaders().setLimit(endpoint.getMaxHeaderCount());
                if (!getInputBuffer().parseHeaders()) {
                    openSocket = true;
                    readComplete = false;
                    break;
                }
                if (!disableUploadTimeout) {
                    setSocketTimeout(connectionUploadTimeout);
                }
            }
        } catch (IOException e) {
            setErrorState(ErrorState.CLOSE_NOW, e);
            break;
        } catch (Throwable t) {
            ExceptionUtils.handleThrowable(t);
            UserDataHelper.Mode logMode = userDataHelper.getNextMode();
            if (logMode != null) {
                String message = sm.getString(
                        "http11processor.header.parse");
                switch (logMode) {
                    case INFO_THEN_DEBUG:
                        message += sm.getString(
                                "http11processor.fallToDebug");
                        //$FALL-THROUGH$
                    case INFO:
                        getLog().info(message, t);
                        break;
                    case DEBUG:
                        getLog().debug(message, t);
                }
            }
            response.setStatus(400);
            setErrorState(ErrorState.CLOSE_CLEAN, t);
            getAdapter().log(request, response, 0);
        }

        if (!getErrorState().isError()) {
            rp.setStage(org.apache.coyote.Constants.STAGE_PREPARE);
            try {
                prepareRequest();
            } catch (Throwable t) {
                ExceptionUtils.handleThrowable(t);
                if (getLog().isDebugEnabled()) {
                    getLog().debug(sm.getString(
                            "http11processor.request.prepare"), t);
                }
                response.setStatus(500);
                setErrorState(ErrorState.CLOSE_CLEAN, t);
                getAdapter().log(request, response, 0);
            }
        }

        if (maxKeepAliveRequests == 1) {
            keepAlive = false;
        } else if (maxKeepAliveRequests > 0 &&
                socketWrapper.decrementKeepAlive() <= 0) {
            keepAlive = false;
        }

        if (!getErrorState().isError()) {
            try {
                rp.setStage(org.apache.coyote.Constants.STAGE_SERVICE);
                getAdapter().service(request, response);
                if(keepAlive && !getErrorState().isError() && (
                        response.getErrorException() != null ||
                                (!isAsync() &&
                                statusDropsConnection(response.getStatus())))) {
                    setErrorState(ErrorState.CLOSE_CLEAN, null);
                }
                setCometTimeouts(socketWrapper);
            } catch (InterruptedIOException e) {
                setErrorState(ErrorState.CLOSE_NOW, e);
            } catch (HeadersTooLargeException e) {
                if (response.isCommitted()) {
                    setErrorState(ErrorState.CLOSE_NOW, e);
                } else {
                    response.reset();
                    response.setStatus(500);
                    setErrorState(ErrorState.CLOSE_CLEAN, e);
                    response.setHeader("Connection", "close"); // TODO: Remove
                }
            } catch (Throwable t) {
                ExceptionUtils.handleThrowable(t);
                getLog().error(sm.getString(
                        "http11processor.request.process"), t);
                response.setStatus(500);
                setErrorState(ErrorState.CLOSE_CLEAN, t);
                getAdapter().log(request, response, 0);
            }
        }

        rp.setStage(org.apache.coyote.Constants.STAGE_ENDINPUT);

        if (!isAsync() && !comet) {
            if (getErrorState().isError()) {
                getInputBuffer().setSwallowInput(false);
            } else {
                checkExpectationAndResponseStatus();
            }
            endRequest();
        }

        rp.setStage(org.apache.coyote.Constants.STAGE_ENDOUTPUT);
        if (getErrorState().isError()) {
            response.setStatus(500);
        }

        if (!isAsync() && !comet || getErrorState().isError()) {
            request.updateCounters();
            if (getErrorState().isIoAllowed()) {
                getInputBuffer().nextRequest();
                getOutputBuffer().nextRequest();
            }
        }

        if (!disableUploadTimeout) {
            if(endpoint.getSoTimeout() > 0) {
                setSocketTimeout(endpoint.getSoTimeout());
            } else {
                setSocketTimeout(0);
            }
        }

        rp.setStage(org.apache.coyote.Constants.STAGE_KEEPALIVE);

        if (breakKeepAliveLoop(socketWrapper)) {
            break;
        }
    }

    rp.setStage(org.apache.coyote.Constants.STAGE_ENDED);

    if (getErrorState().isError() || endpoint.isPaused()) {
        return SocketState.CLOSED;
    } else if (isAsync() || comet) {
        return SocketState.LONG;
    } else if (isUpgrade()) {
        return SocketState.UPGRADING;
    } else {
        if (sendfileInProgress) {
            return SocketState.SENDFILE;
        } else {
            if (openSocket) {
                if (readComplete) {
                    return SocketState.OPEN;
                } else {
                    return SocketState.LONG;
                }
            } else {
                return SocketState.CLOSED;
            }
        }
    }
}
```
Adapter是在Http11ConnectionHandler的createProcessor中设置的，创建是在Connector的initInternal方法中创建的。
service会将org.apache.coyote.Request和org.apache.coyote.Response对象转为org.apache.catalina.connector.Request和org.apache.catalina.connector.Response对象，在Tomcat容器中流转，使用了外观模式。然后调用postParseRequest找到对应的servlet，然后调用connector.getService().getContainer().getPipeline().getFirst().invoke(request, response)让request和response在容器中流转处理。

```java
public void service(org.apache.coyote.Request req,
                    org.apache.coyote.Response res)
    throws Exception {

    Request request = (Request) req.getNote(ADAPTER_NOTES);
    Response response = (Response) res.getNote(ADAPTER_NOTES);

    if (request == null) {

        request = connector.createRequest();
        request.setCoyoteRequest(req);
        response = connector.createResponse();
        response.setCoyoteResponse(res);

        request.setResponse(response);
        response.setRequest(request);

        req.setNote(ADAPTER_NOTES, request);
        res.setNote(ADAPTER_NOTES, response);

        // Set query string encoding
        req.getParameters().setQueryStringEncoding
            (connector.getURIEncoding());

    }

    if (connector.getXpoweredBy()) {
        response.addHeader("X-Powered-By", POWERED_BY);
    }

    boolean comet = false;
    boolean async = false;
    boolean postParseSuccess = false;

    try {
        req.getRequestProcessor().setWorkerThreadName(THREAD_NAME.get());
        postParseSuccess = postParseRequest(req, request, res, response);
        if (postParseSuccess) {
            request.setAsyncSupported(connector.getService().getContainer().getPipeline().isAsyncSupported());
            connector.getService().getContainer().getPipeline().getFirst().invoke(request, response);

            if (request.isComet()) {
                if (!response.isClosed() && !response.isError()) {
                    comet = true;
                    res.action(ActionCode.COMET_BEGIN, null);
                    if (request.getAvailable() || (request.getContentLength() > 0 && (!request.isParametersParsed()))) {
                        event(req, res, SocketStatus.OPEN_READ);
                    }
                } else {
                    request.setFilterChain(null);
                }
            }
        }

        AsyncContextImpl asyncConImpl = (AsyncContextImpl)request.getAsyncContext();
        if (asyncConImpl != null) {
            async = true;
            ReadListener readListener = req.getReadListener();
            if (readListener != null && request.isFinished()) {
                ClassLoader oldCL = null;
                try {
                    oldCL = request.getContext().bind(false, null);
                    if (req.sendAllDataReadEvent()) {
                        req.getReadListener().onAllDataRead();
                    }
                } finally {
                    request.getContext().unbind(false, oldCL);
                }
            }
        } else if (!comet) {
            request.finishRequest();
            response.finishResponse();
        }
    } catch (IOException e) {
    } finally {
        if (!async && !comet) {
            if (postParseSuccess) {
                request.getMappingData().context.logAccess(
                        request, response,
                        System.currentTimeMillis() - req.getStartTime(),
                        false);
            }
        }

        req.getRequestProcessor().setWorkerThreadName(null);
        AtomicBoolean error = new AtomicBoolean(false);
        res.action(ActionCode.IS_ERROR, error);

        if (!comet && !async || error.get()) {
            request.recycle();
            response.recycle();
        } else {
            request.clearEncoders();
            response.clearEncoders();
        }
    }

}
```
#### **postParseRequest** ####
postParseRequest主要是在解析Http Header之后为了request和response对象能够在Tomcat容器中流转进行了处理，然后调用 connector.getService().getMapper().map(serverName, decodedURI,version, request.getMappingData())方法根据MappingData对Engine、Host、Context、servlet进行mapping，找到对应要执行servlet。
```java
protected boolean postParseRequest(org.apache.coyote.Request req, Request request,
            org.apache.coyote.Response res, Response response) throws IOException, ServletException {

    if (! req.scheme().isNull()) {
        request.setSecure(req.scheme().equals("https"));
    } else {
        req.scheme().setString(connector.getScheme());
        request.setSecure(connector.getSecure());
    }


    String proxyName = connector.getProxyName();
    int proxyPort = connector.getProxyPort();
    if (proxyPort != 0) {
        req.setServerPort(proxyPort);
    }
    if (proxyName != null) {
        req.serverName().setString(proxyName);
    }

    MessageBytes undecodedURI = req.requestURI();

    if (undecodedURI.equals("*")) {
        if (req.method().equalsIgnoreCase("OPTIONS")) {
            StringBuilder allow = new StringBuilder();
            allow.append("GET, HEAD, POST, PUT, DELETE");
            if (connector.getAllowTrace()) {
                allow.append(", TRACE");
            }
            allow.append(", OPTIONS");
            res.setHeader("Allow", allow.toString());
        } else {
            res.setStatus(404);
            res.setMessage("Not found");
        }
        connector.getService().getContainer().logAccess(
                request, response, 0, true);
        return false;
    }

    MessageBytes decodedURI = req.decodedURI();

    if (undecodedURI.getType() == MessageBytes.T_BYTES) {
        decodedURI.duplicate(undecodedURI);

        parsePathParameters(req, request);

        try {
            req.getURLDecoder().convert(decodedURI, false);
        } catch (IOException ioe) {
            res.setStatus(400);
            res.setMessage("Invalid URI: " + ioe.getMessage());
            connector.getService().getContainer().logAccess(
                    request, response, 0, true);
            return false;
        }
        if (!normalize(req.decodedURI())) {
            res.setStatus(400);
            res.setMessage("Invalid URI");
            connector.getService().getContainer().logAccess(
                    request, response, 0, true);
            return false;
        }
        convertURI(decodedURI, request);
        if (!checkNormalize(req.decodedURI())) {
            res.setStatus(400);
            res.setMessage("Invalid URI character encoding");
            connector.getService().getContainer().logAccess(
                    request, response, 0, true);
            return false;
        }
    } else {
        decodedURI.toChars();
        CharChunk uriCC = decodedURI.getCharChunk();
        int semicolon = uriCC.indexOf(';');
        if (semicolon > 0) {
            decodedURI.setChars
                (uriCC.getBuffer(), uriCC.getStart(), semicolon);
        }
    }

    MessageBytes serverName;
    if (connector.getUseIPVHosts()) {
        serverName = req.localName();
        if (serverName.isNull()) {
            res.action(ActionCode.REQ_LOCAL_NAME_ATTRIBUTE, null);
        }
    } else {
        serverName = req.serverName();
    }

    String version = null;
    Context versionContext = null;
    boolean mapRequired = true;

    while (mapRequired) {
        connector.getService().getMapper().map(serverName, decodedURI,
                version, request.getMappingData());
        if (request.getContext() == null) {
            res.setStatus(404);
            res.setMessage("Not found");
            Host host = request.getHost();
            if (host != null) {
                host.logAccess(request, response, 0, true);
            }
            return false;
        }

        String sessionID;
        if (request.getServletContext().getEffectiveSessionTrackingModes()
                .contains(SessionTrackingMode.URL)) {
            sessionID = request.getPathParameter(
                    SessionConfig.getSessionUriParamName(
                            request.getContext()));
            if (sessionID != null) {
                request.setRequestedSessionId(sessionID);
                request.setRequestedSessionURL(true);
            }
        }

        parseSessionCookiesId(request);
        parseSessionSslId(request);

        sessionID = request.getRequestedSessionId();

        mapRequired = false;
        if (version != null && request.getContext() == versionContext) {
        } else {
            version = null;
            versionContext = null;

            Context[] contexts = request.getMappingData().contexts;
            if (contexts != null && sessionID != null) {
                for (int i = (contexts.length); i > 0; i--) {
                    Context ctxt = contexts[i - 1];
                    if (ctxt.getManager().findSession(sessionID) != null) {
                        if (!ctxt.equals(request.getMappingData().context)) {
                            version = ctxt.getWebappVersion();
                            versionContext = ctxt;
                            request.getMappingData().recycle();
                            mapRequired = true;
                            request.recycleSessionInfo();
                            request.recycleCookieInfo(true);
                        }
                        break;
                    }
                }
            }
        }

        if (!mapRequired && request.getContext().getPaused()) {
            try {
                Thread.sleep(1000);
            } catch (InterruptedException e) {
            }
            request.getMappingData().recycle();
            mapRequired = true;
        }
    }

    MessageBytes redirectPathMB = request.getMappingData().redirectPath;
    if (!redirectPathMB.isNull()) {
        String redirectPath = URLEncoder.DEFAULT.encode(redirectPathMB.toString());
        String query = request.getQueryString();
        if (request.isRequestedSessionIdFromURL()) {
            redirectPath = redirectPath + ";" +
                    SessionConfig.getSessionUriParamName(
                        request.getContext()) +
                "=" + request.getRequestedSessionId();
        }
        if (query != null) {
            redirectPath = redirectPath + "?" + query;
        }
        response.sendRedirect(redirectPath);
        request.getContext().logAccess(request, response, 0, true);
        return false;
    }

    if (!connector.getAllowTrace()
            && req.method().equalsIgnoreCase("TRACE")) {
        Wrapper wrapper = request.getWrapper();
        String header = null;
        if (wrapper != null) {
            String[] methods = wrapper.getServletMethods();
            if (methods != null) {
                for (int i=0; i<methods.length; i++) {
                    if ("TRACE".equals(methods[i])) {
                        continue;
                    }
                    if (header == null) {
                        header = methods[i];
                    } else {
                        header += ", " + methods[i];
                    }
                }
            }
        }
        res.setStatus(405);
        res.addHeader("Allow", header);
        res.setMessage("TRACE method is not allowed");
        request.getContext().logAccess(request, response, 0, true);
        return false;
    }

    doConnectorAuthenticationAuthorization(req, request);

    return true;
}
```
map方法主要是根据uri进行匹配找到对应的wrapper，wrapper一般与servlet一一对应的，wrapper是在MapperListener\start的时候递归注册host、context、wrapper中注册的。
```java
public void map(MessageBytes host, MessageBytes uri, String version, MappingData mappingData) throws IOException {

	if (host.isNull()) {
		host.getCharChunk().append(defaultHostName);
	}
	host.toChars();
	uri.toChars();
	internalMap(host.getCharChunk(), uri.getCharChunk(), version, mappingData);

}
```
```java
private final void internalMap(CharChunk host, CharChunk uri, String version, MappingData mappingData)
			throws IOException {
	if (mappingData.host != null) {
		throw new AssertionError();
	}

	uri.setLimit(-1);

	MappedHost[] hosts = this.hosts;
	MappedHost mappedHost = exactFindIgnoreCase(hosts, host);
	if (mappedHost == null) {
		if (defaultHostName == null) {
			return;
		}
		mappedHost = exactFind(hosts, defaultHostName);
		if (mappedHost == null) {
			return;
		}
	}
	mappingData.host = mappedHost.object;

	// Context mapping
	ContextList contextList = mappedHost.contextList;
	MappedContext[] contexts = contextList.contexts;
	int pos = find(contexts, uri);
	if (pos == -1) {
		return;
	}
	int lastSlash = -1;
	int uriEnd = uri.getEnd();
	int length = -1;
	boolean found = false;
	MappedContext context = null;
	while (pos >= 0) {
		context = contexts[pos];
		if (uri.startsWith(context.name)) {
			length = context.name.length();
			if (uri.getLength() == length) {
				found = true;
				break;
			} else if (uri.startsWithIgnoreCase("/", length)) {
				found = true;
				break;
			}
		}
		if (lastSlash == -1) {
			lastSlash = nthSlash(uri, contextList.nesting + 1);
		} else {
			lastSlash = lastSlash(uri);
		}
		uri.setEnd(lastSlash);
		pos = find(contexts, uri);
	}
	uri.setEnd(uriEnd);

	if (!found) {
		if (contexts[0].name.equals("")) {
			context = contexts[0];
		} else {
			context = null;
		}
	}
	if (context == null) {
		return;
	}

	mappingData.contextPath.setString(context.name);

	ContextVersion contextVersion = null;
	ContextVersion[] contextVersions = context.versions;
	final int versionCount = contextVersions.length;
	if (versionCount > 1) {
		Context[] contextObjects = new Context[contextVersions.length];
		for (int i = 0; i < contextObjects.length; i++) {
			contextObjects[i] = contextVersions[i].object;
		}
		mappingData.contexts = contextObjects;
		if (version != null) {
			contextVersion = exactFind(contextVersions, version);
		}
	}
	if (contextVersion == null) {
		contextVersion = contextVersions[versionCount - 1];
	}
	mappingData.context = contextVersion.object;
	mappingData.contextSlashCount = contextVersion.slashCount;

	if (!contextVersion.isPaused()) {
		internalMapWrapper(contextVersion, uri, mappingData);
	}
}
```
