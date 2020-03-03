title: Cookie and Session
categories: Java Web
date: 2015-12-10 10:02:30
tags: Cookie and Session
description: Cookie and Session
---

## 前言

存在即合理，探究一样新东西，我们应当从根源出发。它为何存在，它的存在解决了什么问题？就这篇文章的主题来说，Cookie和Session解决了什么问题？我们知道HTTP协议是一种无状态的协议，也就是说，用户的两次访问，服务器不知道他们是不是同一个人。我们知道是不是同一个人有什么作用？这作用当然是很大的，你不可能动用我账户的钱吧？另外，服务器也不可能让你每次请求都得登录一下吧？还有访问一个模块的时候服务器不可能每次都查询数据库获取你的权限吧？

## Cookie

Cookie的作用正是可以解决上述问题，当用户通过HTTP协议访问服务器的时候，服务器会返回一些Key/Value给客户端浏览器，并对这些数据加上一些限制条件，在条件符合时用户再次访问这个服务器时，数据又被完成地返回服务器。因此，我们可以利用Cookie来记录一些用户的信息，这样就可以达到区分用户的作用了。

<!-- more -->

### Cookie配置项

Cookie是HTTP协议的一个配置项，映射到Java中就是一个Cookie对象，那么Cookie具有哪些可以配置的熟悉？这里只挑几个重点要理解的知道一下就行了：

| 属性         | 作用           | 
| ------------|:-------------:|
| Name-Value  | 键值对，设置Cookie的名字和值，比如记录用户名可以设置为：uname=rason； |
| Expires     | 失效时间，设置到达某个时间点Cookie失效，比如expires=Sat, 09-Jan-2016 02:48:45 GMT;      |
| Domain      | 生成该Cookie的域名，比如domain="rason.me"     |
| Path        | 该Cookie在当前哪个路径生成，比如path=/     |
| Secure      | 如果设置了该值，只有在SSL连接时才会回传Cookie    |


### Java Web中如何使用Cookie

服务器端设置Cookie

```
    protected void doGet(HttpServletRequest req, HttpServletResponse resp)
        throws ServletException, IOException
    {
    	String uname = getCookie(request,"uname");
    	String ulevel = getCookie(request,"ulevel");
    	if(uname == null){
    		response.addCookie(new Cookie("uname","rason"));
    	}
    	if(ulevel == null){
    		response.addCookie(new Cookie("ulevel","1"));
    	}

	}

```

服务器端获取Cookie值：

```
	private static Cookie getCookie(HttpServletRequest request, String name) {
		Cookie cookies[] = request.getCookies();
		if (cookies == null || name == null || name.length() == 0) {
			return null;
		}
		Cookie cookie = null;
		for (int i = 0; i < cookies.length; i++) {
			if (cookies[i].getName().equals(name)) {
				cookie = cookies[i];
				break;
			}
		}
		return cookie;
	}
```

当服务器设置了Cookie返回的时候，我们在浏览器中的Response Headers中可以看到，如图：

![浏览器中Set-Cookie](/image/javaweb-set-cookie.png)

从上图可以看出每设置一个Cookie，Response Headers都有一个Set-Cookie熟悉，这里设置了两个Cookie，因此有两个Set-Cookie。图片是随便打开一个网站截取的，因此跟上述代码的Cookie名字不一样。

### 使用Cookie需要考虑的问题

- **Cookie有数量和大小的限制，不同浏览器各不相同，IE浏览器限制50个，大小4KB左右。因此不能滥用，而且滥用给带宽也会带来压力**
- **Cookie的安全性问题，在浏览器中不但可以查看Cookie，还可以修改Cookie。因此Cookie安全性是存在问题的，不能依赖Cookie记录重要数据或者身份权限识别等操作**

## Session

正因为Cookie会有上面两个问题，Session因此应运而生了。Session简单来说就是一个名为JSESSIONID的Cookie，其值就是服务器生成的唯一ID，这样一来，上面两个问题就解决了。

