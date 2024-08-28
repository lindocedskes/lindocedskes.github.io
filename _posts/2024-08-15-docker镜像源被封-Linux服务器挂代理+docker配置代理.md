---
title: docker镜像源被封-Linux服务器挂代理+docker配置代理
description: 解决docker镜像源被封问题
author: #有默认值
date: 2024-08-15 12:13:14 +0800
categories:  [tools, Demo]
tags:  [docker,linux]     # TAG names should always be lowercase
pin:  # 默认false，可填true
math: true
mermaid: true
image:
  path: /assets/bar/backimg.png
  lqip: data:image/webp;base64,UklGRpoAAABXRUJQVlA4WAoAAAAQAAAADwAABwAAQUxQSDIAAAARL0AmbZurmr57yyIiqE8oiG0bejIYEQTgqiDA9vqnsUSI6H+oAERp2HZ65qP/VIAWAFZQOCBCAAAA8AEAnQEqEAAIAAVAfCWkAALp8sF8rgRgAP7o9FDvMCkMde9PK7euH5M1m6VWoDXf2FkP3BqV0ZYbO6NA/VFIAAAA
  alt:  # 图片名
---

## 一、服务器挂代理

1. 安装：https://github.com/wnlen/clash-for-linux（本人ubuntu）

2. 访问UI 管理界面 : 

+ http://<ip>:9090/ui/ （防火墙开启9090，可以访问）
+ 推荐：先使用**本地端口转发**（可以使用vscode插件 remote ssh 插件的 port forward 功能等），再访问 http://127.0.0.1:9090/ui/ <img src="../assets/img/posts24/2024-08-16-19-39-23.png" alt="image-20240810105939148" style="zoom: 25%;" />
3. 登录：

   在`API Base URL`一栏中输入：http://<ip>:9090 ，在`Secret(optional)`一栏中输入启动成功后输出的Secret。
   登录问题：
   [无法登录 http://127.0.0.1:9090/ui/](https://github.com/wnlen/clash-for-linux/issues/18)

   多执行 sudo bash start.sh 和 source /etc/profile.d/clash.sh ，或restart.sh等，`API Base URL`一定不要输入错，每次start.sh后都会重新生成对应Secret

4. 测试：
   执行proxy_on 命令后用**curl或wget**命令测试github、google等网站，ping命令走ICMP协议，本身不通过代理。
   curl http://google.com

5. 更新订阅
   重新 sudo bash start.sh ，restart无效

## 二、docker设置代理

对于刚安装的docker 默认官方源，`wget`通过代理服务器连接到Docker Hub，而`docker pull`没有配置代理。你可以在Docker的配置中添加代理设置。

1. 编辑 `/etc/systemd/system/docker.service.d/http-proxy.conf`(如果没有此文件，可以创建):添加以下内容：

   ```ini
   [Service]
   Environment="HTTP_PROXY=http://127.0.0.1:7890"
   Environment="HTTPS_PROXY=http://127.0.0.1:7890"
   ```

2. 保存后，重新加载Docker服务配置并重启Docker服务：

   ```shell
   sudo systemctl daemon-reload
   sudo systemctl restart docker
   ```



原理：为什么服务器设置了代理，Docker还需要配置？
          在服务器上设置了代理时，通常这个代理配置只影响使用标准网络库的程序（如`wget`、`curl`等），而对于Docker这样的服务进程，它不会自动继承或使用这些系统级的代理设置。因此，Docker需要单独配置代理。


![](/assets/img/posts24/2024-08-16-19-39-42.png)