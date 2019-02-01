title: 从“请求-响应”问题出发看一下SpringMVC的架构（上）
categories: Spring
date: 2015-11-11 10:58:18
tags: SpringMVC
description: SpringMVC spring架构 SpringMVC原理
---

今天看到一句话令我印象深刻，大概意思是公司招聘程序员只是为了**“增加收入，降低支出”**，很具有现实意义。我们为什么需要程序，就是为了降低人力成本，增加工作效率。因此我们在应聘的过程中应该强调自己的代码给公司带来了多少价值，这才是根本。

好吧，上面这段话跟本文没太大关系。提醒了我一下做事情还是得清楚根本的出发点，而用框架要理解框架出现的原因。最近看了点书，就来闲聊一下SpringMVC的架构，大概地弄清楚SpringMVC出现的原因和如何解决它所解决的问题。

```
web开发的根本问题就是**请求-响应**的问题
```

人与人的沟通，简单来说是：我说一句，你回答一句，或者你说一句，我回答一句。人与机器的沟通，简单说来就是：我点一个链接，机器返回了一些内容给我。实际上发生的动作就是浏览器发送一个请求到服务器，服务器返回数据给浏览器呈现，这就是一个请求响应问题。

人与人之间的沟通通过语言，只有共同的语言才能顺利进行沟通。那么浏览器和服务器之间的沟通是基于什么？很显然是http协议。因此web开发中的根本问题可以理解为：
**发起方通过http协议发送请求内容到接收方，接收方收到请求之后经过逻辑处理再通过http协议发送响应内容到接收方**。

那么，这一个过程，用java语言是怎么实现的呢？
<!-- more -->
答案就是servlet技术，相信大家在学习java web开发的时候最初接触到的技术应该就是servlet了。

```
Spring的MVC框架都是基于servlet技术来实现
```

我们很清楚地知道，servlet的核心处理方法就是service方法：

```
    public void service(ServletRequest req, ServletResponse res)
        throws ServletException, IOException
```
这个方法就是用java技术解决了**请求-响应**问题，ServletRequest参数包含了请求发起方的请求内容，service通过逻辑处理之后通过ServletResponse参数返回响应内容到请求发起方。

也许大家会觉得奇怪，为什么这个方法的返回参数是void，为什么它要通过ServletResponse参数来返回内容给请求发起方？因为servlet技术是用java技术解决**请求-响应**问题的最底层封装，已经是最底层了，就不能再把返回参数传递出去给别处处理了，所以只好设计成直接通过参数把内容返回给请求发起方了。

还记得初学servlet技术的时候，我们为每一个servlet都在xml文件中配置一个映射的辛酸往事吗？很显然，servlet技术作为最底层的封装，需要我们编码的地方就很多了。我们伟大的程序员不断地对servlet技术进行封装，才有了struct1.x，struct2.x和我们今天闲聊的springmvc等MVC框架。


```
SpringMVC就是通过DispatcherServlet将一堆组件串联起来的Web框架。
```

终于进入了今天的正题了，SpringMVC究竟做了什么封装使我们能够更简单地编码解决**请求-响应**问题？既然上面提到了直接用servlet开发服务端程序，需要在xml文件中为每个servlet都配置一个映射。那么SpringMVC作为一个优秀的mvc框架，必然不需要这么麻烦，SpringMVC通过DispatcherServlet作为一个全局的servlet分发器，将不同的请求分发给不同的处理器（controller）来处理，因此DispatcherServlet是SpringMVC的直接入口，所以我们会在web.xml中配置这样一段代码，就是指定了SpringMVC的入口。
```
	<servlet>
		<servlet-name>SpringMVC</servlet-name>
		<servlet-class>org.springframework.web.servlet.DispatcherServlet</servlet-class>
		<init-param>
			<param-name>contextConfigLocation</param-name>
			<param-value>classpath:spring-mvc.xml</param-value>
		</init-param>
		<load-on-startup>1</load-on-startup>
		<async-supported>true</async-supported>
	</servlet>
	<servlet-mapping>
		<servlet-name>SpringMVC</servlet-name>
		<url-pattern>*.do</url-pattern>
	</servlet-mapping>
```

除了DispatcherServlet之外，SpringMVC还做了什么改造？既然DispatcherServlet将请求都分发到相应的处理器处理，那么我们就先看一个处理器（controller）的例子，看它对解决**请求-响应**问题做了什么改造：

```
@Controller
public class LoginController{

	@RequestMapping("/login.do")
	public ModelAndView login(@RequestParam String username,@RequestParam String password){
		...
	}

}	
```
可以看出一下几点：

- **参数和返回值不一样了**
	servlet的参数是原生的HttpServletRequest和HttpServletResponse，SpringMVC不用直接操作原生对象了，可以直接获得我们想要的参数了。
	servlet是没有返回值的，原因已经说过；SpringMVC有返回值了，这也是必然需要的，因为DispatcherServlet作为一个分发器，处理器处理完当然要将处理结果返回给DispatcherServlet，DispatcherServlet再将响应通过HttpServletResponse返回给请求发起者。

- **引入Annotation来完成请求-响应的映射关系**
	引入Annotation来完成请求-响应的映射关系，是SpringMVC的一个重大改造,简化了配置。

- **泛化参数和返回值的含义**
	例子中只看到了ModelAndView类型的返回值和@RequestParam这样的请求参数，实际上SpringMVC支持多种请求参数和返回值类型，这在后面再具体看看，心急的可以先看一下SpringMVC的官方文档。
	
从上面这几点可以看出，SpringMVC对servlet的改良并不是面目全非，而是根没变，只是将根包装的更加方便我们使用罢了。由于文章篇幅过长决定分成两篇发布，今天就先写到这里。

## 小结
我们知道了web开发的根本问题就是“请求-响应”问题，而java在这个问题给出的解决方案就是servlet技术。由于servlet技术比较底层，在中大型的系统中编码起来就会相对复杂，因此程序员们就为了简化开发研究出了各种MVC框架，而SpringMVC就是其中的佼佼者。SpringMVC通过一个DispatcherServlet来作为一个全局http的分发器，将请求分发到相应的controller来处理。除此之外还进行了一下其它改良，比如引入Annotation和泛化参数和返回值的含义等。
