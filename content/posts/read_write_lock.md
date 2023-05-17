---
title: "[TIPS] 有关 Read-Write Lock"
date: 2023-05-17T08:42:29+08:00
author: samdlcong
tags: ["Go","TIPS"]
categories: ["技术笔记"]
draft: false
---

## 关于读写锁死锁的问题 

最近在写框架绑定服务提供者的时候，遇到了编译出错，出现了死锁。主要代码如下:

``` Go
func (hade *HadeContainer) Bind(provider ServiceProvider) error {
	hade.lock.Lock()
	defer hade.lock.Unlock()
	key := provider.Name()

	hade.providers[key] = provider

	// if provider is not defer
	if provider.IsDefer() == false {
		if err := provider.Boot(hade); err != nil {
			return err
		}
		// 实例化方法
		// 省略...
	}
	return nil
}
```
这段代码调用了 provider.Boot() 方法
``` Go
func (provider *HadeEnvProvider) Boot(c framework.Container) error {
	app := c.MustMake(contract.AppKey).(contract.App)
	// 省略...
}
func (hade *HadeContainer) MustMake(key string) interface{} {
	serv, err := hade.make(key, nil, false)
	// 省略...
}
func (hade *HadeContainer) make(key string, params []interface{}, forceNew bool) (interface{}, error) {
	hade.lock.RLock()
	defer hade.lock.RUnlock()
	// 查询是否已经注册了这个服务提供者，如果没有注册，则返回错误
	sp := hade.findServiceProvider(key)
	// 省略...
}
```
在 make 方法里调用了读锁，由于前面的写锁是 defer 的，这时候并没有释放，这时调用读锁，造成了死锁。


## 读写锁的数据结构
源码中 src/sync/rwmutex.go:RWMutex 定义了读写锁的数据结构
```Go
type RWMutex struct {
    w Mutex  // 用于控制多个写锁，获得写锁首先要获取该锁
    writerSem uint32 // 写阻塞等待的信号量，最后一个读者释放锁时会释放信号量
    readerSem uint32 // 读阻塞的协程等待的信号量，持有写锁释放锁后会释放信号量
    readerCount32 int32 // 记录读者个数
    readerWait int32 // 记录写阻塞时的读者个数
}
```
读写锁内部仍有一个互斥锁，用于将多个写操作隔离开来，其他几个字段用于隔离读操作和写操作。

RWMutex 提供了 4 个简单的接口
* RLock()
* RUnlock()
* Lock()
* Unlock()

Lock() 的实现逻辑
1. 获取互斥锁
2. 阻塞等待所有读操作结束(如果有的话)

Unlock() 的实现逻辑
1. 唤醒因读锁定而被阻塞的协程(如果有的话)
2. 解除互斥锁

RLock() 的实现逻辑
1. 增加读操作计数，即 readerCount++
2. 阻塞等待写操作结束(如果有的话)

RUnlock() 的实现逻辑
1. 减少读操作计数，即 readerCount--
2. 唤醒等待写操作的协程(如果有的话)

在读写锁中，写操作会阻塞写操作和读操作，读操作会阻塞写操作，但是读操作不会阻塞读操作。如果中间没有及时释放锁，就会造成死锁。

## 读写锁是如何相互阻塞的
1. 写操作是依赖互斥锁来阻止其他写操作的
2. 写操作将 readerCount 变成负值来阻止读操作 这是读写锁的精华
3. 读操作增加 readerCount 的值来阻止写操作
4. 写操作不会被“饿死”。写操作到来时，会把 readCount的值复制到 readerWait 中，用于标记在写操作前面的读者个数。读操作结束后，会递减 readerCount 的值，还会递减 readerWait 的值，readerWait 变为 0 时，唤醒写操作。

所以基于读写锁的特性，读写锁特别适合应用在具有一定并发量且读多写少的场合。
在大量并发读的情况下，多个 Goroutine 可以同时持有读锁，从而减少在锁竞争中等待的时间。

## 对上面代码的修改
知道了死锁的原因后，我们只要手动释放写锁即可，也侧面证明了 defer 很好，但是也不能乱用。代码如下：
```Go 
func (hade *HadeContainer) Bind(provider ServiceProvider) error {
	hade.lock.Lock()
	key := provider.Name()
	hade.providers[key] = provider
	hade.lock.Unlock()

	// if provider is not defer
	if provider.IsDefer() == false {
		if err := provider.Boot(hade); err != nil {
			return err
		}
		// 实例化方法
		// 省略...
	}
	return nil
}
```