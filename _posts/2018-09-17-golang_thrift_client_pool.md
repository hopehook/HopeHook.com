---
date: 2018-09-17
layout: post
title: golang thrift client pool 解决 server 重启连接失效
thread: 2018-09-17-golang_thrift_client_pool.md
categories: 连接池
tags: thrift golang 连接池
---


## 目的

实现一个好用, 可靠的 thrift client 连接池


## 背景故事

最近, 在公司项目中需要用到 thrift rpc 调用, 以往的 thrift client 都是用 python 写的.
因此, 需要写一个新的 golang 版 thrift client, 根据 python 中的经验和以往的经验,
当然是采用连接池比较好, 这样可以减少 TCP 连接的频繁创建销毁带来的开销.

首先, 我翻看了 thrift 的官网和网上的资料, 得出几点小结论:

* 没有发现官方有支持 thrift client 的连接池
* 可以利用第三方连接池库包装 thrift client, 实现连接池

既然如此, 我选择了利用第三方库去实现连接池的方式, 很容易就搞定了.
做法和这篇文章差不多: [链接一](https://studygolang.com/articles/1473), 类似的还有这篇文章: [链接二](http://silenceper.com/blog/201611/tcp_connection_pool/).

在简短运行了一段时间之后, 我敏感的发现了其中的问题. 程序日志中有几个 EOF, write broken pipe 的报错,
我意识到, 这并不是偶然, 很有可能是连接池的问题, 迅速通过 demo 验证, 确定是 thrift server 重启导致的.

回想一下这个场景, 当你通过 rpc 去调用的时候, server 需要更新代码重启, 这个时候 client 的连接都是失效的,
应该及时从连接池中清理掉, 然而第三方连接池似乎都没有这个机制, 也没有提供任何口子给用户.
在 [链接一] [链接二] 中, 两位同仁解决的都是超时失效的问题, 并没有处理重启导致的连接失效.

为了解决这个问题, 我思索了几种方案.

* 方案一 如果官方支持 ping, 可以在每次从连接池获取连接的时候判断一下, 无法 ping 通的连接直接丢弃, 重新获取或者创建新连接
* 方案二 在 server 提供一个 ping 的 rpc 接口, 专门用于判断连通性
* 方案三 继承 thrift 的 client 类, 重写 Call 方法, 通过 send 数据包是否报错来判断连通性, 报错的连接直接丢弃

查找了一圈, 发现 thrift 没有类似 ping 的方法检测连接的连通性, 于是否决方案一;

方案二需要专门提供一个 ping 的接口, 比较 low, 代价较大, 也否定了;

最终, 我选择了方案三, 在 rpc Call 的时候, 做连接池的相关动作, 以及连通性的检测.

这样子改造之后, 代码很简单, 甚至比没有连接池更加方便.
只需要两步: 

* 初始化一次全局的连接池
* 调用的时候通过全局连接池操作

以往没有采用连接池的时候, 每次都要创建连接, 关闭连接, 现在就没必要了

附件文件 thrift.go 是基于 github.com/fatih/pool 这个第三方库, 重写了 Call 的相关代码.
最终实现了个人非常满意的 golang thrift client pool, 分享给大家.


## 后记

回顾以往接触的各种连接池, 都要考虑连接失效的问题. 通过什么方法判断, 如果失效是否重连, 是否重试.
如果想要更好的使用连接池, 通过举一反三就是最好的方式, 把遇到的连接池对比起来看看, 也许还有新的收获.


## 附件

附件: thrift.go
```
package util

import (
	"context"
	"git.apache.org/thrift.git/lib/go/thrift"
	"github.com/hopehook/pool"
	"io"
	"net"
	"time"
)

var (
	maxBadConnRetries int
)

// connReuseStrategy determines how returns connections.
type connReuseStrategy uint8

const (
	// alwaysNewConn forces a new connection.
	alwaysNewConn connReuseStrategy = iota
	// cachedOrNewConn returns a cached connection, if available, else waits
	// for one to become available or
	// creates a new connection.
	cachedOrNewConn
)

type ThriftPoolClient struct {
	*thrift.TStandardClient
	seqId                      int32
	timeout                    time.Duration
	iprotFactory, oprotFactory thrift.TProtocolFactory
	pool                       pool.Pool
}

func NewThriftPoolClient(host, port string, inputProtocol, outputProtocol thrift.TProtocolFactory, initialCap, maxCap int) (*ThriftPoolClient, error) {
	factoryFunc := func() (interface{}, error) {
		conn, err := net.Dial("tcp", net.JoinHostPort(host, port))
		if err != nil {
			return nil, err
		}
		return conn, err
	}

	closeFunc := func(v interface{}) error { return v.(net.Conn).Close() }

	//创建一个连接池： 初始化5，最大连接30
	poolConfig := &pool.PoolConfig{
		InitialCap: initialCap,
		MaxCap:     maxCap,
		Factory:    factoryFunc,
		Close:      closeFunc,
	}

	p, err := pool.NewChannelPool(poolConfig)
	if err != nil {
		return nil, err
	}
	return &ThriftPoolClient{
		iprotFactory: inputProtocol,
		oprotFactory: outputProtocol,
		pool:         p,
	}, nil
}

// Sets the socket timeout
func (p *ThriftPoolClient) SetTimeout(timeout time.Duration) error {
	p.timeout = timeout
	return nil
}

func (p *ThriftPoolClient) Call(ctx context.Context, method string, args, result thrift.TStruct) error {
	var err error
	var errT thrift.TTransportException
	var errTmp error
	var ok bool
	// set maxBadConnRetries equals p.pool.Len(), attempt to retry by all connections
	// if maxBadConnRetries <= 0, set to 2
	maxBadConnRetries = p.pool.Len()
	if maxBadConnRetries <= 0 {
		maxBadConnRetries = 2
	}

	// try maxBadConnRetries times by alwaysNewConn connReuseStrategy
	for i := 0; i < maxBadConnRetries; i++ {
		err = p.call(ctx, method, args, result, cachedOrNewConn)
		if errT, ok = err.(thrift.TTransportException); ok {
			errTmp = errT.Err()
			if errTmp != io.EOF && errTmp != io.ErrUnexpectedEOF && errTmp != io.ErrClosedPipe {
				break
			}
		}
	}

	// if try maxBadConnRetries times failed, create new connection by alwaysNewConn connReuseStrategy
	if errTmp == io.EOF || errTmp == io.ErrUnexpectedEOF || errTmp == io.ErrClosedPipe {
		return p.call(ctx, method, args, result, alwaysNewConn)
	}

	return err
}

func (p *ThriftPoolClient) call(ctx context.Context, method string, args, result thrift.TStruct, strategy connReuseStrategy) error {
	p.seqId++
	seqId := p.seqId

	// get conn from pool
	var connVar interface{}
	var err error
	if strategy == cachedOrNewConn {
		connVar, err = p.pool.Get()
	} else {
		connVar, err = p.pool.Connect()
	}
	if err != nil {
		return err
	}
	conn := connVar.(net.Conn)

	// wrap conn as thrift fd
	transportFactory := thrift.NewTFramedTransportFactory(thrift.NewTTransportFactory())
	trans := thrift.NewTSocketFromConnTimeout(conn, p.timeout)
	transport, err := transportFactory.GetTransport(trans)
	if err != nil {
		return err
	}
	inputProtocol := p.iprotFactory.GetProtocol(transport)
	outputProtocol := p.oprotFactory.GetProtocol(transport)

	if err := p.Send(outputProtocol, seqId, method, args); err != nil {
		return err
	}

	// method is oneway
	if result == nil {
		return nil
	}

	if err = p.Recv(inputProtocol, seqId, method, result); err != nil {
		return err
	}

	// put conn back to the pool, do not close the connection.
	return p.pool.Put(connVar)
}

```

附件 client.go
```
package util

import (
	"git.apache.org/thrift.git/lib/go/thrift"
	"services/articles"
	"services/comments"
	"services/users"
	"log"
	"time"
)

func GetArticleClient(host, port string, initialCap, maxCap int, timeout time.Duration) *articles.ArticleServiceClient {
	protocolFactory := thrift.NewTBinaryProtocolFactoryDefault()
	client, err := NewThriftPoolClient(host, port, protocolFactory, protocolFactory, initialCap, maxCap)
	if err != nil {
		log.Panicln("GetArticleClient error: ", err)
	}
	client.SetTimeout(timeout)
	return articles.NewArticleServiceClient(client)
}

func GetCommentClient(host, port string, initialCap, maxCap int, timeout time.Duration) *comments.CommentServiceClient {
	protocolFactory := thrift.NewTCompactProtocolFactory()
	client, err := NewThriftPoolClient(host, port, protocolFactory, protocolFactory, initialCap, maxCap)
	if err != nil {
		log.Panicln("GetCommentClient error: ", err)
	}
	client.SetTimeout(timeout)
	return comments.NewCommentServiceClient(client)
}

func GetUserClient(host, port string, initialCap, maxCap int, timeout time.Duration) *users.UserServiceClient {
	protocolFactory := thrift.NewTCompactProtocolFactory()
	client, err := NewThriftPoolClient(host, port, protocolFactory, protocolFactory, initialCap, maxCap)
	if err != nil {
		log.Panicln("GetUserClient error: ", err)
	}
	client.SetTimeout(timeout)
	return users.NewUserServiceClient(client)
}
```


