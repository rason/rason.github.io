title: Java Web编码问题
categories: Java Web
date: 2016-02-13 10:31:44
tags: Java Encoding
description: Java编码问题
---

## 前言

在Java Web开发中经常会遇到编码问题，以前解决编码问题总是不断地尝试，而没有认真地分析乱码出现的根本原因。今天简单地梳理一下编码方面的知识，以便遇到编码问题时能够有据可依地解决问题。

## 编码的原因

- 计算机中存储信息的最小单元是一个字节，即8个bit，所以能表示的字符范围是0~255个。
- 人类要表示的字符太多，无法用一个字节来完全表示。

要解决这个矛盾必须要有一个新的数据结构char，从char到byte必须编码。

## 编码格式

常见的编码格式有ASCII，ISO-8859-1，GB2312，GBK，UTF8，UTF-16。编码格式就相当于字典，规定转化的规则。

1. ASCII码

共128个，用一个字节的低7位表示。

2. ISO-8859-1

128个字符显然不够用，所以要扩展。ISO-8859-1涵盖大多数西欧语言字符，是单字节编码，共能表示256个字符。

3. GB2312

双字节编码，包含682个符号，6763个汉字。

GB2312字符集有一个char到byte的码表，不同的字符编码就查这个码表找到与每个字符的对应的字节，码位值大于0xff是双字节，否则是单字节。

<!-- more -->

4. GBK

扩展GB2312，加入更多汉字，能表示21003个汉字，兼容GB2312编码。也就是说用GB2312编码的汉字可以用GBK来解码，并且不会有乱码。

5. GB18030

可能是单字节，双字节或者四字节编码，与GB2312编码兼容，实际使用不广泛。

6. UTF-16

说到UTF必须提到Unicode，ISO试图创建一个全新的超语言字典，世界上所有的语言都可以通过这本字典来相互翻译。

UTF-16用两个字节来表示Unicode转化格式，不论上面字符都可以用两个字节来表示。每两个字节表示一个字符，大大简化了字符串操作，这也是Java以UTF-16作为内存的字符存储格式的一个很重要的原因。

7. UTF-8

UTF-16采用两个字节表示一个字符，虽然表示上简单，但一个字节就能表示的字符还用两个字节来表示就浪费了存储空间。而UTF-8采用了一种变长技术，每个编码区域有不同的字码长度。UTF-8对单字节范围内字符仍然用一个字节表示，对汉字采用三个字节表示。

UTF-8编码与GBK和GB2312不同，不用查码表，在编码效率上更好，所以在存储中文字符时UTF-8编码比较理想。

UTF-8有以下编码规则：

- 如果一个字节，最高位（第8位）为0，表示这是一个ASCII字符。可见，所有ASCII编码已经是UTF-8了。
- 如果一个字节，以11开头，连续的1的个数暗示这个字符的字节数，例如：110xxxxx代表它是双字节UTF-8字符的首字节。
- 如果一个字节，以10开始，表示它不是首字节，需要向前查找才能得到当前字符的首字节。

## Java中编码场景

### I/O操作中的编码

涉及编码的地方一般都在字符到字节或者字节到字符的转换上，而需要这种转换的场景主要是I/O，包含磁盘I/O和网络I/O,网络I/O后面再叙述。

Reader类是Java的I/O中读写字符的父类，而InputStream类是读写字节的父类，InputStreamReader类就是字节到字符的桥梁，它负责在I/O过程中处理读取字节到字符的转换，在解码过程中必须由用户指定Charset编码格式。如果没有指定，将使用本地环境中的默认字符集，如中文环境中将使用GBK编码。

```
FileOutputStream fos = new FileOutputStream("c:/stream.txt");
OutputStreamWriter writer = new OutputStreamWriter(fos,"UTF-8");
writer.write("保存的中文字符串");
```

### 内存中的编码

Java中用String表示字符串，所以String类提供了转换到字节的方法，也支持将字节转换成字符串的构造函数。

```
String s = "中文字符串";
byte[] b = s.getBytes("UTF-8");
String n = new String(b,"UTF-8");
```

Charset类也提供char[]和byte[]的互转功能。

```
Charset charset = Charset.forName("UTF-8");
ByteBuffer byteBuffer = charset.encode(string);
CharBuffer charBuffer = charset.decode(byteBuffer);
```

### 编码格式比较

- GB2312与GBK编码规则类似，但是GBK范围更大，所以GB2312和GBK比较，应选择GBK。

- UTF-16与UTF-8都是处理Unicode编码，它们的编码规则不太相同，相对来说UTF-16编码效率高，字符到字节互相转换更简单，进行字符串操作也更好。它适合在本地磁盘和内存之间使用，可以进行字符和字节之间的快速转换，如Java的内存编码就采用UTF-16编码。

- 但UTF-16不适合在网络之间传输，因为网络容易损坏字节流，一旦字节流损坏将很难恢复，相比较而言UTF-8更适合网络传输。UTF-8在编码效率和编码安全性上做了平衡，是理想的中文编码方式。

## Java Web中的编解码

对于使用中文来说，有I/O的地方就会涉及编码，前面已经提到了I/O操作会引起编码，而大部分I/O引起的乱码都是网络I/O。因为现在几乎所有的应用程序都涉及网络操作，而数据经过网络传输都是以字节为单位的。

