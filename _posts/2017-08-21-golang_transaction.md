---
date: 2017-08-21
layout: post
title: golang transaction 事务使用的正确姿势
thread: 2017-08-21-golang_transaction.md
categories: 数据库
tags: golang
---

#### 第一种写法
这种写法非常朴实，程序流程也非常明确，但是事务处理与程序流程嵌入太深，容易遗漏，造成严重的问题

```
func DoSomething() (err error) {
    tx, err := db.Begin()
    if err != nil {
        return
    }


    defer func() {
        if p := recover(); p != nil {
            tx.Rollback()
            panic(p)  // re-throw panic after Rollback
        }
    }()


    if _, err = tx.Exec(...); err != nil {
        tx.Rollback()
        return
    }
    if _, err = tx.Exec(...); err != nil {
        tx.Rollback()
        return
    }
    // ...


    err = tx.Commit()
    return
}
```


#### 第二种写法
下面这种写法把事务处理从程序流程抽离了出来，不容易遗漏，但是作用域是整个函数，程序流程不是很清晰

```
func DoSomething() (err error) {
    tx, err := db.Begin()
    if err != nil {
        return
    }


    defer func() {
        if p := recover(); p != nil {
            tx.Rollback()
            panic(p) // re-throw panic after Rollback
        } else if err != nil {
            tx.Rollback()
        } else {
            err = tx.Commit()
        }
    }()


    if _, err = tx.Exec(...); err != nil {
        return
    }
    if _, err = tx.Exec(...); err != nil {
        return
    }
    // ...
    return
}
```



#### 第三种写法
写法三是对写法二的进一步封装，写法高级一点，缺点同上

```
func Transact(db *sql.DB, txFunc func(*sql.Tx) error) (err error) {
    tx, err := db.Begin()
    if err != nil {
        return
    }


    defer func() {
        if p := recover(); p != nil {
            tx.Rollback()
            panic(p) // re-throw panic after Rollback
        } else if err != nil {
            tx.Rollback()
        } else {
            err = tx.Commit()
        }
    }()


    err = txFunc(tx)
    return err
}


func DoSomething() error {
    return Transact(db, func (tx *sql.Tx) error {
        if _, err := tx.Exec(...); err != nil {
            return err
        }
        if _, err := tx.Exec(...); err != nil {
            return err
        }
    })
}
```


#### 我的写法
经过总结和实验，我采用了下面这种写法，defer tx.Rollback() 使得事务回滚始终得到执行。
当 tx.Commit() 执行后，tx.Rollback() 起到关闭事务的作用，
当程序因为某个错误中止，tx.Rollback() 起到回滚事务，同事关闭事务的作用。

* 普通场景
```
func DoSomething() (err error) {
    tx, _ := db.Begin()
    defer tx.Rollback()

    if _, err = tx.Exec(...); err != nil {
        return
    }
    if _, err = tx.Exec(...); err != nil {
        return
    }
    // ...


    err = tx.Commit()
    return
}
```


* 循环场景

(1) 小事务 每次循环提交一次
在循环内部使用这种写法的时候，defer 不能使用，所以要把事务部分抽离到独立的函数当中

```
func DoSomething() (err error) {
    tx, _ := db.Begin()
    defer tx.Rollback()

    if _, err = tx.Exec(...); err != nil {
        return
    }
    if _, err = tx.Exec(...); err != nil {
        return
    }
    // ...


    err = tx.Commit()
    return
}


for {
    if err := DoSomething(); err != nil{
         // ...
    }
}
```

(2) 大事务 批量提交
大事务的场景和普通场景是一样的，没有任何区别

```
func DoSomething() (err error) {
    tx, _ := db.Begin()
    defer tx.Rollback()

    for{
        if _, err = tx.Exec(...); err != nil {
            return
        }
        if _, err = tx.Exec(...); err != nil {
            return
        }
        // ...
    }

    err = tx.Commit()
    return
}
```


参考链接:

[https://stackoverflow.com/questions/16184238/database-sql-tx-detecting-commit-or-rollback](https://stackoverflow.com/questions/16184238/database-sql-tx-detecting-commit-or-rollback)








