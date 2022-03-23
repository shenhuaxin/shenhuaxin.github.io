---
title: 记录一次SQL超时问题
author: 渡边
date: 2022-03-21
categories: [mysql]
tags: [mysql]
math: true
mermaid: true
image:
path: /commons/devices-mockup.png
width: 800
height: 500
---

## 1. 背景
最近，在迁移某个环境，遇到了一次SQL超时问题，记录一下。

## 2. 现象
开始我们遇到的情况如图所示，显示一个 update sql 因为超时而被取消了，但是这是一条根据主键更新的SQL, 按道理来说不会很慢，
所以猜测是不是在这个事务中，有其他耗时长的 sql 或者是因为这条记录被锁住了。
![](/assets/img/2022-03-21-error-transacation/2f942b73.png)

## 3.分析
后续在对不同的数据进行相似操作时发现，第一次操作会报 NoClassDefFoundError ，后续进行 update 都超时。这时候就判断是否是事务未提交。但是
更新后的数据却能够在页面上查询的到。但是数据库中查询不到。
![](/assets/img/2022-03-21-error-transacation/87062065.png)

然后使用 `SELECT * FROM INFORMATION_SCHEMA.INNODB_LOCKS;` 对锁信息进行查询， 发现确实出现了锁等待的情况。
![](/assets/img/2022-03-21-error-transacation/8df080b7.png)

这时候猜测应该是事务未提交，所以第一次占有行锁未释放，第二次获取行锁失败，所以导致超时，代码经过简化后如图：

![](/assets/img/2022-03-21-error-transacation/eaa219bf.png)

我们看到这里使用的是手动控制事务，并且 rollback 是在 catch(Exception) 代码块中，而之前报的错是一个 Error ，所以导致的是事务未进行提交。
而后续的操作则因为获取不到锁而失败。

> @Transactional 注解默认会回滚 RuntimeException 和 Error
{: .prompt-tip }

那么为什么事务未提交，却能在页面上查询到更新后的数据呢，这是因为数据库连接池中默认只有一条连接，所以能查到之前未提交的数据。
