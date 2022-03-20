---
title: Eureka源码-EurekaServer初始化
author: 渡边
date: 2022-03-19
categories: [Eureka源码]
tags: [eureka]
math: true
mermaid: true
image:
path: /commons/devices-mockup.png
width: 800
height: 500
---

## 1. 源码
首先我们看下 eureka 源码的模块。

![eureka模块](/assets/img/2022-03-19-eureka-server-init/3ef12081.png){: .normal}

archaius 是配置管理组件，jersey 是类似于 springmvc 的 web 框架，governator 是一些实验性的代码。

在这些模块中，eureka-core、eureka-client、eureka-server 是三个重要的模块。


## 2. eureka server 初始化
在 eureka-server 中，主要就是一个 web.xml，在这里面有一个 listener 和五个 filter。listener 是会随着 web 服务器的启动而启动，所以
EurekaBootStrap 就是 eureka 服务端的入口。五个 filter 是对一些请求的拦截。
```xml
  <listener>
    <listener-class>com.netflix.eureka.EurekaBootStrap</listener-class>
  </listener>
```
1. 下面就是 EurekaBootStrap 在web容器启动时被调用的函数。初始化了 eureka 环境 和 eureka server 上下文。
```java
    public void contextInitialized(ServletContextEvent event) {
        try {
            // 初始化eureka环境， datacenter和environment 没有配置，设置默认值
            initEurekaEnvironment();
            // 初始化eureka server上下文
            initEurekaServerContext();

            ServletContext sc = event.getServletContext();
            sc.setAttribute(EurekaServerContext.class.getName(), serverContext);
        } catch (Throwable e) {
            logger.error("Cannot bootstrap eureka server :", e);
            throw new RuntimeException("Cannot bootstrap eureka server :", e);
        }
    }
```
2. initEurekaEnvironment 中初始化 eureka 环境，这里使用到了 archaius ，并没有什么重要的东西，只是对用户覆盖的一些配置进行设置。
```java
    protected void initEurekaEnvironment() throws Exception {
        logger.info("Setting the eureka configuration..");

        String dataCenter = ConfigurationManager.getConfigInstance().getString(EUREKA_DATACENTER);
        if (dataCenter == null) {
            logger.info("Eureka data center value eureka.datacenter is not set, defaulting to default");
            ConfigurationManager.getConfigInstance().setProperty(ARCHAIUS_DEPLOYMENT_DATACENTER, DEFAULT);
        } else {
            ConfigurationManager.getConfigInstance().setProperty(ARCHAIUS_DEPLOYMENT_DATACENTER, dataCenter);
        }
        String environment = ConfigurationManager.getConfigInstance().getString(EUREKA_ENVIRONMENT);
        if (environment == null) {
            ConfigurationManager.getConfigInstance().setProperty(ARCHAIUS_DEPLOYMENT_ENVIRONMENT, TEST);
            logger.info("Eureka environment value eureka.environment is not set, defaulting to test");
        }
    }
```
3. initEurekaServerContext 是一个重要的函数，在这里面进行了 eureka 服务的启动。
   1. 创建 EurekaServerConfig，加载eureka-server.properties
   2. 创建一个 eureka client，用于集群之间的同步
   3. 创建一个 PeerAwareInstanceRegistry，用于感知 eureka 集群的服务实例注册表 PeerAwareInstanceRegistry
   4. 创建一个 PeerEurekaNodes，代表 eureka 集群中的其他节点
   5. 创建一个 EurekaServerContext, 代表了 eureka server 上下文。
   6. 通过 eureka 集群注册表 registry 从其他 eureka server 节点获取服务注册信息
   7. 注册 eureka monitor

## 3. 创建 eurekaServerConfig
在创建 eurekaServerConfig 时，会从 eureka-server.properties 中读取配置项。ConfigurationManager 会读取配置信息，后面也是从
ConfigurationManager 中获取配置信息。
```java
    private void init() {
        String env = ConfigurationManager.getConfigInstance().getString(
                EUREKA_ENVIRONMENT, TEST);
        ConfigurationManager.getConfigInstance().setProperty(
                ARCHAIUS_DEPLOYMENT_ENVIRONMENT, env);
        String eurekaPropsFile = EUREKA_PROPS_FILE.get();
        try {
            // 读取 配置文件中的配置项。
            ConfigurationManager
                    .loadCascadedPropertiesFromResources(eurekaPropsFile);
        } catch (IOException e) {
        }
    }
```

## 4. 创建 eureka client
首先创建了一个 EurekaInstanceConfig，读取 eureka-client.properties，第二步根据 instanceConfig 创建一个 InstanceInfo 用于代表
当前的 eureka server。需要注意的是，在集群模式下， 一个 eureka server 也是一个 eureka client。最后，创建一个 DiscoveryClient，
```java
    // eureka-client.properties创建EurekaInstanceConfig
    EurekaInstanceConfig instanceConfig = isCloud(ConfigurationManager.getDeploymentContext())
            ? new CloudInstanceConfig()
            : new MyDataCenterInstanceConfig();

    // 基于instanceConfig创建InstanceInfo, 根据instanceConfig和InstanceInfo创建ApplicationInfoManager
    applicationInfoManager = new ApplicationInfoManager(
            instanceConfig, new EurekaConfigBasedInstanceInfoProvider(instanceConfig).get());

    EurekaClientConfig eurekaClientConfig = new DefaultEurekaClientConfig();
    // 创建EurekaClient
    eurekaClient = new DiscoveryClient(applicationInfoManager, eurekaClientConfig);
```
### 4.1. 创建 DiscoveryClient
创建 DiscoveryClient走的是下面这个构造函数，在这个构造函数中，做了很多事情。
`DiscoveryClient#DiscoveryClient(ApplicationInfoManager, EurekaClientConfig, AbstractDiscoveryClientOptionalArgs, Provider<BackupRegistry>)`
下面就来看看主要做的几件事。

