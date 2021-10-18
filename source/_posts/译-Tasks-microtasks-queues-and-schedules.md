---
title: '[译]Tasks,microtasks,queues,and schedules'
date: 2018-07-01 16:47:20
tags:
---

> 当我告诉我的同事 [Matt Gaunt](https://twitter.com/gauntface) 我正打算写一篇关于 microtask 组织队列以及在浏览器事件循环中运行的文章时，他说“老实讲，Jake，打死我都不看”。好吧，不管怎样，文章嘛，写都写了╮(╯_╰)╭
>
> 实际上，如果你倾向于看视频，[Philip Roberts](https://twitter.com/philip_roberts) 在 JSConf 上关于 event loop 的[演讲](https://www.youtube.com/watch?v=8aGhZQkoFbQ)会是不错的选择——虽然没有提到 microtask，但是其余部分还是挺出彩的



看看这段代码：

```javascript
console.log('script start');

setTimeout(function() {
  console.log('setTimeout');
});

Promise.resolve().then(function() {
  console.log('promise1');
}).then(function() {
  console.log('promise2');
});

console.log('script end');
```

你觉得结果的打印顺序是什么？



#### 小试牛刀

<iframe src="https://cinyearchan.github.io/task-microtask-demo/page/1-log-test.html" height="300" width="600" style="border: none;"></iframe>
正确答案是：

`script start`、`script end`、`promise1`、`promise2`、`setTimeout`

但就浏览器的支持来说，有些浏览器上打印的结果却有出入（备注：原文写于2015年）

Microsoft Edge、Firefox 40、iOS Safari 以及 桌面版 Safari 8.0.0 会在 `promise1` 和 `promise2` 前打印 `setTimeout`，尽管看起来像是一种罕见的情况（备注：原文为 a race condition 个人认为应该是 rare condition）。很奇怪，因为 Firefox 39 和 Safari 8.0.7 打印的结果是正常的



#### 结果为什么会这样

要想明白其中的原理，你需要搞清楚 event loop 是如何处理 task 和 microtask

> 每一个“线程”都有自己的 **event loop** ，同样，每一个 web worker 也都有自己的 event loop，它们能独立运行，但同时，同源下的所有窗口共享一个 event loop 以便能够同步的交流。Event loop 不停地运行，执行队列里的每一个 task。一个 event loop 会有多个 task source(任务源)，以此保证每个源下的 task 的执行顺序（例如 [IndexedDB](https://w3c.github.io/IndexedDB/#database-access-task-source) 定义了自己的源），但浏览器会在每一轮循环里挑选源来获取要执行的 task。这使得浏览器能够给那些性能敏感（performance sensitive）的 task 给予优先权，例如 用户输入

> 任务（task）会被编排（schedule）好，以便浏览器可以从其内部进入 JavaScript/DOM 领域并保证这些操作按顺序发生。从获取鼠标的单击事件到事件回调需要编排一个 task，同样的还有解析 HTML 以及上面提到的例子—— `setTimeout`

`setTimeout` 会在给定的延迟内等待，然后给它自己的回调编排一个新的 task ——这就是为什么 `setTimeout` 会在 `script end` 后面打印了—— `script end` 是第一个 task 的一部分，而 `setTimeout` 是在另一个隔离的 task 内打印的。

> 微任务（microtask）通常被编排用于那些在当前正在执行的脚本后直接发生的事物，例如对一组批量操作作出反应，或者不需要通过创建一个额外的、完整的任务（task）来获取异步的结果。只要没有其他 JavaScript 处于执行中期，并且处于每个任务（task）的末尾时，微任务队列（microtask queue）就会在回调后处理。在微任务队列中任何编排的其他微任务都会被添加到队列的末尾并进行处理。微任务包括 mutation observer callbacks 以及上面例子中的 promise callback

一个 promise 实例的状态一旦变为凝滞，或者已经处于凝滞，它就会为其反应的回调向微任务队列中添加一个微任务——如此可以保证即使 promise 的状态已经凝滞，promise callback 一定是异步的。

因此调用一个紧挨着状态凝滞的 promise 的 `then(yey, nay)` 会立即编排一个微任务——这就是 `promise1` 和 `promise2` 会在 `script end` 之后打印的原因——当前运行的脚本一定会在微任务执行前结束；而 `promise1` 和 `promise2` 出现在 `setTimeout` 之前，是因为（当前任务的）微任务总是发生在下一个任务之前。



好吧，一步一步来：

<iframe src="https://cinyearchan.github.io/task-microtask-demo/page/2-step-test.html" height="550" width="600" style="border: none;"></iframe>
##### 为什么部分浏览器执行的结果会有差异

部分浏览器打印的结果是：`script start`、`script end`、`setTimeout`、`promise1`、`promise2`，从结果来看，它们会在 `setTimeout` 后面执行 promise 的回调——很有可能是因为它们将调用 promise 的回调当做新任务的一部分而不是微任务。

这是可以理解的，因为 promise 来源于 ECMAScript 而不是 HTML:)

ECMAScript 中有“作业（job）”的概念，这类似于微任务，但除了 [vague mailing list duscussions](https://esdiscuss.org/topic/the-initialization-steps-for-web-browsers#content-16) ，这两个概念的区别不是很明确。但普遍的共识是：有充足的理由要求我们将 promise 当做微任务队列的一部分

将 promise 视为任务会导致性能问题，因为其回调可能会因与渲染等与任务相关的事务而不必要的延迟，它还会因与其他任务源的交互而导致非确定性，并且可能会破坏与其他 API 的交互，后面会对这个展开叙述

[Edge正尝试在 promise 中使用 microtask](https://connect.microsoft.com/IE/feedback/details/1658365) （2015版 Edge，现在的 Edge 是基于 chrome 内核的），webkit nightly 一直都是这么做的，Firefox 43 也已经修复了这一点，所以我猜 Safari 最终也会选择修复这点



#### 如何判断用的是 task 还是 microtask

测试是一种方法。see when logs appear relative to promises & setTimeout，尽管你依赖于正确的实现（代码实现正确为前提，观察打印的结果，剖析 promise 以及 setTimeout 表现出的关系）

查看规范也是一种方法。比如说，[`setTimeout` 的第14步](https://html.spec.whatwg.org/multipage/webappapis.html#timer-initialisation-steps) 会编排一个任务，而 [queue a mutation record 的第5步](https://dom.spec.whatwg.org/#queue-a-mutation-record) 会编排一个微任务

如上所述，在 ECMAScript 领域，microtask 被称为“作业（job）”。在 [PerformPromiseThen 的第8步](https://www.ecma-international.org/ecma-262/6.0/#sec-performpromisethen) 中，`EnqueueJob` 会被调用去编排一个微任务



好吧，试试更复杂的例子吧

#### 第一关

在写这边文章之前，我弄错了一点。看看下面的代码片段：

```html
<div class="outer">
  <div class="inner"></div>
</div>
```

给出以下 JS，如果我点击 `div.inner`，打印的结果是什么？

```javascript
// Let's get hold of those elements
var outer = document.querySelector('.outer');
var inner = document.querySelector('.inner');

// Let;s listen for attribute changes on the
// outer element
new MutationObserver(function() {
  console.log('mutate');
}).observe(outer, {
  attributes: true
});

// Here's a click listener...
function onClick() {
  console.log('click');
	
  setTimeout(function() {
    console.log('timeout');
  }, 0);
  
  Promise.resolve().then(function() {
    console.log('promise');
  });
  
  outer.setAttribute('data-random', Math.random());
}

// ...which we'll attach to both elements
inner.addEventListener('click', onClick);
outer.addEventListener('click', onClick);
```

在揭示答案之前可以用上述代码写个例子放到浏览器中测试——提示：打印会发生不止一次



#### 测试

点击内部的方块触发点击事件

<iframe src="https://cinyearchan.github.io/task-microtask-demo/page/3-square-click.html" height="450" width="600" style="border: none;"></iframe>

与你的猜测有出入？如果有出入，你仍有可能是对的，毕竟浏览器之间是有差异的：





#### 到底谁是对的

派发 click 事件是一个任务；Mutation observer 以及 promise callbacks 会被编排为微任务；`setTimeout` 的回调会被编排为一个任务——以下便是代码运行的详细步骤：

<iframe src="https://cinyearchan.github.io/task-microtask-demo/page/4-step-mutation.html" height="550" width="600" style="border: none;"></iframe>

Chrome 里运行的结果是正确的。对我来说比较“新”的一点是微任务会在回调之后处理（只要没有其他 JavaScript 在执行中），我认为它是限于任务的末尾——这一规则来源于 HTML 规范中调用回调：

> If the [stack of script settings objects](https://html.spec.whatwg.org/multipage/webappapis.html#stack-of-script-settings-objects) is now empty, [perform a microtask checkpoint](https://html.spec.whatwg.org/multipage/webappapis.html#perform-a-microtask-checkpoint)
>
> -- [HTML: Cleaning up after a callback](https://html.spec.whatwg.org/multipage/webappapis.html#clean-up-after-running-a-callback) step 3 

并且微任务检查点涉及遍历微任务队列，除非我们已经在处理微任务队列。类似的，ECMAScript 是这样描述作业的：

> 仅当不存在正在运行的执行上下文，并且执行上下文的栈是空时，作业的执行**可以**被初始化
>
> —— ECMAScript: Jobs and Job Queues

尽管 ECMAScript 中作业的“可以”变成了 HTML 上下文中的“必须”



#### 浏览器出了什么问题？

正如 mutation callbacks 所示，**Firefox** 和 **Safari** 能够正常处理点击监听之间的微任务队列，但对 promise 的编排却有不同。这是可以理解的，毕竟作业与微任务之间的联系是模糊的，但我仍然希望他们能在 listener callbacks 之间执行。[Firefox ticket](https://bugzilla.mozilla.org/show_bug.cgi?id=1193394)、[Safari ticket](https://bugs.webkit.org/show_bug.cgi?id=147933)

至于 **Edge**，我们观察到它不仅不能正常编排 promise，也无法正常的处理点击监听之间的微任务队列，反而在调用所有监听器过后很久才会去处理微任务队列，因此表现为在两次click打印之后才记录到单个的mutate log。[Bug ticket](https://connect.microsoft.com/IE/feedbackdetail/view/1658386/microtasks-queues-should-be-processed-following-event-listeners)



#### 第一关升级版

同样的例子，如果我们执行下面这段代码，会发生什么？

```javascript
inner.click();
```

这会像先前那样开始事件派发，但不是通过现实的交互而是脚本触发



#### 小试牛刀

<iframe src="https://cinyearchan.github.io/task-microtask-demo/page/5-click-test.html" height="300" width="600" style="border: none;"></iframe>

各个浏览器的结果：



我坚信我会在 Chrome 一直获取不同的结果，这张图我已经更新过无数次，我曾一度以为我错误的测试了 Canary（Chrome开发版）



#### 为什么Chrome的结果会不确定

请看执行的步骤：

<iframe src="https://cinyearchan.github.io/task-microtask-demo/page/6-step-mutation.html" height="550" width="600" style="border: none;"></iframe>
