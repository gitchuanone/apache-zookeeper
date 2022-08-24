

### 单机模式服务端启动

#### 执行过程概述

单机模式的ZK服务端逻辑写在 ZooKeeperServerMain 类中，由里面的 main 函数启动，整个过程如下  

![](./imgs/5-3-服务器-单机版服务器启动.png)

单机模式的委托启动类为： ZooKeeperServerMain  



#### 服务端启动过程

看下 ZooKeeperServerMain 里面的main 函数代码， org.apache.zookeeper.server.ZooKeeperServerMain#initializeAndRun

```java
protected void initializeAndRun(String[] args)
        throws ConfigException, IOException, AdminServerException
    {
        try {
            ManagedUtil.registerLog4jMBeans();
        } catch (JMException e) {
            LOG.warn("Unable to register log4j JMX control", e);
        }

        // 如果入参只有一个，则认为是配置文件的路径。
        ServerConfig config = new ServerConfig();
        if (args.length == 1) {
            config.parse(args[0]);
        } else {
            // 否则是各个参数。
            config.parse(args);
        }

        runFromConfig(config);
    }
```

——> org.apache.zookeeper.server.ZooKeeperServerMain#runFromConfig

```java
public void runFromConfig(ServerConfig config)
            throws IOException, AdminServerException {
        LOG.info("Starting server");
        FileTxnSnapLog txnLog = null;
        try {
            // Note that this thread isn't going to be doing anything else,
            // so rather than spawning another thread, we will just call
            // run() in this thread.
            // create a file logger url from the command line args
            // 初始化日志文件。
            txnLog = new FileTxnSnapLog(config.dataLogDir, config.dataDir);
            // 初始化 ZkServer 对象。
            final ZooKeeperServer zkServer = new ZooKeeperServer(txnLog,
                    config.tickTime, config.minSessionTimeout, config.maxSessionTimeout, null);

            // Registers shutdown handler which will be used to know the
            // server error or shutdown state changes.
            final CountDownLatch shutdownLatch = new CountDownLatch(1);
            zkServer.registerServerShutdownHandler(
                    new ZooKeeperServerShutdownHandler(shutdownLatch));

            // Start Admin server
            adminServer = AdminServerFactory.createAdminServer();
            adminServer.setZooKeeperServer(zkServer);
            adminServer.start();

            boolean needStartZKServer = true;
            if (config.getClientPortAddress() != null) {
                // 初始化 server 端IO对象，默认是 NIOServerCnxnFactory。
                cnxnFactory = ServerCnxnFactory.createFactory();
                // 初始化配置信息。
                cnxnFactory.configure(config.getClientPortAddress(), config.getMaxClientCnxns(), false);
                // 启动服务
                cnxnFactory.startup(zkServer);
                // zkServer has been started. So we don't need to start it again in secureCnxnFactory.
                needStartZKServer = false;
            }
            if (config.getSecureClientPortAddress() != null) {
                secureCnxnFactory = ServerCnxnFactory.createFactory();
                secureCnxnFactory.configure(config.getSecureClientPortAddress(), config.getMaxClientCnxns(), true);
                secureCnxnFactory.startup(zkServer, needStartZKServer);
            }

            /*
            container ZNodes是3.6版本之后新增的节点类型， Container类型的节点会在它没有⼦节点时
            被删除（新创建的Container节点除外），该类就是⽤来周期性的进⾏检查清理⼯作。
             */
            containerManager = new ContainerManager(zkServer.getZKDatabase(), zkServer.firstProcessor,
                    Integer.getInteger("znode.container.checkIntervalMs", (int) TimeUnit.MINUTES.toMillis(1)),
                    Integer.getInteger("znode.container.maxPerMinute", 10000)
            );
            containerManager.start();

            // Watch status of ZooKeeper server. It will do a graceful shutdown
            // if the server is not running or hits an internal error.
            shutdownLatch.await();

            shutdown();

            if (cnxnFactory != null) {
                cnxnFactory.join();
            }
            if (secureCnxnFactory != null) {
                secureCnxnFactory.join();
            }
            if (zkServer.canShutdown()) {
                zkServer.shutdown(true);
            }
        } catch (InterruptedException e) {
            // warn, but generally this is ok
            LOG.warn("Server interrupted", e);
        } finally {
            if (txnLog != null) {
                txnLog.close();
            }
        }
    }
```

zk 单机模式启动主要流程：

1. 注册jmx。
2. 解析 ServerConfig 配置对象。
3. 根据配置对象，运行单机zk服务。
4. 创建管理事务日志和快照 FileTxnSnapLog 对象， zookeeperServer 对象，并设置 zkServer 的统计对象。
5. 设置zk服务钩子，原理是通过设置 CountDownLatch，调用 ZooKeeperServerShutdownHandler 的 handle 方法，可以将触发 shutdownLatch.await 方法继续执行，即调用 shutdown 关闭单机服务。
6. 基于 jetty 创建 zk 的 admin 服务。
7. 创建连接对象 cnxnFactory 和 secureCnxnFactory（安全连接才创建该对象），用于处理客户端的请求。
8. 创建定时清除容器节点管理器，⽤于处理容器节点下不存在⼦节点的清理容器节点⼯作等。

可以看到关键点在于解析配置跟启动两个⽅法，先来看下解析配置逻辑，对应上⾯的configure⽅法，org.apache.zookeeper.server.NIOServerCnxnFactory#configure

