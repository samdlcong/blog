---
title: "关于并发和 I/O 模型"
date: 2023-02-27T13:28:04+08:00
author: samdlcong
tags: ["PHP"]
categories: ["源码阅读"]
draft: false
---

## 并发和并行

一提到并发，通常就会提起并行，那么什么是并发，什么又是并行？

并行意味着一个任务同时有多个可执行单元在运行。

从操作系统的角度来看，并行意味着一个任务被多个CPU 核心执行。 单独一个 CPU 核心不能执行并行计算，它可以是在多个任务中来回切换地进行运算，但是不能被称为并行。并行的必要条件是多CPU 核心 或多 CPU 的计算环境。

并发意味着在同一时期内执行多个任务，可以在一个 CPU 核心上，也可以在多个 CPU 核心上。但是在同一时期，这些任务不一定都处于运行状态，这取决于 CPU 核心或者 CPU的数量。



## 阻塞 IO 、 非阻塞 IO和多路复用 IO

阻塞 IO 最传统的一种 IO 模型，即在读写数据的过程中会发生阻塞现象。

非阻塞 IO 用户发起 IO 请求后， 并不需要等待，而是马上就得到一个结果。用户线程需要不断地询问内核数据是否就绪，会一直占用 CPU

多路复用 IO 会有一个线程不断去轮询多个 socket 的状态，只有当 socket 真正有读写事件时，才真正调用实际的 IO读写操作，因为只需要用一个线程就可以管理多个 socket, 系统不需要建立新的进程或者线程，也不必维护这些线程和进程，所以大大减少了资源占用。多路复用有多种方案，比如 select poll epoll 之后另开一篇再表。

另外IO 模型还有 信号驱动 IO， 异步IO， 也在之后的文章中再讲。



## 并发 IO 模型

### 1. 单进程阻塞IO

最原始的一种 IO 模型，任务只能一个接着一个按序执行，哪怕 CPU 空闲着。只有一个 PHP脚本在执行。

### 2. 多进程阻塞IO

这是目前 PHP-FPM 和大部分 CGI 脚本(Python、Ruby)所用的模型

![multiple_process_blockingio](http://img.samdlcong.com/multiple_process_blockingio.jpg)

这个模型的缺点是：如果并发量很大的情况下，服务器会来不及处理，会引发 502 



### 3. 多线程阻塞 IO

这是 Java Web Application所用的模型

![multiple_threads_blocking_io](http://img.samdlcong.com/multiple_threads_blocking_io.jpg)

线程比进程轻量，并且能在应用间共享内存和状态。

每个请求都单独的占用一个线程，线程由线程池管理来复用，减少重新创建线程的系统开销。

Java 语言支持非阻塞 IO 模型， 但是大多数 Web 服务器都没在使用。



### 4. 单线程非阻塞 IO

这是 Node.js 使用的模型

![single_thread_non_blocking_io](http://img.samdlcong.com/single_thread_non_blocking_io.jpg)

采用的是 Event Loop 的模式，每次发出请求后，会给出回调函数，一旦完成 IO 操作后，就会被执行。底层是将任务放入队列，处理好任务后，会将结果返回给回调函数。

这个模型的缺点是：每一个请求都在同一个周期内执行，一个请求可能会在同一周期内阻塞其他请求，处理的链接越多，每个请求的响应时间就越慢。如果请求很大，无法发挥多核 CPU 应有的性能。



### 5. Reactor 和单线程 IO 多路复用

这是 Redis 6.0 之前采用的模型， 6.0 之后采用的多线程IO，只是用来处理网络数据的读写和协议的解析，而执行命令依旧是单线程。

单线程的优点：单线程操作，避免了不必要的上下文切换和竞争条件，也不存在多进程或者多线程导致的切换而消耗CPU，不用去考虑各种锁的问题，不存在加锁释放锁操作，没有因为可能出现死锁而导致的性能消耗。

所以 Redis 的单线程处理速度仍然很快。



### 6. Reactor 和多线程

这是 Java Netty 采用的模型

![thread_pools_non_blocking_io](http://img.samdlcong.com/thread_pools_non_blocking_io.jpg)

Reactor 模式下主程序只负责监听文件描述符上是否有事件发生，并不处理文件描述符的读写。读写由其他的工作程序做。主程序用来抗并发，不阻塞，且非常的轻便，事件可以通过队列的方式等待被工作程序执行。





### 7. 多线程、协程 非阻塞IO

这是 Go 采用的模型

![multiple_threads_coroutines_non_blocking_io](http://img.samdlcong.com/multiple_threads_coroutines_non_blocking_io.jpg)

Go 原生支持并发，采用轻量级的线程，协程(大小默认 2KB)来处理，并且在程序端来管理这些协程，可以充分利用多核 CPU的性能。是一种非常好的并发解决方案。



以上就是目前流行的或者历史存在过的并发模型，当然还有其他的变体，但都是大同小异。通过这些模型方法最终需要解决的依然是 IO 速度和 CPU 速度之间的差。
