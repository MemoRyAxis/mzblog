---
title: 谈谈 SLF4j 和 javax.Validation 实现类的加载方式
date: 2018-03-05 23:50:20
tags:
---

但是你也许注意到了, 在我们项目中使用到日志, 校验等功能时, 只是使用 SLF4J, Validation 的接口, 而没有去获取具体实现类的对象. 
那么, 他们是如何获取和加载的实现类的呢?


## 1. 前言

SLF4J 是最经典的门面模式, SLF4J 以接口的形式提供的实现日志框架的核心 API.
但其并不提供具体的实现, 而是由 log4j2, logback 等项目负责实现.

javax.Validation 是随着 J2EE 6 发布的 Java 规范, 其抽象了 Java 开发过程中对领域模型和方法的校验. 
其提供了验证元数据和校验接口, 但一样没有提供具体的实现, 而是由 Hibernate-Validator 负责实现.

下面是项目中我们初始化这些接口类的代码:

```Java
    org.slf4j.Logger log = org.slf4j.LoggerFactory.getLogger(A.class);
```

```Java
    javax.validation.ValidatorFactory factory = javax.validation.Validation.buildDefaultValidatorFactory();
    javax.validation.Validator validator = factory.getValidator();
```

仔细的读者可以看到上面代码没有将包名放入 import 中, 而是直接带在类名上.
也许更细心的你可以理解手写这段代码的用心良苦, 你可能发现了我们在声明这些接口时, __并没有声明任何相关的实现类__.

那么大家可能有下面的疑问:
1. SLF4J 怎么知道我使用的日志框架是 log4j 还是 logback 呢?
2. 使用 javax.validation 如果不引用相关的实现框架还能正常校验对象么?

这篇文章便尝试为大家解开这些疑惑.


## 2. Bean Validation 实现类的加载方式

___基于接口 (SPI)___

启动类 `Validation` 在获取校验器时, __会使用 `ServiceLoader` 从类加载器中获取所有实现接口 `ValidationProvider` 的类__.

源码如下:

```Java
        /**
         * @see javax.validation.Validation.GetValidationProviderListAction#loadProviders(ClassLoader)) 
         */
        private List<ValidationProvider<?>> loadProviders(ClassLoader classloader) {
            ServiceLoader<ValidationProvider> loader = ServiceLoader.load( ValidationProvider.class, classloader );
            Iterator<ValidationProvider> providerIterator = loader.iterator();
            List<ValidationProvider<?>> validationProviderList = new ArrayList<ValidationProvider<?>>();
            while ( providerIterator.hasNext() ) {
                try {
                    validationProviderList.add( providerIterator.next() );
                }
                catch ( ServiceConfigurationError e ) {
                    // ignore, because it can happen when multiple
                    // providers are present and some of them are not class loader
                    // compatible with our API.
                }
            }
            return validationProviderList;
        }
```

> ps: 
> java.validation 并没有为接口提供默认实现, 所以如果没有引入具体的实现框架时, 上面代码初始化后其实是没有实现类 (校验器) 的!
> 声明接口时并不会抛出异常, 但在使用校验器时便会抛出异常, 并提示没有引入具体的实现类.

+ 优点: 
    + 相对来说容易理解, ServiceLoader 封装具体的细节, 实现框架的开发只需要实现具体的接口并满足 SPI 的要求即可.
+ 缺点:
    + 使用时加载, 如果没有提供任何具体实现时, 程序会在运行时抛出异常.


## 3. SLF4J 实现类的加载方式

___基于约定___

约定实现 SLF4J 的项目中一定要有类 `org.slf4j.impl.StaticLoggerBinder`, 并且实现 `#getSingleton()` 等一系列方法.

源码如下: 

A: __调用_日志实现框架_中的 `StaticLoggerBinder.getSingleton()` 完成初始化 (绑定), 由静态代码块实现__
B: 如果找不到 `StaticLoggerBinder` 使用默认的接口实现 `NOPLoggerFactory`, 即没有任务操作 (不打印日志) 的接口实现
C: 如果找不到 `#getSingleton()` 方法, 则提示日志实现框架绑定失败

