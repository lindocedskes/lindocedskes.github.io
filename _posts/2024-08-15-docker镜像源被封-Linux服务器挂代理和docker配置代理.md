---
title: Linux挂clash代理+docker配置代理
description: Linux挂clash代理、全局代理、解决docker镜像源被封问题
author: #有默认值
date: 2024-08-15 12:13:14 +0800
categories:  [tools, Demo]
tags:  [docker,linux]     # TAG names should always be lowercase
pin:  # 默认false，可填true
math: true
mermaid: true
image:
  path: /assets/bar/clashMeta.png
  lqip: data:image/webp;base64,UklGRpoAAABXRUJQVlA4WAoAAAAQAAAADwAABwAAQUxQSDIAAAARL0AmbZurmr57yyIiqE8oiG0bejIYEQTgqiDA9vqnsUSI6H+oAERp2HZ65qP/VIAWAFZQOCBCAAAA8AEAnQEqEAAIAAVAfCWkAALp8sF8rgRgAP7o9FDvMCkMde9PK7euH5M1m6VWoDXf2FkP3BqV0ZYbO6NA/VFIAAAA
  alt:  # 图片名
---

## 实验室服务器挂代理 （无root权限用户）
### 需求说明
linux无sudo权限以及非root的用户，用apt安装软件

`一般来说使用apt安装软件，必须要有root权限，因为apt安装时需要写/usr/bin、/usr/share等目录，而这些目录只有root用户(或有sudo权限)才有写入权限的，所以没有sudo权限的普通用户，要用apt安装软件的话，就只能以源码安装方式来安装了`
具体操作如下:

1. 获取源代码；常用`wget/curl`下载，也可以用类似`apt-get source`的方式获取仓库中软件源代码；
2. 解压源代码安装包；例如对于gzip格式的tar包：`tar -zxvf xxx.tar.gz`；
#### 一般过程：使用wget 下载源代码安装包后编译安装:

下载源码，编译安装

```shell
#1.获取压缩包 wget
wget https://XXXX.tar.gz 
#2.解压，进入文件
tar -zxvf XXXX.tar.gz 
cd XXXX
```

```shell
#3.设置安装路径，并编译，安装
./configure --prefix=/home/your_username/install # /home/username/install/是我放置的位置，该目录下有个bin
make -j && make install
```


### 第一步：设置非root用户命令的安装路径 
> 具有root权限的用户直接跳过
{: .prompt-tip}
2. 设置 `PATH`（只需要第一次的时候设置）:

`vim ~/.bashrc` ，目的是指明用户安装的位置为 `your_username/install` 安装好的命令路径在对应`/bin`位置

```shell
export PATH=/home/username/install/bin:$PATH
```

保存并退出编辑器，然后执行以下命令使更改生效：

```shell
source ~/.bashrc
```

### 第二步：服务器安装clash

视频教程：https://www.youtube.com/watch?v=0OmO5_HsVPU  6:30 开始看 （一般的安装）

**下面是我自己写的步骤：** 适合偏向非root用户