### URL的编解码

```
http://localhost:8080/examples/servlets/servlet/测试?author=香吉士
```
对于上面的URL，在火狐浏览器中测试结果是：

- servlet路径，即“测试”是UTF-8编码。
- 参数，即“香吉士”是GBK编码。

不同的浏览器对servlet路径的编码也可能不一样，这对服务器的解码造成很大困难。在Tomcat中，对URL的URI部分进行解码的字符集是在connector的`<Connector URIEncoding="UTF-8">`中定义，如果没有定义，那么将以默认编码ISO-8859-1解析。

那么参数是根据什么解码？GET方式HTTP请求的参数与POST方式HTTP请求的参数都是作为Parameters保存的，都通过request.getParameter获取参数值。对它们的解码是在`request.getParameter`方法第一次被调用时进行的。request.getParameter方法被调用时将会调用org.apache.catalina.connnector.Request的parseParameters方法。这个方法将会对GET和POST方式传递的参数进行解码，但是它们的解码字符集有可能不一样。

GET方式的参数解码字符集要么是在Header中ContentType定义的Charset，要么就是默认的ISO-8859-1,要使用ContentType中定义的编码就要将connector的`<Connector URIEncoding="UTF-8" useBodyEncodingForURI="true">`中的useBodyEncodingForURI设置为true。

从上面的URL编码和解码过程来看，比较复杂，而且编码和解码并不是我们在应用程序中能完全控制，所以在应用程序中应该尽量避免在URL中使用非ASCII字符。

### HTTP Header的编解码

客户端发起一个HTTP请求时，除了上面的URL外还可能会在Header中传递其它参数，如Cookie，redirectPath等，这些值很可能会存在编码问题。不要在Header中传递非ASCII字符，如果一定要传递，可以先将这些字符用org.apache.catalina.util.URLEncoder编码，然后再添加到Header中。

### POST表单的编解码

前面提到POST表单提交的参数的解码是在第一次调用request.getParameter时发生的，POST表单参数传递方式和GET不同，它是通过HTTP的BODY传递到服务端的。

浏览器会根据ContentType的Charset编码格式对表单填写的参数进行编码，然后提交到服务器，在服务器端同样也是用ContentType中的字符集进行解码的。所以通过POST表单提交的参数一般不会出现问题，而且这个字符集编码是我们自己设置的，可以通过request.setCharacterEncoding(charset)来设置。

**注意：**一定要在第一次调用request.getParameter方法之前就设置request.setCharacterEncoding(charset)，否则你的POST表单提交上来的数据也可能出现乱码。

Tomcat在解析Parameter参数集合之前会获取Header的content-type请求头，并且检查这个content-type中的charset值，在默认情况下浏览器在提交form表单时，提交的content-type是不会含有charset信息的。所以如果没有设置request.setCharacterEncoding(charset)，那么表单提交的数据将会按照系统的默认编码方式解析。

### HTTP BODY的编解码

通过Response返回给客户端浏览器的内容会先经过编码再到浏览器进行解码，编解码字符集可以通过response.setCharacterEncoding(charset)来设置，它将覆盖request.getCharacterEncoding的值，并且通过Header的Content-Type返回客户端。如果没有设置，那么浏览器将根据HTML的`<meta HTTP-equiv="Content-Type"> content="text/html;charset=GBK"`中的charset来解码。

## JS中的编码问题

### encodeURI()

该函数可以将整个URL中的字符（除了一些特殊字符，如!#$'()*+,-./:;=?@_~0-9a-zA-Z）进行UTF-8编码，在每个码值前加上“%”。

### encodeURIComponent()

该函数比encodeURI()编码还要彻底，它除了对!'()*-._~0-9a-zA-Z这几个字符不编码外，其它所以字符都编码，这个函数通常用于将一个URL当做一个参数放在另一个URL中。

### Java与JS编解码问题

JS进行的编码，在服务器端Java怎么解码？

Java端处理URL编解码有两个类，分别是java.net.URLEnocder和java.net.URLDecoder。这两个类可以将所有“%”加UTF-8解码，从而得到原始的字符。

Java端的这两个类与前端JS对应的是encodeURIComponent和decodeURIComponent。如果前端用encodeURIComponent编码后，到服务器端用URLDecoder解码可能会出现乱码，一定是两个字符编码类型不一致导致的，JS编码默认是UTF-8，服务器端可能是GBK或者GB2312，所以会出现乱码。解决办法是用encodeURIComponent两次编码，这样在Java端通过request.getParameter()用GBK解码后取得的就是UTF-8编码的字符串，再用UTF-8解码一次即可。

## 总结

- 几种常见的编码格式的区别和使用场景
- Java中涉及编码问题的地方
- 重点是HTTP请求中存在编码的地方

要解决编码问题，首先要搞清楚哪些地方会银企字符到字节的编码，以及字节到字符的编码，最常见地方就是存储数据到磁盘或者数据要经过网络传输。然后针对这些地方搞清楚操作这些数据的框架或系统是如何控制编码的。最后正确设置编码格式，避免使用默认的编码格式。
