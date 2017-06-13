---
date: 2017-01-08
layout: post
title: websocket协议学习
thread: 2017-01-08-websocket.md
categories: 计算机原理
tags: 协议
---

<br/>
#### 一 实验代码
<br/>
1. client.html
<br/>
<pre>
<html>
    <head>
        <script type="text/javascript" src="/media/js/jquery-1.7.1.min.js">
        </script>
    </head>
    
    <body>
        <input type="button" id="connect" value="socket connect" />
        <input type="button" id="send" value="socket send" />
        <input type="button" id="close" value="socket close" />
    </body>
    <script type="text/javascript">
        var socket;
        $("#connect").click(function(event) {
            socket = new WebSocket("ws://127.0.0.1:8000");
            socket.onopen = function() {
                alert("Socket has been opened");
            }
            socket.onmessage = function(msg) {
                alert(msg.data);
            }
            socket.onclose = function() {
                alert("Socket has been closed");
            }
        });

        $("#close").click(function(event) {
            socket.close();
        })
		
        $("#send").click(function(event) {
            socket.send("Websocket server, I am browser.")
        });
    </script>
</html>
</pre>

<br/>
2. websocket_server.go
<br/>
<pre>
package main

import (
	"crypto/sha1"
	"encoding/base64"
	"encoding/binary"
	"io"
	"log"
	"net"
	"strings"
	"time"
)

type WsSocket struct {
	Conn net.Conn
}

// 帧类型(OPCODE). RFC 6455, section 11.8.
const (
	FRAME_CONTINUE = 0  //继续帧
	FRAME_TEXT     = 1  //文本帧
	FRAME_BINARY   = 2  //二进制帧
	FRAME_CLOSE    = 8  //关闭帧
	FRAME_PING     = 9  //ping帧
	FRAME_PONG     = 10 //pong帧
)

func init() {
	//初始化日志打印格式
	log.SetFlags(log.Lshortfile | log.LstdFlags)
}

func main() {
	ln, err := net.Listen("tcp", ":8000")
	defer ln.Close()
	if err != nil {
		log.Panic(err)
	}

	for {
		conn, err := ln.Accept()
		if err != nil {
			log.Println("accept err:", err)
		}
		go handleConnection(conn)
	}

}

func handleConnection(conn net.Conn) {
	// http request open websocket
	content := make([]byte, 1024)
	conn.Read(content)
	log.Printf("http request:\n%s\n", string(content))
	headers := parseHttpHeader(string(content))
	secWebsocketKey := headers["Sec-WebSocket-Key"]

	// http response open websocket
	response := "HTTP/1.1 101 Switching Protocols\r\n"
	response += "Sec-WebSocket-Accept: " + computeAcceptKey(secWebsocketKey) + "\r\n"
	response += "Connection: Upgrade\r\n"
	response += "Upgrade: websocket\r\n\r\n"
	log.Printf("http response:\n%s\n", response)
	if lenth, err := conn.Write([]byte(response)); err != nil {
		log.Println(err)
	} else {
		log.Println("send http response len:", lenth)
	}

	// websocket established
	wssocket := &WsSocket{Conn: conn}

	//begin test case
	for {
		time.Sleep(5 * time.Second)
		// frame ping
		wssocket.SendIframe(FRAME_PING, []byte("hello"))
		// frame read //浏览器响应同样负载数据的pong帧
		log.Printf("server read data from client:\n%s\n", string(wssocket.ReadIframe()))
	}
	//end test case
}

