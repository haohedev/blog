---
title: Ogre+WebUI(基于QWebView的实现)
date: 2018-07-20 09:08:39
tags:
    - QT
    - OGRE
categories:
---

## 背景
公司有自主研发的基于[OGRE](!https://www.ogre3d.org/)的三维引擎，同时引擎封装了`python2`的脚本，现想将三维引擎进一步封装成平台。考虑到前端框架更多，效果丰富，开发周期更短等优势，打算利用`WebUI`进行界面的开发，三维平台提供`API`接口供前端调用。

## WebUI概念
`WebUI`是指在`QMainWindow`中设置个渲染`OGRE`的`QWidget`，同时创建个透明背景的`QWebView`覆盖在`OGRE`窗口上方。因为设置了透明背景，因此页面中透明的部分可以直接看到`OGRE`窗口。设置`WA_TransparentForMouseEvents`属性，使得在透明的部分鼠标事件穿过页面直接送达给`OGRE`窗口。

## QWebView VS QWebEngineView 
`Qt`从5.6开始，移除了`WebKit`组件， 从而采用了`Chrome`内核的`WebEngine`组件。想使用`WebKit`内核并不重新编译的情况下，可以选择的`Qt`版本最高只有`5.5.1`。 虽然基于`Chrome`内核效率提升；但是由于架构的改变，使得`QWebEngineView`没有办法穿透鼠标事件，即使设置了`WA_TransparentForMouseEvents`也没有效果，相信这是`Qt`的`Bug`， 当以后的版本解决这个问题后，可以更新至`QWebEngineView`，会拥有更佳的体验效果。

## 现有架构

![现有架构](http://pc5axiqo2.bkt.clouddn.com/IMG_0821.JPG)

我们在项目中使用过上面的架构，通`QWebFrame::addToJavaScriptWindowObject`接口，可以添加宿主对象供前端调用。提供的对象包括场景的定位，模型的显示隐藏、高亮等操作。  
优势： 底层只要提供一些宿主对象即可，业务功能都由前端最来完成，开发比较便捷。  
缺点：
1. 由于`OGRE`渲染比较吃紧cpu，而前端页面的渲染同样也依赖于cpu。 当我们设置`OGRE`为60帧时，页面渲染大量数据时，避免不了的出现卡顿等现象。降帧自然可以解决问题，但降帧会导致不太友好的三维体验。  
1. 由于我们提供的`QWebView`类似于浏览器，而`QWebView`自带的`QInspector`调试起来并不是很方便，同时接口调用严重依赖于宿主对象，导致前端开发调试不便捷。


## 目标架构

![目标架构](http://pc5axiqo2.bkt.clouddn.com/IMG_0823.JPG)

我们来看现在这个新的架构：  
进程1：包含了`QWebView`以及提供`HTTPServer`的服务。 同时利用`QWidget::createWindowContainer`来嵌入**进程2**的窗体。  
进程2：实现了基于`QWidget`的`OGRE`渲染部分， 创建渲染引擎，处理并派发鼠标和键盘事件。  
脚本：**进程2**只需要实现创建三维引擎并派发事件即可。更多的功能实现在脚本中来完成，这样可以动态扩展三维引擎的功能部分。  

由于拆分了多个进程，那么我们就必须要考虑`IPC`技术了。 `Qt`提供了`QLocalServer + QLocalSocket`和`QSharedMemory`两种方案。个人觉得前者的方案更优一些。
> 基于`Python3`的`PyQt5`和基于`Python2`的`PyQt4`在利用`QLocalSocket`进行通讯时，当数据量比较大的时候，会出现数据传输不完整的bug， 这是需要利用`QSharedMemory`来配合完成`IPC`。

## 具体实现

**进程1**我们采用`Python3.4` + `PyQt5.5.1`来做。`Python`有`pywebview`库，通过修改`pywebview`源码可以实现在`windows`平台上利用`Qt`框架的`QWebView`来实现界面部分。 同时在`pywebview`的`examples`中`flask_app`实现了前后端分离的结构，后端采用了`flask`提供了`HTTP API`的接口。考虑到我们要将场景中的数据都以`Restful API`形式提供，我们可以采用基于`flask`的`Eve-Sqlalchemy`框架来实现，利用`ORM`技术将数据映射到内存的`Sqlite`。通过数据库的触发器部分，来实现对三维对象的更新操作。除此之外，还应提供一些常见操作的`HTTP API`接口， 如实体的定位、视口的跳转等。  