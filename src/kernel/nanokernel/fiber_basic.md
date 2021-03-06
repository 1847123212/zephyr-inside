---
title: Zephys OS nano 内核篇：fiber 服务 - 基础
date: 2016-09-21 23:23:32
categories: ["Zephyr OS"]
tags: [Zephyr]
---

本文是关于 fiber 的纯理论的部分，先熟悉熟悉这些理论有助于理解后面的代码。当学习了后面的代码后，你可以再回过头来看看这些理论，你将有新的收获。

怎样才能算理解了这些概念？当你看到下面的每句话的时候，脑海里能复现出其对应的代码时，你就是真的理解了！

- [fiber 的概念](#fiber-的概念)
- [fiber 的生命周期](#fiber-的生命周期)
- [fiber 的调度](#fiber-的调度)
  - [fiber 的状态](#fiber-的状态)
  - [fiber 的优先级](#fiber-的优先级)
  - [fiber 的调度算法](#fiber-的调度算法)
  - [时间片](#时间片)

<!--more-->

# fiber 的概念
fiber 是一个轻量级的、**不可抢占**的线程。fiber 主要用于处理设备驱动程序和一些对性能比较挑剔的工作。

一个应用程序可以使用多个 fiber。fiber 是匿名的，当它开始执行后，不能被其它任何 fiber 和 task 引用。创建 fiber 时，必须指定以下属性：
- 一段内存区域：用做 fiber 的栈，并保存上下文相关的信息。
- 一个入口函数：当 fiber 开始执行时需要调用的函数。
- 参数：传递给入口函数的参数。
- 优先级：该 fiber 的优先级。内核调度器按照优先级顺序调用 fiber。
- 选项：fiber 中需要使用的选项。

在系统启动进行初始化时，内核可能会自动创建零个或多个 fiber，这依赖于：
1. 应用程序所配置的内核的功能
2. 编译应用程序镜像时的板级配置

# fiber 的生命周期

fiber 可以被另一个 fiber，或 task，或进行系统初始化时的内核创建。通常，fiber 在创建后将立即成为可执行状态(被加入就绪链表中)，但是我们也可以根据需要调用指定的接口让其等待一段指定的时间后再变为可执行状态(即 fiber 的延迟启动功能)。例如，当 fiber 需要使用设备时，可以等待设备准备就绪后再被调度。内核也支持取消延迟启动的 fiber，即当延迟期限到达前，发现该 fiber 不再需要了，直接将其删除。

fiber 一旦被启动了，它一般会永远执行下去。以下两种方法可以**终止**一个 fiber：
- 优雅地终止：fiber 自己从它的入口函数中返回(return)。在这种情况下，fiber 需要自己释放它所拥有的系统资源(比如信号量)。
- 非优雅地终止：
  - 被内核终止(aborting)：当内核产生致命错误(例如使用了一个空指针 NULL)时，会自动终止 fiber。
  - 被 fiber_abort() 函数终止。

> 注意，这里所说的 *fiber 的终止*与后面所说的* fiber 放弃 CPU *是两个不同个概念，请注意区分
# fiber 的调度
nanokernel 的调度器负责选择某个线程运行，被选中的线程叫做“当前上下文(current context)”。由于 isr 的优先级比 fiber 和 task 的优先级高，因此只有在没有 isr 需要执行时，调度器才会选择某个线程来运行。

由于 fiber 的优先级比 task 的优先级高，且 task 是可抢占式的，因此如果当前的上下文是 task，那么当有 fiber 需要执行时，调度器将会抢占 task 的执行。由于 fiber 是不可抢占的，因此调度器永远不会抢占 fiber 的执行(即使存在优先级更高的 fiber)，除非它被终止或主动释放 CPU。
## fiber 的状态
fiber 有一个隐含的状态，用以判断 fiber 是否可被调度执行。这个状态记录了 fiber 不能被执行的原因：
- fiber 还没被创建
- fiber 正在等待某个内核服务，例如信号量或定时器
- fiber 已被终止了

如果 fiber 不包含上述状态，我们就说 fiber 是可执行的，它将被放入 fiber 就绪链表中去，等待被调度器调度执行。

## fiber 的优先级
fiber 支持的优先级的数量非常多，其范围由 0(最高优先级) 到 2^32-1(最低优先级)。fiber 的优先级不能为负数。

虽然在创建的时候就指定了优先级，但是 fiber 的优先级可以在其被创建后依然可以修改。
## fiber 的调度算法
nanokernel 的调度器总是选择优先级最高的 fiber 去执行。当 fiber 的就绪链表中同时存在多个优先级系统的 fiber 时，它会调度等待时间最久的 fiber 去执行。

当不存在可执行的 fiber (即 fiber 的就绪链表为空)时，调度器将选择一个  task 来执行。task 的选择依赖于应用程序是 nanokernel 应用程序还是 microkernel 应用程序。如果是 nanokernel 应用程序，调度器总是选择后台 task 来执行。如果是 microkernel 应用程序，将由 microkernel 的调度器选择执行哪个 task。

一个 fiber 一旦成为“当前上下文”后，它将一直执行，除非发生下列情况：
- 它调用了一个内核的 API 将自己阻塞了(例如它尝试获取一个无效的信号量)
- 从入口函数中返回了。
- 它执行了某个操作，导致了一个致命的错误，或调用了函数 fiber_abort()。

## 时间片
由于 fiber 具有是不可抢占的特性，为了避免某个 fiber 长时间占用 CPU，fiber 可以自愿放弃 CPU，将机会让给其它 fiber。

fiber **放弃 CPU **有两种方法：
- 调用 fiber_yield()，将本 fiber 放回 nanokernel 调度器的可执行 fiber 链表中，让所有高于或等于该 fiber 优先级的其它 fiber 先执行。如果没有高于或等于该 fiber 优先级的 fiber，那么该 fiber 继续执行，不用进行上下文切换。
- 调用 fiber_sleep()，将本 fiber 阻塞一段指定的时间，让可执行 fiber 链表中所有优先级的 fiber 都有可能得到执行。