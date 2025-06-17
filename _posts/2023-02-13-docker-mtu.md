---
title: docker https 接口访问不通
author: 渡边
date: 2023-02-13
categories: [docker]
tags: [docker,https]
math: true
mermaid: true
image:
path: /commons/devices-mockup.png
width: 800
height: 500
---


### 问题
在容器中访问 https 接口不通，但是宿主机上访问是通的。通过 curl -v 再次访问，发现是握手失败。

经过查阅，发现是 docker 的 mtu 设置的问题，在宿主机上的 mtu 为 1450， 但是在容器中是 1500。

如果docker的网卡mtu大于宿主机的网卡mtu，在大数据包传输时可能会丢失数据。这也是造成了通信到一半就中断的原因。


解决方法：
1. 在 daemon.json 中设置 mtu
```java
{
    "mtu": 1450
}
```
注： 如果有 live-restore: true 那么可能导致 docker 容器未重启，那么 docker0 的 mtu 一直未能变更，删除改配置后有效。
