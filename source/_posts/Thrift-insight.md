---
title: Thrift insight
date: 2017-10-31 09:29:52
categories: Distributed
tags: Thrift
description: Thrift insight
---

## 前言

上两篇文章从基本使用和基础架构两个方面初步了解了一下 Thrift 。今天继续以之前的入门例子为基础，深入了解一下其调用过程。

## 客户端调用过程

### 本地发起调用

```
HelloService.Client client = new HelloService.Client(protocol);

client.hello("rason");
```

我们重点是要了解 Thrift 编译器生成的客户端代码是如何进行远程调用的，也就是`client.hello("rason")`这个方法进行了什么样的操作。

```
public void hello(String name) throws org.apache.thrift.TException
{
    send_hello(name);
    recv_hello();
}
```

该方法抽象得很简单，就做两件事情：发送请求和接收响应。认真观察这两个方法，会发现其特点：发送方法的格式是`send_` + 方法名称，参数为远程方法所需要的参数；接收响应方法格式为`recv_` + 方法名称。

## 发送请求

```
public void send_hello(String name) throws org.apache.thrift.TException
{
    hello_args args = new hello_args();
    args.setName(name);
    sendBase("hello", args);
}
```

Thrift 会为每个方法的参数生成一个参数内部类，类名的格式为：方法名称 + `_args`，如上的`hello_args`。

参数设置完毕之后开始发送，调用`sendBase("hello", args)`方法。方法的第一个参数为远程方法名，第二个参数为远程方法所需的参数对象。

```
protected void sendBase(String methodName, TBase<?,?> args) throws TException {
    sendBase(methodName, args, TMessageType.CALL);
}
```

发送请求的消息类型为调用`TMessageType.CALL`， Thrift 的消息类型一共有以下四种：

```
public final class TMessageType {
    public static final byte CALL  = 1;
    public static final byte REPLY = 2;
    public static final byte EXCEPTION = 3;
    public static final byte ONEWAY = 4;
}
```

<!-- more -->

然后，调用所使用的传输协议来将数据写网络：

```
private void sendBase(String methodName, TBase<?,?> args, byte type) throws TException {
    oprot_.writeMessageBegin(new TMessage(methodName, type, ++seqid_));
    args.write(oprot_);
    oprot_.writeMessageEnd();
    oprot_.getTransport().flush();
}
```

我们的例子中使用的是二进制协议`TBinaryProtocol`。先写消息头，消息头的内容包括所调用方法的名称，消息类型和消息序号。这个消息序号的用途是将请求和响应对应起来，响应消息会原封不动地将这个序号返回以确定是哪个请求的响应。然后写参数，最后将缓冲区刷新进行写网络。

## 接收响应

```
public void recv_hello() throws org.apache.thrift.TException
{
    hello_result result = new hello_result();
    receiveBase(result, "hello");
    return;
}
```

同样的，响应信息也会自动生成一个响应结果类。然后接收`hello` 方法的响应：

```
protected void receiveBase(TBase<?,?> result, String methodName) throws TException {
    TMessage msg = iprot_.readMessageBegin();
    if (msg.type == TMessageType.EXCEPTION) {
        TApplicationException x = new TApplicationException();
        x.read(iprot_);
        iprot_.readMessageEnd();
        throw x;
    }
    System.out.format("Received %d%n", msg.seqid);
    if (msg.seqid != seqid_) {
        throw new TApplicationException(TApplicationException.BAD_SEQUENCE_ID,
            String.format("%s failed: out of sequence response: expected %d but got %d", methodName, seqid_, msg.seqid));
    }
    result.read(iprot_);
    iprot_.readMessageEnd();
}
```

- 先判断是否调用有发生异常
- 再判断消息序列号对不对
- 然后才读取结果

```
public void read(org.apache.thrift.protocol.TProtocol iprot) throws org.apache.thrift.TException {
    scheme(iprot).read(iprot, this);
}
```

根据协议读取结果：

