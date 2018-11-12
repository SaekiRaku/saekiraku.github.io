---
title: 微服务 gRPC 入门教程
abbrlink: 35492
date: 2018-08-03 15:57:58
category: 微服务
banner: https://saekiraku-1253597019.coscd.myqcloud.com/image/qt6up.png
tags:
 - gRPC
 - 微服务
---

gRPC & Protobuf 入门。

<!-- more -->

# 基础概念

RPC（Remote Procedure Call） —— 远程过程调用，它是一种通过网络从远程计算机程序上请求服务，而不需要了解底层网络技术的协议。举例来说：一些大公司的系统都是由成千上万大大小小的服务组成的，每个服务部署在不同的机器上，由不同的团队负责。各个服务之间又会相互依赖和关联，此时，服务之间互相依赖和调用，就需要基于 RPC 协议。虽然也可以基于 HTTP/1 协议进行通信，但是其效率远不如 RPC。

gRPC 是由 Google 主导开发的高性能、跨语言的远程过程调用框架，使用 HTTP/2 协议并用 ProtoBuf 作为序列化工具。其主要优点是通信速度更快更高效，尤其是在成百上千的微服务环境下，服务间的相互调用如果效率低下，就会非常影响应用性能。

# 基本流程

1. 使用 Protobuf 定义 `接口描述文件`，即 .proto 文件。
2. 然后使用 compile 工具生成特定语言的 `执行代码` ，比如 Golang、Python 等。
3. 引用 `执行代码` ，编写业务逻辑，启动 `Server` 端，Server 端通过侦听指定的端口，来等待 Client 端的链接请求。
4. 引用 `执行代码` ，编写业务逻辑，启动 `Client` 端，Client 通过与 Server 建立 TCP 长链接，并发送请求。

# Protobuf

## 基本语法

```golang
syntax = "proto3";
// 声明 proto 文件语法版本

package helloworld;
// 声明包名

service Greeter {
    // 定义服务
    rpc SayHello (HelloRequest) returns (HelloReply) {}
    // 定义服务下 sayHello 方法，传入 HelloRequest 消息结构，输出 HelloReply 消息结构
}

message HelloRequest {
    // 定义 HelloRequest 消息结构
    string name = 1;
    // 定义字符串类型 name 字段，并且其位置为1。
    int32 age = 2;
    // 定义变长编码的整数类型 age 字段，并且其位置为2。
}

message HelloReply {
    // 定义 HelloRequest 消息结构
    string message = 1;
    // 定义字符串类型 message 字段，并且其位置为1。
}
```

## 编译

double		double	double	float	float64	Float	double	float
float		float	float	float	float32	Float	float	float
int32	使用变长编码，对于负值的效率很低，如果你的域有可能有负值，请使用sint64替代	int32	int	int	int32	Fixnum 或者 Bignum（根据需要）	int	integer
uint32	使用变长编码	uint32	int	int/long	uint32	Fixnum 或者 Bignum（根据需要）	uint	integer
uint64	使用变长编码	uint64	long	int/long	uint64	Bignum	ulong	integer/string
sint32	使用变长编码，这些编码在负值时比int32高效的多	int32	int	int	int32	Fixnum 或者 Bignum（根据需要）	int	integer
sint64	使用变长编码，有符号的整型值。编码时比通常的int64高效。	int64	long	int/long	int64	Bignum	long	integer/string
fixed32	总是4个字节，如果数值总是比总是比228大的话，这个类型会比uint32高效。	uint32	int	int	uint32	Fixnum 或者 Bignum（根据需要）	uint	integer
fixed64	总是8个字节，如果数值总是比总是比256大的话，这个类型会比uint64高效。	uint64	long	int/long	uint64	Bignum	ulong	integer/string
sfixed32	总是4个字节	int32	int	int	int32	Fixnum 或者 Bignum（根据需要）	int	integer
sfixed64	总是8个字节	int64	long	int/long	int64	Bignum	long	integer/string
bool		bool	boolean	bool	bool	TrueClass/FalseClass	bool	boolean
string	一个字符串必须是UTF-8编码或者7-bit ASCII编码的文本。	string	String	str/unicode	string	String (UTF-8)	string	string
bytes	可能包含任意顺序的字节数据。	string	ByteString	str	[]byte	String (ASCII-8BIT)	ByteString	string
