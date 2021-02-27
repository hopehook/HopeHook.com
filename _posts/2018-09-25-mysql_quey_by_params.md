---
date: 2018-09-25
layout: post
title: Mysql 参数化查询 in 和 like
thread: 2018-09-25-mysql_quey_by_params.md
categories: 数据库
tags: mysql
---


背景:

为了防范 SQL 注入攻击, 在查询 mysql 的时候, 我们会选择参数化查询. 但是, 有些情况比较特别, 传入的参数需要特别处理才可以传入, 常见的就是 in 和 like 的场景.

1 模糊查询 like

```
login = "%" + login + "%"
db.query(`select * from test where login like ? or id like ?`, login, login)
```

有的 mysql 驱动库需要对 % 符号进行转义, 比如替换成 %%

2 in 查询

```
ids = [1, 2, 3]
db.query(`select * from test where id in (?, ?)`, ids[0], ids[1], ids[2])
```

日常开发中, ids 的数量往往是不确定的, 因此要填多少个参数也不确定.
如果是纯数字的 in 查询, 可以考虑转为字符串拼接的方式, 也不会有 sql 注入的问题 (不会改变 sql 的语义).

也可以考虑下面这种做法:
先循环拼接出 sql 语句部分的 ? 占位符号: 
`select * from test where id in (?, ?)`

然后把参数的 list 展开传入: 
```
db.query(`select * from test where id in (?, ?)`, ids…)
```

如果还有其他条件的参数需要占位, 可以 append 到 ids 中, 依次展开: 
```
ids = append(ids, login)
db.query(`select * from test where id in (?, ?) and login = ?`, ids…)
```

