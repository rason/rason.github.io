title: 'Facade&Observer'
categories: "DesignPattern"
date: 2016-01-11 16:21:38
tags: Facade Observer
description: Facade&Observer DesignPattern
---

## 前言

以前为了面试找工作，会专门地去学习一下设计模式。但长时间不注意去应用，以致现在很多模式只知道个名字，所以说，要学以致用才行。接着上篇文章，继续概括一下另外两种设计模式：门面模式（Facade）和观察者模式（Observer）。

# 门面模式（Facade）

门面模式，就是将一个东西封装成一个门面好与人家更加容易地进行交流。主要用在一个很大的系统中有多个子系统时，多个子系统肯定要互相通信，但是每个子系统又不能将自己的内部数据过多地暴露给其他系统。每个子系统都会设计一个门面，把别的系统感兴趣的数据封装起来，通过这个门面来进行访问。门面设计模式示意图：

![门面设计模式](https://raw.githubusercontent.com/rason/rason.github.io/master/image/designpatternfacade.png)

Client只能访问Facade中提供的数据是门面设计模式的关键，至于Client如何访问Facade和SubSystem，如何提供Facade门面设计模式并没有规定死。

在Tomcat中，有很多地方都运用了Facade设计模式。比如Request和Response的封装，StandWrapper到ServletConfig的封装，ApplicationContext到ServletContext的封装等。

<!-- more -->

举个例子，在Tomcat的StandardWrapper中，会创建一个facade对象用于Servlet的初始化：

```
//this为StandardWrapper实例
StandardWrapperFacade facade = new StandardWrapperFacade(this);
```

```
public final class StandardWrapperFacade
    implements ServletConfig {
    public StandardWrapperFacade(StandardWrapper config) {

        super();
        this.config = config;

    }
    private final ServletConfig config;
    private ServletContext context = null;
    @Override
    public String getServletName() {
        return config.getServletName();
    }
    @Override
    public ServletContext getServletContext() {
        if (context == null) {
            context = config.getServletContext();
            if ((context != null) && (context instanceof ApplicationContext))
                context = ((ApplicationContext) context).getFacade();
        }
        return (context);
    }
    @Override
    public String getInitParameter(String name) {
        return config.getInitParameter(name);
    }
    @Override
    public Enumeration<String> getInitParameterNames() {
        return config.getInitParameterNames();
    }


}

```

我们知道StandardWrapper本身具有很多属性和方法，为此封装了一个StandardWrapperFacade类来暴露一些Servlet需要的数据，而自身另外的数据并没有暴露出来，达到了隔离数据的效果。同样的例子可以在Request和Response，ApplicationContext到ServletContext的封装中查看到，此处不一一举例了。

## 观察者模式（Observer）

观察者模式也就发布-订阅模式，也就是事件监听机制，通常在发生某个事情前后会触发一些操作。具体来说就是你在做事的时候有人在盯着你，当你在做的事情是它感兴趣的事，它就会跟着做另外一些事情。但是盯着你的人必须到你那里注册，否则你无法通知它。因此，通常观察者模式会有一下几个角色：

- **Subject抽象主题**：它主责管理所有观察者的引用，同时定义主要的实际操作。
- **ConcreteSubject具体主题**：它实现了抽象主题的所有接口，当自己发生变化时，会通知所有观察者。
- **Observer观察者**：监听主题发生变化的操作接口。

在Tomcat中，这一模式最经典的应用就是控制组件生命周期的Lifecycle。在之前分析Tomcat源码的时候，我们知道Server，Engine，Host，Context，Wrapper这些组件都实现了Lifecycle接口，当这些组件的生命周期变化时，会通知相应的监听者进行一些操作。举例来说：

```
public interface Lifecycle {
	public static final String BEFORE_INIT_EVENT = "before_init";
	public static final String AFTER_INIT_EVENT = "after_init";
	public static final String CONFIGURE_START_EVENT = "configure_start";
	...
	public void addLifecycleListener(LifecycleListener listener);
	public LifecycleListener[] findLifecycleListeners();
	public void removeLifecycleListener(LifecycleListener listener);
	public void init() throws LifecycleException;
	public void start() throws LifecycleException;
	public void stop() throws LifecycleException;
	public void destroy() throws LifecycleException;
	...
}

public final class StandardServer extends LifecycleMBeanBase implements Server {
    public StandardServer() {

        super();

        globalNamingResources = new NamingResourcesImpl();
        globalNamingResources.setContainer(this);

        if (isUseNaming()) {
            namingContextListener = new NamingContextListener();
            addLifecycleListener(namingContextListener);
        } else {
            namingContextListener = null;
        }

    }

    protected void startInternal() throws LifecycleException {

        fireLifecycleEvent(CONFIGURE_START_EVENT, null);
        setState(LifecycleState.STARTING);
    	...
    }
}
```

- 这里的Lifecycle就是Subject抽象主题，可以看出它有addLifecycleListener等方法来管理观察者，同时还定义了BEFORE_INIT_EVENT等实际操作。
- StandardServer就是ConcreteSubject具体主题，实现了Lifecycle抽象主题的所有接口（这里是间接实现，可自行查看Tomcat源码），当自己发生变化时，比如上述的开始方法startInternal，会调用fireLifecycleEvent通知所有的观察者。

那么这里的fireLifecycleEvent是怎么通知所有的观察者的？请看下面代码：

```
public void fireLifecycleEvent(String type, Object data) {
    LifecycleEvent event = new LifecycleEvent(lifecycle, type, data);
    for (LifecycleListener listener : listeners) {
        listener.lifecycleEvent(event);
    }
}

public interface LifecycleListener {

    public void lifecycleEvent(LifecycleEvent event);
}
```

可以看出，fireLifecycleEvent循环通知了每一个观察者，这里的LifecycleListener属于抽象的观察者，这个抽像观察者比较简单，只定义了一个监听主题变化时进行操作的接口。那么具体观察者当然是实现了LifecycleListener接口的类，在StandardServer中就是NamingContextListener类了。

```
public class NamingContextListener
    implements LifecycleListener, ContainerListener, PropertyChangeListener {
	public void lifecycleEvent(LifecycleEvent event) {

        container = event.getLifecycle();

        if (container instanceof Context) {
            namingResources = ((Context) container).getNamingResources();
            logger = log;
            token = ((Context) container).getNamingToken();
        } else if (container instanceof Server) {
            namingResources = ((Server) container).getGlobalNamingResources();
            token = ((Server) container).getNamingToken();
        } else {
            return;
        }

        if (Lifecycle.CONFIGURE_START_EVENT.equals(event.getType())) {
        	...
		}else if (Lifecycle.CONFIGURE_STOP_EVENT.equals(event.getType())) {
			...
		}
		...
}
```

可以看出，这个具体的观察者，会根据主题的不同操作而进行相应的操作，这就是一个观察者模式的典型例子了。

## 总结

- **门面模式**：封装一个或多个门面接口与不同系统交流，隔离不必要的数据。
- **观察者模式**：主题发生变化时，通知观察者进行相应的操作。
