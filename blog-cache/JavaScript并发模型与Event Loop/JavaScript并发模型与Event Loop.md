# # JavaScript并发模型与Event Loop

javascript在主线程运行时，会产生栈（stack）和堆（heap），程序中的代码会进入栈中等待执行，而程序中的对象会被分配在一个堆中。若代码执行中遇上异步方法，该异步方法就会被添加到用于回调的队列（queue）中。整个模型结构如下图所示：

![](F:\github-blog\blog-cache\JavaScript并发模型与Event Loop\模型结构.png)

