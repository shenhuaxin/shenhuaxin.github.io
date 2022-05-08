---
title: Windows Terminal连接堡垒机
author: 渡边
date: 2022-05-01
categories: [软件]
tags: [工作效率]
math: true
mermaid: true
image:
path: /commons/devices-mockup.png
width: 800
height: 500
---

windows terminal 现在足够的好看，做为一个终端进行使用。

### 自定义自己的默认终端
打开json文件，设置 defaultProfile 的值为自己想用的终端guid。

![](/assets/img/2022-05-01-window-terminal/34db23c9.png){: width="800" height="500" .left}

### 作为SSH工具使用
1. 首先确保安装了 open ssh 客户端。
![](/assets/img/2022-05-01-window-terminal/639fe00a.png){: width="800" height="500" .left}
2. 使用 ssh 连接远程服务器
![](/assets/img/2022-05-01-window-terminal/ab612bac.png){: width="800" height="500" .left}
3. open ssh 是交互式的连接，所以需要自己输入密码，那么如果想不输入密码，那么我们就需要一些自动化
工具帮忙。在 windows 中没有一些其他在 linux 中好用的工具，有一个 go 语言编写的 expect 工具 + lua 脚本可以帮助我们
进行一些自动化的操作。在登录的时候，脚本就会帮助我们输入密码，免去了我们一步操作。
![](/assets/img/2022-05-01-window-terminal/a93fe6fb.png){: width="800" height="500" .left}
![](/assets/img/2022-05-01-window-terminal/9c09854a.png){: width="800" height="500" .left}

lua脚本如下所示：
```lua
echo(true)
if spawn([[ssh]],"shen@ubuntu01.com") then
    expect("password:")
    echo(false)
    send("shen\r")
    expect("~]$")
    echo(true)
    send("exit\r")
end
```
### 连接堡垒机
连接堡垒机有点特殊，不知为何用 powershell 使用 ssh 连接堡垒机，就会报错。
![](/assets/img/2022-05-01-window-terminal/d4c2d726.png){: width="800" height="500" .left}


所以使用了 Git Bash 连接堡垒机。或许以后 powershell 就可以了，就不用这么麻烦了。使用 gitbash 连接堡垒机命令如下
> D:\programefile\Git\bin\bash.exe -c "ssh 188xxx260@139.xxx.xxx.2 -p60022"

windows terminal 和 lua脚本如下所示：
![](/assets/img/2022-05-01-window-terminal/80efe94f.png){: width="800" height="500" .left}
![](/assets/img/2022-05-01-window-terminal/15822b72.png){: width="800" height="500" .left}
