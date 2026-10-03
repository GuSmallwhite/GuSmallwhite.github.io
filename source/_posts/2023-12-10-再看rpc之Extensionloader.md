---
title: "再看rpc之Extensionloader"
date: "2023-12-10 12:00:00"
permalink: "2023/12/10/再看rpc之Extensionloader/"
categories: ["rpc项目"]
tags: ["rpc"]
---

## 前言

看代码的时候一致不知道这个extensionloader和spi是干什么的

我们今天就来看一看

## SPI

dubbo里自定义了spi机制，与jdk的spi机制不同

```
package github.javaguide.extension;

import java.lang.annotation.Documented;
import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;

@Documented
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.TYPE)
public @interface SPI {
}
```

就是自定义了一个SPI注解，以后有这个注解的都能被dubbo的spi机制实现注入

## Holder

一个用来保存值的loader可以线程安全

```
package gu.extension;

public class Holder <T>{
    private volatile T value;
    public T get(){return value;}
    public void set(T value){this.value=value;}

}
```

## ExtensionLoader

核心类

### 静态变量

```
private static final String SERVICE_DIRECTORY = "META-INF/extensions/";//配置文件的目录,在resource的metaxxxxx包下
private static final Map<Class<?>,ExtensionLoader<?>> EXTENSIONLOADER_LOADERS=new ConcurrentHashMap<>();//每个类抖都有对应的唯一extensionloader
private static final Map<Class<?>, Object>EXTENSION_INSTANCES=new ConcurrentHashMap<>();//每个类对应的具体实例
private final Class<?>type;//接口类型
private ExtensionLoader(Class<?> type) {
    this.type = type;
}
private final  Map<String,Holder<Object>>cachedInstances=new ConcurrentHashMap<>();//用于缓存类的实现的实例
private final Holder<Map<String, Class<?>>> cachedClasses = new Holder<>();//用于换成扩展类的信息
```

**理一下EXTENSION\_INSTANCES和cachedInstances的区别**

扩展是什么？

> 在软件开发中，”扩展” 通常指的是对系统或框架的功能进行增加、定制或替换。在 Dubbo 框架中，”扩展” 特指对框架提供的接口（或扩展点）进行实现的具体类。这些实现类可以根据业务需求进行定制和替换，使系统具有更强大的功能。
>
> 在 Dubbo 中，扩展点是指框架定义的接口，而扩展是指这些接口的具体实现。框架通过扩展点和扩展的机制来实现灵活的功能定制和替换。Dubbo 使用 Java 的 SPI（Service Provider Interface）机制来实现扩展点和扩展的加载。
>
> 让我们以一个具体的例子来说明：
>
> 1. **扩展点（Extension Point）：** 假设有一个 `Car` 接口，表示汽车，其中定义了一些基本的汽车行为方法，如 `drive`、`stop` 等。
>
> ```
> >public interface Car {
>    void drive();
>    void stop();
> >}
> ```
>
> 2. **扩展（Extension）：** 然后，有两个实现类 `ElectricCar` 和 `GasolineCar`，分别表示电动汽车和汽油汽车，它们实现了 `Car` 接口。
>
> ```
> >public class ElectricCar implements Car {
>    @Override
>    public void drive() {
>        System.out.println("Electric car is driving.");
>    }
>
>    @Override
>    public void stop() {
>        System.out.println("Electric car stopped.");
>    }
> >}
>
> >public class GasolineCar implements Car {
>    @Override
>    public void drive() {
>        System.out.println("Gasoline car is driving.");
>    }
>
>    @Override
>    public void stop() {
>        System.out.println("Gasoline car stopped.");
>    }
> >}
> ```
>
> 3. **扩展加载（Extension Loading）：** Dubbo 框架通过扩展加载机制，根据配置或条件加载相应的扩展实现类。在上述例子中，可以通过配置选择使用 `ElectricCar` 或 `GasolineCar`。
>
> 这样，`Car` 接口就是一个扩展点，而 `ElectricCar` 和 `GasolineCar` 就是这个扩展点的扩展。 Dubbo 框架会负责加载这些扩展实现，使得开发者可以根据实际需求进行选择和定制。这就是 Dubbo 中所说的 “扩展”。

所以这里面

```
private static final Map<Class<?>, Object>EXTENSION_INSTANCES=new ConcurrentHashMap<>();//每个接口类对应的扩展实例，你可以理解为策略模式里面，实现虚拟接口的类，比如接口是Car,那么扩展就是bus,suv...
```

而**cachedInstances**就是用来存放扩展名和他对应的holder，holder用来实现单例模式，懒加载

**cachedInstances**作为一个局部缓存，我们获得instance会优先从cachedInstances里面获得

如果没有，再从全局缓存**EXTENSION\_INSTANCES**里面去啊找

### getExtensionLoader

逻辑都在注释里面，比较好理解

