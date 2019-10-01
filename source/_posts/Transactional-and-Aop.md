---
title: Transactional 注解与 AOP
date: 2019-08-29 20:18:49
tags:
---

当我们在一个 private 方法上打上 @Transactional 注解，IDEA 会提示 `Methods annotated with '@Transactional' must be overridable`。比如下面的例子：

```
@Service
public class UserService {

    public void method1(){
        method2();
    }

    @Transactional
    private void method2(){

    }
}
```

有时候我们想让事务的范围尽可能的小，可能就会写出这样的代码，在 method1 先做一些非事务的事情，然后在 method2 中做事务相关的事情。此时，如果我们用 IDEA 的话，应该就会出现上面的提示信息，eclipse 我没用不清楚会不会有这样的其实。为什么被 @Transactional 注解的方法必须是可重写的呢？

如果我们忽略掉这个信息，你会发现编译运行是没有问题的。在外部调用 UserService.method1() 时，如果 method2 没有发生异常的话，你会发现一切都是正常的。为什么 IDEA 要这样提示呢，真是怪了！

但是，当 method2 发生异常时，你会惊奇的发现 method2 的事务并**没有回滚！**

### AOP

要理解事务为什么没有回滚，我们就要回顾一下 AOP 的知识。在 Spring 中默认是通过 JDK 动态代理的方式来实现 AOP 的，对于打了 @Transactional 注解的类，Spring 动态代理会生成一个代理 bean 和一个真实的目标 bean。

我们回头看看 method1 并没有打上注解，所以 method1 并不会被事务切面环绕。而 method2 是通过 method1 调用的，隐藏的调用对象是真实的目标 bean，真实的目标 bean 是没有切面逻辑的，切面逻辑都在代理 bean 上。这就是为什么事务没有回滚的原因，如下图所示：

![transactional-aop](https://raw.githubusercontent.com/rason/rason.github.io/master/image/transactional-aop.gif)

按照这样的理解，只要我们在 method1 方法上打上 @Transactional 注解，事务就能生效了，method2 的注解是多余的。此时，我们应该就能理解 IDEA 的善意提示了。因为 UserService 没有接口，所以只能通过 CGLIB 的方式来实现动态代理，而 CGLIB 是通过继承的方式来进行代理，需要对目标 bean 的方法进行重写，但是 private 修饰的方法是不能重写的，所以就会出现这样的提示。

但是，在 method1 上打注解，method2 上不打，那不是违背了我们当初让事务范围最小化的出发点了吗？一切又回到了原地。还有没有其他解决方法？

1. 把 method1 方法搬到 controller 层，嘿嘿，这个方法看起来很鸡贼，但是好像是更加合理的？
2. 如果代理类和目标类合为一个，有注解的方法有切面环绕，没有注解的方法没有切面环绕，这样是不是就行了？AspectJ 静态代理就是这样的一门技术，在编译成 class 字节码的时候在方法周围织入切面逻辑。

### 总结

不仅仅是 @Transactional 会这样，其他的注解也是同理的，所以我们要深入理解 AOP 的原理，面对这些问题才能游刃有余。


