# # JavaScript并发模型与Event Loop

## 简要描述

javascript在主线程运行时，会产生**栈（stack）**和**堆（heap）**，程序中的代码会进入栈中等待执行，而程序中的对象会被分配在一个堆中。若代码执行中遇上异步方法，该异步方法就会被添加到用于回调的**队列（queue）**中。整个模型结构如下图所示：

![](模型结构.png)

这里要先说明一下，所有任务都要在**执行栈（stack）****里执行，不管是同步任务还是异步任务，只不过同步任务现进入执行栈，而异步任务要在队列里等待执行栈的调用。那么执行栈什么时候才会去读取任务**队列（queue）**呢，很显然只有当**执行栈**里所有的任务都执行完了，执行栈才会去任务**队列**里读取任务，总不能闲着啊对不对。这样不断循环去读取任务**队列**里的任务并执行的过程就叫它**事件循环（Event Loop）**。

## Macrotasks 和 Microtasks

上面我们说了**事件循环**就是**执行栈**循环不断地去**任务队列**里去读取任务，但是一次读取一个任务吗？这里还稍微有些区别。

**任务队列**其实分两种，**Macrotasks队列**和**Microtasks队列**，我们一般所说的任务队列是**Macrotasks队列**，在每一次循环中只会读取一个，而**Microtasks队列**却会一直读取，也就是直接把队列里所有的任务全拿过来到执行栈里执行。并且**Microtasks队列**有优先性，只有当**Microtasks队列**里的任务清空，才会读**Macrotasks队列**里的任务。

那么哪些是**Macrotasks**，哪些是**Microtasks**呢？可以看下下面。

```
1. Macrotasks: setTimeout, setInterval, setImmediate, I/O, UI rendering
2. Microtasks: process.nextTick, Promises, Object.observe(废弃), MutationObserver
```





