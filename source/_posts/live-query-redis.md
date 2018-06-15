---
title: 基于LiveQuery的数据主动推送
date: 2018-06-15 20:11:44
tags: 
    - node
    - redis
    - LiveQuery
categories: Parse
---


## 背景
我们知道，当配置了需要推送的类名称之后，通过`Parse`平台更改数据时，会将数据通知到`LiveQuery Server`, `LiveQuery Server`通过`WebSocket`协议对数据进行推送。`Parse`提供了各个平台相应的`SDK`，具体的实现可以参考[官方网站](http://docs.parseplatform.org/parse-server/guide/#live-queries)。  
基础架构如下图所示：
![BASIC ARCHITECTURE](http://docs.parseplatform.org/assets/images/lq_local.png)   
> 通常来说，`ParseServer`和`Parse LiveQuery Server`是运行在同一个服务中的。
<!-- more -->

不难发现，这种基础的架构也仅仅适合本地开发或测试时使用。在生产环境中，我们通常用类似`pm2`之类的工具对`Parse Server`进行负载均衡和进程守护。这样一来，当推送的`Server`不是客户端所订阅的，那么客户端将接受不到消息的推送。  
不用担心，`Parse`提供了可扩展的架构：
![SCALABILITY ARCHITECTURE](http://docs.parseplatform.org/assets/images/lq_multiple.png)

可以看到，在`Parse Server`和`Parse LiveQuery Server`之间增加了`redis`作为中间层，`Parse Server`将数据推送至`Redis`中， `Parse LiveQuery Server`从`Redis`中订阅数据的变化。这样有效的避免了上述的问题。