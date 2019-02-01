---
title: Thrift concept
date: 2017-10-27 13:30:01
categories: Distributed
tags: Thrift
description: Thrift concept
---

## 前言

上一篇文章对Thrift RPC 及其跨语言的特性进行了初步的了解，Thrift 对消息的打包和发送操作进行了一些抽象和封装。那么，今天就来了解一下Thrift 框架中进行的一些抽象概念。

## Thrift 网络栈

下图代表Thrift 的网络栈：

```
  +-------------------------------------------+
  | Server                                    |
  | (single-threaded, event-driven etc)       |
  +-------------------------------------------+
  | Processor                                 |
  | (compiler generated)                      |
  +-------------------------------------------+
  | Protocol                                  |
  | (JSON, compact etc)                       |
  +-------------------------------------------+
  | Transport                                 |
  | (raw TCP, HTTP etc)                       |
  +-------------------------------------------+
```

## Transport

传输层作为从网络读/写的简单抽象。使底层传输与系统的其余部分解耦（例如序列化/反序列化）。

Transport 接口暴露以下一些方法：

- open 
- close
- read
- write
- flush

除了 Transport 接口之外，还有用于服务端的 ServerTransport 来接收请求连接：

- open
- listen
- accept
- close

## Protocol

协议抽象定义了将内存数据结构映射成线上格式的一种机制。换句话说，就是指定数据类型如何使用底层传输编码/解码自己。因此，协议实现控制编码方案，并负责（反）序列化。从这个意义上讲，协议的一些例子包括JSON、XML、纯文本、二进制等。

<!-- more -->

下面是 Protocol 接口内容：

```
writeMessageBegin(name, type, seq)
writeMessageEnd()
writeStructBegin(name)
writeStructEnd()
writeFieldBegin(name, type, id)
writeFieldEnd()
writeFieldStop()
writeMapBegin(ktype, vtype, size)
writeMapEnd()
writeListBegin(etype, size)
writeListEnd()
writeSetBegin(etype, size)
writeSetEnd()
writeBool(bool)
writeByte(byte)
writeI16(i16)
writeI32(i32)
writeI64(i64)
writeDouble(double)
writeString(string)

name, type, seq = readMessageBegin()
                  readMessageEnd()
name = readStructBegin()
       readStructEnd()
name, type, id = readFieldBegin()
                 readFieldEnd()
k, v, size = readMapBegin()
             readMapEnd()
etype, size = readListBegin()
              readListEnd()
etype, size = readSetBegin()
              readSetEnd()
bool = readBool()
byte = readByte()
i16 = readI16()
i32 = readI32()
i64 = readI64()
double = readDouble()
string = readString()
```

Thrift 协议是面向流设计的，在进行序列化之前不需要知道字符串的长度或者列表的项目数量。

## Processor

处理器封装了从输入流读取数据并写入输出流的能力。输入输出流代表 Protocol 对象。Processor 接口极其的简单：

```
interface TProcessor {
    bool process(TProtocol in, TProtocol out) throws TException
}
```

特定服务的处理器实现是由编译器生成的。处理器本质就是从网络上读取数据（使用 input protocol），委托handler（由用户实现，即接口的实现类） 来进行处理，然后返回响应信息（使用 output protocol）。

## Server

服务器将上面描述的所有特性结合起来：

- 创建一个transport
- 为 transport 创建 input/output protocols
- 基于input/output protocols 来创建一个processor
- 等待请求连接并将其发送到processor

## 整体架构

![Thrift 架构](https://raw.githubusercontent.com/rason/rason.github.io/master/image/Apache_Thrift_architecture.png)
