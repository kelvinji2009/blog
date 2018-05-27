---
title: "加强Docker容器与Java10集成(译)"
date: 2018-05-27T01:47:22+08:00
draft: false
---

# 加强Docker容器与Java10集成(译)

[原文链接](https://blog.docker.com/2018/04/improved-docker-container-integration-with-java-10/)

很多运行在JAVA虚拟机(JVM)中的应用，包括数据服务如Apache Spark和Kafka以及传统企业应用，都运行在容器中。最近，运行在容器里的JVM出现了由于内存和CPU资源限制和使用率导致性能损失问题。这是因为JAVA意识不到自己运行在容器中。随着JAVA 10的发布，JVM终于可以通过设置cgroup来实现这些约束。内存和CPU约束都可以直接在容器里被用来管理JAVA应用，包括：
* 在容器里面设置内存约束
* 在容器里设置可用的CPUs个数
* 在容器里设置CPU约束

JAVA 10的这些优化在Docker for mac/windows和Docker企业版均可正常运行。

## 容器内存限制
一直到JAVA 9，JVM依然不能够通过在容器中使用标识来识别内存或CPU限制。在Java 10中，内存限制可以被自动识别，而且这个特性是默认支持的。

JAVA对服务器进行分级定义，如果一台服务器有2CPUs和2GB内存，那么默认的堆栈大小是1/4的物理内存。假设一个安装了docker企业版的服务器有4CPUs和2GB内存。我们来比较分别比较运行JAVA 8和JAVA 10的容器的区别。先看JAVA 8：

```shell
docker container run -it -m512 --entrypoint bash openjdk:latest

$ docker-java-home/bin/java -XX:+PrintFlagsFinal -version | grep MaxHeapSize
    uintx MaxHeapSize                              := 524288000                          {product}
openjdk version "1.8.0_162"
```

我们可以看到最大的堆栈大小是512MB，刚好（1/4）X2GB，只是通过docker EE自动设置的，而不是通过设置容器来实现的。做个比较，我们看下在JAVA 10环境下运行同样的命令是什么结果。

```shell
docker container run -it -m512M --entrypoint bash openjdk:10-jdk

$ docker-java-home/bin/java -XX:+PrintFlagsFinal -version | grep MaxHeapSize
   size_t MaxHeapSize                              = 134217728                                {product} {ergonomic}
openjdk version "10" 2018-03-20
```
上面的结果显示，容器里的内存限制非常接近我们期望的128MB。

## 设置可用CPUs个数

默认情况下，每个容器可以获取到主机的CPU时钟周期是没有限制的。各种限制可以被设置来约束某个指定的容器可以获取的主机CPU时钟周期。
JAVA 10能够识别这些限制：

```shell
docker container run -it --cpus 2 openjdk:10-jdk
jshell> Runtime.getRuntime().availableProcessors()
$1 ==> 2
```

所有分配给Docker EE的CPUs均获得同样比例的CPU时钟周期。这个比例可以通过调整CPU共享比重（相对于其他所有运行的容器的比重）来进行修改。
这个比例只在运行CPU敏感的进程才生效。当一个容器中的任务是空闲状态，那么其他容器可以使用剩余的全部CPU时间。然后真实的CPU时间统计则相当依赖系统上运行的容器的个数。这些在JAVA 10中都可以设置：

```shell
docker container run -it --cpu-shares 2048 openjdk:10-jdk
jshell> Runtime.getRuntime().availableProcessors()
$1 ==> 2
```

JAVA 10中也可以设置CPU集合约束（允许哪些CPU执行）。

```shell
docker run -it --cpuset-cpus="1,2,3" openjdk:10-jdk
jshell> Runtime.getRuntime().availableProcessors()
$1 ==> 3
```

## 分配内存和CPU

使用Java 10，可以使用容器设置来估计部署应用程序所需的内存和CPU的分配。我们假设已经确定了容器中运行的每个进程的内存堆和CPU要求，并设置了JAVA_OPTS。例如，如果您有一个跨10个节点分布的应用程序，五个节点需要512Mb的内存，每个需要1024个CPU-shares，另外五个节点需要256Mb，每个512个CPU-shares。请注意，1个CPU份额比例由1024表示。


对于内存，应用程序需要至少分配5Gb。
512Mb x 5 = 2.56 Gb
256Mb x 5 = 1.28 Gb


该应用程序需要8个CPU才能高效运行。
1024 x 5 = 5 CPUs
512 x 5 = 3 CPUs


最佳实践建议分析应用程序以确定运行在JVM中的每个进程的内存和CPU分配。但是，在调整容器大小以防止Java应用程序中的内存不足错误以及分配足够的CPU来处理工作负载时，Java 10消除了这些猜测。





































