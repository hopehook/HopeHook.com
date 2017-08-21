---
date: 2017-08-21
layout: post
title: golang transaction 事务使用的正确姿势
thread: 2017-08-21-golang_transaction.md
categories: 数据库
tags: golang
---


### 写法1

点评: 最常规，稳妥的写法，但是事务的处理痕迹太多

<pre>
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
</pre>



### 写法2

点评: 精简了很多，但事务的作用域是到函数结束，不方便限制作用范围

<pre>
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
</pre>


### 写法3

点评: 很高端的写法，可读性稍微差一点

<pre>
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
</pre>


### 我的写法

点评: 咋看起来没什么特点，但是简洁，安全，作用范围可控

<pre>
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
</pre>


### 循环场景的写法

1 小事务  // 每次循环提交一次

<pre>
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
</pre>


2 大事务 // 批量提交

<pre>
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
</pre>


参考: 
https://stackoverflow.com/questions/16184238/database-sql-tx-detecting-commit-or-rollback