//发送帧给客户端(不考虑分片)(只做服务端,无掩码)
func (this *WsSocket) SendIframe(OPCODE int, frameData []byte) {
	dataLen := len(frameData)
	var n int
	var err error

	//第一个字节b1
	b1 := 0x80 | byte(OPCODE)
	n, err = this.Conn.Write([]byte{b1})
	if err != nil {
		log.Printf("Conn.Write() error,length:%d;error:%s\n", n, err)
		if err == io.EOF {
			log.Println("客户端已经断开WsSocket!")
		} else if err.(*net.OpError).Err.Error() == "use of closed network connection" {
			log.Println("服务端已经断开WsSocket!")
		}
	}

	//第二个字节
	var b2 byte
	var payloadLen int
	switch {
	case dataLen <= 125:
		b2 = byte(dataLen)
		payloadLen = dataLen
	case 126 <= dataLen && dataLen <= 65535:
		b2 = byte(126)
		payloadLen = 126
	case dataLen > 65535:
		b2 = byte(127)
		payloadLen = 127
	}
	this.Conn.Write([]byte{b2})

	//如果payloadLen不够用,写入exPayLenByte,用exPayLenByte表示负载数据的长度
	switch payloadLen {
	case 126:
		exPayloadLenByte := make([]byte, 2)
		exPayloadLenByte[0] = byte(dataLen >> 8) //高8位
		exPayloadLenByte[1] = byte(dataLen)      //低8位
		this.Conn.Write(exPayloadLenByte)        //扩展2个字节表示负载数据长度, 最高位也可以用
	case 127:
		exPayloadLenByte := make([]byte, 8)
		exPayloadLenByte[0] = byte(dataLen >> 56) //第1个字节
		exPayloadLenByte[1] = byte(dataLen >> 48) //第2个字节
		exPayloadLenByte[2] = byte(dataLen >> 40) //第3个字节
		exPayloadLenByte[3] = byte(dataLen >> 32) //第4个字节
		exPayloadLenByte[4] = byte(dataLen >> 24) //第5个字节
		exPayloadLenByte[5] = byte(dataLen >> 16) //第6个字节
		exPayloadLenByte[6] = byte(dataLen >> 8)  //第7个字节
		exPayloadLenByte[7] = byte(dataLen)       //第8个字节
		this.Conn.Write(exPayloadLenByte)         //扩展8个字节表示负载数据长度, 最高位不可以用,必须为0
	}
	this.Conn.Write(frameData) //无掩码,直接在表示长度的区域后面写入数据
	log.Printf("real payloadLen=%d:该数据帧的真实负载数据长度(bytes).\n", dataLen)
	log.Println("MASK=0:没有掩码.")
	log.Printf("server send data to client:\n%s\n", string(frameData))

}

//读取客户端发送的帧(考虑分片)
func (this *WsSocket) ReadIframe() (frameData []byte) {
	var n int
	var err error

	//第一个字节
	b1 := make([]byte, 1)
	n, err = this.Conn.Read(b1)
	if err != nil {
		log.Printf("Conn.Read() error,length:%d;error:%s\n", n, err)
		if err == io.EOF {
			log.Println("客户端已经断开WsSocket!")
		} else if err.(*net.OpError).Err.Error() == "use of closed network connection" {
			log.Println("服务端已经断开WsSocket!")
		}
	}
	FIN := b1[0] >> 7
	OPCODE := b1[0] & 0x0F
	if OPCODE == 8 {
		log.Println("OPCODE=8:连接关闭帧.")
		this.SendIframe(FRAME_CLOSE, formatCloseMessage(1000, "因为收到客户端的主动关闭请求,所以响应."))
		this.Conn.Close()
		return
	}

	//第二个字节
	b2 := make([]byte, 1)
	this.Conn.Read(b2)
	payloadLen := int64(b2[0] & 0x7F) //payloadLen:表示数据报文长度(可能不够用),0x7F(16) > 01111111(2)
	MASK := b2[0] >> 7                //MASK=1:表示客户端发来的数据,且表示采用了掩码(客户端传来的数据必须采用掩码)
	log.Printf("second byte:MASK=%d, raw payloadLen=%d\n", MASK, payloadLen)

	//扩展长度
	dataLen := payloadLen
	switch {
	case payloadLen == 126:
		// 如果payloadLen=126,启用2个字节作为拓展,表示更长的报文
		// 负载数据的长度范围(bytes):126~65535(2) 0xffff
		log.Println("raw payloadLen=126,启用2个字节作为拓展（最高有效位可以是1,使用所有位）,表示更长的报文")
		exPayloadLenByte := make([]byte, 2)
		n, err := this.Conn.Read(exPayloadLenByte)
		if err != nil {
			log.Printf("Conn.Read() error,length:%d;error:%s\n", n, err)
		}
		dataLen = int64(exPayloadLenByte[0])<<8 + int64(exPayloadLenByte[1])

	case payloadLen == 127:
		// 如果payloadLen=127,启用8个字节作为拓展,表示更长的报文
		// 负载数据的长度范围(bytes):65536~0x7fff ffff ffff ffff
		log.Println("payloadLen=127,启用8个字节作为拓展（最高有效位必须是0,舍弃最高位）,表示更长的报文")
		exPayloadLenByte := make([]byte, 8)
		this.Conn.Read(exPayloadLenByte)
		dataLen = int64(exPayloadLenByte[0])<<56 + int64(exPayloadLenByte[1])<<48 + int64(exPayloadLenByte[2])<<40 + int64(exPayloadLenByte[3])<<32 + int64(exPayloadLenByte[4])<<24 + int64(exPayloadLenByte[5])<<16 + int64(exPayloadLenByte[6])<<8 + int64(exPayloadLenByte[7])
	}
	log.Printf("real payloadLen=%d:该数据帧的真实负载数据长度(bytes).\n", dataLen)

	//掩码
	maskingByte := make([]byte, 4)
	if MASK == 1 {
		this.Conn.Read(maskingByte)
		log.Println("MASK=1:负载数据采用了掩码.")
	} else if MASK == 0 {
		log.Println("MASK=0:没有掩码.")
	}

	//数据
	payloadDataByte := make([]byte, dataLen)
	this.Conn.Read(payloadDataByte)
	dataByte := make([]byte, dataLen)
	for i := int64(0); i < dataLen; i++ {
		if MASK == 1 { //解析掩码数据
			dataByte[i] = payloadDataByte[i] ^ maskingByte[i%4]
		} else {
			dataByte[i] = payloadDataByte[i]
		}
	}

	//如果没有数据,强制停止递归
	if dataLen <= 0 {
		return
	}
	//最后一帧,正常停止递归
	if FIN == 1 {
		return dataByte
	}
	//中间帧
	nextData := this.ReadIframe()
	//汇总
	return append(frameData, nextData...)
}

