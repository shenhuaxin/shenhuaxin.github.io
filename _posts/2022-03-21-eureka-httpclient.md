---
title: Eureka源码02-EurekaHttpClient装饰模式的使用
author: 渡边
date: 2022-03-21
categories: [Eureka源码]
tags: [eureka]
math: true
mermaid: true
image:
path: /commons/devices-mockup.png
width: 800
height: 500
---

## 1. EurekaTransport
EurekaTransport 持有了用于 http 通讯的组件，用于向 eureka server 发送服务注册等信息的有 registrationClient，用于向 eureka server
查询服务信息的有 queryClient。bootstrapResolver 这个是用于对 `service-url.defaultZone` 这个配置或者其他的 zone 进行解析。
```java
     private static final class EurekaTransport {
         private ClosableResolver bootstrapResolver;
         private TransportClientFactory transportClientFactory;

         private EurekaHttpClient registrationClient;
         private EurekaHttpClientFactory registrationClientFactory;

         private EurekaHttpClient queryClient;
         private EurekaHttpClientFactory queryClientFactory;
    }
```

## 2. ClosableResolver
ClosableResolver 的实现类为 ConfigClusterResolver，在 getClusterEndpoints -> getClusterEndpointsFromConfig -> EndpointUtils.getServiceUrlsMapFromConfig
-> EurekaClientConfig.getEurekaServerServiceUrls 从配置文件中获取到了 eureka 集群信息。
```java
    public List<String> getEurekaServerServiceUrls(String myZone) {
        String serviceUrls = configInstance.getStringProperty(
                namespace + CONFIG_EUREKA_SERVER_SERVICE_URL_PREFIX + "." + myZone, null).get();
        if (serviceUrls == null || serviceUrls.isEmpty()) {
            serviceUrls = configInstance.getStringProperty(
                    namespace + CONFIG_EUREKA_SERVER_SERVICE_URL_PREFIX + ".default", null).get();

        }
        if (serviceUrls != null) {
            return Arrays.asList(serviceUrls.split(","));
        }

        return new ArrayList<String>();
    }
```

## 3. registrationClient
如果`registration.enabled = true`则需要向 eureka server 进行注册，那么就会创建 registrationClient。
```java
    if (clientConfig.shouldRegisterWithEureka()) {
        EurekaHttpClientFactory newRegistrationClientFactory = null;
        EurekaHttpClient newRegistrationClient = null;
        try {
            newRegistrationClientFactory = EurekaHttpClients.registrationClientFactory(
                    eurekaTransport.bootstrapResolver,
                    eurekaTransport.transportClientFactory,
                    transportConfig
            );
            newRegistrationClient = newRegistrationClientFactory.newClient();
        } catch (Exception e) {
            logger.warn("Transport initialization failure", e);
        }
        eurekaTransport.registrationClientFactory = newRegistrationClientFactory;
        eurekaTransport.registrationClient = newRegistrationClient;
    }
```
newRegistrationClientFactory 用于创建 newRegistrationClient。

```java
    static EurekaHttpClientFactory canonicalClientFactory(final String name,
                                                          final EurekaTransportConfig transportConfig,
                                                          final ClusterResolver<EurekaEndpoint> clusterResolver,
                                                          final TransportClientFactory transportClientFactory) {

        return new EurekaHttpClientFactory() {
            @Override
            public EurekaHttpClient newClient() {
                return new SessionedEurekaHttpClient(
                        name,
                        RetryableEurekaHttpClient.createFactory(
                                name,
                                transportConfig,
                                clusterResolver,
                                RedirectingEurekaHttpClient.createFactory(transportClientFactory),
                                ServerStatusEvaluators.legacyEvaluator()),
                        transportConfig.getSessionedClientReconnectIntervalSeconds() * 1000
                );
            }

            @Override
            public void shutdown() {
                wrapClosable(clusterResolver).shutdown();
            }
        };
    }
```
我们可以看到当调用 factory.newClient 函数时，最终创建的是一个 SessionedEurekaHttpClient。至于 RetryableEurekaHttpClient 和
RedirectingEurekaHttpClient 则是在执行 execute 函数时创建的。

现在我们看到了三种代理的 EurekaHttpClient，但是 MetricsCollectingEurekaHttpClient 是在哪里创建的呢？

在创建 RedirectingEurekaHttpClient 的时候，最终会使用到 eurekaTransport.transportClientFactory，也就是 Jersey1TransportClientFactories。

```java
    public TransportClientFactory newTransportClientFactory(final Collection<ClientFilter> additionalFilters,
                                                                   final EurekaJerseyClient providedJerseyClient) {
        ApacheHttpClient4 apacheHttpClient = providedJerseyClient.getClient();
        if (additionalFilters != null) {
            for (ClientFilter filter : additionalFilters) {
                if (filter != null) {
                    apacheHttpClient.addFilter(filter);
                }
            }
        }

        final TransportClientFactory jerseyFactory = new JerseyEurekaHttpClientFactory(providedJerseyClient, false);
        final TransportClientFactory metricsFactory = MetricsCollectingEurekaHttpClient.createFactory(jerseyFactory);

        return new TransportClientFactory() {
            @Override
            public EurekaHttpClient newClient(EurekaEndpoint serviceUrl) {
                return metricsFactory.newClient(serviceUrl);
            }

            @Override
            public void shutdown() {
                metricsFactory.shutdown();
                jerseyFactory.shutdown();
            }
        };
    }
```
{: file='Jersey1TransportClientFactories'}
至此，我们可以得到 EurekaHttpClient 的调用顺序，SessionedEurekaHttpClient -> RetryableEurekaHttpClient -> RedirectingEurekaHttpClient
-> MetricsCollectingEurekaHttpClient -> JerseyApplicationClient

## 4. EurekaHttpClientDecorator
就以从实现 EurekaHttpClient 接口的 register 函数举例，将 register 的行为下放到了实现类。在子类中，当调用 RequestExecutor#execute 时，
会委托给代理进行 register，这样的话装饰器可以在实际调用函数前后做一些装饰操作。
```java
    protected abstract <R> EurekaHttpResponse<R> execute(RequestExecutor<R> requestExecutor);

    @Override
    public EurekaHttpResponse<Void> register(final InstanceInfo info) {
        return execute(new RequestExecutor<Void>() {
            @Override
            public EurekaHttpResponse<Void> execute(EurekaHttpClient delegate) {
                return delegate.register(info);
            }

            @Override
            public RequestType getRequestType() {
                return RequestType.Register;
            }
        });
    }
```

## 5. EurekaHttpClientDecorator实现类的作用

* MetricsCollectingEurekaHttpClient： 对不同的请求结果进行统计。
* RedirectingEurekaHttpClient：在创建 client 的时候，会进行一次请求，如果返回结果需要进行重定向，就会根据重新重定向地址重新生成一个client
* RetryableEurekaHttpClient：如果请求失败，则会换一个 eureka server 进行重试
* SessionedEurekaHttpClient： 超过一定时间没有使用过这个 client，就会重新创建一个新的 client。防止一直使用相同的 client。
