---
date: 2017-06-14
layout: post
title: redis mysql 连接池 之 golang 实现
thread: 2017-06-14-golang_db_pool.md
categories: 数据库
tags: golang
---

分享一下 golang 实现的 redis 和 mysql 连接池，可以在项目中直接引用连接池句柄，调用对应的方法。 
<br/>
<br/>
#### 举个栗子：
<p></p>
1 mysql 连接池的使用
<p></p>
（1） 在项目子目录放置 mysql.go
<p></p>
（2）在需要调用的地方导入连接池句柄 DB
<p></p>
（3）调用 DB.Query()
<p></p>
 
2 redis 连接池的使用
<p></p>
（1）在项目子目录放置 redis.go
<p></p>
（2）在需要调用的地方导入连接池句柄 Cache
<p></p>
（3）调用 Cache.SetString (“test_key”, “test_value”)
<br/>
<br/>
#### 最新代码地址：
<p></p>
[https://github.com/hopehook/golang-db](https://github.com/hopehook/golang-db)
<br/>
<br/>
#### 附件:
<p></p>
1 mysql 连接池代码
<p></p>
<pre>
package lib

import (
	"database/sql"
	"fmt"
	"strconv"

	"github.com/arnehormann/sqlinternals/mysqlinternals"
	_ "github.com/go-sql-driver/mysql"
)

var MYSQL map[string]string = map[string]string{
	"host":         "127.0.0.1:3306",
	"database":     "",
	"user":         "",
	"password":     "",
	"maxOpenConns": "0",
	"maxIdleConns": "0",
}

type SqlConnPool struct {
	DriverName     string
	DataSourceName string
	MaxOpenConns   int64
	MaxIdleConns   int64
	SqlDB          *sql.DB // 连接池
}

var DB *SqlConnPool

func init() {
	dataSourceName := fmt.Sprintf("%s:%s@tcp(%s)/%s", MYSQL["user"], MYSQL["password"], MYSQL["host"], MYSQL["database"])
	maxOpenConns, _ := strconv.ParseInt(MYSQL["maxOpenConns"], 10, 64)
	maxIdleConns, _ := strconv.ParseInt(MYSQL["maxIdleConns"], 10, 64)

	DB = &SqlConnPool{
		DriverName:     "mysql",
		DataSourceName: dataSourceName,
		MaxOpenConns:   maxOpenConns,
		MaxIdleConns:   maxIdleConns,
	}
	if err := DB.open(); err != nil {
		panic("init db failed")
	}
}

// 封装的连接池的方法
func (p *SqlConnPool) open() error {
	var err error
	p.SqlDB, err = sql.Open(p.DriverName, p.DataSourceName)
	p.SqlDB.SetMaxOpenConns(int(p.MaxOpenConns))
	p.SqlDB.SetMaxIdleConns(int(p.MaxIdleConns))
	return err
}

func (p *SqlConnPool) Close() error {
	return p.SqlDB.Close()
}

func (p *SqlConnPool) Query(queryStr string, args ...interface{}) ([]map[string]interface{}, error) {
	rows, err := p.SqlDB.Query(queryStr, args...)
	if err != nil {
		return []map[string]interface{}{}, err
	}
	defer rows.Close()
	// 返回属性字典
	columns, err := mysqlinternals.Columns(rows)
	// 获取字段类型
	scanArgs := make([]interface{}, len(columns))
	values := make([]sql.RawBytes, len(columns))
	for i, _ := range values {
		scanArgs[i] = &values[i]
	}
	rowsMap := make([]map[string]interface{}, 0, 10)
	for rows.Next() {
		rows.Scan(scanArgs...)
		rowMap := make(map[string]interface{})
		for i, value := range values {
			rowMap[columns[i].Name()] = bytes2RealType(value, columns[i].MysqlType())
		}
		rowsMap = append(rowsMap, rowMap)
	}
	if err = rows.Err(); err != nil {
		return []map[string]interface{}{}, err
	}
	return rowsMap, nil
}

func (p *SqlConnPool) execute(sqlStr string, args ...interface{}) (sql.Result, error) {
	return p.SqlDB.Exec(sqlStr, args...)
}

func (p *SqlConnPool) Update(updateStr string, args ...interface{}) (int64, error) {
	result, err := p.execute(updateStr, args...)
	if err != nil {
		return 0, err
	}
	affect, err := result.RowsAffected()
	return affect, err
}

func (p *SqlConnPool) Insert(insertStr string, args ...interface{}) (int64, error) {
	result, err := p.execute(insertStr, args...)
	if err != nil {
		return 0, err
	}
	lastid, err := result.LastInsertId()
	return lastid, err

}

func (p *SqlConnPool) Delete(deleteStr string, args ...interface{}) (int64, error) {
	result, err := p.execute(deleteStr, args...)
	if err != nil {
		return 0, err
	}
	affect, err := result.RowsAffected()
	return affect, err
}

type SqlConnTransaction struct {
	SqlTx *sql.Tx // 单个事务连接
}

//// 开启一个事务
func (p *SqlConnPool) Begin() (*SqlConnTransaction, error) {
	var oneSqlConnTransaction = &SqlConnTransaction{}
	var err error
	if pingErr := p.SqlDB.Ping(); pingErr == nil {
		oneSqlConnTransaction.SqlTx, err = p.SqlDB.Begin()
	}
	return oneSqlConnTransaction, err
}

// 封装的单个事务连接的方法
func (t *SqlConnTransaction) Rollback() error {
	return t.SqlTx.Rollback()
}

func (t *SqlConnTransaction) Commit() error {
	return t.SqlTx.Commit()
}

func (t *SqlConnTransaction) Query(queryStr string, args ...interface{}) ([]map[string]interface{}, error) {
	rows, err := t.SqlTx.Query(queryStr, args...)
	if err != nil {
		return []map[string]interface{}{}, err
	}
	defer rows.Close()
	// 返回属性字典
	columns, err := mysqlinternals.Columns(rows)
	// 获取字段类型
	scanArgs := make([]interface{}, len(columns))
	values := make([]sql.RawBytes, len(columns))
	for i, _ := range values {
		scanArgs[i] = &values[i]
	}
	rowsMap := make([]map[string]interface{}, 0, 10)
	for rows.Next() {
		rows.Scan(scanArgs...)
		rowMap := make(map[string]interface{})
		for i, value := range values {
			rowMap[columns[i].Name()] = bytes2RealType(value, columns[i].MysqlType())
		}
		rowsMap = append(rowsMap, rowMap)
	}
	if err = rows.Err(); err != nil {
		return []map[string]interface{}{}, err
	}
	return rowsMap, nil
}

func (t *SqlConnTransaction) execute(sqlStr string, args ...interface{}) (sql.Result, error) {
	return t.SqlTx.Exec(sqlStr, args...)
}

func (t *SqlConnTransaction) Update(updateStr string, args ...interface{}) (int64, error) {
	result, err := t.execute(updateStr, args...)
	if err != nil {
		return 0, err
	}
	affect, err := result.RowsAffected()
	return affect, err
}

func (t *SqlConnTransaction) Insert(insertStr string, args ...interface{}) (int64, error) {
	result, err := t.execute(insertStr, args...)
	if err != nil {
		return 0, err
	}
	lastid, err := result.LastInsertId()
	return lastid, err

}

func (t *SqlConnTransaction) Delete(deleteStr string, args ...interface{}) (int64, error) {
	result, err := t.execute(deleteStr, args...)
	if err != nil {
		return 0, err
	}
	affect, err := result.RowsAffected()
	return affect, err
}

// others
func bytes2RealType(src []byte, columnType string) interface{} {
	srcStr := string(src)
	var result interface{}
	switch columnType {
	case "TINYINT":
		fallthrough
	case "SMALLINT":
		fallthrough
	case "INT":
		fallthrough
	case "BIGINT":
		result, _ = strconv.ParseInt(srcStr, 10, 64)
	case "CHAR":
		fallthrough
	case "VARCHAR":
		fallthrough
	case "BLOB":
		fallthrough
	case "TIMESTAMP":
		fallthrough
	case "DATETIME":
		result = srcStr
	case "FLOAT":
		fallthrough
	case "DOUBLE":
		fallthrough
	case "DECIMAL":
		result, _ = strconv.ParseFloat(srcStr, 64)
	default:
		result = nil
	}
	return result
}
</pre>
<br/>
2 redis 连接池代码
<p></p>
<pre>
package lib

import (
	"strconv"
	"time"

	"github.com/garyburd/redigo/redis"
)

var REDIS map[string]string = map[string]string{
	"host":         "127.0.0.1:6379",
	"database":     "0",
	"password":     "",
	"maxOpenConns": "0",
	"maxIdleConns": "0",
}

var Cache *RedisConnPool

type RedisConnPool struct {
	redisPool *redis.Pool
}

func init() {
	Cache = &RedisConnPool{}
	maxOpenConns, _ := strconv.ParseInt(REDIS["maxOpenConns"], 10, 64)
	maxIdleConns, _ := strconv.ParseInt(REDIS["maxIdleConns"], 10, 64)
	database, _ := strconv.ParseInt(REDIS["database"], 10, 64)

	Cache.redisPool = newPool(REDIS["host"], REDIS["password"], int(database), int(maxOpenConns), int(maxIdleConns))
	if Cache.redisPool == nil {
		panic("init redis failed！")
	}
}

func newPool(server, password string, database, maxOpenConns, maxIdleConns int) *redis.Pool {
	return &redis.Pool{
		MaxActive:   maxOpenConns, // max number of connections
		MaxIdle:     maxIdleConns,
		IdleTimeout: 10 * time.Second,
		Dial: func() (redis.Conn, error) {
			c, err := redis.Dial("tcp", server)
			if err != nil {
				return nil, err
			}
			if _, err := c.Do("AUTH", password); err != nil {
				c.Close()
				return nil, err
			}
			if _, err := c.Do("select", database); err != nil {
				c.Close()
				return nil, err
			}
			return c, err
		},
		TestOnBorrow: func(c redis.Conn, t time.Time) error {
			_, err := c.Do("PING")
			return err
		},
	}
}

// 关闭连接池
func (p *RedisConnPool) Close() error {
	err := p.redisPool.Close()
	return err
}

// 当前某一个数据库，执行命令
func (p *RedisConnPool) Do(command string, args ...interface{}) (interface{}, error) {
	conn := p.redisPool.Get()
	defer conn.Close()
	return conn.Do(command, args...)
}

//// String（字符串）
func (p *RedisConnPool) SetString(key string, value interface{}) (interface{}, error) {
	conn := p.redisPool.Get()
	defer conn.Close()
	return conn.Do("SET", key, value)
}

func (p *RedisConnPool) GetString(key string) (string, error) {
	// 从连接池里面获得一个连接
	conn := p.redisPool.Get()
	// 连接完关闭，其实没有关闭，是放回池里，也就是队列里面，等待下一个重用
	defer conn.Close()
	return redis.String(conn.Do("GET", key))
}

func (p *RedisConnPool) GetBytes(key string) ([]byte, error) {
	conn := p.redisPool.Get()
	defer conn.Close()
	return redis.Bytes(conn.Do("GET", key))
}

func (p *RedisConnPool) GetInt(key string) (int, error) {
	conn := p.redisPool.Get()
	defer conn.Close()
	return redis.Int(conn.Do("GET", key))
}

func (p *RedisConnPool) GetInt64(key string) (int64, error) {
	conn := p.redisPool.Get()
	defer conn.Close()
	return redis.Int64(conn.Do("GET", key))
}

//// Key（键）
func (p *RedisConnPool) DelKey(key string) (interface{}, error) {
	conn := p.redisPool.Get()
	defer conn.Close()
	return conn.Do("DEL", key)
}

func (p *RedisConnPool) ExpireKey(key string, seconds int64) (interface{}, error) {
	conn := p.redisPool.Get()
	defer conn.Close()
	return conn.Do("EXPIRE", key, seconds)
}

func (p *RedisConnPool) Keys(pattern string) ([]string, error) {
	conn := p.redisPool.Get()
	defer conn.Close()
	return redis.Strings(conn.Do("KEYS", pattern))
}

func (p *RedisConnPool) KeysByteSlices(pattern string) ([][]byte, error) {
	conn := p.redisPool.Get()
	defer conn.Close()
	return redis.ByteSlices(conn.Do("KEYS", pattern))
}

//// Hash（哈希表）
func (p *RedisConnPool) SetHashMap(key string, fieldValue map[string]interface{}) (interface{}, error) {
	conn := p.redisPool.Get()
	defer conn.Close()
	return conn.Do("HMSET", redis.Args{}.Add(key).AddFlat(fieldValue)...)
}

func (p *RedisConnPool) GetHashMapString(key string) (map[string]string, error) {
	conn := p.redisPool.Get()
	defer conn.Close()
	return redis.StringMap(conn.Do("HGETALL", key))
}

func (p *RedisConnPool) GetHashMapInt(key string) (map[string]int, error) {
	conn := p.redisPool.Get()
	defer conn.Close()
	return redis.IntMap(conn.Do("HGETALL", key))
}

func (p *RedisConnPool) GetHashMapInt64(key string) (map[string]int64, error) {
	conn := p.redisPool.Get()
	defer conn.Close()
	return redis.Int64Map(conn.Do("HGETALL", key))
}
</pre>