//计算Sec-WebSocket-Accept
func computeAcceptKey(secWebsocketKey string) string {
	var keyGUID = []byte("258EAFA5-E914-47DA-95CA-C5AB0DC85B11")
	h := sha1.New()
	h.Write([]byte(secWebsocketKey))
	h.Write(keyGUID)
	return base64.StdEncoding.EncodeToString(h.Sum(nil))
}

//HTTP报文头部map
func parseHttpHeader(content string) map[string]string {
	headers := make(map[string]string, 10)
	lines := strings.Split(content, "\r\n")
	for _, line := range lines {
		if len(line) >= 0 {
			words := strings.Split(line, ":")
			if len(words) == 2 {
				headers[strings.Trim(words[0], " ")] = strings.Trim(words[1], " ")
			}
		}
	}
	return headers
}

//二进制逐位打印字节数组
func printBinary(data []byte) {
	for i := 0; i < len(data); i++ {
		byteData := data[i]
		var j uint
		for j = 7; j > 0; j-- {
			log.Printf("%d", ((byteData >> j) & 0x01))
		}
		log.Printf("%d\n", ((byteData >> j) & 0x01))
	}
}

//关闭码 + 关闭原因 = 关闭帧的负载数据
func formatCloseMessage(closeCode int, text string) []byte {
	buf := make([]byte, 2+len(text))
	binary.BigEndian.PutUint16(buf, uint16(closeCode))
	copy(buf[2:], text)
	return buf
}
</pre>
> [代码链接，仅供学习](https://github.com/hopehook/go-lab/blob/master/4.websocket)
>
> websocket_server.go: websocket 基于 tcp socket 的粗糙实现, 只提供 websocket 服务
> 
> websocket_http_server.go: 把该实现移植到了 http socket 环境(也可以是某个 golang web 框架), 实现了 websocket http 利用同一个端口，同时对> 外服务。原理：
> <pre>
> // 通过Hijacker拿到http连接下的tcp连接，Hijack()之后该连接完全由自己接管
> conn, _, err := w.(http.Hijacker).Hijack()
> </pre>

<br/>
#### 二 websocket协议阅读要点记录
* [RFC协议中文版](https://github.com/zhangkaitao/websocket-protocol)
* [RFC协议英文版](https://tools.ietf.org/html/rfc6455)
<br/>
1.客户端必须掩码(mask)它发送到服务器的所有帧(更多详细信息请参见 5.3 节)。
<br/>
2.当收到一个没有掩码的帧时,服务器必须关闭连接。在这种情况下,服务器可能发送一个定义在 7.4.1 节的状态码 1002(协议错误)的 Close 帧。
<br/>
3.服务器必须不掩码发送到客户端的所有帧。如果客户端检测到掩码的帧,它必须关闭连接。在这种情况下,它可能使用定义在 7.4.1节的状态码 1002(协议错误)。
<br/>
4.一个没有分片的消息由单个带有 FIN 位设置(5.2 节)和一个非 0 操作码的帧组成。
<br/>
5.一个分片的消息由单个带有 FIN 位清零(5.2 节)和一个非 0 操作码的帧组成,跟随零个或多个带有 FIN 位清零和操作码设置为 0 的帧,且终止于一个带有 FIN 位设置且 0 操作码的帧。一个分片的消息概念上是等价于单个大的消息,其负载是等价于按顺序串联片段的负载
<br/>
6.控制帧(参见 5.5 节)可能被注入到一个分片消息的中间。控制帧本身必须不被分割。
<br/>
7.消息分片必须按发送者发送顺序交付给收件人。
<br/>
8.一个消息的所有分片是相同类型,以第一个片段的操作码设置。
<br/>
9.关闭帧可以包含内容体(“帧的“应用数据”部分)指示一个关闭的原因,例如端点关闭了、端点收到的帧太大、或端点收到的帧不符合端点期望的格式。如果有内容体,内容体的头两个字节必须是 2 字节的无符号整数(按网络字节顺序)代表一个在 7.4 节的/code/值定义的状态码。跟着 2 字节的整数,内容体可以包含 UTF-8 编码的/reason/值,本规范没有定义它的解释。数据不必是人类可读的但可能对调试或传递打开连接的脚本相关的信息是有用的。由于数据不保证人类可读,客户端必须不把它显示给最终用户。
<br/>
10.在应用发送关闭帧之后,必须不发送任何更多的数据帧。
<br/>
11.发送并接收一个关闭消息后,一个端点认为 WebSocket 连接关闭了且必须关闭底层的 TCP 连接。服务器必须立即关闭底层 TCP 连接,客户端应该等待服务器关闭连接但可能在发送和接收一个关闭消息之后的任何时候关闭连接,例如,如果它没有在一个合理的时间周期内接收到服务器的 TCP 关闭。
<br/>
12.一个端点可以在连接建立之后并在连接关闭之前的任何时候发送一个 Ping 帧。注意:一个 Ping 即可以充当一个 keepalive,也可以作为验证远程端点仍可响应

<br/>
#### 三 小经验
1.浏览器目前没有提供js接口发送ping帧,浏览器可能单向的发送pong帧(可以利用文本帧当作ping帧来使用)
<br/>
2.服务端给浏览器发送ping帧,浏览器会尽快响应同样负载数据的pong帧
<br/>
3.浏览器发送的websocket负载数据太大的时候会分片
<br/>
4.不管是浏览器,还是服务器,收到close帧都回复同样内容的close帧,然后做后续的操作

<br/>
#### 四 连接断开情况分析
* server:s
* browser:b
<br/>
<br/>
**情况0**
<br/>
动作:b发s连接关闭帧,s无操作
<br/>
现象:
<br/>
0) b过很久之后触发了onclose
<br/>
1) s写入: *net.OpError: write tcp 127.0.0.1:8000->127.0.0.1:34508: write: broken pipe
<br/>
2) s读取: *errors.errorString: EOF
<br/>
<br/>
**情况1**
<br/>
动作:b发s连接关闭帧,s回应连接关闭帧
<br/>
现象:
<br/>
0) b马上触发了onclose
<br/>
1) s写入: *net.OpError: write tcp 127.0.0.1:8000->127.0.0.1:34482: write: broken pipe
<br/>
2) s读取: *errors.errorString: EOF
<br/>
<br/>
**情况2**
<br/>
动作:b发s连接关闭帧,s回应连接关闭帧,s关闭tcp socket
<br/>
现象:
<br/>
0) b马上触发了onclose
<br/>
1) s写入: *net.OpError: write tcp 127.0.0.1:8000->127.0.0.1:34502: use of closed network connection
<br/>
2) s读取: *net.OpError: read tcp 127.0.0.1:8000->127.0.0.1:34502: use of closed network connection
<br/>
<br/>
**情况3**
<br/>
动作:s发b连接关闭帧,b无操作
<br/>
现象:
<br/>
0) b马上回应相同数据的关闭帧, 接着触发onclose
<br/>
1) s写入: *net.OpError: write tcp 127.0.0.1:8000->127.0.0.1:34482: write: broken pipe
<br/>
2) s读取: *errors.errorString: EOF
<br/>
<br/>
**情况4**
<br/>
动作:s发b连接关闭帧,s关闭tcp socket
<br/>
现象:
<br/>
0) b马上触发了onclose
<br/>
1) s写入: *net.OpError: write tcp 127.0.0.1:8000->127.0.0.1:34542: use of closed network connection
<br/>
2) s读取: *net.OpError: tcp 127.0.0.1:8000->127.0.0.1:34542: use of closed network connection

