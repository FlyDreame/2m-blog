# 深度解析vue的$nextTick的实现原理 

## 前提

vue 中有一个我们经常会用到的api，**nextTick**。我们都知道他是个可以在Dom更新后才执行的回调，比如下面的代码：

```javascript
this.msg = 'hello'  // 假设msg是data上的值

// Dom还没更新

this.$nextTick(() => {
    // Dom更新了
})
```

每次用它的时候，我都会想，nextTick是怎么实现的呢，难道是监听了Dom的变化吗？于是我去看了下nextTick的实现源码，根据源码，我们可以详细了解下这个货。

（注：在阅读之前需要了解 **事件循环机制** 、**microtask** 和 **task/macrotask** 的基本概念，可参考我写的文章 [JavaScript并发模型与Event Loop](https://github.com/FlyDreame/blog/issues/1)）

## 正文

前提中我们猜想是不是监听了Dom的更新，在HTML5中的确是有个api：**MutationObserver**，他可以监听Dom对象的变动（节点的删除、属性修改等） 。代码示例如下：

```javascript
let mo = new MutationObserver(callback) //callback 是Dom更新后的回调函数
let domTarget = 你想要监听的dom节点
mo.observe(domTarget, {
      characterData: true //说明监听文本内容的修改。
})
```

那么vue是不是用**MutationObserver**来监听Dom是否更新完毕的呢？然后我们打开vue的部分源码看看：

```javascript
// 创建一个MutationObserver,observer监听到dom改动之后后执行回调nextTickHandler
    var observer = new MutationObserver(nextTickHandler)
    var textNode = document.createTextNode(counter)
    // 调用MutationObserver的接口,观测文本节点的字符内容
    observer.observe(textNode, {
      characterData: true
    })
```

很明显我们在代码中看到了**MutationObserver**，但是**MutationObserver**监听Dom不对啊，源码中监听的是自己创建的文本节点，难道这个文本节点变化就能代表其他Dom的变化吗，很明显这个结论不成立。

其实nextTick的实现原理并不是基于**MutationObserver**，而是稍微借用的了**MutationObserver**的**microtask**特性，下面让我们看下nextTick的全部代码（代码版本为2.4.x）：

```javascript
export const nextTick = (function () {
  var callbacks = []
  var pending = false
  var timerFunc
  
  function nextTickHandler () {
    pending = false
    // 之所以要slice复制一份出来是因为有的cb执行过程中又会往callbacks中加入内容
    // 比如$nextTick的回调函数里又有$nextTick
    // 这些是应该放入到下一个轮次的nextTick去执行的,
    // 所以拷贝一份当前的,遍历执行完当前的即可,避免无休止的执行下去
    var copies = callbacks.slice(0)
    callbacks = []
    for (var i = 0; i < copies.length; i++) {
      copies[i]()  // 遍历执行回调
    }
  }
    
  /*--------------------------确定timerFunc---------------------------*/
  /* 这一坨代码就是为了确定timerFunc*/
  // ios9.3以上的WebView的MutationObserver有bug，
  //所以在hasMutationObserverBug中存放了是否是这种情况
  if (typeof MutationObserver !== 'undefined' && !hasMutationObserverBug) {
    var counter = 1
    // 创建一个MutationObserver,observer监听到dom改动之后后执行回调nextTickHandler
    var observer = new MutationObserver(nextTickHandler)
    var textNode = document.createTextNode(counter)
    // 调用MutationObserver的接口,观测文本节点的字符内容
    observer.observe(textNode, {
      characterData: true
    })
    // 每次执行timerFunc都会让文本节点的内容在0/1之间切换,
    // 不用true/false可能是有的浏览器对于文本节点设置内容为true/false有bug？
    // 切换之后将新值赋值到那个我们MutationObserver观测的文本节点上去
    timerFunc = function () {
      counter = (counter + 1) % 2
      textNode.data = counter
    }
  } else {
    // webpack attempts to inject a shim for setImmediate
    // if it is used as a global, so we have to work around that to
    // avoid bundling unnecessary code.
	// webpack默认会在代码中插入setImmediate的垫片
    // 没有MutationObserver就优先用setImmediate，不行再用setTimeout
    const context = inBrowser
      ? window
      : typeof global !== 'undefined' ? global : {}
    timerFunc = context.setImmediate || setTimeout
  }
  /*--------------------------确定timerFunc---------------------------*/
    
  return function (cb, ctx) {
    var func = ctx
      ? function () { cb.call(ctx) }
      : cb
    callbacks.push(func)
    // 如果pending为true, 就其实表明本轮事件循环中已经执行过timerFunc(nextTickHandler, 0)
    if (pending) return
    pending = true
    timerFunc(nextTickHandler, 0)
  }
})()
```

