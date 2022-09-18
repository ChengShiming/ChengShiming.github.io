---
title: "可熔断的数据源：Java Fusible DataSource "
categories: 
  - Java 
toc: true
toc_sticky: true
---

Java Web服务接入第三方数据源时，可能会间歇性出现第三方数据源网络异常的情况，对服务的可用性产生影响，例如在服务启动时数据源不可用，那么会导致服务启动失败。本文针对该情况实现一种可熔断的数据源，能根据数据源的网络状况**自动切换**两种运行状态（正常、熔断），同时对状态切换提供回调方法，可实现一些异常通知功能；并对外暴露当前运行状态，让业务调用方能根据运行状态实现**fast-fail**，避免大量无用的调用、异常日志。

# 问题分析
数据源网络异常可能出现在两个时机：数据源创建时、数据源运行时。通过分析网络异常发生时打印的方法调用栈，可以看到数据源创建时由`HicariPool#checkFailFast()——>HicariPool#createPoolEntry()——>PoolBase#newConnect()——>DriverDataSource#getConnection()——>Socket#connect()`抛出网络连接异常`ConnectException`，同时服务会异常中止；数据源运行时网络异常由`DataSource#getConnection()`抛出SQL执行异常`SQLException`。 出现网络异常会导致这两个时机的异常需要分别处理。

# 开发实现
参考微服务架构中熔断器的实现，对资源进行隔离并捕获异常来完成熔断降级。本项目对上述两个网络异常时机的异常进行捕获并同时不断监听网络情况的方式来达到运行状态自动切换、对外暴露当前运行状态这两个目标。
对目标的访问进行控制，可以采用[代理模式（Proxy Pattern）](https://www.liaoxuefeng.com/wiki/1252599548343744/1281319432618017)来实现，。代理模式分为静态代理、动态代理，本项目需要代理的对象单一，所以采用静态代理更合适。

`HikariDataSourceProxy`作为`HikariDataSource`的代理类，代理类中包含运行状态字段、`HikariDataSource`的引用，在构造方法中实例化`HikariDataSource`，并创建`WatchDog`线程用于定期检查更新数据源的运行状态。重写`getConnection()`方法，捕获运行时中的网络异常并更新运行状态。

`JdbcTemplateProxy`作为`JdbcTemplate`的代理类，持有`HikariDataSourceProxy`引用并对外暴露数据源的运行状态。

当通过`JdbcTemplate`执行sql时，先调用`getCircuitState()`判断数据源运行状态，数据源处于熔断状态时可以让调用方快速失败。

___
**本文完整源代码地址：<https://github.com/ChengShiming/Java-web-wheel>**