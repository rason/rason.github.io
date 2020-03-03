---
title: Spring transaction propagation
date: 2019-08-22 14:06:55
tags:
---

有些简单的知识点理解的比较含糊，在使用的时候总是会卡壳，Spring 事务的传播行为就是一个例子。今天再次复习一下。

事务的传播行为指的是两个方法调用之间事务怎么传播的问题。比如，方法A 有事务，调用的方法B也有事务，那么方法B 的事务是直接加入方法A 的事务还是让方法A 的事务挂起呢？当然，Spring 的事务传播行为不仅仅只有这两种情况。

### 事务传播属性

- **PROPAGATION_REQUIRED：** 如果方法A没有事务，就新建一个事务；如果有，就加入当前事务。这是 Spring 提供的默认事务传播行为。
- **PROPAGATION_SUPPORTS：** 如果方法A没有事务，就以非事务方式执行；如果有，就使用当前事务。
- **PROPAGATION_MANDATORY：** 如果方法A没有事务，就抛出异常；如果有，就使用当前事务。
- **PROPAGATION_REQUIRES_NEW：** 如果方法A没有事务，就新建一个事务；如果有，就将当前事务挂起。
- **PROPAGATION_NOT_SUPPORTED：** 如果方法A没有事务，就以非事务方式执行；如果有，就将当前事务挂起。
- **PROPAGATION_NEVER：** 如果方法A没有事务，就以非事务方式执行；如果有，就抛出异常。
- **PROPAGATION_NESTED：** 如果方法A没有事务，就新建一个事务；如果有，就作为当前事务的嵌套事务。

前六个应该都很好理解，就是最后一个传播行为可能会比较难理解，很容易和 PROPAGATION_REQUIRES_NEW 混淆，下面我们来分析一下这两种行为的区别。

### PROPAGATION_NESTED 与 PROPAGATION_REQUIRES_NEW 的区别

PROPAGATION_REQUIRES_NEW 是新开一个事务，是独立的，不会受外部事务的影响。当新开的事务开始执行时，外部事务会被挂起，内部事务结束了，外部事务继续执行。
![PROPAGATION_REQUIRES_NEW](/image/transaction-propagation-1.png)

PROPAGATION_NESTED 是一个 "嵌套的" 事务，它是已经存在事务的一个真正的子事务。嵌套事务开始执行时，它将取得一个 savepoint。如果这个嵌套事务失败，我们将回滚到此 savepoint。本质上，外部事务和嵌套事务属于同一个物理事务，使用保存点实现嵌套事务。嵌套事务和它的父事务是相依的，它的提交要和它的父事务一起。也就是说，如果父事务最后回滚，它也要回滚。如果子事务回滚或提交，不会导致父事务回滚或提交，但父事务回滚将导致子事务回滚。
![PROPAGATION_NESTED](/image/transaction-propagation-2.png)

### PROPAGATION_NESTED 用法示例

```
ServiceA {  
      
    /** 
     * 事务属性配置为 PROPAGATION_REQUIRED 
     */  
    void methodA() {  
        try {  
            ServiceB.methodB();  
        } catch (SomeException) {  
            // 执行其他业务, 如 ServiceC.methodC();  
        }  
    }  
  
}  
```

这种方式也是嵌套事务最有价值的地方，它起到了分支执行的效果, 如果 ServiceB.methodB 失败, 那么执行 ServiceC.methodC(), 而 ServiceB.methodB 已经回滚到它执行之前的 SavePoint, 所以不会产生脏数据(相当于此方法从未执行过), 这种特性可以用在某些特殊的业务中, 而 PROPAGATION_REQUIRED 和 PROPAGATION_REQUIRES_NEW 都没有办法做到这一点。