```
//获得类型对应的Loader
public static <S> ExtensionLoader<S>getExtensionLoader(Class<S> type)//前面一个S表示这个泛型方法的泛型参数类型是S
{
    if (type == null) {
        throw new IllegalArgumentException("Extension type should not be null.");
    }
    if (!type.isInterface()) {
        throw new IllegalArgumentException("Extension type must be an interface.");//必须是个接口，你联系策略模式理解
    }
    if (type.getAnnotation(SPI.class) == null) {
        throw new IllegalArgumentException("Extension type must be annotated by @SPI");//必须有SPI注解
    }
    ExtensionLoader<S> extensionLoader=(ExtensionLoader<S>) EXTENSIONLOADER_LOADERS.get(type);//从map里面获得
    //如果没有就加入
    if(extensionLoader==null)
    {
        EXTENSIONLOADER_LOADERS.putIfAbsent(type,new ExtensionLoader<S>(type));
        extensionLoader = (ExtensionLoader<S>) EXTENSION_LOADERS.get(type);
    }
    return  extensionLoader;
}
```

### getExtension

获得扩展，在没有实例的情况下调用了create方法

```
public T getExtension(String name)//其实是从局部换成里面寻找
{
    if(StringUtil.isBlank(name))
    {
        throw new IllegalArgumentException("Extension name should not be null or empty.");
    }
    Holder<Object> holder = cachedInstances.get(name);//检查局部缓存里面是不是有了，没有就创造然后put
    if(holder==null)
    {
        cachedInstances.put(name,new Holder<>());
        holder=cachedInstances.get(name);
    }
    Object instance=holder.get();
    //单例模式
    if (instance == null) {
        synchronized (holder) {
            instance = holder.get();
            if (instance == null) {
                instance = createExtension(name);//调用方法从全局缓存里面找
                holder.set(instance);//放到局部缓存里
            }
        }
    }
    return (T)instance;
}
```

### createExtension

虽然名字是create，但是真实逻辑是先从全局缓存找，没有才去创键

```
private T createExtension(String name)//创建扩展实例
{
    Class<?> clazz = getExtensionClasses().get(name);//这个方法后面细说
    if (clazz == null) {
        throw new RuntimeException("No such extension of name " + name);
    }
    T instance = (T) EXTENSION_INSTANCES.get(clazz);//获取实例
    if(instance==null)
    {
        try {
            EXTENSION_INSTANCES.putIfAbsent(clazz, clazz.newInstance());
            instance = (T) EXTENSION_INSTANCES.get(clazz);
        } catch (Exception e) {
            log.error(e.getMessage());
        }
    }
    return  instance;
}
```

### getExtensionClasses

从缓存里取得所有扩展的类

> 1. 第一次检查，再次尝试从缓存中获取，防止在等待锁的过程中其他线程已经加载了扩展类。
> 2. 如果仍然没有获取到，进入同步块，再次尝试从缓存中获取，如果仍然为空，就创建一个新的 `HashMap` 对象，通过 `loadDirectory` 方法加载目录下的扩展类，并将加载的扩展类放入新创建的 `HashMap` 中，最后将这个 `HashMap` 缓存起来。

```
private Map<String, Class<?>> getExtensionClasses()
  {
      Map<String,Class<?>>classes=cachedClasses.get();
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
      return  classes;
  }
```

### loadDirectory

这个方法通过 `ClassLoader` 获取指定目录下的所有配置文件，并且逐个加载这些配置文件。然后调用 `loadResource` 方法来解析每个配置文件的内容。

```
private void loadDirectory(Map<String, Class<?>> extensionClasses)
{
    String fileName=ExtensionLoader.SERVICE_DIRECTORY+type.getName();//配置文件的相对路径
    try
    {
        Enumeration<URL>urls;//存放所有配置文件的url，我的理解就是文件里的内容
        ClassLoader classLoader = ExtensionLoader.class.getClassLoader();
        urls=classLoader.getResources(fileName);
        if(urls!=null)
        {
            while (urls.hasMoreElements())
            {
                URL resourceUrl = urls.nextElement();
                loadResource(extensionClasses, classLoader, resourceUrl);//调用方法加载资源
            }
        }
    }catch (IOException e)
    {
        log.error(e.getMessage());
    }

}
```

### loaderResource

真正进行加载资源的方法

```
//加载类资源，把名字和资源的对应关系放到map里
private void loadResource(Map<String, Class<?>> extensionClasses, ClassLoader classLoader, URL resourceUrl)
{
    try(BufferedReader reader = new BufferedReader(new InputStreamReader(resourceUrl.openStream(), UTF_8)))
    {
        String line;
        while ((line=reader.readLine())!=null)
        {
            //先获得注释的部分，把注释切掉
            final int ci = line.indexOf('#');
            if (ci >= 0) {
                // string after # is comment so we ignore it
                line = line.substring(0, ci);
            }
            line=line.trim();//把空格去掉
            //如果把杂七杂八的东西全去掉之后，还有内容，说明是有效的
            if(line.length()>0)
            {
                try
                {
                    final int ei = line.indexOf('=');//寻找等号的位置，前面就是扩展点名字，后面就是具体的类所在的位置
                    String name = line.substring(0, ei).trim();
                    String clazzName = line.substring(ei + 1).trim();
                    //检查两个全部是否有效，有就加载了，然后放到map里
                    if (name.length() > 0 && clazzName.length() > 0) {
                        Class<?> clazz = classLoader.loadClass(clazzName);
                        extensionClasses.put(name, clazz);
                    }
                }catch (ClassNotFoundException e)
                {
                    log.error(e.getMessage());
                }
            }
        }
    }catch (IOException e)
    {
        log.error(e.getMessage());
    }
}
```