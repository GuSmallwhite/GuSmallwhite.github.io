---
title: "zk之watcher机制"
date: "2024-02-07 12:00:00"
permalink: "2024/02/07/zk之watcher机制/"
categories: ["算法", "rpc项目"]
tags: ["算法", "rpc"]
---

## 前言

zk之watcher≠curator之cachelistener

## Watch 机制是如何实现的

我们可以通过向 ZooKeeper 客户端的构造方法中传递 Watcher 参数的方式实现：

```
new ZooKeeper(String connectString, int sessionTimeout, Watcher watcher)
```

上面代码的意思是定义了一个了 ZooKeeper 客户端对象实例，并传入三个参数：

```
connectString 服务端地址

sessionTimeout：超时时间

Watcher：监控事件
```

这个 Watcher 将作为整个 ZooKeeper 会话期间的上下文 ，一直被保存在客户端 ZKWatchManager 的 defaultWatcher 中。

除此之外，ZooKeeper 客户端也可以通过 getData、exists 和 getChildren 三个接口来向 ZooKeeper 服务器注册 Watcher，从而方便地在不同的情况下添加 Watch 事件：

```
getData(String path, Watcher watcher, Stat stat)
```

### 触发通知的事件

![image.png](https://learn.lianglianglee.com/%E4%B8%93%E6%A0%8F/ZooKeeper%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90%E4%B8%8E%E5%AE%9E%E6%88%98-%E5%AE%8C/assets/Ciqc1F61ILaAb7sQAAC6T3wMHDU651.png)

客户端连接服务端的时候，可以对数据节点的创建、删除、数据变更、子节点的更新等操作进行监控。

而在验证失败、断开连接、服务端销毁的情况下啥都干不了

## watcher机制原理

![image](https://learn.lianglianglee.com/%E4%B8%93%E6%A0%8F/ZooKeeper%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90%E4%B8%8E%E5%AE%9E%E6%88%98-%E5%AE%8C/assets/Ciqc1F61IL-AEQuUAABdpaAsy2k628.png)

我们可以将 Watch 机制理解为是分布式环境下的观察者模式。所以接下来我们就以观察者模式的角度点来看看 ZooKeeper 底层 Watch 是如何实现的。

![image](https://learn.lianglianglee.com/%E4%B8%93%E6%A0%8F/ZooKeeper%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90%E4%B8%8E%E5%AE%9E%E6%88%98-%E5%AE%8C/assets/Ciqc1F61IMWAbWW9AABzXk9xuOs953.png)

通常我们在实现观察者模式时，最核心或者说关键的代码就是创建一个列表来存放观察者。 而在 ZooKeeper 中则是在客户端和服务器端分别实现两个存放观察者列表，即：ZKWatchManager 和 WatchManager。其核心操作就是围绕着这两个展开的。

### 客户端 Watch 注册实现过程

在发送一个 Watch 监控事件的会话请求时，ZooKeeper 客户端主要做了两个工作：

* 标记该会话是一个带有 Watch 事件的请求
* 将 Watch 事件存储到 ZKWatchManager

当发送一个带有 Watch 事件的请求时，客户端首先会把该会话标记为带有 Watch 监控的事件请求，之后通过 DataWatchRegistration 类来保存 watcher 事件和节点的对应关系：

```
public byte[] getData(final String path, Watcher watcher, Stat stat){

  ...

  WatchRegistration wcb = null;

  if (watcher != null) {

    wcb = new DataWatchRegistration(watcher, clientPath);

  }

  RequestHeader h = new RequestHeader();

  request.setWatch(watcher != null);

  ...

  GetDataResponse response = new GetDataResponse();

  ReplyHeader r = cnxn.submitRequest(h, request, response, wcb);

  }
```

之后客户端向服务器发送请求时，是将请求封装成一个 Packet 对象，并添加到一个等待发送队列 outgoingQueue 中：

```
public Packet queuePacket(RequestHeader h, ReplyHeader r，...) {

    Packet packet = null;

    ...

    packet = new Packet(h, r, request, response, watchRegistration);

    ...

    outgoingQueue.add(packet); 

    ...

    return packet;

}
```

最后，ZooKeeper 客户端就会向服务器端发送这个请求，完成请求发送后。调用负责处理服务器响应的 SendThread 线程类中的 readResponse 方法接收服务端的回调，并在最后执行 finishPacket（）方法将 Watch 注册到 ZKWatchManager 中：

```
private void finishPacket(Packet p) {

        int err = p.replyHeader.getErr();

        if (p.watchRegistration != null) {

            p.watchRegistration.register(err);

        }

       ...

}
```

### 服务端 Watch 注册实现过程

Zookeeper 服务端处理 Watch 事件基本有 2 个过程：

* 解析收到的请求是否带有 Watch 注册事件
* 将对应的 Watch 事件存储到 WatchManager

下面我们分别对这 2 个步骤进行分析：

当 ZooKeeper 服务器接收到一个客户端请求后，首先会对请求进行解析，判断该请求是否包含 Watch 事件。这在 ZooKeeper 底层是通过 FinalRequestProcessor 类中的 processRequest 函数实现的。当 getDataRequest.getWatch() 值为 True 时，表明该请求需要进行 Watch 监控注册。并通过 zks.getZKDatabase().getData 函数将 Watch 事件注册到服务端的 WatchManager 中。

```
public void processRequest(Request request) {

...

byte b[] =                zks.getZKDatabase().getData(getDataRequest.getPath(), stat,

        getDataRequest.getWatch() ? cnxn : null);

rsp = new GetDataResponse(b, stat);

..

}
```

### 服务端 Watch 事件的触发过程

以 setData 接口即“节点数据内容发生变更”事件为例。在 setData 方法内部执行完对节点数据的变更后，会调用 WatchManager.triggerWatch 方法触发数据变更事件。

```
public Stat setData(String path, byte data[], ...){

        Stat s = new Stat();

        DataNode n = nodes.get(path);

        ...

        dataWatches.triggerWatch(path, EventType.NodeDataChanged);

        return s;

    }
```

我们进入 triggerWatch 函数内部来看看他究竟做了哪些工作。首先，封装了一个具有会话状态、事件类型、数据节点 3 种属性的 WatchedEvent 对象。之后查询该节点注册的 Watch 事件，如果为空说明该节点没有注册过 Watch 事件。如果存在 Watch 事件则添加到定义的 Wathcers 集合中，并在 WatchManager 管理中删除。最后，通过调用 process 方法向客户端发送通知。

### 客户端回调的处理过程

客户端首先反序列化服务器发送请求头信息 并判断相属性字段 xid 的值为 -1，表示该请求响应为通知类型。在处理通知类型时，首先将己收到的字节流反序列化转换成 WatcherEvent 对象。接着判断客户端是否配置了 chrootPath 属性，如果为 True 说明客户端配置了 chrootPath 属性。需要对接收到的节点路径进行 chrootPath 处理。最后调用 eventThread.queueEvent( ）方法将接收到的事件交给 EventThread 线程进行处理

```
if (replyHdr.getXid() == -1) {

    ...

    WatcherEvent event = new WatcherEvent();

    event.deserialize(bbia, "response");

    ...

    if (chrootPath != null) {

        String serverPath = event.getPath();

        if(serverPath.compareTo(chrootPath)==0)

            event.setPath("/");

            ...

            event.setPath(serverPath.substring(chrootPath.length()));

            ...

    }

    WatchedEvent we = new WatchedEvent(event);

    ...

    eventThread.queueEvent( we );

}
```

接下来我们来看一下 EventThread.queueEvent() 方法内部的执行逻辑。其主要工作分为 2 点： 第 1 步按照通知的事件类型，从 ZKWatchManager 中查询注册过的客户端 Watch 信息。客户端在查询到对应的 Watch 信息后，会将其从 ZKWatchManager 的管理中删除。因此这里也请你多注意，客户端的 Watcher 机制是一次性的，触发后就会被删除。

```
public Set<Watcher> materialize(...)

{

	Set<Watcher> result = new HashSet<Watcher>();

	...

	switch (type) {

    ...

	case NodeDataChanged:

	case NodeCreated:

	    synchronized (dataWatches) {

	        addTo(dataWatches.remove(clientPath), result);

	    }

	    synchronized (existWatches) {

	        addTo(existWatches.remove(clientPath), result);

	    }

	    break;

    ....

	}

	return result;

}
```

完成了第 1 步工作获取到对应的 Watcher 信息后，将查询到的 Watcher 存储到 waitingEvents 队列中，调用 EventThread 类中的 run 方法会循环取出在 waitingEvents 队列中等待的 Watcher 事件进行处理。

最后调用 processEvent(event) 方法来最终执行实现了 Watcher 接口的 process（）方法。

## 总结

大体上讲 ZooKeeper 实现的方式是通过客服端和服务端分别创建有观察者的信息列表。客户端调用 getData、exist 等接口时，首先将对应的 Watch 事件放到本地的 ZKWatchManager 中进行管理。服务端在接收到客户端的请求后根据请求类型判断是否含有 Watch 事件，并将对应事件放到 WatchManager 中进行管理。

在事件触发的时候服务端通过节点的路径信息查询相应的 Watch 事件通知给客户端，客户端在接收到通知后，首先查询本地的 ZKWatchManager 获得对应的 Watch 信息处理回调操作。这种设计不但实现了一个分布式环境下的观察者模式，而且通过将客户端和服务端各自处理 Watch 事件所需要的额外信息分别保存在两端，减少彼此通信的内容。大大提升了服务的处理性能。

* 客户端注册wathcer：

  + 首先会把该会话标记为带有 Watch 监控的事件请求，之后将watcher存储到zkwatchermanager
* 服务端注册watcher：

  + 解析收到的请求是否带有 Watch 注册事件
  + 将对应的 Watch 事件存储到 WatchManager
* 服务端触发watcher：

  + 在 注册watcher的方法内部执行完对节点数据的变更后，会调用 WatchManager.triggerWatch 方法触发数据变更事件。
  + 首先，封装了一个具有会话状态、事件类型、数据节点 3 种属性的 WatchedEvent 对象。
  + 之后查询该节点注册的 Watch 事件，如果为空说明该节点没有注册过 Watch 事件。
  + 如果存在 Watch 事件则添加到定义的 Wathcers 集合中，并在 WatchManager 管理中删除。最后，通过调用 process 方法向客户端发送通知。
* 客户端回调：

  + 客户端首先反序列化服务器发送请求头信息 并判断相属性字段 xid 的值为 -1，表示该请求响应为通知类型。
  + 在处理通知类型时，首先将己收到的字节流反序列化转换成 WatcherEvent 对象。一系列操作后，将接收到的事件交给 EventThread 线程进行处理
  + 线程内部做两个事情：1. 按照通知的事件类型，从 ZKWatchManager 中查询注册过的客户端 Watch 信息并将其从 ZKWatchManager 的管理中删除。

    2.将watcher存储到队列当中，调用函数循环从队列中取出事件处理