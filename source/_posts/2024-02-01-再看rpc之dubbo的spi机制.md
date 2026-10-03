---
title: "再看rpc之dubbo的spi机制"
date: "2024-02-01 12:00:00"
permalink: "2024/02/01/再看rpc之dubbo的spi机制/"
categories: ["rpc项目", "面向面试"]
tags: ["zk", "rpc", "面试"]
---

## 前言

简历上写了spi，我就得搞明白，项目里的spi是怎么用的，当然是参考了dubbo，那么今天来看看如何使用和理解

## SPI是什么

SPI 全称为 Service Provider Interface，是一种服务发现机制。SPI 的本质是将接口实现类的全限定名配置在文件中，并由服务加载器读取配置文件，加载实现类。这样可以在运行时，动态为接口替换实现类。

## 我的spi（dubbo的

Dubbo 并未使用 Java SPI，而是重新实现了一套功能更强的 SPI 机制。

Dubbo SPI 的相关逻辑被封装在了 ExtensionLoader 类中，通过 ExtensionLoader，我们可以加载指定的实现类。Dubbo SPI 所需的配置文件需放置在 META-INF/dubbo 路径下，配置内容示例如下：

```
optimusPrime = org.apache.spi.OptimusPrime
bumblebee = org.apache.spi.Bumblebee
```

与 Java SPI 实现类配置不同，Dubbo SPI 是通过键值对的方式进行配置，这样我们可以按需加载指定的实现类。另外，在测试 Dubbo SPI 时，需要在 Robot 接口上标注 @SPI 注解。示例：

```
public class DubboSPITest {

    @Test
    public void sayHello() throws Exception {
        ExtensionLoader<Robot> extensionLoader = 
            ExtensionLoader.getExtensionLoader(Robot.class);
        Robot optimusPrime = extensionLoader.getExtension("optimusPrime");
        optimusPrime.sayHello();
        Robot bumblebee = extensionLoader.getExtension("bumblebee");
        bumblebee.sayHello();
    }
}
```

自己的项目中的用法：

```
serviceRegistry=ExtensionLoader.getExtensionLoader(ServiceRegistry.class).getExtension(ServiceRegistryEnum.ZK.getName());

this.serviceDiscovery=ExtensionLoader.getExtensionLoader(ServiceDiscovery.class).getExtension(ServiceDiscoveryEnum.ZK.getName())

/.../
```

都是通过获取这个类对应的extensionloader再通过键值的方式，根据名称获取对应的实现类

## 项目、dubbo之spi源码分析

从 ExtensionLoader 的 getExtension 方法作为入口，对拓展类对象的获取过程进行详细的分析

**getExtension**

```
public T getExtension(String name) {
    if (name == null || name.length() == 0)
        throw new IllegalArgumentException("Extension name == null");
    if ("true".equals(name)) {
        // 获取默认的拓展实现类
        return getDefaultExtension();
    }
    // Holder，顾名思义，用于持有目标对象
    Holder<Object> holder = cachedInstances.get(name);
    if (holder == null) {
        cachedInstances.putIfAbsent(name, new Holder<Object>());
        holder = cachedInstances.get(name);
    }
    Object instance = holder.get();
    // 双重检查
    if (instance == null) {
        synchronized (holder) {
            instance = holder.get();
            if (instance == null) {
                // 创建拓展实例
                instance = createExtension(name);
                // 设置实例到 holder 中
                holder.set(instance);
            }
        }
    }
    return (T) instance;
}
```

这个简单，就是看缓存里有没有这样的目标对象，如果有就直接返回

如果没有就通过`createExtension`创建一个单例

下面我们来看一下创建拓展对象的过程是怎样的。

**createExtension**

这里我的项目的实现和dubbo的实现有所区别

我的项目的：

```
private T createExtension(String name) {
    // 从配置文件里读所有的拓展类，得到配置项名称和配置类的映射表
    Class<?> clazz = getExtensionClasses().get(name);
    if (clazz == null) {
        throw new RuntimeException("No such extension of name " + name);
    }
    T instance = (T) EXTENSION_INSTANCES.get(clazz);
    if (instance == null) {
        try {
            EXTENSION_INSTANCES.putIfAbsent(clazz, clazz.newInstance());
            instance = (T) EXTENSION_INSTANCES.get(clazz);
        } catch (Exception e) {
            log.error(e.getMessage());
        }
    }
    return instance;
}
```

dubbo的

