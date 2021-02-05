---
title: 使用免费阿里云服务器和FRP搭建远程桌面服务
author: ZhangMapler
date: 2021-02-05 17:33:00 +0800
categories: [RDP]
math: false
---

# 使用免费阿里云服务器搭建远程桌面服务

## 背景

[RDP，Remote Desktop Protocol](https://en.wikipedia.org/wiki/Remote_Desktop_Protocol#:~:text=By%20default%2C%20the%20server%20listens,128%20application%20sharing%20protocol.)是微软推动开发的远程桌面协议，它本身也会默认内置在win10专业版或者以上版本中，使用TCP和UDP协议，监听端口默认3389。

但是问题是只能在局域网内或者公司企业的内网中使用。当我们无法访问到内网时，就不能连接公司的电脑进行远程操作。怎么解决这个问题呢，当然如果公司有VPN的话，可以直接连接VPN，进行远程连接。如果没有VPN，我们就需要使用点手段了。

目前市面上免费的远程服务主要是Teamviewer，网速也还可以，但是有连接设备数的限制，对于那些频繁需要远程和在多个设备上连接远程服务的人来说，就不够用了。那么有什么办法可以解决这个问题吗？

没错，我们可以自己搭建一个远程桌面服务（或者叫远程桌面的中转服务）。当然前提条件是有一台公网IP的机器（C）（可以选择阿里云和腾讯云等厂商，见文末推荐）。假设笔记本电脑（A）要远程连接公司的电脑（B），因为B在内网，A在公网无法直接访问。那么我们可以这样，B向C、A向C分别主动建立一条双向通道连接（TCP）。

当我们需要远程时，流程大概如下所示：
1. A将远程连接请求发到C；
2. C收到请求后将此请求转发给B；
3. B收到请求后，通过BC通道向C发送用户认证请求；
4. C收到请求后，将认证请求再转发给A；
5. A收到认证请求后，由A机器的RDP协议处理请求（显示输入用户名和密码弹窗）；
6. A填完用户信息后，发送给C，C再将其转发给B；
7. B验证用户信息正确，将电脑实时信息发送给C，C将其转发给A；
8. A收到B的实时信息后，由A的RDP协议处理（页面上打开远程窗口，显示B的电脑远程窗口）
9. 随后A将操作命令（点击，键盘输入等）通过C发送给B；B再将响应信息通过C发送A；

很合理，但是连接通道谁来建立呀，这里就要提到FRP工具啦。

## FRP

[FRP](https://github.com/fatedier/frp)是一款开源的内网暴露，内网穿透工具。分为服务端和客户端。

再回到上文的方案中，C扮演的角色就是服务端，用于接受A和B机器的连接，那么C机器上运行的就是FRP的服务端。A和B上运行的就是客户端。

首先我们来下载，先去FRP的[Releases](https://github.com/fatedier/frp/releases)页面，下载对应的包就行了。
> windows系统，使用intel或者amd的cpu，根据32位和64位分别下载frp_0.35.1_windows_386.zip和frp_0.35.1_windows_amd64.zip。linux上也同理。如果访问不了github，也可以直接在文末的下载链接中下载。

我们先下载服务端的包frp_0.35.1_linux_amd64.tar.gz，可以直接执行命令 `wget下载链接` 下载，如：

```
wget https://github.com/fatedier/frp/releases/download/v0.35.1/frp_0.35.1_linux_amd64.tar.gz
```

下载后，解压：

```
tar -zxvf frp_0.35.1_linux_amd64.tar.gz
```

之后，进入到解压的文件夹，文件树如下：
```
├── frpc
├── frpc_full.ini
├── frpc.ini
├── frps
├── frps_full.ini
├── frps.ini
```
其中frpc（client）是客户端的可执行程序，frps（server）是服务端的可以执行程序。接着编辑frps.ini文件，将内容改为：
```
[common]
bind_port = 7000
```
这样FRP服务端就运行在了C机器上，监听7000端口，运行成功会输出以下日志：
```
2021/02/05 18:48:45 [I] [root.go:108] frps uses config file: ./frps.ini
2021/02/05 18:48:45 [I] [service.go:190] frps tcp listen on 0.0.0.0:7000
2021/02/05 18:48:45 [I] [root.go:217] frps started successfully
```
另外需要配置好网关规则，允许端口7000和3389的TCP和UDP流量进出。

接着我们分别将frp_0.35.1_windows_amd64.zip下载到电脑A。解压后，编辑frpc.ini，将内容改为：
```
[common]
server_addr = C机器的公网IP地址或者域名
server_port = 7000
auto_token=mstsc

[mstsc]
type = tcp
local_ip = 127.0.0.1
local_port = 3389   
remote_port = 3389  # B机器连接C机器的端口，这里使用默认
```
之后，打开powershell或者cmd，进入到此文件夹，执行命令：
```
.\frpc.exe -c .\frpc.ini
```
运行后，会输出日志：
```
2021/02/05 18:59:04 [I] [service.go:290] [1f8d33888eb10657] login to server success, get run id [1f8d33888eb10657], server udp port [0]
2021/02/05 18:59:04 [I] [proxy_manager.go:144] [1f8d33888eb10657] proxy added: [mstsc]
2021/02/05 18:59:04 [I] [control.go:180] [1f8d33888eb10657] [mstsc] start proxy success
```
C机器输出日志，表示连接成功：
```
[control.go:446] [a091bfe7d86960ee] new proxy [mstsc] success
```
同时C机器FRP服务端会根据客户端的remote_port开启监听：
```
netstat -ano|grep 3389
tcp6       0      0 :::3389                 :::*                    LISTEN      off (0.00/0/0)
```

## 连接

上述步骤连接完成后，就可以进行连接了。手机端可以应用市场搜索微软的官方APP `RD Client`，电脑上可以直接使用远程桌面连接，输入C机器的公网IP或者域名就可以了。

## 云服务器推荐

笔者使用的是阿里云免费[试用活动](https://free.aliyun.com/?spm=5176.21103406.J_3012903320.9.3444597cHPTjMt)中的1核2G机器，5m带宽，一个月试用时间，春节回家是够用了。5m带宽基本上700KB/s的网速够用了，实测比Teamviewer速度稍快，写代码刷网页很流畅。

如果不差钱，也可以购买阿里云最新活动中的的[2核2G机器](https://www.aliyun.com/minisite/goods?userCode=wlxyid9q&share_source=copy_link)，5m带宽，99元/年。

## 下载链接

1. frp_0.35.1_windows_amd64.zip

    链接: https://pan.baidu.com/s/1fLRWoQs4Kta-rQoAnreXEg 提取码: zztw
2. frp_0.35.1_linux_amd64.tar.gz

    链接: https://pan.baidu.com/s/1MONsWoHfy8EXalJTN26lqA 提取码: 9cuu