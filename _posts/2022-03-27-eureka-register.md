---
title: Eureka源码03-Eureka服务注册流程
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

eureka server 接受 eureka client 服务注册的请求， 在 eureka-core 项 com/netflix/eureka/resources 包下。

有一个 ApplicationsResource 类。
```java
    @Path("{appId}")
    public ApplicationResource getApplicationResource(
            @PathParam("version") String version,
            @PathParam("appId") String appId) {
        CurrentRequestVersion.set(Version.toEnum(version));
        // 没有加资源指示器， 是一个子资源定位器
        return new ApplicationResource(appId, serverConfig, registry);
    }
```
将 {appId} 请求路径下的所有 Http 请求， 都交给 ApplicationResource 这个类去处理。

而服务注册就在下面这段代码中。
```java
public Response addInstance(InstanceInfo info,
                                @HeaderParam(PeerEurekaNode.HEADER_REPLICATION) String isReplication) {
        registry.register(info, "true".equals(isReplication));
        return Response.status(204).build();  // 204 to be backwards compatible
    }
```

```java
    public void register(final InstanceInfo info, final boolean isReplication) {
        int leaseDuration = Lease.DEFAULT_DURATION_IN_SECS;
        if (info.getLeaseInfo() != null && info.getLeaseInfo().getDurationInSecs() > 0) {
            leaseDuration = info.getLeaseInfo().getDurationInSecs();
        }
        super.register(info, leaseDuration, isReplication);
        replicateToPeers(Action.Register, info.getAppName(), info.getId(), info, null, isReplication);
    }
```

1. 调用服务的注册方法
1. 将注册信息同步到其他的 Eureka Server 上。

我们再来看看父类中的 register 函数。
```java
    public void register(InstanceInfo registrant, int leaseDuration, boolean isReplication) {
        try {
            read.lock();
            // 通过服务名称拿到服务的实例Map
            Map<String, Lease<InstanceInfo>> gMap = registry.get(registrant.getAppName());
            REGISTER.increment(isReplication);
            if (gMap == null) {
                final ConcurrentHashMap<String, Lease<InstanceInfo>> gNewMap = new ConcurrentHashMap<String, Lease<InstanceInfo>>();
                gMap = registry.putIfAbsent(registrant.getAppName(), gNewMap);
                if (gMap == null) {
                    gMap = gNewMap;
                }
            }
            Lease<InstanceInfo> existingLease = gMap.get(registrant.getId());

            Lease<InstanceInfo> lease = new Lease<InstanceInfo>(registrant, leaseDuration);
            // 加入到服务注册表中
            gMap.put(registrant.getId(), lease);
            // 保存最近注册服务的队列
            synchronized (recentRegisteredQueue) {
                recentRegisteredQueue.add(new Pair<Long, String>(
                        System.currentTimeMillis(),
                        registrant.getAppName() + "(" + registrant.getId() + ")"));
            }
            // If the lease is registered with UP status, set lease service up timestamp
            if (InstanceStatus.UP.equals(registrant.getStatus())) {
                lease.serviceUp();
            }
            registrant.setActionType(ActionType.ADDED);
            recentlyChangedQueue.add(new RecentlyChangedItem(lease));
            registrant.setLastUpdatedTimestamp();

            // 主动过期掉 readwriteMap中 对应的 服务的信息 和 ALL_APPS
            invalidateCache(registrant.getAppName(), registrant.getVIPAddress(), registrant.getSecureVipAddress());
        } finally {
            read.unlock();
        }
    }
```

1. 往注册表中添加这个新的实例
1. 更新新的期望心跳值和心跳阈值
1. 主动过期掉 读写缓存 中对应的一些数据
