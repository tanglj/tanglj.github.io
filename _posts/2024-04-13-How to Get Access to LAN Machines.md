---
title: 如何实现局域网机器的外部访问
date: 2024-04-13 09:56:00 +0800
categories: [计算机基础]
tags: [linux]
---

### 1. 问题的产生

大部分企业或家庭会拥有自己的局域网，以便多个设备能够联网以及共享网络服务。

局域网中机器之间的访问很容易，但想要通过公网访问局域网内的机器则存在一个问题：该机器没有公网IP，无法直接访问。

为解决这一问题，出现了一种技术：内网穿透。

### 2. 内网穿透

`内网穿透` 是一种技术，它允许外部网络中的计算机或设备通过互联网访问内部网络中的资源或服务。

内网穿透技术通过在内部网络和外部网络之间建立一条安全的通道来解决这个问题。通常使用一个位于公网可被认证设备访问的代理服务来实现。

下图中的 `花生壳服务器` 即是代理：

![内网穿透的原理](assets/posts-img/002-oray-overview.png)

### 3. 花生壳内网穿透

[贝锐花生壳](https://hsk.oray.com/) 是一个简单易用的内网穿透代理服务，免费的体验版服务支持配置2条映射和1G/月的流量，可满足低频基本使用。

![官方介绍](assets/posts-img/002-oray-site.png)

具体的安装配置方法可查看官方文档，比如 [Linux版使用教程](https://service.oray.com/question/11630.html)。

在添加映射时，映射类型选择 `TCP`，内网地址填写为 `127.0.0.1:22`，即可将 `SSH` 服务暴露出去。

在另一台可访问公网的电脑上，通过花生壳随机生成的外网地址访问，例如以下配置的访问方法为：`ssh -p 32757 user@1r39463n06.yicp.fun`。

![内网穿透的原理](assets/posts-img/002-oray-mapping.png)

### 4. frp自建代理

除了贝锐花生壳以外，还有很多其他的服务提供商，例如 [NATAPP](https://natapp.cn/)、[cpolar](https://www.cpolar.com/)、[Ngrok](https://ngrok.com/)、[Sunny-Ngrok](https://www.ngrok.cc/) 等，均有免费体验版和付费服务。

> 备注：以上各服务商，包括贝锐花生壳，仅为列举，不做任何推荐。

除了选择服务商外，还可自行搭建代理服务，高频使用的情况下可节省费用，并且能够保证数据安全。

[frp](https://github.com/fatedier/frp) 是一个开源的内网穿透代理应用，使用它搭建代理服务需要有一台具有公网IP的服务器。

frp的 `release` 安装包中包含一个服务端程序 `frps` 及配置 `frps.toml `，以及一个客户端程序 `frpc` 及配置 `frpc.toml `。服务端程序即是运行于公网的代理服务，而客户端程序是内网中用于和外网代理服务建立通道的本地服务。

具体的安装配置方法可查看 [官方文档](https://gofrp.org/zh-cn/docs/examples/ssh/)。

### 5. 参考资料

- 花生壳官网对内网穿透的说明 [https://www.oray.com/news/5571.html](https://www.oray.com/news/5571.html)

- frp介绍 [https://gofrp.org/zh-cn/docs/overview/](https://gofrp.org/zh-cn/docs/overview/https://gofrp.org/zh-cn/docs/overview/)