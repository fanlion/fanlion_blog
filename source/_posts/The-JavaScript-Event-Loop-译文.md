---
title: The JavaScript Event Loop 译文
date: 2019-03-04 17:44:53
tags: JavaScript
categories: 翻译
toc: true
---

文本是[The JavaScript Event Loop](https://flaviocopes.com/javascript-event-loop/)的译文,以下为译文正文部分。

---

# JavaScript 事件循环

事件循环是了解JavaScript的最重要方面之一，本文通过简单的术语去解释它。

## 简介

事件循环是了解JavaScript的最重要方面之一，本文通过简单的术语去解释它。

> 我已经用JavaScript编程好几年了，但我从来没完整的理解它是如何运作的。不去知道这些概念的细节也是完全没问题的，但是通常，这对于我们知道是它如何运作的十分帮助，你也许只是对它有一点点好奇心。

<!-- more -->

这篇文章的目标是解释JavaScript是如何在一个单线程中工作，以及它是如何处理异步函数的内部细节。

你的JavaScript代码运行在一个单线程里。这意味着同一时刻只能发生一件事。

这个限制通常也十分有用，因为它可以简化很多，让你的程序不用担心并发的问题。

你只需要将注意力放在如何去写你的代码并避免任何可能会阻塞这个线程的事情，例如同步的网络调用或者无限循环。

通常，大多数浏览器每个浏览器tab都存在一个事件循环，让每个进程隔离，避免一个存在无限循环或者大量处理任务的web页面阻塞了你的整个浏览器。

例如，环境管理多个并发的事件循环，以处理API调用。同样，[Web Workers](https://flaviocopes.com/web-workers/)运行在它们自己的事件循环里。

你主要需要关心的是你的代码将运行在一个单事件循环里，然后在写代码的时候记在脑海中去避免阻塞它。

## 阻塞事件循环

任何需要很长时间才能将控制权返回给事件循环的JavaScript代码都会阻止页面中任何JavaScript代码的执行，甚至阻塞UI线程，导致用户不能点击、滚动页面等。

几乎JavaScript中的所有的`I/O`原语都是非阻塞的。网络请求、Node.js文件系统操作等。被阻塞是个例外，这也是为什么JavaScript基于如此多的回调，以及最近的`Promise`和`async/await`。

## 调用堆栈

调用堆栈是一个`LIFO`队列（后进、先出）

事件循环不断的检查调用堆栈去查看是否还存在任何需要被执行的函数。

在执行此操作时，它会将它找到的任何函数调用添加到调用堆栈并按顺序执行每个调用

你了解调试器或浏览器控制台中你可能熟悉的错误堆栈跟踪吗？浏览器在调用堆栈中查找函数名称以通知你哪个函数发起当前调用：

![exception-call-stack](exception-call-stack.png)

## 一个简单的事件循环说明

让我们选择一个例子：

``` js
const bar = () => console.log('bar')

const baz = () => console.log('baz')

const foo = () => {
  console.log('foo')
  bar()
  baz()
}

foo()
```

上面代码打印

``` js
foo
bar
baz
```

正如我们所预期一样。

当代码开始执行，首先`foo()`被调用。在`foo()`内部我们首先调用`bar()`，然后我们调用`barz()`。

在这个时候，调用栈看起来像这样：

![call-stack-first-example](call-stack-first-example.png)

事件循环的每次迭代都会查看调用栈中是否存内容，如果存在则执行它：

![hexecution-order-first-example](execution-order-first-example.png)

直到调用栈为空。

## 队列中函数执行

上面的例子看起来很寻常，没有什么特殊之处：JavaScript找到一些东西去执行，按顺序执行。

让我们看一下如何延迟一个函数直到堆栈清空了再执行。

`setTimeout(() => {}), 0)`的用法是去调用一个函数，但是，一旦代码中的其他函数都执行了才会去执行它。

举个例子：

``` js
const bar = () => console.log('bar')

const baz = () => console.log('baz')

const foo = () => {
  console.log('foo')
  setTimeout(bar, 0)
  baz()
}

foo()
```

下面是它的代码打印，也许出乎意料：

``` js
foo
baz
bar
```

当代码运行，首先`foo()`被调用。在`foo()`内部我们先调用`setTimeout()`，通过把`bar`作为参数传递，并传递`0`作为定时器，我们希望尽可能快的立即执行它，然后再去执行`baz()`。

在这个时候调用栈看起来像这样：

![call-stack-second-example](call-stack-second-example.png)

这是我们程序中所有函数的执行顺序：

![execution-order-second-example](execution-order-second-example.png)

为什么会这样？

## 消息队列

当`setTimeout()`被调用，浏览器或者Node.js启动定时器。一旦时间到了，在这个例子中是立刻，因为我们把`0`设置为了超时时长，这个回调函数被放入了消息队列。

消息队列也是像用户所产生的点击、键盘事件或者是`fetch`响应在你的代码有机会响应它们之前所排队的地方，或者是像`onLoad`这种`DOM`事件。

事件循环优先考虑调用栈，它会首先处理在调用栈中找到的内容，一旦调用栈为空，它会去拾取消息队列中的内容。

我们不用去等待函数（像`setTimeout`、`fetch`或者是其他的东西完）成它们自己的工作，因为它们是浏览器提供的，它们存货在自己的线程里。例如，你设置`setTimeout`的超时时间到2秒，你不用等待2秒，这个等待发生在其他地方。

## ES6作业队列

`ECMAScript 2015`引入了`作业队列`的概念，它被`Promises`使用（ES6/ES2015引入的）。 这是一种尽快执行异步函数结果的方法，而不是放在调用栈的末尾。

我发现把它比喻为在游乐园里的过山车再恰当不过了：消息队列把你反正队列的后面，在所有其他人之后，在那里你不得不等到轮到你，而作业队列是快速通过的票，让你完成上一次乘坐后立马再次乘坐。

例如：

``` js
const bar = () => console.log('bar')

const baz = () => console.log('baz')

const foo = () => {
  console.log('foo')
  setTimeout(bar, 0)
  new Promise((resolve, reject) =>
    resolve('should be right after baz, before bar')
  ).then(resolve => console.log(resolve))
  baz()
}

foo()
```

它的输出

``` js
foo
baz
should be right after baz, before bar
bar
```

这是`Promises`（以及 async/await, 它们是构建与`promises`之上的）和普通的老式异步函数（通过`setTimeout()`或者其他的平台`APIs`）之间的巨大差异。

---

END