```java
public void configure(InetSocketAddress addr, int maxcc, boolean secure) throws IOException {
        if (secure) {
            throw new UnsupportedOperationException("SSL isn't supported in NIOServerCnxn");
        }
        configureSaslLogin();

        maxClientCnxns = maxcc;
        // 会话超时时间。
        sessionlessCnxnTimeout = Integer.getInteger(
            ZOOKEEPER_NIO_SESSIONLESS_CNXN_TIMEOUT, 10000);
        // We also use the sessionlessCnxnTimeout as expiring interval for
        // cnxnExpiryQueue. These don't need to be the same, but the expiring
        // interval passed into the ExpiryQueue() constructor below should be
        // less than or equal to the timeout.
        // 过期队列。
        cnxnExpiryQueue =
            new ExpiryQueue<NIOServerCnxn>(sessionlessCnxnTimeout);
        // 过期线程，从cnxnExpiryQueue中读取数据，如果已经过期则关闭。
        expirerThread = new ConnectionExpirerThread();

        // 根据CPU个数计算selector线程的数量。
        int numCores = Runtime.getRuntime().availableProcessors();
        // 32 cores sweet spot seems to be 4 selector threads
        numSelectorThreads = Integer.getInteger(
            ZOOKEEPER_NIO_NUM_SELECTOR_THREADS,
            Math.max((int) Math.sqrt((float) numCores/2), 1));
        if (numSelectorThreads < 1) {
            throw new IOException("numSelectorThreads must be at least 1");
        }

        // 计算worker线程的数量。
        numWorkerThreads = Integer.getInteger(
            ZOOKEEPER_NIO_NUM_WORKER_THREADS, 2 * numCores);
        // worker线程关闭时间。
        workerShutdownTimeoutMS = Long.getLong(
            ZOOKEEPER_NIO_SHUTDOWN_TIMEOUT, 5000);

        LOG.info("Configuring NIO connection handler with "
                 + (sessionlessCnxnTimeout/1000) + "s sessionless connection"
                 + " timeout, " + numSelectorThreads + " selector thread(s), "
                 + (numWorkerThreads > 0 ? numWorkerThreads : "no")
                 + " worker threads, and "
                 + (directBufferBytes == 0 ? "gathered writes." :
                    ("" + (directBufferBytes/1024) + " kB direct buffers.")));
        // 初始化selector线程。
        for(int i=0; i<numSelectorThreads; ++i) {
            selectorThreads.add(new SelectorThread(i));
        }

        this.ss = ServerSocketChannel.open();
        ss.socket().setReuseAddress(true);
        LOG.info("binding to port " + addr);
        ss.socket().bind(addr);
        ss.configureBlocking(false);
        // 初始化accept线程，只有一个，里面会注册监听Accept事件。
        acceptThread = new AcceptThread(ss, addr, selectorThreads);
    }
```

再来看下启动逻辑， org.apache.zookeeper.server.NIOServerCnxnFactory#startup

```java
public void startup(ZooKeeperServer zkServer) throws IOException,
InterruptedException {
startup(zkServer, true);
}

	// 启动分了好几块，一个一个看。
    @Override
    public void startup(ZooKeeperServer zks, boolean startServer)
            throws IOException, InterruptedException {
        start();
        setZooKeeperServer(zks);
        if (startServer) {
            zks.startdata();
            zks.startup();
        }
    }

	@Override
    public void start() {
        stopped = false;
        // 初始化worker线程池。
        if (workerPool == null) {
            workerPool = new WorkerService(
                "NIOWorker", numWorkerThreads, false);
        }
        // 挨个启动select线程。
        for(SelectorThread thread : selectorThreads) {
            if (thread.getState() == Thread.State.NEW) {
                thread.start();
            }
        }
        // ensure thread is started once and only once
        // 启动 acceptThread 线程。
        if (acceptThread.getState() == Thread.State.NEW) {
            acceptThread.start();
        }
        // 启动 expirerThread 线程。
        if (expirerThread.getState() == Thread.State.NEW) {
            expirerThread.start();
        }
    }

```

——>  org.apache.zookeeper.server.ZooKeeperServer#startdata

```java
// 初始化数据结构。
    public void startdata()
    throws IOException, InterruptedException {
        //check to see if zkDb is not null
        // 初始化ZKDatabase，该数据结构用来保存zk上面存储的所有数据。
        if (zkDb == null) {
            // 初始化数据，这里会加入一些原始节点，例如/zookeeper。
            zkDb = new ZKDatabase(this.txnLogFactory);
        }
        // 加载磁盘上已经存储的数据，如果有的话。
        if (!zkDb.isInitialized()) {
            loadData();
        }
    }

    // 启动剩余项目。
    public synchronized void startup() {
        // 初始化session追踪器。
        if (sessionTracker == null) {
            createSessionTracker();
        }
        // 启动session追踪器。
        startSessionTracker();
        // 建立请求处理链路。
        setupRequestProcessors();

        registerJMX();

        setState(State.RUNNING);
        notifyAll();
    }

    // 这⾥可以看出，单机模式下请求的处理链路为：PrepRequestProcessor -> SyncRequestProcessor -> FinalRequestProcessor。
    protected void setupRequestProcessors() {
        RequestProcessor finalProcessor = new FinalRequestProcessor(this);
        RequestProcessor syncProcessor = new SyncRequestProcessor(this,
                finalProcessor);
        ((SyncRequestProcessor)syncProcessor).start();
        firstProcessor = new PrepRequestProcessor(this, syncProcessor);
        ((PrepRequestProcessor)firstProcessor).start();
    }
```

