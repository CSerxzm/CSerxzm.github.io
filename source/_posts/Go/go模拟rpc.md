---
title: go模拟rpc
comments: false
date: 2022-05-10 11:08:10
categories:
  - go
tags:
  - rpc
---

&emsp;&emsp;服务器开发中会使用 RPC（Remote Procedure Call，远程过程调用）简化进程间通信的过程。RPC 能有效地封装通信过程，让远程的数据收发通信过程看起来就像本地的函数调用一样。

&emsp;&emsp;本例中，使用通道代替 Socket 实现 RPC 的过程。客户端与服务器运行在同一个进程，服务器和客户端在两个 goroutine 中运行。

<!-- more -->

&emsp;&emsp;对于代码主要分为客户端程序和服务端程序，其中传输的结构体是 Name，它其中有变量 firstname 和 lastname。通过创建一个没有缓冲区的结构体通道来交换数据。函数`func RPCClient(ch chan Name, req Name) (Name, error) `用来模拟客户端，`func RPCServer(ch chan Name) `用来模拟服务端。在主程序中，首先使用并行方式开启一个服务端来监听通道，而后客户端向其发送数据。服务端响应后将数据原封不动的返回。

```go
package main

import (
	"errors"
	"fmt"
	"time"
)

type Name struct{
	firstname string
	lastname string
}

// 模拟RPC客户端的请求和接收消息封装
func RPCClient(ch chan Name, req Name) (Name, error) {
	// 向服务器发送请求
	ch <- req
	// 等待服务器返回
	select {
	case ack := <-ch: // 接收到服务器返回数据
		return ack, nil
	case <-time.After(time.Second): // 超时
		return Name{}, errors.New("Time out")
	}
}

// 模拟RPC服务器端接收客户端请求和回应
func RPCServer(ch chan Name) {
	for {
		// 接收客户端请求
		data := <-ch
		// 打印接收到的数据
		fmt.Println("server received:", data)
		// 反馈给客户端收到
		myname := Name{firstname:"roger",lastname:"roger"}
		//time.Sleep(time.Second * 2)
		ch <- myname
	}
}

func main() {
	// 创建一个无缓冲字符串通道
	ch := make(chan Name)
	// 并发执行服务器逻辑
	go RPCServer(ch)
	// 客户端请求数据和接收数据
	myname := Name{firstname:"123",lastname:"234"}
	recv, err := RPCClient(ch,myname)
	if err != nil {
		// 发生错误打印
		fmt.Println(err)
	} else {
		// 正常接收到数据
		fmt.Println("client received", recv)
	}
}

```

&emsp;&emsp;对于服务端程序的`time.Sleep(time.Second * 2)`是为了延时操作，当时间超过 1 秒，就会触发 select 的超时通道，打印出`Time out`错误信息。
&emsp;&emsp;该教程只是一个模拟的程序，用来学习 select,后面将会深入的了解和学习 go 语言。
