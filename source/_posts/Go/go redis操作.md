---
title: go redis操作
comments: false
date: 2022-06-11 15:44:52
categories:
  - go
tags:
  - redis
---

在本文将会讲述 go 语言操作`redis`。在学习过程中，首先使用`github.com/go-redis/redis`,但是版本的原因，导致不能成功连接`redis`的服务器。经过调研可使用`github.com/garyburd/redigo/redis`进行 redis 的操作。

<!--more-->

# 使用 go-redis

```go
package main

import (
	"fmt"
	"github.com/go-redis/redis"
)

var redisDb *redis.Client

// 根据redis配置初始化一个客户端
func initClient() (err error) {
	redisDb = redis.NewClient(&redis.Options{
		Addr:     "localhost:6379", // redis地址
		Password: "123456",         // redis密码，没有则留空
		DB:       0,                // 默认数据库，默认是0
	})
	//通过 *redis.Client.Ping() 来检查是否成功连接到了redis服务器
	_, err = redisDb.Ping().Result()
	if err != nil {
		return err
	}
	return nil
}

func main() {
	err := initClient()
	if err != nil {
		panic(err)
	}
	//第三个参数代表key的过期时间，0代表不会过期。
	err = redisDb.Set("name", "xzm", 0).Err()
	if err != nil {
		panic(err)
	}
	var val string
	//Result函数返回两个值，第一个是key的值，第二个是错误信息
	val, err = redisDb.Get("name").Result()
	//判断查询是否出错
	if err != nil {
		panic(err)
	}
	fmt.Println("name的值：", val)
}
```

# 使用 redigo

## 非连接池

```
package main

import (
	"fmt"
	"github.com/garyburd/redigo/redis"
)

func main() {
	conn, err := redis.Dial("tcp", "127.0.0.1:6379", redis.DialPassword("123456"))
	if err != nil {
		panic(err)
	}
	defer conn.Close()
	//向redis写入数据
	n, err := conn.Do("Set", "name", "xzm")
	if err != nil {
		panic(err)
	}
	fmt.Println("n:", n)

	//向redis读取数据,返回的r是interface{},因此需要转换
	r, err := redis.String(conn.Do("Get", "name"))
	if err != nil {
		panic(err)
	}
	fmt.Println("conn succ...", r)
}
```

## 连接池

```go
package main

import (
	"fmt"

	"github.com/garyburd/redigo/redis"
)

var pool *redis.Pool //创建redis连接池

func init() {
	//实例化一个连接池
	pool = &redis.Pool{
		MaxIdle:     16,      //最初的连接数量
		MaxActive:   1000000, //最大连接数量,不确定可以用0（0表示自动定义）按需分配
		IdleTimeout: 300,     //连接关闭时间 300秒 （300秒不使用自动关闭）
		Dial: func() (redis.Conn, error) { //要连接的redis数据库
			return redis.Dial("tcp", "localhost:6379", redis.DialPassword("123456"))
		},
	}
}

func main() {
	c := pool.Get() //从连接池，取一个链接
	defer c.Close() //函数运行结束 ，把连接放回连接池

	_, err := c.Do("set", "name", "xzm") // 这里set/get大小写都可以的
	if err != nil {
		panic(err)
	}
	r, err := redis.String(c.Do("get", "name"))
	if err != nil {
		panic(err)
	}
	fmt.Println(r)
	pool.Close() //关闭连接池
}
```
