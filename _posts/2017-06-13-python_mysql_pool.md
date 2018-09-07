---
date: 2017-06-13
layout: post
title: python 的 mysql 连接池（基于 mysql 官方的 mysql-connector-python）
thread: 2017-06-13-python_mysql_pool.md
categories: 数据库
tags: python
---


#### 背景


网上看了一圈，没有发现比较顺手的 python mysql 连接池，甚至很多都是错误的实现。于是自己写了一个简单实用的，经过了生产环境的考验，分享出来。


#### 使用方法

1 安装 mysql-connector-python 库

2 实例化一个 Pool 全局单例“数据库连接池句柄”

3 通过“数据库连接池句柄”调用对应的方法


#### 代码片段

<pre>
from mysqllib import Pool

# 配置信息
db_conf = {
    "user": "",
    "password": "",
    "database": "",
    "host": "",
    "port": 3306,
    "time_zone": "+8:00",
    "buffered": True,
    "autocommit": True,
    "charset": "utf8mb4",
}

# 实例化一个 Pool 全局单例“数据库连接池句柄”
db = Pool(pool_reset_session=False, **db_conf)

# 通过连接池操作
rows = db.query("select * from table")

# 事务操作
transaction = db.begin()
try:
    transaction.insert("insert into talble...")
except:
    transaction.rollback()
else:
    transaction.commit()
finally:
    transaction.close()

# 事务操作语法糖
with db.begin() as transaction:
    transaction.insert("insert into talble...")

</pre>



#### 附件 mysqllib.py: 连接池实现源码

<pre>
# -*- coding:utf-8 -*-
import logging
from mysql.connector.pooling import CNX_POOL_MAXSIZE
from mysql.connector.pooling import MySQLConnectionPool, PooledMySQLConnection
from mysql.connector import errors
import threading
CONNECTION_POOL_LOCK = threading.RLock()


class Pool(MySQLConnectionPool):

    def connect(self):
        try:
            return self.get_connection()
        except errors.PoolError:
            # Pool size should be lower or equal to CNX_POOL_MAXSIZE
            if self.pool_size < CNX_POOL_MAXSIZE:
                with threading.Lock():
                    new_pool_size = self.pool_size + 1
                    try:
                        self._set_pool_size(new_pool_size)
                        self._cnx_queue.maxsize = new_pool_size
                        self.add_connection()
                    except Exception as e:
                        logging.exception(e)
                    return self.connect()
            else:
                with CONNECTION_POOL_LOCK:
                    cnx = self._cnx_queue.get(block=True)
                    if not cnx.is_connected() or self._config_version != cnx._pool_config_version:
                        cnx.config(**self._cnx_config)
                        try:
                            cnx.reconnect()
                        except errors.InterfaceError:
                            # Failed to reconnect, give connection back to pool
                            self._queue_connection(cnx)
                            raise
                        cnx._pool_config_version = self._config_version
                    return PooledMySQLConnection(self, cnx)
        except Exception:
            raise

    def query(self, operation, params=None):
        cnx = cursor = None
        try:
            cnx = self.connect()
            cursor = cnx.cursor(buffered=True, dictionary=True)
            cursor.execute(operation, params)
            data_set = cursor.fetchall()
        except Exception:
            raise
        finally:
            if cursor:
                cursor.close()
            if cnx:
                cnx.close()
        return data_set

    def get(self, operation, params=None):
        cnx = cursor = None
        try:
            cnx = self.connect()
            cursor = cnx.cursor(buffered=True, dictionary=True)
            cursor.execute(operation, params)
            data_set = cursor.fetchone()
        except Exception:
            raise
        finally:
            if cursor:
                cursor.close()
            if cnx:
                cnx.close()
        return data_set

    def insert(self, operation, params=None):
        cnx = cursor = None
        try:
            cnx = self.connect()
            cursor = cnx.cursor()
            cursor.execute(operation, params)
            last_id = cursor.lastrowid
        except Exception:
            raise
        finally:
            if cursor:
                cursor.close()
            if cnx:
                cnx.close()
        return last_id

    def insert_many(self, operation, seq_params):
        cnx = cursor = None
        try:
            cnx = self.connect()
            cursor = cnx.cursor()
            cursor.executemany(operation, seq_params)
            row_count = cursor.rowcount
        except Exception:
            raise
        finally:
            if cursor:
                cursor.close()
            if cnx:
                cnx.close()
        return row_count

    def execute(self, operation, params=None):
        cnx = cursor = None
        try:
            cnx = self.connect()
            cursor = cnx.cursor()
            cursor.execute(operation, params)
            row_count = cursor.rowcount
        except Exception:
            raise
        finally:
            if cursor:
                cursor.close()
            if cnx:
                cnx.close()
        return row_count

    def update(self, operation, params=None):
        return self.execute(operation, params)

    def delete(self, operation, params=None):
        return self.execute(operation, params)

    def begin(self, consistent_snapshot=False, isolation_level=None, readonly=None):
        cnx = self.connect()
        cnx.start_transaction(consistent_snapshot, isolation_level, readonly)
        return Transaction(cnx)


class Transaction(object):

    def __init__(self, connection):
        self.cnx = None
        if isinstance(connection, PooledMySQLConnection):
            self.cnx = connection
            self.cursor = connection.cursor(buffered=True, dictionary=True)
        else:
            raise AttributeError("connection should be a PooledMySQLConnection")

    def __enter__(self):
        return self

    def __exit__(self, exc_type, exc_val, exc_tb):
        if exc_type is None and exc_val is None and exc_tb is None:
            self.commit()
        else:
            # will raise with-body's Exception, should deal with it
            self.rollback()
        self.close()

    def query(self, operation, params=None):
        cursor = self.cursor
        cursor.execute(operation, params)
        data_set = cursor.fetchall()
        return data_set

    def get(self, operation, params=None):
        cursor = self.cursor
        cursor.execute(operation, params)
        data_set = cursor.fetchone()
        return data_set

    def insert(self, operation, params=None):
        cursor = self.cursor
        cursor.execute(operation, params)
        last_id = cursor.lastrowid
        return last_id

    def insert_many(self, operation, seq_params):
        cursor = self.cursor
        cursor.executemany(operation, seq_params)
        row_count = cursor.rowcount
        return row_count

    def execute(self, operation, params=None):
        cursor = self.cursor
        cursor.execute(operation, params)
        row_count = cursor.rowcount
        return row_count

    def update(self, operation, params=None):
        return self.execute(operation, params)

    def delete(self, operation, params=None):
        return self.execute(operation, params)

    def commit(self):
        self.cnx.commit()

    def rollback(self):
        self.cnx.rollback()

    def close(self):
        self.cursor.close()
        self.cnx.close()

</pre>