1. 创建一个支持调度的线程池，scheduler
2. 创建一个用于心跳定时任务的调度器，cacheRefreshExecutor
3. 创建一个用于缓存刷新任务的调度器，heartbeatExecutor
4. 创建一个 eureka 用于网络传输组件
5. 从其他 eureka server 中拉取注册表
6. 初始化定时任务，这里有三个任务，缓存刷新任务，定时心跳任务。使用 scheduler调度心跳和缓存刷新的定时任务的定时执行，
用各自的executor用于执行任务。并且会向其他 eureka server 进行注册
```java
    try {
        // default size of 2 - 1 each for heartbeat and cacheRefresh
        // 支持调度的线程池
        scheduler = Executors.newScheduledThreadPool(2,
                new ThreadFactoryBuilder()
                        .setNameFormat("DiscoveryClient-%d")
                        .setDaemon(true)
                        .build());
        // 支持心跳的线程池
        heartbeatExecutor = new ThreadPoolExecutor(
                1, clientConfig.getHeartbeatExecutorThreadPoolSize(), 0, TimeUnit.SECONDS,
                new SynchronousQueue<Runnable>(),
                new ThreadFactoryBuilder()
                        .setNameFormat("DiscoveryClient-HeartbeatExecutor-%d")
                        .setDaemon(true)
                        .build()
        );  // use direct handoff
        // 支持缓存刷新的线程池
        cacheRefreshExecutor = new ThreadPoolExecutor(
                1, clientConfig.getCacheRefreshExecutorThreadPoolSize(), 0, TimeUnit.SECONDS,
                new SynchronousQueue<Runnable>(),
                new ThreadFactoryBuilder()
                        .setNameFormat("DiscoveryClient-CacheRefreshExecutor-%d")
                        .setDaemon(true)
                        .build()
        );  // use direct handoff

         // EurekaTransport, 初始化eureka网络传输组件
        eurekaTransport = new EurekaTransport();
        scheduleServerEndpointTask(eurekaTransport, args);

    } catch (Throwable e) {
    }

    // 拉取注册表信息，如果没有成功， 从备用的注册表地址中拉取
    if (clientConfig.shouldFetchRegistry() && !fetchRegistry(false)) {
        fetchRegistryFromBackup();
    }

    // call and execute the pre registration handler before all background tasks (inc registration) is started
    if (this.preRegistrationHandler != null) {
        this.preRegistrationHandler.beforeRegistration();
    }
    // 初始化调度任务
    initScheduledTasks();
```


## 5. 创建服务注册表 registry
根据 eurekaServerConfig、eurekaClientConfig、eurekaClient 创建了 PeerAwareInstanceRegistry。
```java
    registry = new PeerAwareInstanceRegistryImpl(
            eurekaServerConfig,
            eurekaClient.getEurekaClientConfig(),
            serverCodecs,
            eurekaClient
    );
```

## 6. 创建 PeerEurekaNodes
peerEurekaNodes 代表了 eureka server 集群中的其他节点，使用到了 registry、applicationInfoManager。
```java
    PeerEurekaNodes peerEurekaNodes = getPeerEurekaNodes(
            registry,
            eurekaServerConfig,
            eurekaClient.getEurekaClientConfig(),
            serverCodecs,
            applicationInfoManager
    );
```

## 7. 创建 serverContext 并初始化
这里最重要的就是 `serverContext.initialize()` 这行代码。 首先根据配置的 serviceUrl 创建 PeerEurekaNode 集合，并且创建一个10分钟
更新一次集群节点的定时任务。然后会创建一个15分钟更新一次心跳阈值的定时任务。
```java
    serverContext = new DefaultEurekaServerContext(
            eurekaServerConfig,
            serverCodecs,
            registry,
            peerEurekaNodes,
            applicationInfoManager
    );
    // ServerContext的Holder， 用于持有serverContext，后面好拿出来使用。
    EurekaServerContextHolder.initialize(serverContext);

    serverContext.initialize();
```

## 8. 初始化服务注册表
从其他 eureka 节点中获取客户端实例，并且添加到注册表中。然后将自身的状态设置为 UP，开始接收外部的服务请求。在 web.xml 中的 StatusFilter，
使用到了这个状态，只有 eureka server 状态为 UP 时，才会对请求进行放行。否则返回服务还未准备好，请尝试其他的 server 节点。
```java
    // 同步实例信息
    int registryCount = registry.syncUp();
    // 开放注册
    registry.openForTraffic(applicationInfoManager, registryCount);
```


## 9. 注册 eureka 监控器
```java
    EurekaMonitors.registerAllStats();
```




## 10. 小结
在 eureka server 启动的时候， 创建了 eureka client 用于获取获取其他 eureka server 中的信息和将自身注册到其他的 eureka server 中。
创建了注册表 registry 用于存储服务实例。创建了定时任务，用于集群间的有清除缓存任务、发送心跳任务、更新集群节点任务。用于处理客户端的更新心跳阈值定时任务。

还有两个概念需要注意。Applications 代表了注册到 eureka server 中的所有服务，Application 代表的是某一个服务，InstanceInfo 代表的是
某一个服务下的一个实例节点。还有就是 eureka client 中用来 http 通讯的 EurekaHttpClient，使用了装饰器模式，对 jersey 进行了各种装饰。
有指标统计、重试、重定向、定时重建session这四个。











