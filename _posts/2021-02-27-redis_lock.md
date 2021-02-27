---
date: 2021-02-27
layout: post
title: redis 分布式锁的实用方案
thread: 2021-02-27-redis_lock.md
categories: 计算机原理
tags: redis
---

## 前言

在编程中， 经常需要开多进程或者多线程， 甚至多协程， 来提高并发性能。
但是，当 多进程/多线程/多协程 需要访问同一个全局变量的时候，会导致 "线程不安全" 问题。
为了解决 "线程不安全" 问题，多进程/多线程/多协程 间访问到同一个变量的时候， 就需要加锁，实现互斥访问。

但是，上述场景只是单实例部署的情况下， 访问实例内部变量的情况，如果涉及到多个实例，就没法在代码中直接加锁了。
针对这种情况，最常见的解决方案就是利用 redis set 命令实现 "分布式锁" 的方案。

## 选用 Redis 实现分布式锁原因

- Redis单进程单线程运行的特性 （采用单线程，避免了不必要的上下文切换和竞争条件）
- Redis有很高的IO性能 （内部实现采用非阻塞 IO 和 epoll，基于 epoll 自己实现的简单的事件框架。epoll 中的读、写、关闭、连接都转化成了事件，然后利用 epoll 的多路复用特性，绝不在 IO 上浪费一点时间。）
- Redis命令对此支持较好，实现起来比较方便 （早期版本加锁使用 SETNX + EXPIRE， 新版本加锁可以直接使用 SET NX EX）
- Redis纯内存操作 （比利用 Mysql 唯一索引需要访问磁盘的 IO 快）


## 实用的单机版 Redis 分布式锁


### 前提条件

- Redis 单实例
- Redis 版本 >= 2.6.12


### 加锁

- EX seconds ： 将键的过期时间设置为 seconds 秒。 执行 SET key value EX seconds 的效果等同于执行 SETEX key seconds value 。
- NX ： 只在键不存在时， 才对键进行设置操作。 执行 SET key value NX 的效果等同于执行 SETNX key value 。


```
def redis_acquire_lock(key, acquire_timeout=3, lock_timeout=10):
    """ 加锁

    :param key: 锁名
    :param acquire_timeout: 获取锁的超时时间，默认 3 秒
    :param lock_timeout: 锁住 key 的超时时间，默认 10 秒
    :return:
    """
    token = str(uuid4())
    end = time.time() + acquire_timeout
    while time.time() < end:
        if redis_client.set(key, token, ex=lock_timeout, nx=True):
            return token
        time.sleep(0.001)
    return None
```



### 释放锁

- pipeline:

Redis 的 pipeline(管道)功能在命令行中没有，但 Redis 是支持 pipeline 的，而且在各个语言版的 client 中都有相应的实现。
由于网络开销延迟，即使 redis server 端有很强的处理能力，也由于收到的 client 消息少，而造成吞吐量小。
当 client 使用 pipelining 发送命令时，redis server 必须部分请求放到队列中（使用内存）执行完毕后一次性发送结果.
需要注意的是 Redis pipeline 并非是原子性的操作， 所以经常搭配事务一起使用。原生批命令(mset, mget) 支持原子性。

- WATCH:

WATCH 命令可以监控一个或多个键，一旦其中有一个键被修改（或删除），之后的事务就不会执行。
监控一直持续到 EXEC 命令（事务中的命令是在 EXEC 之后才执行的，所以在 MULTI 命令后可以修改 WATCH 监控的键值）


```
def redis_release_lock(key, token):
    """ 释放锁

    :param key: 锁名
    :param token: 当前锁持有者获得的锁标识
    :return:
    """
    # 使用 pipeline 技术， 将命令批量提交到 redis
    with redis_client.pipeline() as pipe:
        while True:
            try:
                # watch 锁, multi 后如果该 key 被其他客户端改变, 事务操作会抛出 WatchError 异常
                pipe.watch(key)
                data = pipe.get(key)
                # 对比锁标识，确保锁的当前持有者无误
                if data and data.decode() == token:
                    pipe.multi()
                    pipe.delete(key)
                    pipe.execute()
                    return True
                else:
                    # 取消所有的 watch
                    pipe.unwatch()
                    break
            except WatchError:
                # 再次进入 while 循环
                pass
        return False
```

基于 Redis 单实例，假设这个单实例总是可用，这种方法已经足够安全，可以应对大部分业务场景。