```Java
    /**
     * @see org.slf4j.LoggerFactory#bind()
     */
    private final static void bind() {
        try {
            Set<URL> staticLoggerBinderPathSet = null;
            // skip check under android, see also
            // http://jira.qos.ch/browse/SLF4J-328
            if (!isAndroid()) {
                staticLoggerBinderPathSet = findPossibleStaticLoggerBinderPathSet();
                reportMultipleBindingAmbiguity(staticLoggerBinderPathSet);
            }
            // the next line does the binding
(A)→        StaticLoggerBinder.getSingleton();
            INITIALIZATION_STATE = SUCCESSFUL_INITIALIZATION;
            reportActualBinding(staticLoggerBinderPathSet);
            fixSubstituteLoggers();
            replayEvents();
            // release all resources in SUBST_FACTORY
            SUBST_FACTORY.clear();
(B)→    } catch (NoClassDefFoundError ncde) {
            String msg = ncde.getMessage();
            if (messageContainsOrgSlf4jImplStaticLoggerBinder(msg)) {
                INITIALIZATION_STATE = NOP_FALLBACK_INITIALIZATION;
                Util.report("Failed to load class \"org.slf4j.impl.StaticLoggerBinder\".");
                Util.report("Defaulting to no-operation (NOP) logger implementation");
                Util.report("See " + NO_STATICLOGGERBINDER_URL + " for further details.");
            } else {
                failedBinding(ncde);
                throw ncde;
            }
(C)→    } catch (java.lang.NoSuchMethodError nsme) {
            String msg = nsme.getMessage();
            if (msg != null && msg.contains("org.slf4j.impl.StaticLoggerBinder.getSingleton()")) {
                INITIALIZATION_STATE = FAILED_INITIALIZATION;
                Util.report("slf4j-api 1.6.x (or later) is incompatible with this binding.");
                Util.report("Your binding is version 1.5.5 or earlier.");
                Util.report("Upgrade your binding to version 1.6.x.");
            }
            throw nsme;
        } catch (Exception e) {
            failedBinding(e);
            throw new IllegalStateException("Unexpected initialization failure", e);
        }
    }
```

> ps: 
> 大家在这里可以拿旁边的项目动手试一下, 使用源码阅读工具定位到 `LoggerFactory#bind()`. 
> 然后将项目排除所有日志实现框架, 如 log4j2, logback 等, 但保留 SLF4J.
> 如果 (A) 处代码没有报错, 那么进入 `StaticLoggerBinder`, 看看是哪个包实现的它, 再去排除一下依赖.
> 如果报错了, Congratulations! 不妨回头看一看, 你是不是已经理解这种加载方式了呢呢?

+ 优点:
    + 静态代码块在类加载的时候执行, 所以在项目启动时即可知道是否加载了接口的实现类, 可以更早的发现风险.
+ 缺点:
    + 基于约定需要对约定有足够的了解, 相对于基于接口编码更为复杂.


## 4. 结语

文章到这就结束了, 这篇文章是否对大家有用呢? 本文还涉及了大量知识点, 最后也会有一些参考链接, 大家有兴趣可以继续扩展扩展.


## 参考链接:

1. [从源码来理解slf4j的绑定，以及logback对配置文件的加载](https://www.cnblogs.com/youzhibing/p/6849843.html)
1. [Bean Validation specification](http://beanvalidation.org/2.0/spec/)
1. [ServiceLoader (Java Platform SE 7 )](https://docs.oracle.com/javase/7/docs/api/java/util/ServiceLoader.html)
1. [详谈ServiceLoader实现原理](http://www.jb51.net/article/106516.htm)
1. [difference-between-spi-and-api](https://stackoverflow.com/questions/2954372/difference-between-spi-and-api)
1. slf4j-api 报错的代码也能打包?