首先，客户端只保存一个Cookie就行了，这样Cookie的数量大小和传输量就不存在问题了。服务器根据Cookie传过来的JSESSIONID获取到唯一的Session对象，该对象就可以保存一些用户的信息了，解决了Cookie解决的问题。
其次，Session是保存在服务器中的，所以浏览器不能修改，安全性得到了一定的保证。至于JSESSIONID被窃取的问题这里就不展开讨论了，自行搜索吧。

如果浏览器不支持Cookie会怎么样？浏览器会将JSESSIONID作为Path parameters写到请求URL中，这样服务器就能拿到JSESSIONID了。

### Session工作原理

当我们调用`HttpSession session = request.getSession();`时，如果请求中包含了requestedSessionId，则会根据这个requestedSessionId查找出这个Session。如果请求中没有requestedSessionId，则服务器会创建一个新的Session，并将其添加到`org.apache.catalina.Manager`中，该接口的作用就是管理应用的Session生命周期，过期回收，服务器关系，Session序列化到磁盘等。

这里贴一段Tomcat8中获取Session的源码，注意留意英文的注释，很好地解析了这一流程。

```
protected Session doGetSession(boolean create) {

        // There cannot be a session if no context has been assigned yet
        Context context = getContext();
        if (context == null) {
            return (null);
        }

        // Return the current session if it exists and is valid
        if ((session != null) && !session.isValid()) {
            session = null;
        }
        if (session != null) {
            return (session);
        }

        // Return the requested session if it exists and is valid
        Manager manager = context.getManager();
        if (manager == null) {
            return (null);      // Sessions are not supported
        }
        if (requestedSessionId != null) {
            try {
                session = manager.findSession(requestedSessionId);
            } catch (IOException e) {
                session = null;
            }
            if ((session != null) && !session.isValid()) {
                session = null;
            }
            if (session != null) {
                session.access();
                return (session);
            }
        }

        // Create a new session if requested and the response is not committed
        if (!create) {
            return (null);
        }
        if (response != null
                && context.getServletContext()
                        .getEffectiveSessionTrackingModes()
                        .contains(SessionTrackingMode.COOKIE)
                && response.getResponse().isCommitted()) {
            throw new IllegalStateException(
                    sm.getString("coyoteRequest.sessionCreateCommitted"));
        }

        // Attempt to reuse session id if one was submitted in a cookie
        // Do not reuse the session id if it is from a URL, to prevent possible
        // phishing attacks
        // Use the SSL session ID if one is present.
        if (("/".equals(context.getSessionCookiePath())
                && isRequestedSessionIdFromCookie()) || requestedSessionSSL ) {
            session = manager.createSession(getRequestedSessionId());
        } else {
            session = manager.createSession(null);
        }

        // Creating a new session cookie based on that session
        if (session != null
                && context.getServletContext()
                        .getEffectiveSessionTrackingModes()
                        .contains(SessionTrackingMode.COOKIE)) {
            Cookie cookie =
                ApplicationSessionCookieConfig.createSessionCookie(
                        context, session.getIdInternal(), isSecure());

            response.addSessionCookieInternal(cookie);
        }

        if (session == null) {
            return null;
        }

        session.access();
        return session;
    }
```

这里的重点就是刚才提到的Manager接口，该接口在Tomcat的标准实现是StandardManager，负责管理Servlet容器中的Session对象。当容器重启或者关闭时，StandardManager负责持久化没有过期的Session到一个“SESSION.ser”文件中。当容器重启时，会重新解析该文件恢复Session。但是当我们用kill线程的方法来终止容器的工作，这样是无法持久化Session的。

另外，StandardManager中管理的Session是有有效时间的，否则太多的Session会导致容器的内存不足。在Tomcat中默认的有效实际是60秒，当然我们也可以调用HttpSession的`setMaxInactiveInterval`方法来修改这个时间。

Tomcat中有一个后台线程会调用session.isValid()方法检查Session是否过期，我们从request.getSession()方法中也能检测出Session是否过期，当我们获取到了Session，但是Session里面设置的一些属性值没有了，那就是过期了。getSession()方法获取不到Session会创建一个新的Session，如果不想创建新的Session可以调用 getSession(boolean create)方法。

## 总结

通过这篇文章应该可以了解到下面几点：

- **Cookie和Session解决了什么问题**
- **Cookie如何使用和存在的问题**
- **Session的工作原理**
