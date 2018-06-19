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
通过上面的架构，我们可以知道性能的瓶颈在于更新数据。当我们有很多的点需要实时数据的更新时，这样的操作无疑会非常慢，毕竟数据的更新需要经历[CLP](http://docs.parseplatform.org/parse-server/guide/#class-level-permissions)和[ACL](http://docs.parseplatform.org/js/guide/#object-level-access-control)。而现在面临的需求是大批量的更新实时数据，但不需要将数据通过`Parse`平台进行存储。虽然有其他的解决方案，比如通过`rabbitmq`来进行数据的推送，`rabbitmq`有`Web STOMP`插件，其底层也通过`WebSockets`实现，但我们更希望所有的工作都在`Parse`上来完成，充分的利用`LiveQuery Server`基于条件的事件订阅机制。因此我们的面临的问题是在`Parse`平台上如何将数据高效的推送？
> 更多关于`Parse`平台`CLP`和`ACL`和差异[官方文档](http://docs.parseplatform.org/js/guide/#clp-and-acl-interaction)的例子解释的很清晰。

## 解决方案  

### 分析
首先，通过分析源码，我们看到`LiveQuery Server`对数据的解析不会经过底层的IO， 仅仅是将推送的数据构造出`ParseObject`。其次，通过上面的可扩展架构图，我们看到`Parse Server`将数据推送至`Redis`，而`LiveQuery Server`订阅`Redis`从而将数据推送至`LiveQuery Client`，因此我们可以对`Redis`做文章。如果知道`ParseServer`推送的`主题`和`消息体`，那么我们就可以绕过`Parse Server`从而实现数据的主动推送。

### 实现
同样还是通过分析`Parse`的源码,可以看到推送的`主题`的格式`APP_ID`+ `事件类型`。用`APP_ID`作为前缀，目的是用以区分不同的应用。下面我们总结一下各个事件的`主题`和`消息体`:
**新增数据：**  `主题`为`${APP_ID}afterSave`，`消息体`中仅有`currentParseObject`。  
**更新数据：**  `主题`为`${APP_ID}afterSave`，`消息体`中包含`currentParseObject`和`originalParseObject`。  
**删除数据：**  `主题`为`${APP_ID}afterDelete`，`消息体`中仅有`currentParseObject`。  
`currentParseObject`和`originalParseObject`的结构是带有类型信息的`JSON`对象。之所以更新数据时需要`currentParseObject`和`originalParseObject`，是因为`LiveQuery`中有[enter-event](http://docs.parseplatform.org/js/guide/#enter-event)和[leave-event](http://docs.parseplatform.org/js/guide/#leave-event)。  
``` json
{
    "currentParseObject": {
        "score": 1337,
        "playerName": "Sean Plott",
        "cheatMode": false,
        "createdAt": "2018-06-17T02:42:31.542Z",
        "updatedAt": "2018-06-18T07:28:39.752Z",
        "objectId": "c6m5FUyRqo",
        "__type": "Object",
        "className": "GameScore"
    },
    "originalParseObject": {
        "score": 1337,
        "playerName": "Sean Plott",
        "cheatMode": true,
        "createdAt": "2018-06-17T02:42:31.542Z",
        "updatedAt": "2018-06-17T02:43:45.665Z",
        "objectId": "c6m5FUyRqo",
        "__type": "Object",
        "className": "GameScore"
    }
}
```

上面提到过，`LiveQuery Server`从消息体中构造出`Parse Object`对象。也就是说消息体中有的属性，都会被传递到客户端，而消息体中不存在的属性，客户端自然也是接收不到的。所以我们只需要将我们的数据组织成上面的格式，对于不同的订阅条件，会由`LiveQuery Server`进行处理，从而实现数据的推送。

### 数据模型
既然已经知道了如何实现数据的主动推送，我们现在应该讨论一下如何应用到项目中，也就是说如何设计数据模型。

**方案一：** 将实时数据和定义表存放在一起。  
&emsp;&emsp;在定义表中可以不体现实时数据的字段，利用`Parse`[云函数](http://docs.parseplatform.org/cloudcode/guide/)中的`afterFind`触发器，获取实时数据并添加到查询后的对象中。这样当用户获取定义表数据时，实时数据字段自然也在其中。这样做的缺点是，当推送数据时，你必须从`Parse`平台取出来定义数据，然后将定义数据和实时数据合并在一起。可以想到，如果数据更新的比较频繁时，这无疑增大了服务器的压力。当然，可以利用`Redis`做缓存，这里我们就不展开讨论了。  
**方案二：** 将实时数据表分开存储。  
&emsp;&emsp;通过设备编号将两个表关联起来。这样仍然可以像`方案一`一样利用触发器来实现获取定义时包含完整的数据。但是订阅数据时必须订阅实时表，而实时表中仅包含编号和实时数据。这样的好处是你不需要取定义数据，避免了`方案一`来带的问题，但推送的数据仅仅为实时数据，相对来说缺少了定义属性。  

### 总结
在实际项目中，应该根据不同应用来确定方案。在数据变化不是很多的情况下，利用`Parse`原生的数据推送机制也是不错的选择。我们可以看到，`方案一`和`方案二`都有相应的好处，但同时也都存在不足，还需我们自己将模型进行完善。

---
> **本文作者：** 郝赫   
> **本文链接：** https://zyzz.xyz/live-query-redis/   
> **版权声明：** 本作品采用 [知识共享署名-非商业性使用-相同方式共享 4.0 国际许可协议](https://creativecommons.org/licenses/by-nc-sa/4.0/deed.zh) 进行许可。转载请注明出处！  
> ![license](https://i.creativecommons.org/l/by-nc-sa/4.0/88x31.png)