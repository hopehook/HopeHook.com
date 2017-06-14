---
date: 2017-06-13
layout: post
title: python 的 mysql 连接池（基于 mysql 官方的 mysql-connector-python）
thread: 2017-06-13-python_mysql_pool.md
categories: 数据库
tags: Python
---

## 背景
<p></p>
网上看了一圈，没有发现顺手的 mysql python 连接池，甚至很多都是错误的实现。于是自己写了一个，经过了生产环境的考验，特地分享出来。
<p></p>
近期，多说关闭了（伤心），所以欢迎盆友们到本人的 [github](https://github.com/hopehook/hopehook.com) 提 issue 进行交流，也许你有更好的实现方式，而我却不知道，大家一起进步吧 :)
<br/>
## 使用方法
1 安装 mysql-connector-python 库
2 通过“数据库连接池句柄”调用对应的方法
<br/>
## 源码片段
1 setting.py

<pre>
# mysql 基本配置
MYSQL_CONFIG = {
    "slave_db": {
        "user": "",
        "password": "",
        "database": "",
        "host": "",
        "port": 3306,
        "time_zone": "+8:00",
        "buffered": True,
        "autocommit": True,
    },
    "master_db": {
        "user": "",
        "password": "",
        "database": "",
        "host": "",
        "port": 3306,
        "time_zone": "+8:00",
        "buffered": True,
        "autocommit": True,
    }

}

# mysql 连接池配置
MYSQL_POOL_CONFIG = {
    "slave_db": {
        "pool_name": "slave_db",
        "pool_size": 5,
        "pool_reset_session": True,
    },
    "master_db": {
        "pool_name": "master_db",
        "pool_size": 5,
        "pool_reset_session": True,
    }
}
</pre>


<br/>
2 mysqllib.py

<pre>
# -*- coding:utf-8 -*-
import sys
import traceback
import mysql.connector
from mysql.connector.pooling import CNX_POOL_MAXSIZE
from mysql.connector.pooling import MySQLConnectionPool, PooledMySQLConnection
from mysql.connector import errors
from mysql.connector import errorcode
from mysql.connector.cursor import MySQLCursorBufferedDict
from config.setting import MYSQL_CONFIG, MYSQL_POOL_CONFIG
from lib.log import logger
import threading
CONNECTION_POOL_LOCK = threading.RLock()


class MySQLDriverPool(MySQLConnectionPool):

    def connect(self):
        try:
            logger.info("pool name: %s, pool size: %s" % (self.pool_name, self.pool_size))
            connection = self.get_connection()
            return connection
        except errors.PoolError:
            # Pool size should be lower or equal to CNX_POOL_MAXSIZE
            if self.pool_size < CNX_POOL_MAXSIZE:
                # 连接数在限定范围内，可以新增连接
                logger.warn('need add connection to this pool: %s' % self.pool_name)
                new_pool_size = self.pool_size + 1
                self._set_pool_size(new_pool_size)
                self._cnx_queue.maxsize = new_pool_size
                self.add_connection()
                logger.info('add connection to this pool successfully: %s' % self.pool_name)
                logger.info('pool name: %s, free connections (%s): %s' % (self.pool_name, self._cnx_queue.qsize(), self._cnx_queue.queue))
                return self.connect()
            else:
                # 连接数达到极限，进入 “阻塞模式”
                logger.warn('pool "%s" has no free connections, executor turn to block status' % self.pool_name)
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

        raise errors.PoolError("failed getting connection")

    def query(self, statement, *args, **kwargs):

        cnx = cursor = None
        try:
            cnx = self.connect()
            cursor = cnx.cursor(buffered=True, cursor_class=MySQLCursorBufferedDict)
            cursor.execute(statement, *args, **kwargs)
            data_set = cursor.fetchall()
        except Exception, e:
            logger.error(e)
            raise Exception('query error')
        finally:
            if cursor:
                cursor.close()
            if cnx:
                cnx.close()
        return data_set

    def get(self, statement, *args, **kwargs):

        cnx = cursor = None
        try:
            cnx = self.connect()
            cursor = cnx.cursor(buffered=True, cursor_class=MySQLCursorBufferedDict)
            cursor.execute(statement, *args, **kwargs)
            data_set = cursor.fetchone()
        except Exception, e:
            logger.error(e)
            raise Exception('get error')
        finally:
            if cursor:
                cursor.close()
            if cnx:
                cnx.close()
        return data_set

    def insert(self, statement, *args, **kwargs):

        cnx = cursor = None
        try:
            cnx = self.connect()
            cursor = cnx.cursor()
            cursor.execute(statement, *args, **kwargs)
            last_id = cursor.lastrowid
        except Exception, e:
            logger.error(e)
            raise Exception('insert error')
        finally:
            if cursor:
                cursor.close()
            if cnx:
                cnx.close()
        return last_id

    def insert_many(self, statement, *args, **kwargs):

        cnx = cursor = None
        try:
            cnx = self.connect()
            cursor = cnx.cursor()
            cursor.executemany(statement, *args, **kwargs)
            row_count = cursor.rowcount
        except Exception, e:
            logger.error(e)
            raise Exception('insert_many error')
        finally:
            if cursor:
                cursor.close()
            if cnx:
                cnx.close()
        return row_count

    def execute(self, statement, *args, **kwargs):

        cnx = cursor = None
        try:
            cnx = self.connect()
            cursor = cnx.cursor()
            cursor.execute(statement, *args, **kwargs)
            row_count = cursor.rowcount
        except Exception, e:
            logger.error(e)
            raise Exception('execute error')
        finally:
            if cursor:
                cursor.close()
            if cnx:
                cnx.close()
        return row_count

    def update(self, statement, *args, **kwargs):
        return self.execute(statement, *args, **kwargs)

    def delete(self, statement, *args, **kwargs):
        return self.execute(statement, *args, **kwargs)

    def begin(self, consistent_snapshot=False, isolation_level=None, readonly=None):

        cnx = self.connect()
        cnx.start_transaction(consistent_snapshot, isolation_level, readonly)
        return MySQLDriverConnection(cnx)


class MySQLDriverConnection(object):

    def __init__(self, connection):
        self.cnx = None
        if isinstance(connection, PooledMySQLConnection):
            self.cnx = connection
            self.cursor = connection.cursor(buffered=True, cursor_class=MySQLCursorBufferedDict)
        else:
            raise AttributeError("connection should be a PooledMySQLConnection")

    def query(self, statement, *args, **kwargs):

        cursor = self.cursor
        cursor.execute(statement, *args, **kwargs)
        data_set = cursor.fetchall()
        return data_set

    def get(self, statement, *args, **kwargs):

        cursor = self.cursor
        cursor.execute(statement, *args, **kwargs)
        data_set = cursor.fetchone()
        return data_set

    def insert(self, statement, *args, **kwargs):

        cursor = self.cursor
        cursor.execute(statement, *args, **kwargs)
        last_id = cursor.lastrowid
        return last_id

    def insert_many(self, statement, *args, **kwargs):

        cursor = self.cursor
        cursor.executemany(statement, *args, **kwargs)
        row_count = cursor.rowcount
        return row_count

    def execute(self, statement, *args, **kwargs):

        cursor = self.cursor
        cursor.execute(statement, *args, **kwargs)
        row_count = cursor.rowcount
        return row_count

    def update(self, statement, *args, **kwargs):
        return self.execute(statement, *args, **kwargs)

    def delete(self, statement, *args, **kwargs):
        return self.execute(statement, *args, **kwargs)

    def commit(self):

        self.cnx.commit()
        self.cursor.close()
        self.cnx.close()

    def rollback(self):

        self.cnx.rollback()
        self.cursor.close()
        self.cnx.close()


def init_mysql_connect_pool(config=None, pool=None):
    try:
        if pool is None:
            db_pool = MySQLDriverPool(5, 'defualt_pool', True, **config)
        else:
            db_pool = MySQLDriverPool(pool['pool_size'], pool['pool_name'], pool['pool_reset_session'], **config)
    except errors.OperationalError:
        logger.error(traceback.format_exc())
        sys.exit(-10001)
    except errors.InterfaceError:
        logger.error(traceback.format_exc())
        sys.exit(-10002)
    except mysql.connector.Error as err:
        if err.errno == errorcode.ER_ACCESS_DENIED_ERROR:
            logger.error('Connect to database server error: Access deny')
        elif err.errno == errorcode.ER_BAD_DB_ERROR:
            logger.error('Connect to database server error: Database not exists')
        else:
            logger.error(err)
        sys.exit(-10003)
    except Exception as err:
        logger.error(err)
        sys.exit(-10004)

    logger.info('''Connect to database server success, pool_name:{0}, pool_size:{1}
                '''.format(db_pool.pool_name, db_pool.pool_size))
    return db_pool

# mysql 连接池句柄
dbm = init_mysql_connect_pool(config=MYSQL_CONFIG['master_db'], pool=MYSQL_POOL_CONFIG['master_db'])
dbs = init_mysql_connect_pool(config=MYSQL_CONFIG['slave_db'], pool=MYSQL_POOL_CONFIG['slave_db'])

</pre>