[Clash Meta](https://github.com/MetaCubeX/mihomo/releases/latest)  改名为mihomo

根据官网教程下载系统对应文件：（实验室服务器是debian  x86）
https://wiki.metacubex.one/startup/#__tabbed_1_2 
如：mihomo-linux-amd64.deb

由于没有 root 权限，使用 `dpkg-deb` 命令手动解压 `.deb` 包，然后将文件移动到你指定的目录（！！！）

```shell
#创建一个临时目录来解压 .deb 包
mkdir ~/temp_deb
# 使用 dpkg-deb 命令解压 .deb 包：
dpkg-deb -x mihomo-linux-amd64.deb ~/temp_deb
# 将解压后的文件移动到你指定的安装目录
mv ~/temp_deb/* /home/your_username/install/
# 将解压后的文件移动到合适的位置，由于目标目录中已经存在同名文件或目录，使用 cp 命令合并目录内容 
cp -r /home/your_username/install/usr/* /home/your_username/install/
mv /home/your_username/install/etc/* /home/your_username/install/
#（若之前已经写过path环境变量可跳过）确保 bin 目录中的可执行文件在你自定义的 PATH 环境变量中
export PATH=/home/your_username/install/bin:$PATH

#运行程序
mihomo
```

/home/your_username/install/bin 是我自定义的非root用户 指定命令安装的位置。

将你的机场得到的配置文件 config.yaml 复制到 /home/your_username/install/etc/mihomo:

```
cp config.yaml /home/your_username/install/etc/mihomo
```

在config.yam 中添加控制面板 ui  

```yaml
# 控制面板
external-controller: 0.0.0.0:9090 # 访问端口
secret: "sdvwa@4af" # 控制面板设置的密码
external-ui: ui
external-ui-url: "https://github.com/MetaCubeX/metacubexd/archive/refs/heads/gh-pages.zip"
```

发现页面空白，检查后ui文件夹下没东西。将https://github.com/MetaCubeX/metacubexd/archive/refs/heads/gh-pages.zip 手动下载 解压放入  ，并改名为ui

>  /home/your_username/install/etc/mihomo/ui

编辑systemd 配置文件 /home/your_username/instal/etc/systemd/system/mihomo.servic`

注意修改 `ExecStart` 以匹配你自定义的安装位置。

```
[Unit]
Description=mihomo Daemon, Another Clash Kernel.
After=network.target NetworkManager.service systemd-networkd.service iwd.service

[Service]
Type=simple
LimitNPROC=500
LimitNOFILE=1000000
CapabilityBoundingSet=CAP_NET_ADMIN CAP_NET_RAW CAP_NET_BIND_SERVICE CAP_SYS_TIME CAP_SYS_PTRACE CAP_DAC_READ_SEARCH CAP_DAC_OVERRIDE
AmbientCapabilities=CAP_NET_ADMIN CAP_NET_RAW CAP_NET_BND_SERVICE CAP_SYS_TIME CAP_SYS_PTRACE CAP_DAC_READ_SEARCH CAP_DAC_OVERRIDE
Restart=always
ExecStartPre=/usr/bin/sleep 1s
ExecStart=/home/luoyutao24/install/bin/mihomo -d /home/your_username/install/etc/mihomo
ExecReload=/bin/kill -HUP $MAINPID

[Install]
WantedBy=multi-user.target
```

**手动运行**-control c 关闭进程时关闭，不建议一直开着，实验室服务器还是按需开比较安全。

```
/home/your_username/install/bin/mihomo -d /home/your_username/install/etc/mihomo
```

访问控制面板：

```
http://127.0.0.1:9090/ui/#/setup
```

让linux 走代理端口：7890

```
export ALL_PROXY=http://127.0.0.1:7890
```

ALL_PROXY 通常包括包括 HTTP、HTTPS、FTP 等的流量。

按照rule 规则匹配。线路选择通过控制面板 切换国外流量线路。

docker 应该就走7890的代理流量了，应该不用再配置（还没试过）

> 下面是较旧的内容，在云服务器上安装，尝试使用clash-core 内核+ yacd 的web ui 控制面板 来使用，但clash-core内核源作者删库，来源于第三方备份的版本，不如使用上面的由 [Clash Meta](https://github.com/MetaCubeX/mihomo/releases/latest)  改名为mihomo 在维护的内核。
{: .prompt-warning}

## 一、服务器挂代理

参考教程：https://blog.myxuechao.com/post/36
### 1. 下载安装包
[clash-core](https://github.com/Kuingsmile/clash-core?tab=readme-ov-file) 在 releases中，备份仓库，源clash作者已经删库
   linux 相关的有 3 个安装包:
   + `386`，指的是i386指的是intel80386,32位架构
   + `amd64`，指的是amd的64位架构，新的指令集，支持64位系统
   + `amd64-v3，`指的是使用 golang v3 环境变量，环境变量版本越高,兼容性越差,但性能可能因使用新指令而得到提升

> linux **32 位系统使用 `-386`, 64 位系统默认使用 `-amd64-v3`，如果存在不兼容的情况在使用`-amd64`**
{: .prompt-info }

### 2. 安装包位置

   ```shell
   cd /root
   mkdir clash
   # 下载好文件后直接放在 clash 文件夹里面,我自己下了 clash-linux-amd64-v1.18.0.gz在终端执行 `gunzip  clash-linux-amd64-latest.gz` ，解压.gz文件。
   ```

   2.1 在终端执行 `gunzip  clash-linux-amd64-v1.18.0.gz` ，解压.gz文件

   2.2 将解压后的文件重命名为 `clash`。

   ```shell
   mv clash-linux-amd64-v1.18.0.gz clash #重命名为 clash
   chmod +x clash #赋予 Clash 执行权限
   ```

### 3. 复制订阅链接
   通过订阅链接 获取对应配置文件
   3.1 在终端执行 `wget -O config.yaml` + 刚刚复制的托管链接
   3.2 在终端执行 `./clash -d .` 即可启动 `Clash`

>首次运行之后会在**`当前运行配置目录生成配置文件`，产生的目录为 `/root/clash` ，里面有 `config.yaml` 和 `Country.mmdb` 两个文件。**
{: .prompt-info }

> 或者直接手动复制自己pc上的clash配置文件，`config.yaml` 和 `Country.mmdb` 到/root/clash文件夹下
{: .prompt-tip }

### 4. 修改系统代理(永久)

   ```shell
   cd ~ #切换到 root 账户目录
   ```

   运行 `vi .bashrc`编辑，增加一键开启/关闭 代理函数 到文件末尾

   ```shell
   # 开启代理
   function proxy_on(){
       export all_proxy=socks5://127.0.0.1:7890  # 注意你的端口号可能不是7890，注意修改
       export http_proxy=http://127.0.0.1:7890
       export https_proxy=http://127.0.0.1:7890
       echo -e "已开启代理"
   }
   
   # 关闭代理
   function proxy_off(){
       unset all_proxy
       unset http_proxy
       unset https_proxy
       echo -e "已关闭代理"
   }
   ```

   重新加载终端配置文件

   ```shell
   source ~/.bashrc
   ```

   使用函数

   ```shell
   #在终端输入 proxy_on 代表使用代理
   proxy_on
   #在终端输入 proxy_off 代表关闭代理
   proxy_off
   ```

   测试：

   ```shell
   env | grep proxy #查看代理的地址
   curl https://www.google.com # 测试代理效果
   ```

### 5. 使用守护进程Clash 自启动及后台运行
   5.1 运行`cp clash /usr/local/bin`，可执行文件放入 bin 目录 使全局可访问
   5.2 创建 `/etc/systemd/system/clash.service`

   ```shell
   [Unit]
   Description=Clash daemon, A rule-based proxy in Go.
   After=network.target
   
   [Service]
   Type=simple
   Restart=always
   ExecStart=/usr/local/bin/clash -d   /root/clash
   
   [Install]
   WantedBy=multi-user.target
   ```

   1. 运行 `systemctl enable clash`设置 clash 服务在系统启动时运行；

   2. 运行 `systemctl start clash`立即运行 clash 服务；

   3. 运行 `systemctl status clash`查看 clash 服务运行状态；

   4. 运行 `systemctl stop clash` 停止服务

   5. 运行 `systemctl restart clash` **重启服务**

   6. 运行 `systemctl is-enabled clash` **查看服务是否正在运行**

   7. 运行 `journalctl -xe`查看运行日志；

> 利用 systemd 创建守护进程，关键在于ExecStart执行命令 -d 和/root/clash都是执行clash文件的参数
{: .prompt-danger }

### 6. 配置Clash可视化 web 页面控制

   6.1 下载面板文件

   ```shell
   # 1.切换到 /root/clash/文件夹里
   cd /root/clash/
   # 2.下载 ui
   wget https://github.com/haishanh/yacd/archive/gh-pages.zip
   # 3.解压
   unzip gh-pages.zip
   # 4.修改文件名为 dashboard
   mv yacd-gh-pages/ dashboard/
   ```

   6.2 修改/root/clash/ 下的config.yaml

   - `secret`就是`api的访问秘钥` 如果没有的话任何人都可以访问你的clash面板的api 不安全
   - 这里的 `port` 是 http/https 代理端口
   - `socks-port` 是 socks 流量代理端口
   - `external-controller` 是外部控制端口，用于面板控制(前端页面的端口)
   - `external-ui` 是本地控制页面的源码（前端面板的路由）

   ```yaml
   port: 7890
   socks-port: 7891
   secret: 123456789
   external-controller: 0.0.0.0:9090  #别忘记在服务器厂商开放端口号
   external-ui: dashboard  #打开面板
   ```

   6.3 重启服务

   ```shell
   systemctl restart clash
   ```

   6.4 访问页面 : 宿主机 ip + 9090/ui

   API base URL 是127.0.0.1:9090 ，secret是config.yaml配置文件的字段 ，这2个必填

   + http://<ip>:9090/ui/ （防火墙开启9090，可以访问）
   + 推荐：先使用**本地端口转发**（可以使用vscode插件 remote ssh 插件的 port forward 功能等），再访问 http://127.0.0.1:9090/ui/ <img src="../assets/img/posts24/2024-08-16-19-39-23.png" alt="image-20240810105939148" style="zoom: 25%;" />

   

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


![](../assets/img/posts24/2024-08-16-19-39-42.png)