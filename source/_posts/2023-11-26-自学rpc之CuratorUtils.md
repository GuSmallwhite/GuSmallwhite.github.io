---
title: "自学rpc之CuratorUtils"
date: "2023-11-26 12:00:00"
permalink: "2023/11/26/自学rpc之CuratorUtils/"
categories: ["rpc项目"]
tags: ["rpc", "动态代理", "zk"]
---

## 前言

有一说一这个比较简单，会curator的那些api就能看懂

## 成员变量

```
private static final int BASE_SLEEP_TIME = 1000;//基础等待时间，也就是过多少秒重试
private static final int MAX_RETRIES = 3;//最大重试次数
public static final String ZK_REGISTER_ROOT_PATH = "/my-rpc";//根路径名字
private static final Map<String, List<String>> SERVICE_ADDRESS_MAP = new ConcurrentHashMap<>();//缓存服务名称和对应服务实例列表的映射关系
private static final Set<String> REGISTERED_PATH_SET = ConcurrentHashMap.newKeySet();//储已经注册的服务路径
private static CuratorFramework zkClient;//CuratorFramework 实例，用于与 ZooKeeper 服务器建立连接。
private static final String DEFAULT_ZOOKEEPER_ADDRESS = "127.0.0.1:2181";//默认zookeeper服务地址
```

### createPersistentNode

```
public static void createPersistentNode(CuratorFramework zkClient, String path) {
    try {
        if (REGISTERED_PATH_SET.contains(path) || zkClient.checkExists().forPath(path) != null) {
            log.info("The node already exists. The node is:[{}]", path);
        } else {
            //eg: /my-rpc/github.javaguide.HelloService/127.0.0.1:9999
            zkClient.create().creatingParentsIfNeeded().withMode(CreateMode.PERSISTENT).forPath(path);//创建持久化节点
            log.info("The node was created successfully. The node is:[{}]", path);
        }
        REGISTERED_PATH_SET.add(path);//加入已经注册过了的set中
    } catch (Exception e) {
        log.error("create persistent node for path [{}] fail", path);
    }
}
```

用于在path下创建一个持久节点

### getChildrenNodes

```
public static List<String> getChildrenNodes(CuratorFramework zkClient, String rpcServiceName) {
    if (SERVICE_ADDRESS_MAP.containsKey(rpcServiceName)) {
        return SERVICE_ADDRESS_MAP.get(rpcServiceName);//已经是存到map里了那就直接返回就行了
    }
    List<String> result = null;
    String servicePath = ZK_REGISTER_ROOT_PATH + "/" + rpcServiceName;
    try {
        result = zkClient.getChildren().forPath(servicePath);//从ZooKeeper中获取指定路径（servicePath）下的子节点列表。
        SERVICE_ADDRESS_MAP.put(rpcServiceName, result);//放进map里
        registerWatcher(rpcServiceName, zkClient);//调用registerWatcher(rpcServiceName, zkClient)方法注册一个监听器，以便在子节点发生变化时能够及时更新缓存。
    } catch (Exception e) {
        log.error("get children nodes for path [{}] fail", servicePath);
    }
    return result;
}
```

用于获取服务名称对应的所有子节点

registerwatcher是什么？看下面

### registerWatcher

```
private static void registerWatcher(String rpcServiceName, CuratorFramework zkClient) throws Exception {
    String servicePath = ZK_REGISTER_ROOT_PATH + "/" + rpcServiceName;//服务名
    //创建一个监听指定路径下子节点变更的缓存对象pathChildrenCache。
    PathChildrenCache pathChildrenCache = new PathChildrenCache(zkClient, servicePath, true);
    //当子节点变更事件发生时，会重新获取指定路径下的所有子节点，并更新SERVICE_ADDRESS_MAP缓存。
    PathChildrenCacheListener pathChildrenCacheListener = (curatorFramework, pathChildrenCacheEvent) -> {
        List<String> serviceAddresses = curatorFramework.getChildren().forPath(servicePath);
        SERVICE_ADDRESS_MAP.put(rpcServiceName, serviceAddresses);
    };
    pathChildrenCache.getListenable().addListener(pathChildrenCacheListener);//将监听器添加到缓存对象中。
    pathChildrenCache.start();//启动监听器，使其开始监听指定路径下子节点的变化。
}
```

这个函数的作用是创建并启动一个监听器，监控指定路径下子节点的变化，并在变化发生时更新`SERVICE_ADDRESS_MAP`缓存。这样，当ZooKeeper上的子节点发生变化时，`SERVICE_ADDRESS_MAP`会得到及时的更新。

只要初始化时调用了`registerWatcher`并执行了`pathChildrenCache.start()`，监听器就会一直在后台运行。

`getChildrenNodes`方法只是为了初始化子节点信息，并在初始化时启动了监听器，但监听器的生命周期是独立于`getChildrenNodes`方法的。

### clearRegistry

```
public static void clearRegistry(CuratorFramework zkClient, InetSocketAddress inetSocketAddress) {
    REGISTERED_PATH_SET.stream().parallel().forEach(p -> {
        try {
            if (p.endsWith(inetSocketAddress.toString())) {
                zkClient.delete().forPath(p);
            }
        } catch (Exception e) {
            log.error("clear registry for path [{}] fail", p);
        }
    });
    log.info("All registered services on the server are cleared:[{}]", REGISTERED_PATH_SET.toString());
}
```

这个很简单就是删除这个地址对应的所有节点

### getZkClient

```
public static CuratorFramework getZkClient() {
    // 检查用户是否已经设置了 ZooKeeper 的地址，通过读取配置文件中的配置项获取
    Properties properties = PropertiesFileUtil.readPropertiesFile(RpcConfigEnum.RPC_CONFIG_PATH.getPropertyValue());
    //如果 zkClient 不为 null 且其状态为 CuratorFrameworkState.STARTED，表示 ZooKeeper 客户端已经启动，直接返回这个实例。
    String zookeeperAddress = properties != null && properties.getProperty(RpcConfigEnum.ZK_ADDRESS.getPropertyValue()) != null ? properties.getProperty(RpcConfigEnum.ZK_ADDRESS.getPropertyValue()) : DEFAULT_ZOOKEEPER_ADDRESS;
    // if zkClient has been started, return directly
    if (zkClient != null && zkClient.getState() == CuratorFrameworkState.STARTED) {
        return zkClient;
    }
    //指定重试策略
    RetryPolicy retryPolicy = new ExponentialBackoffRetry(BASE_SLEEP_TIME, MAX_RETRIES);
    //新建zk实例
    zkClient = CuratorFrameworkFactory.builder()
            // the server to connect to (can be a server list)
            .connectString(zookeeperAddress)
            .retryPolicy(retryPolicy)
            .build();
    zkClient.start();
    try {
        // wait 30s until connect to the zookeeper
        if (!zkClient.blockUntilConnected(30, TimeUnit.SECONDS)) {
            throw new RuntimeException("Time out waiting to connect to ZK!");
        }
    } catch (InterruptedException e) {
        e.printStackTrace();
    }
    return zkClient;
}
```

**PropertiesFileUtil**见别的博客