```
private T createExtension(String name) {
    // 从配置文件中加载所有的拓展类，可得到“配置项名称”到“配置类”的映射关系表
    Class<?> clazz = getExtensionClasses().get(name);
    if (clazz == null) {
        throw findException(name);
    }
    try {
        T instance = (T) EXTENSION_INSTANCES.get(clazz);
        if (instance == null) {
            // 通过反射创建实例
            EXTENSION_INSTANCES.putIfAbsent(clazz, clazz.newInstance());
            instance = (T) EXTENSION_INSTANCES.get(clazz);
        }
        // 向实例中注入依赖
        injectExtension(instance);
        Set<Class<?>> wrapperClasses = cachedWrapperClasses;
        if (wrapperClasses != null && !wrapperClasses.isEmpty()) {
            // 循环创建 Wrapper 实例
            for (Class<?> wrapperClass : wrapperClasses) {
                // 将当前 instance 作为参数传给 Wrapper 的构造方法，并通过反射创建 Wrapper 实例。
                // 然后向 Wrapper 实例中注入依赖，最后将 Wrapper 实例再次赋值给 instance 变量
                instance = injectExtension(
                    (T) wrapperClass.getConstructor(type).newInstance(instance));
            }
        }
        return instance;
    } catch (Throwable t) {
        throw new IllegalStateException("...");
    }
}
```

dubbo的实现步骤是：

1. 通过 `getExtensionClasses` 获取所有的拓展类
2. 通过反射创建拓展对象
3. 向拓展对象中注入依赖
4. 将拓展对象包裹在相应的 Wrapper 对象中

相较于我的，多了向拓展对象中注入依赖和包裹在响应的wrapper当中

**getExtensionClasses**

dubbo的实现：

```
private Map<String, Class<?>> getExtensionClasses() {
    // 从缓存中获取已加载的拓展类
    Map<String, Class<?>> classes = cachedClasses.get();
    // 双重检查
    if (classes == null) {
        synchronized (cachedClasses) {
            classes = cachedClasses.get();
            if (classes == null) {
                // 加载拓展类
                classes = loadExtensionClasses();
                cachedClasses.set(classes);
            }
        }
    }
    return classes;
}
```

首先还是从缓存中加载，没有再通过`loadExtensionClasses()`方法加载拓展类

我的实现略有不同：

```
private Map<String, Class<?>> getExtensionClasses() {
    // get the loaded extension class from the cache
    Map<String, Class<?>> classes = cachedClasses.get();
    // double check
    if (classes == null) {
        synchronized (cachedClasses) {
            classes = cachedClasses.get();
            if (classes == null) {
                classes = new HashMap<>();
                // load all extensions from our extensions directory
                loadDirectory(classes);
                cachedClasses.set(classes);
            }
        }
    }
    return classes;
}
```

我没找到就直接`loadDirectory`了，直接从指定目录读取配置文件，而dubbo还需要对spi注解进行解析，看是否符合规范

**loadDirectory**

dubbo和我的做的是同样的事，就是先通过 classLoader 获取所有资源链接，然后再通过 loadResource 方法加载资源

```
private void loadDirectory(Map<String, Class<?>> extensionClasses, String dir) {
    // fileName = 文件夹路径 + type 全限定名 
    String fileName = dir + type.getName();
    try {
        Enumeration<java.net.URL> urls;
        ClassLoader classLoader = findClassLoader();
        // 根据文件名加载所有的同名文件
        if (classLoader != null) {
            urls = classLoader.getResources(fileName);
        } else {
            urls = ClassLoader.getSystemResources(fileName);
        }
        if (urls != null) {
            while (urls.hasMoreElements()) {
                java.net.URL resourceURL = urls.nextElement();
                // 加载资源
                loadResource(extensionClasses, classLoader, resourceURL);
            }
        }
    } catch (Throwable t) {
        logger.error("...");
    }
}
```

**loadResource**

```
private void loadResource(Map<String, Class<?>> extensionClasses, ClassLoader classLoader, URL resourceUrl) {
    try (BufferedReader reader = new BufferedReader(new InputStreamReader(resourceUrl.openStream(), UTF_8))) {
        String line;
        // 按行读取
        while ((line = reader.readLine()) != null) {
            // #后面的是注释，不要了
            final int ci = line.indexOf('#');
            if (ci >= 0) {
                // string after # is comment so we ignore it
                line = line.substring(0, ci);
            }
            line = line.trim();
            if (line.length() > 0) {
                try {
                    //以‘=’为界，分割键值
                    final int ei = line.indexOf('=');
                    String name = line.substring(0, ei).trim();
                    String clazzName = line.substring(ei + 1).trim();
                    // our SPI use key-value pair so both of them must not be empty
                    if (name.length() > 0 && clazzName.length() > 0) {
                        //合法则读取类并且缓存
                        Class<?> clazz = classLoader.loadClass(clazzName);
                        extensionClasses.put(name, clazz);
                    }
                } catch (ClassNotFoundException e) {
                    log.error(e.getMessage());
                }
            }

        }
    } catch (IOException e) {
        log.error(e.getMessage());
    }
}
```

## dubbo这样的啊好处

Dubbo SPI 更加灵活，可扩展性更强，能够更好地适应分布式服务框架的需求。