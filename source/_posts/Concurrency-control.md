---
title: 并发控制
date: 2017-09-13 09:47:35
categories: Concurrency
tags: Concurrency
description: Concurrency control
---

在信息技术和计算机科学领域，特别是在计算机编程，操作系统，多处理器和数据库领域，并发控制可以**确保并发操作的正确结果**。

计算机系统，软件和硬件都由模块或组件组成。每个组件被设计为正确地操作，即，服从或满足某些一致性规则。当组件通过**消息传递**或**共享访问的数据**（在存储器或存储器中）同时进行操作时，某个组件的一致性可能被另一个组件破坏。

并发控制领域提供规则，方法，设计方法和理论，以保持组件在交互时并发运行的一致性，从而保持整个系统的一致性和正确性。在系统中引入并发控制意味着对一些操作进行了限制，这通常会导致某些性能上的下降。操作一致性和正确性应尽可能高效地实现，而不会将性能降低到合理的水平以下。与简单的顺序算法相比，并发控制可能需要大量额外的复杂性和并发算法的开销。

## 数据库中的并发控制

数据库管理系统，其他事务对象以及相关的分布式应用程序中的并发控制可以确保数据库事务同时执行，而不会违反各数据库的数据完整性。因此，在两个数据库事务或多个数据库事务或多个数据库事务随时间重叠执行的任何系统中，并发控制是正确性的基本要素，可以访问相同的数据。前人提出了一个完善的数据库系统并发控制理论：[序列化理论](https://en.wikipedia.org/wiki/Serializability)，可以有效地设计和分析并发控制方法和机制。

为了确保正确性，DBMS通常保证只生成[可串行化](https://en.wikipedia.org/wiki/Serializability)的事务计划，除非有意放宽序列化以提高性能，而且仅在应用程序正确性不受损害的情况下。为了在事务调度失败（中止）的情况下维持正确性还需要具有[可恢复](https://en.wikipedia.org/wiki/Serializability#Correctness_-_recoverability)（从中止）属性。DBMS还保证不会丢失提交的事务的影响，并且相关数据库中没有受中止（回滚）事务的影响。

## 数据库事务与ACID原则

数据库事务是工作单元，通常在数据库中封装多个操作（例如，读取数据库对象，写入，获取锁定等）。每个数据库事务都遵循以下规则：

- **原子性**：事务中的每一项操作是一个整体，要么全部完成，要么全部放弃。

- **一致性**：每个事务必须使数据库保持一致（正确）状态，即维护数据库的预定完整性规则（对数据库对象之间的约束）。事务必须将数据库从一个一致的状态转变为另一个一致状态（然而，程序员有责任确保事务本身是正确的，即正确地执行它打算执行的操作（从应用程序的角度来看视图），而预定义的完整性规则由DBMS执行）。

- **隔离性**：事务之间不能相互干扰（作为其执行的最终结果）。此外，通常（根据并发控制方法），未完成的事务的影响对于另一个事务来说甚至不可见。 **提供隔离性是并发控制的主要目标**。

- **持久性**：成功（提交）事务的影响是持久的。

## 为什么需要并发控制？

如果事务按顺序执行，即顺序地在时间上不重叠，则不存在事务并发性。然而，如果以不受控制的方式允许具有交织操作的并发事务，则可能会发生一些意外的不期望的结果，例如：

- 更新丢失问题：第二个事务在由第一个并发事务写入的第一个值之上写入数据项（数据）的第二个值。
- 脏读问题：事务读取由稍后回滚的事务写入的值。
- 不正确的摘要问题：当一个事务对重复数据项的所有实例的值进行摘要总结，但第二个事务更新了该数据项的某些实例。

大多数高性能事务系统需要并发运行事务以满足其性能要求。因此，没有并发控制，这样的系统既不能提供正确的结果也不能保持数据库的一致性。

## 并发控制机制

### 分类

并发控制机制主要分类如下：

- 乐观并发控制：延迟检查事务是否满足隔离和其他完整性规则（例如，可串行化和可恢复性），直到其结束，而不会阻止其任何（读取和写入）操作。如果规则在其提交时被违反，则中止事务。中断的事务立即重新启动并重新执行，这导致了明显的开销（而不是执行到最后一次）。如果不是太多的事务被中止，那么乐观通常是一个很好的策略。

- 悲观并发控制：阻塞可能会破坏完整性等规则的事务。阻塞操作通常会带来性能上的下降。

- 半乐观并发控制：一些情况下阻塞，一些情况下不阻塞。

不同的类别提供不同的性能，即不同的平均事务完成率（吞吐量），这取决于事务类型的组合，并行度的计算水平和其他因素。阻塞，死锁和中止都会导致性能下降，从而导致类别之间的权衡。

### 方法

并发控制有许多方法。它们中的大多数可以在上述主要类别中实现。主要的方法，有各种各样的变体，在某些情况下可能重叠或组合，它们是：

- 1.锁（例如[二阶段锁](https://en.wikipedia.org/wiki/Two-phase_locking)）：在数据上加锁控制其访问。

- 2.[优先图检测](https://en.wikipedia.org/wiki/Serializability#Testing_conflict_serializability)：检查调度图是否形成一个回路，可以通过中止某个事务来中止。

- 3.[时间戳顺序](https://en.wikipedia.org/wiki/Timestamp-based_concurrency_control)：在事务上标记时间戳，然后通过时间戳先后顺序来控制或者检查数据访问。

- 4.[提交顺序](https://en.wikipedia.org/wiki/Commitment_ordering)：控制或检查事务提交的时间顺序与它们各自的优先级一致。

与上述方法一起使用的其他主要并发控制类型包括：

- [多版本并发控制](https://en.wikipedia.org/wiki/Multiversion_concurrency_control)（MVCC）:通过每次写对象时生成一个数据库对象的新版本来增加并发性和性能，并根据调度方法允许对几个最后相关版本（每个对象）的事务读操作。

- [索引并发控制](https://en.wikipedia.org/wiki/Index_locking)
