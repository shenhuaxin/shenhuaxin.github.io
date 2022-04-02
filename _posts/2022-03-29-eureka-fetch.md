---
title: Eureka源码04-Eureka 抓取注册表流程
author: 渡边
date: 2022-03-29
categories: [Eureka源码]
tags: [eureka]
math: true
mermaid: true
image:
path: /commons/devices-mockup.png
width: 800
height: 500
---



在 DiscoveryClient 中有一段抓取服务注册表的代码， 当时没有深入， 现在继续看看。
```java
        if (clientConfig.shouldFetchRegistry() && !fetchRegistry(false)) {
            fetchRegistryFromBackup();
        }
```
如果需要抓取注册表， 那么就进行一次注册表的抓取， 抓取失败了， 从备用的地址中继续抓取。

深入看下 fetchRegistry 函数干了什么？
```java
    private boolean fetchRegistry(boolean forceFullRegistryFetch) {
        // 拉取注册表
        Stopwatch tracer = FETCH_REGISTRY_TIMER.start();
        try {
            Applications applications = getApplications();

            if (clientConfig.shouldDisableDelta()
                    || (!Strings.isNullOrEmpty(clientConfig.getRegistryRefreshSingleVipAddress()))
                    || forceFullRegistryFetch
                    || (applications == null)
                    || (applications.getRegisteredApplications().size() == 0)
                    || (applications.getVersion() == -1)) //Client application does not have latest library supporting delta
            {
                // 拉取全量注册表
                getAndStoreFullRegistry();
            } else {
                // 增量更新
                getAndUpdateDelta(applications);
            }
            // 设置一致性hash值
            applications.setAppsHashCode(applications.getReconcileHashCode());
            logTotalInstances();
        } finally {
            if (tracer != null) {
                tracer.stop();
            }
        }
        // Notify about cache refresh before updating the instance remote status
        onCacheRefreshed();
        // Update remote status based on refreshed data held in the cache
        updateInstanceRemoteStatus();
        // registry was fetched successfully, so return true
        return true;
    }
```

1. 先从本地获取一下 Applications 缓存，
1. 如果本地有， 进行一次增量的注册表更新
1. 如果本地没有， 进行一次全量的注册表拉取。
1. 计算 applications 的 一致性 hash code。


全量抓取注册表， 调用的是  /apps 的请求 ， 对应了 ApplicationsResource.getContainers 方法， 删掉一些不重要的函数，代码如下
```java
	public Response getContainers(@PathParam("version") String version,
                                  @HeaderParam(HEADER_ACCEPT) String acceptHeader,
                                  @HeaderParam(HEADER_ACCEPT_ENCODING) String acceptEncoding,
                                  @HeaderParam(EurekaAccept.HTTP_X_EUREKA_ACCEPT) String eurekaAccept,
                                  @Context UriInfo uriInfo,
                                  @Nullable @QueryParam("regions") String regionsStr) {
        // 缓存key
        Key cacheKey = new Key(Key.EntityType.Application,
                ResponseCacheImpl.ALL_APPS,
                keyType, CurrentRequestVersion.get(), EurekaAccept.fromString(eurekaAccept), regions
        );

        Response response;
        if (acceptEncoding != null && acceptEncoding.contains(HEADER_GZIP_VALUE)) {
            response = Response.ok(responseCache.getGZIP(cacheKey))
                    .header(HEADER_CONTENT_ENCODING, HEADER_GZIP_VALUE)
                    .header(HEADER_CONTENT_TYPE, returnMediaType)
                    .build();
        } else {
            response = Response.ok(responseCache.get(cacheKey))
                    .build();
        }
        return response;
    }
```
发现 是从  responseCache 中取出 ALL_APPS 的数据。然后把数据缓存到 Eureka Client本地。


