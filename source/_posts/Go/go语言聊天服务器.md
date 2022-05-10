---
title: go语言聊天服务器
comments: false
date: 2022-05-10 12:12:24
categories:
  - go
tags:
---

&emsp;&emsp;本节将开发一个聊天的示例程序，它可以在几个用户之间相互广播文本消息。其中广播是通过通道 chan 来实现的。主要文件有服务端`netsever.go`和客户端`netclient.go`。

## 服务端 netsever.go

&emsp;&emsp;对于服务端程序，有全局变量`entering`、`leaving`和 `messages`。此后通过监听这三个通道，当有消息数据到来时`select`会感知到，而后就会向保存的连接轮流发送消息，也就实现了广播，而当连接上下线的时候，`func handleConn(conn net.Conn)`会向`leaving`和 `messages`中写入通道变量，然后根据这个变量在`clients := make(map[client]bool)`中添加连接或删除连接。

&emsp;&emsp;对于服务端的`main`函数，主要是监听 tcp 的连接，开启广播，而后`for`循环中监听是否有新的连接到来，到来后新建`goroutine`来处理连接，执行函数`func handleConn(conn net.Conn)`,首先会新建`goroutline`来执行函数`func clientWriter(conn net.Conn, ch <-chan string)`,使用创建的`chan`来沟通服务端和客户端，使用函数`conn.RemoteAddr().String()`来获得地址，此后将向通道写入创建的通道变量，并在`clients` 里面进行保存,连接`IO`缓冲区，来打印沟通的信息，直到下线，然后关闭连接。

&emsp;&emsp;对于广播函数`func broadcaster()`,则用于维护局部变量`clients`,当有信息的时，使用`for`进行轮流发送信息,当有上下线时，修改该局部变量。

```go
package main

import (
	"bufio"
	"fmt"
	"log"
	"net"
)

func main() {
	//监听
	listener, err := net.Listen("tcp", "localhost:8000")
	if err != nil {
		log.Fatal(err)
	}
	//开启广播
	go broadcaster()
	fmt.Println("服务器启动成功!")
	for {
		//新的连接到来
		conn, err := listener.Accept()
		if err != nil {
			log.Print(err)
			continue
		}
		//处理该连接
		go handleConn(conn)
	}
}

type client chan<- string //对外发送消息的通道

//全局变量，用于监听是否有连接加入或离开，以及广播消息
var (
	entering = make(chan client)
	leaving  = make(chan client)
	messages = make(chan string) //所有连接的客户端，即发送到客户端的消息发送到messages中
)

//广播函数
func broadcaster() {
	clients := make(map[client]bool)
	for {
		select {
		case msg := <-messages:
			// 把所有接收到的消息广播给所有客户端
			for cli := range clients {
				cli <- msg
			}
		case cli := <-entering:
			//有用户进入
			clients[cli] = true
		case cli := <-leaving:
			//有用户退出
			delete(clients, cli)
			close(cli)
		}
	}
}

//处理连接
func handleConn(conn net.Conn) {
	ch := make(chan string) //对外发送客户消息的通道
	go clientWriter(conn, ch)
	who := conn.RemoteAddr().String()
	ch <- "欢迎 " + who
	messages <- who + " 上线"
	entering <- ch
	input := bufio.NewScanner(conn)
	for input.Scan() {
		messages <- who + ": " + input.Text()
	}
	leaving <- ch
	messages <- who + " 下线"
	conn.Close()
}

func clientWriter(conn net.Conn, ch <-chan string) {
	for msg := range ch {
		fmt.Fprintln(conn, msg)
	}
}
```

## 客户端 netclient.go

&emsp;&emsp;对于客户端则比较简单，主要是将`os.Stdout`、
`os.Stdin`和连接`conn`进行映射。让其不断地接受服务端的数据和发送输入缓冲区的数据。

&emsp;&emsp;在客户端主要注意下使用匿名函数创建 `goroutine`。

```go
// netcat 是一个简单的TCP服务器读/写客户端
package main

import (
	"fmt"
	"io"
	"log"
	"net"
	"os"
)

func main() {
	conn, err := net.Dial("tcp", "localhost:8000")
	if err != nil {
		log.Fatal(err)
	}
	done := make(chan struct{})
	go func() {
		//该线程用于接受数据
		io.Copy(os.Stdout, conn)
		//逐一分析，阻塞的，这后面是没有执行的
		fmt.Println("done")
		done <- struct{}{} // 向主Goroutine发出完成信息
	}()
	mustCopy(conn, os.Stdin)
	conn.Close()
	<-done // 等待后台goroutine完成
}

//发送数据
func mustCopy(dst io.Writer, src io.Reader) {
	_, err := io.Copy(dst, src)
	if err != nil {
		log.Fatal(err)
	}
}
```