```
private static class hello_resultStandardScheme extends org.apache.thrift.scheme.StandardScheme<hello_result> {

    public void read(org.apache.thrift.protocol.TProtocol iprot, hello_result struct) throws org.apache.thrift.TException {
    org.apache.thrift.protocol.TField schemeField;
    iprot.readStructBegin();
    while (true)
    {
        schemeField = iprot.readFieldBegin();
        if (schemeField.type == org.apache.thrift.protocol.TType.STOP) { 
        break;
        }
        switch (schemeField.id) {
        default:
            org.apache.thrift.protocol.TProtocolUtil.skip(iprot, schemeField.type);
        }
        iprot.readFieldEnd();
    }
    iprot.readStructEnd();

    // check for required fields of primitive type, which can't be checked in the validate method
    struct.validate();
    }

    public void write(org.apache.thrift.protocol.TProtocol oprot, hello_result struct) throws org.apache.thrift.TException {
    struct.validate();

    oprot.writeStructBegin(STRUCT_DESC);
    oprot.writeFieldStop();
    oprot.writeStructEnd();
    }

}
```

这里的重点是会根据`schemeField` 的ID逐一读取，由于我们的例子没有返回值，所以只有一个default处理。

## 服务端调用过程

客户端将消息写网络，然后传输到服务端，服务器是如何对消息进行处理的？从之前的 Thrift 基本架构的文章中了解到是通过一个 Processor 来进行处理的：

```
public abstract class TBaseProcessor<I> implements TProcessor {
  private final I iface;
  private final Map<String,ProcessFunction<I, ? extends TBase>> processMap;

  protected TBaseProcessor(I iface, Map<String, ProcessFunction<I, ? extends TBase>> processFunctionMap) {
    this.iface = iface;
    this.processMap = processFunctionMap;
  }

  public Map<String,ProcessFunction<I, ? extends TBase>> getProcessMapView() {
    return Collections.unmodifiableMap(processMap);
  }

  @Override
  public boolean process(TProtocol in, TProtocol out) throws TException {
    TMessage msg = in.readMessageBegin();
    ProcessFunction fn = processMap.get(msg.name);
    if (fn == null) {
      TProtocolUtil.skip(in, TType.STRUCT);
      in.readMessageEnd();
      TApplicationException x = new TApplicationException(TApplicationException.UNKNOWN_METHOD, "Invalid method name: '"+msg.name+"'");
      out.writeMessageBegin(new TMessage(msg.name, TMessageType.EXCEPTION, msg.seqid));
      x.write(out);
      out.writeMessageEnd();
      out.getTransport().flush();
      return true;
    }
    fn.process(msg.seqid, in, out, iface);
    return true;
  }
}
```

Processor 接口只有一个`process`方法，这个方法有以下几个关键点：

- 读取消息头，将消息名字（也就是所调用的远程方法名称）读出来；
- 根据方法名称获取相应的`ProcessFunction`，这个是重点内容；Thrift 会为接口的每一个方法都生成一个处理函数类`ProcessFunction`，类的名称就为方法名称，如下：

```
public static class hello<I extends Iface> extends org.apache.thrift.ProcessFunction<I, hello_args> {
    public hello() {
    super("hello");
    }

    public hello_args getEmptyArgsInstance() {
    return new hello_args();
    }

    protected boolean isOneway() {
    return false;
    }

    public hello_result getResult(I iface, hello_args args) throws org.apache.thrift.TException {
    hello_result result = new hello_result();
    iface.hello(args.name);
    return result;
    }
}
```

- 调用处理函数类`ProcessFunction`的process方法，该方法有一个重要的参数`iface`，也就是服务端接口的实现类的实例，最终会委托该类来进行处理。


## 总结

客户端发送消息头（包括所调用方法的名称，消息类型和消息序号）和消息参数，然后等待接收响应；

服务端根据所调用的方法找到相应的处理函数类`ProcessFunction`，然后委托真正的接口实现类的实例对请求进行处理。
