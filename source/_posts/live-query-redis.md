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
我们知道，当配置了需要推送的类名称之后，通过`Parse Server`平台更改数据时，会将数据通知到`LiveQuery Server`, `LiveQuery Server`通过`WebSocket`协议对数据进行推送。`Parse`提供了各个平台相应的`SDK`，具体的实现可以参考[官方网站](http://docs.parseplatform.org/parse-server/guide/#live-queries)。  
基础架构如下图所示：
![BASIC ARCHITECTURE](http://docs.parseplatform.org/assets/images/lq_local.png)   
> 通常来说，`ParseServer`和`Parse LiveQuery Server`是运行在同一个服务中的。
<!-- more -->

不难发现，这种基础的架构也仅仅适合本地开发或测试时使用。在生产环境中，我们通常用类似`pm2`之类的工具对`Parse Server`进行负载均衡和进程守护。这样一来，当推送的`Server`不是客户端所订阅的，那么客户端将接受不到消息的推送。  
不用担心，`Parse`提供了可扩展的架构：
![SCALABILITY ARCHITECTURE](http://docs.parseplatform.org/assets/images/lq_multiple.png)

可以看到，在`Parse Server`和`Parse LiveQuery Server`之间增加了`redis`作为中间层，`Parse Server`将数据推送至`Redis`中， `Parse LiveQuery Server`从`Redis`中订阅数据的变化。这样有效的避免了上述的问题。

## 问题
通过上面的架构，我们可以知道性能的瓶颈在于更新数据。当我们有很多的点需要实时数据的更新时，这样的操作无疑会非常慢，毕竟数据的更新需要经历[CLP](http://docs.parseplatform.org/parse-server/guide/#class-level-permissions)和[ACL](http://docs.parseplatform.org/js/guide/#object-level-access-control)。而现在面临的需求是大批量的更新实时数据，但不需要将数据通过`ParseServer`平台进行存储。虽然有其他的解决方案，比如通过`rabbitmq`来进行数据的推送，但我们更希望 ***all in Parse*** 。通过分析源码，我们看到`LiveQuery Server`对数据的解析不会经过底层的IO， 仅仅是将推送的数据构造出`ParseObject`,因此我们的面临的问题是如何将数据高效的推送。
> 更多关于`Parse`平台`CLP`和`ACL`和差异[官方文档](http://docs.parseplatform.org/js/guide/#clp-and-acl-interaction)的例子解释的很清晰。

## 解决方案  

### 分析
通过上面的可扩展架构图，我们看到`Parse Server`将数据推送至`Redis`，而`LiveQuery Server`订阅`Redis`从而将数据推送至`LiveQuery Client`，因此我们可以对`Redis`做文章。如果知道`ParseServer`推送的主题和消息体，那么我们就可以绕过`Parse Server`从而实现实时数据的主动推送。

### 实现
**敬请期待** 


---
> **本文作者：** 郝赫   
> **本文链接：** https://zyzz.xyz/live-query-redis/   
> **版权声明：** 本作品采用 [知识共享署名-非商业性使用-相同方式共享 4.0 国际许可协议](https://creativecommons.org/licenses/by-nc-sa/4.0/deed.zh) 进行许可。转载请注明出处！  
> ![license](https://i.creativecommons.org/l/by-nc-sa/4.0/88x31.png)