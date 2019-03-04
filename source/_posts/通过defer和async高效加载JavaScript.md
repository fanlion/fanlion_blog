---
title: 通过defer和async高效加载JavaScript
date: 2019-03-04 14:21:47
tags: JavaScript
categories: 翻译
toc: true
---

文本是[Efficiently load JavaScript with defer and async](https://flaviocopes.com/javascript-async-defer/)的译文,以下为译文正文部分。

---

# 通过defer和async高效加载JavaScript

当在一个HTML页面上加载一个script时，你需要小心应对以免引起页面的加载性能问题。

一个script传统上通过下面这种方式引入：

``` js
  <script src="script.js"></script>
```

当HTML解析器发现这行代码时，一个请求将被准备好去获取这个script并执行这个script.

<!-- more -->

一旦这个过程完成，解析过程将被恢复，剩余的HTML才能完成分析。

正如你可以想象得到的，这个操作会对这个页面的加载时间产生巨大的影响。

如果这个script的加载花费了比预期时间还要稍微长一点的时间，例如网络有点慢或者你在手机设备上并且网络连接有点糟糕，访客很有可能在script被加载和执行之前看到一个空白页面。

## 脚本位置问题

当你第一次学习HTML的时候，你被告知script 标签需要放置在`<head>`标签里：

``` html
<html>
  <head>
    <title>Title</title>
    <script src="script.js"></script>
  </head>
  <body>
    ...
  </body>
</html>

```

正如我之前所说的，当解析器发现这行时，解析器会去获取这个script并执行这行它。然后，在获取到script并执行script后，解析器才会接着去解析body里的内容。

这是糟糕的，因为这会引进一些延时。对于这个问题，一个十分通用的解决方案是把`script`标签放在页面的底部，`在</body>`这个闭合标签之前。

通过上面的方法，script标签将会在所有页面完成解析和加载后再完成加载和执行。相对于把`script`放在`head`中是个巨大的提升。

如果你需要支持那些不支持`HTML`最近特性标签：`async`和`defer`的老式浏览器，上面的方法已经是你所能做的事情里面做得最好的了。

## Async 和 Defer

`async`和`defer`都是`boolean`类型的属性，它们的用法相似：

``` js
<script async src="script.js"></script>
```

``` js
<script defer src="script.js"></script>
```

如果你同时指定了`async`和`defer`, `async`在现代浏览器中将会优先，然而那些只支持`defer`不支持`async`的老式浏览器将会使用`defer`。

> 通过caniuse.com可以检测浏览器是否支持`async` https://caniuse.com/#feat=script-async 和 `defer` https://caniuse.com/#feat=script-defer。

这两属性仅仅在把`script`放在页面的`head`中才会起作用，如果放在我们前面所说的`body`标签底部是没用的。

## 表现对比

### 不用defer或者async，放在head中

下面是没有`defer`和`async`，并把script放在页面`head`里script的加载情况。

![without-defer-async-head](without-defer-async-head.png)

解析过程在script被下载和执行完成之前是处于暂停状态的。一旦script的下载和执行完成，解析过程才会恢复。

### 不用defer或者async，放在body中

下面是没有`defer`和`async`，并把script放在页面`body`里, 在闭合标签之前script的加载情况。

![without-defer-async-body](without-defer-async-body.png)

解析过程的完成没有任何暂停，直到解析完成后，script完成获取和执行。解析过程甚至是在script被下载完之前完成，所以与上一个例子相比页面会更早的呈现给用户。

### 使用asyn，放在head中

下面是使用`async`，并把script放在页面`head`标签里script的加载情况。

![with-async](with-async.png)

script通过异步的方式获取，当完成下载后HTML解析将被暂停并去执行这个script，然后HTML恢复解析。

### 使用defer，放在head中

下面是使用`async`，并把script放在页面`head`标签里script的加载情况。

![with-defer](with-defer.png)

script脚本通过异步的方式获取，它只会在HTML解析完成之后执行。

解析完成的过程就像当我们把script放在`body`标签的结尾处一样。但是总的来说，script的执行结束要比前者好，因为script的下载是与HTML解析并行的。

所以在提速的术语中,这是胜出方案。🏆

### 阻塞解析

`async`会阻塞页面的解析，而`defer`不会。

### 阻塞渲染

`async`和`defer`对于阻塞渲染都不能做任何保证，这取决于你和你的script脚本(例如，确保你的那些script脚本在`onLoad`事件之后才运行)

### domInteractive

被标记位`defer`的脚本在`domInteractive`事件后都能被正确的执行，`domInteractive`事件发生在HTML加载、解析完成、`DOM`已经被创建后。

CSS和图片在这个时候仍然处于被解析和加载的路上。

一旦上述过程完成，浏览器将会触发`domComplete`事件，然后触发`onLoad`事件。

`domInteractive`是十分重要的，因为它的时间测定可以被用来衡量人所感知的加载速度。[查看MDN](https://developer.mozilla.org/en-US/docs/Web/API/PerformanceTiming/domInteractive)获取更多信息。

### 保持有序

另一个支持使用`defer`的场景：多个被标记为`async`的script脚本是在他们可用的时候以随意的顺序执行。多个被标记位`defer`的script脚本是按它们在html中定义的顺序执行（在HTML解析完成以后）。

### 直接告诉我最好的方式就好了

当用到`script`时，最好的提升你的页面加载速度的方式是把`script`放在`head`标签中，然后给`script`标签添加上`defer`属性：

``` js
<script defer src="script.js"></script>
```

这是更快触发`domInteractive`事件的方案。

考虑那些推荐使用`defer`情况，似乎在大多数场景下`defer`比`async`是个更好的选择。

如果你不在意页面第一次渲染的延时，确保你的JavaScript在页面已经被解析前已经被执行了。

---

END