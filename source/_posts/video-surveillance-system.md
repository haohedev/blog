---
title: 基于SRS的流媒体服务器在安防行业的集成应用
date: 2018-11-11 09:44:45
tags:
categories: Video
---

## 背景
在给企业或者政府做安全生产相关的系统时，都绕不开视频监控这一重要环节，而通常涉及到的企业会有大量的`摄像头`或者`硬盘录像机（DVR）`等设备。在日益趋向BS架构的应用，如何可以在`现代浏览器`里播放这些摄像头呢？

## 思路
在互联网点播技术非常盛行的今天，有着各种各样的优秀方案，不乏一些高质量的开源框架如：`SRS`、`Nginx+RTMP`等。我们今天就来聊聊[SRS](https://github.com/ossrs/srs)这个优秀的框架。本文将提供一个思路以及带领大家动手实现一个简单的示例。更多的细节参见官方的[WIKI](https://github.com/ossrs/srs/wiki)。

## 前提
1. 本文的`SRS`将在`docker`中运行，所以必须有一个可以运行`docker`的系统。本文是在`ubuntu 18.04`下采用`docker`中运行。
1. 身边有一个`硬盘录像机(DVR)`或`网络摄像机（IPC）`，本文采用的是`海康DVR`。
1. 自行安装`ffmpeg`，本文将采用`ffmpeg`对视频流进行转码。

<!-- more -->

## 协议的转换
我们通过输入和输出来对`SRS流媒体服务器`做个最基本的了解。    
**输入**：`RTMP`协议的视频流。  
**输出**：`RTMP`、`HLS`、`FLV`等协议的输出。  
现在市面上大部分的安防监控的协议都是`RTSP`，那么就很清晰了，我们只需要将`RTSP`协议转换成`RTMP`协议之后推送给`SRS流媒体服务器`，我们的任务就完成了。至于协议[转封成HLS](https://github.com/ossrs/srs/wiki/v2_CN_SampleHLS)或[转封成FLV](https://github.com/ossrs/srs/wiki/v2_CN_SampleHttpFlv)以及视频存储`（基于SRS的DVR）`等就交给`SRS流媒体服务器`了。  
海康威视RTSP地址举例说明：  
主码流取流: `rtsp://admin:12345@192.0.0.64:554/h264/ch1/main/av_stream`  
子码流取流:   `rtsp://admin:12345@192.0.0.64:554/h264/ch1/sub/av_stream` 

不难看出，我们只需要知道`硬盘录像机(DVR)`或`IPC`的**验证信息**以及**通道号**，即可以得到`rtsp`协议的地址。
> 可以在`VLC`中的`打开网络串流中`输入`rtsp`的`uri`地址，即可测试是否可以正常获取`rtsp`流。

## 架构图
![BASIC ARCHITECTURE](http://pc5axiqo2.bkt.clouddn.com/%E5%AE%89%E9%98%B2%E8%A7%86%E9%A2%91%E9%9B%86%E6%88%90%E7%9B%91%E6%8E%A7%E6%96%B9%E6%A1%88.png)   
我们对这个图进行解释说明：  
**采集服务**：我们自己实现的服务，可以通过`硬盘录像机(DVR)`的SDK获取`摄像头`的信息，并且可以启动`FFMPEG`对流进行转码。  
**srs流媒体服务器**：指我们在docker中部署的`SRS`服务。  
**①**:我们通过`http`服务，像`采集服务`提供`硬盘录像机（DVR）`登陆验证信息，从而获取所有摄像头的基本信息(如摄像头的名称等)。  
**②**:当`①`成功后，我们启动`FFMPEG`对`硬盘录像机(DVR）`的各个通道的`RTSP协议`的视频流转换成`RTMP协议`并推送到`SRS流媒体服务器`。  
**③**:当我们需要获取视频数据的时候，从`SRS流媒体服务器`进行拉流进行播放。

## 实践

1、在Docker中启动srs:
```docker
docker run -d --name srs -p 8080:8080 -p 1935:1935 -p 1985:1985 --restart=always ossrs/srs:2.0  
```
> 这里`8080`端口是指`hls`协议端口号，`1935`是`RTMP`协议的端口号, 而`1985`则是srs的后台管理界面，可以查看srs的服务器信息，包括CPU、内存、带宽等。还包括虚拟主机以及视频流的情况。具体的可以参见[srs控制台](http://ossrs.net:1985/console/ng_index.html#/connect?host=ossrs.net&port=1985)。  
进入到docker中，默认进到/srs目录，会看到3个文件夹：  
**conf文件夹**：里面存在各种各样的配置，可以通过在docker的命令行中修改启动时的命令来加载不同的配置，默认执行的conf中的`docker.conf`，支持`RTMP`，支持`转封HLS`，支持`API`后台管理接口。  
**objs文件夹**：包含`srs`的可执行程序，同时该目录下有个`nginx`文件夹，`转封HLS`时会将`m3u8`文件生成到该目录中的live文件夹中（在本示例中）。
**etc文件夹**：包含启动时的执行脚本。  

2、 启动FFMPEG，将硬盘录像机的`RTSP`协议的视频流转换成`RTMP`协议的视频流并推送至`SRS流媒体服务器`：
```bash
ffmpeg -rtsp_transport tcp -i rtsp://admin:12345@192.0.0.64:554/h264/ch1/main/av_stream -vcodec copy -acodec aas -f flv rtmp://localhost:1935/live/livestream
```
> **-rtsp_transport tcp**：rtsp协议默认采用的udp传输，我们指定表示强制用tcp的方式进行传输，因为海康是tcp方式。  
**-i**： 表示输入源，我们输入`硬盘录像机(DVR)`的`RTSP`协议地址。  
**-vcodec copy**: 表示视频编码方式，因为海康默认就是h.264编码，因为我们拷贝源视流即可。  
**-acodec aas**：表示音频编码方式，这里需要我们注意一下，海康的音频默认**不是**`aac`或`mp3`格式，因为不能拷贝原始流，需要进行一次转换。  
**-f flv**：表示格式化成flv格式  
最后推送到我们的流媒体服务器  
***注意：*** `localhost`替换成流媒体服务器地址。

3、 启动`VLC`播放`RTMP`或`HLS`流：
```
RTMP流地址：rtmp://localhost:1935/live/livestream
HLS流地址：http://localhost:8080/live/livestream.m3u8
```
> 这里需要把`localhost`换成`srs`的地址。
## 问题

- *为什么没有看见在浏览器里打开视频的部分？*  
已经可以将协议转成`RTMP`、`HLS`、`FLV`了，前端有很多优秀的`js`库对`HLS`或`FLV`进行支持。
- *为什么RTMP流可以播放但是HLS流不能播放？*  
这个需要看SRS流媒体服务器的输出日志，我遇到的问题是因为音频编码不一直导致SRS不能正常的将流进行HLS的切分。如果`FFMPEG`指定`acodec copy`，但原始流不是`aac`或`mp3`，则会导致`RTMP`流可以看，但`HLS`不能看。
- *有没有完整的实现代码？* 
这个没有，本文只是提供了一种技术思路，走通了技术路线，但真正实现起来并不困难。
- *推流和拉流有什么区别？* 
这个就好像在商场门上一面写着`推`，而另一面写着`拉`一样，如果你站在外面就是`推`，但如果站在里面就是`拉`。如果在SRS里获取，我们就认为是`拉`流，否则就是`推`流。
- *SRS有拉流的操作，为什么还单独开启FFMPEG的进程来推流？*  
的确`SRS`里集成了`FFMPEG`，但是由于海康的`RTSP`流的传输方式是`TCP`，而如果在FFMPEG里不指定协议，通常是以`UDP`的方式获取数据。我暂时没有找到在SRS中拉流中指定`RTSP`流配置`TCP`协议的配置项，因此采用独立的进程来推流。有人在Issue中提到[此问题](https://github.com/ossrs/srs/issues/975),但是我没有试过。
- *如果我要接入的视频不提供RTSP协议怎么办？*  
没有关系，我们只需要通过`FFMPEG`将视频转成`h.264`，将音频转成`aac`或`mp3`推到`SRS流媒体服务器`即可。
- *其他未知的问题？*  
可以参考官方的[wiki](https://github.com/ossrs/srs/wiki)或在[Issues](https://github.com/ossrs/srs/issues)中查找答案，基本上你想知道的，上面都会有答案的。