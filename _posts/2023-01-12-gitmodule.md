---
title: 使用 git module 集成前后端项目
author: 渡边
date: 2023-01-12
categories: [git]
tags: [git]
math: true
mermaid: true
image:
path: /commons/devices-mockup.png
width: 800
height: 500
---


### 需求
最近有个项目需要在流水线中集成前后端项目一起部署

使用流水线部署时，怎么同时拉取前后端项目，避免本地打包前端项目放到后端项目中。

### 解决
研究后发现使用 gitmodules 特别好处理这个问题。

在后端项目中，使用以下命令将前端项目添加进后端项目的子模块中。
> git submodule add https://codeup.aliyun.com/xxxx.git xxxx

本地也会生成一个 .gitmodules 文件，可以在该文件中添加 branch = dev ，设置子模块的分支。

```git
[submodule "xxxx"]
	path = xxxx
	url = https://codeup.aliyun.com/xxxx.git
	branch = dev
```
也可以使用以下命令设置分支
> git config -f .gitmodules submodule.xxxx.branch dev


### 后续
后续发现在流水线使用 sh 命令更加方便。
> git clone https://codeup.aliyun.com/xxxx.git
>
还可以增加一个 . 号，让拉取的代码直接放在当前文件夹
> git clone https://codeup.aliyun.com/xxxx.git .
