---
date: 2018-12-20
layout: post
title: Tcp 连接处于异常状态的一点总结 (close wait & time wait)
thread: 2018-12-20-time_wait_close_wait.md
categories: 经验
tags: linux
---


* 出现大量 close wait
 * 我方被动关闭连接导致的, 我方代码有问题, 可能没有关闭连接, 导致服务方因为超时等原因强制关闭了连接
 * 解决的方法如果是单连接, 确保连接关闭; 如果是连接池, 确保连接回收到池子中

* 出现大量 time wait
 * 我方主动关闭连接导致的, 我方代码有问题, 可能频繁的创建连接, 所以频繁关闭连接, 而 time wait 需要 2 MSL 时间才会消失, 所以形成堆积.
 * 解决的方法是要么使用连接池，要么在同一个连接做尽可能多的事情，避免频繁开关.


* 模拟 TIME_WAIT

运行附件一 time_wait.go 的代码, 可以模拟 TIME_WAIT 形成的情况.
每次 http 请求都会新创建一个 client, 请求完毕之后主动关闭 http 连接,
由于这个过程极为频繁, TIME_WAIT 需要 2 MSL 的时间才会结束, 于是短时间形成了堆积.

在这个例子中, 我们可以使用 `http.DefaultClient` 代替 `http.Client{}`, 每次请求的时候使用全局默认的 client,
避免频繁的开关连接, 使用 DefaultClient 本质就是使用了连接池, 而不是每次都新建连接.


* 模拟 CLOSE_WAIT

我们最常见 close wait 的情形是事务.

正确的事务代码例子:
```
transaction = db.begin()
try:
        sql = 'update table set deleted = 1 where id = 1'
        transaction.exec(sql)
        transaction.commit()
catch:
        transaction.rollback()
finally:
        transaction.close()
```

错误的事务处理例子:
```
transaction = db.begin()
try:
        sql = 'update table set deleted = 1 where id = 1'
        transaction.exec(sql)
        transaction.commit()
catch:
        transaction.rollback()
```

在错误的例子中, 我们忘记了调用 transaction.close() 关闭事务连接或者回收事务占用的 tcp 连接



* 各个状态存在的生命周期
 * time wait 的生存时间是 2 MSL. RFC 定义的 MSL 是 2 分钟, 因此总共 4 分钟; linux 默认实现是 60s (定义在 Linux 内核源码 /usr/src/linux/include/net/tcp.h 中).
 * close wait 的生存时间是一直到 tcp 生命结束才会结束. 所以受到 tcp keep alive 时间限制, 以 centos 为力, tcp_keepalive_time = 1200 s, 这个配置可以更改. 
 (KeepAlive并不是TCP协议规范的一部分，但在几乎所有的TCP/IP协议栈（不管是Linux还是Windows）中，都实现了KeepAlive功能)




附件一: time_wait.go
``` 
package main

import (
        "net/http"
        "io/ioutil"
        "sync"
        "log"
)

var (
        goPool   *GoPool
)

func main()  {
        goPool = NewGPool(100)
        url := "http://www.ifeng.com"
        req, _ := http.NewRequest("GET", url, nil)
        for {
                goPool.Add()
                go HttpGet(req)
        }
}

func HttpGet(req *http.Request) ([]byte, error) {
        defer goPool.Done()
        client := http.Client{}
        var body []byte
        var resp *http.Response
        var err error
        resp, err = client.Do(req)
        if err != nil {
                return body, err
        }
        log.Println("http status: ", resp.Status)
        body, err = ioutil.ReadAll(resp.Body)
        log.Println("http body:", string(body))
        resp.Body.Close()
        return body, err
}


type GoPool struct {
        queue chan int
        wg    sync.WaitGroup
}

func NewGPool(size int) *GoPool {
        if size <= 0 {
                log.Panicln("Size must over than zero.")
        }
        return &GoPool{
                queue: make(chan int, size),
                wg:    sync.WaitGroup{},
        }
}

func (p *GoPool) Add() {
        p.queue <- 1
        p.wg.Add(1)
}

func (p *GoPool) Done() {
        p.wg.Done()
        <-p.queue
}

func (p *GoPool) Wait() {
        p.wg.Wait()
}
```


