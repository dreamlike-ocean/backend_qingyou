# Table of contents

* [概述](README.md)

## Java基础

* [反射和注解](java基础/反射和注解.md)
* [泛型和容器类](java基础/泛型和容器类.md)
* [内部类和lambda](java基础/内部类和lambda.md)
  * [invokedynamic与字符串拼接](java基础/stringConcatViaInvokeDynamic.md) 

* [IO流](java基础/io流.md)
* [FileChannel](java基础/FileChannel.md)
* [ByteBuffer和native内存](java基础/ByteBuffer和native内存.md)
* [MMap](java基础/Mmap.md)
* [selector与nio](java基础/selector.md)
* [异常](java基础/异常.md)
* [并发编程](java基础/多线程和并发导论.md)
  * [线程池](java基础/线程池.md)
  * [线程安全-1](java基础/线程安全(1).md)
  * [线程安全-2](java基础/线程安全(2).md)
  * [volatile,JMM和JDK9内存顺序简述](java基础/volatile，JMM和jdk9内存顺序简论.md)
* [java socket api及其native实现](java基础/socket.md)
* [拓展内容](java基础/拓展内容.md)

## jvm

- [概述](jvm/JVM概述.md)
- [常量池](jvm/常量池.md)
- [JVM方法调用(一)](jvm/JVM方法调用(一).md)

## GIT

- [GIT](git/AllInOne.md)

## 构建工具

* [Maven](构建工具/maven_get_start.md)
  * [依赖](构建工具/maven_dependencies.md) 
  * [仓库](构建工具/maven_repositories.md)
  * [插件](构建工具/maven_plugin.md)
  * [idea](构建工具/maven_idea.md)

## web开发

- [mybatis](mybatis/get-start.md)
  - [select](mybatis/select.md)
  - [insert update delete](mybatis/insert%2Cupdate%2Cdelete.md)
  - [动态sql标签](mybatis/%E9%82%A3%E4%BA%9Bxml%E6%A8%A1%E6%9D%BF%E6%A0%87%E7%AD%BE.md)
  - [配置杂谈](mybatis/%E9%85%8D%E7%BD%AE%E6%9D%82%E8%B0%88.md)

## MySQL

  * [MySQL基础](MySQL/MySQL基础.md)
  * [储存引擎与索引](MySQL/MySQL高级-1存储引擎与索引.md)
  * [索引的使用和优化](MySQL/MySQL高级-2索引的使用和优化.md)
  * [事务和锁](MySQL/MySQL高级-3事务和锁.md)
  * [总集](MySQL/MySQL高级-总集.md)

## Redis
* [Redis](redis/Redis%E7%AC%94%E8%AE%B0.md)

## 网络协议
* [TCP]
* [HTTP](网络协议/http/http(1)概述.md)
  * [URI](网络协议/http/http(2)uri.md)
  * [协议组成](网络协议/http/http(3)协议构成.md)
  * [REST](网络协议/http/http(4)restful.md)
  * [httpclient](网络协议/http/http(5)jdk的httpclient.md)
  * [http版本比较](网络协议/http/http(6)http版本比较.md)

## os

* [什么是进程](os/什么是进程.md)

## dreamlike的私货


* Loom
  * [openjdk loom](https://openjdk.org/projects/loom/) 
  * [fiber and continuation](dreamlike的私货/Project%20Loom%20Java虚拟机的纤程和计算续体.md)
  * [State of Loom - Part 1](/dreamlike的私货/state_of_loom_part1.md)
  * [State of Loom - Part 2](/dreamlike的私货/state_of_loom_part2.md)
  * [Spring的Loom化改造](dreamlike的私货/Spring的loom化改造.md)
  * [loom的实现](dreamlike的私货/loom的实现.md)
  * [为什么jvm需要有栈协程](dreamlike的私货/为什么jvm需要有栈协程.md) 
* [响应式的锁](dreamlike的私货/%E5%93%8D%E5%BA%94%E5%BC%8F%E7%9A%84%E9%94%81.md)
* [[翻译]为什么现代的jvm仍然偏爱安全点](dreamlike的私货/【翻译】为什么现代的JVM分析器仍然偏爱安全点？.md)
* [io_uring介绍](dreamlike的私货/io_uring.md)
* CPP

  * [C++20协程入门](dreamlike的私货/cpp_coroutine/first.md)
  * [async scope和通用回调转协程](dreamlike的私货/cpp_coroutine/async_scope.md)
* Panama FFI

  * [Panama浅析](dreamlike的私货/Panama浅析.md)