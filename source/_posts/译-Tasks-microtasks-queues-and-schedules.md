---
title: '[译]Tasks,microtasks,queues,and schedules'
date: 2018-07-01 16:47:20
tags:
---

如果你更喜欢视频讲解的话，推荐 [Philip Roberts](https://twitter.com/philip_roberts) 的 [great talk at JSConf on the event loop](https://www.youtube.com/watch?v=8aGhZQkoFbQ)，虽然没有涵盖 `microtasks` ，但却详细介绍了 `tasks、queues、schedules`

[原文地址](https://jakearchibald.com/2015/tasks-microtasks-queues-and-schedules/)

思考以下JS片段
```javascript
console.log('script start');

setTimeout(function() {
    console.log('setTimeout');
}, 0);

Promise.resolve().then(function() {
    console.log('promise1');
}).then(function() {
    console.log('promise2');
});

console.log('script end');
```

控制台会打印什么呢？

### 尝试

<style type='text/css'>
.Btn { 
	display: inline-block;
    padding: 4px 12px;
    margin-bottom: 0;
    font-size: 14px;
    line-height: 20px;
    text-align: center;
    text-shadow: 0 1px 1px rgba(255, 255, 255, 0.75);
    vertical-align: middle;
    cursor: pointer;
    background: #f5f5f5;
    background-image: linear-gradient(to bottom, #fff, #e6e6e6);
    background-repeat: repeat-x;
    border: 1px solid #bbb;
    border-color: rgba(0, 0, 0, 0.15) rgba(0, 0, 0, 0.15) rgba(0, 0, 0, 0, 25);
    border-radius: 4px;
    box-shadow: inset 0 1px 0 rgba(255, 255, 255, 0.2), 0 1px 2px rgba(0, 0, 0, 0.05);  
}
.Btn:hover {
    background-position: 0 -15px;
    background-color: #e6e6e6;
    transition: background-position 0.1s linear;
}
.Btn:focus {
    outline: 5px auto -webkit-focus-ring-color;
    outline-offset: -2px;
}
.log-output{
    width: 100%;
    box-sizing: border-box;
    height: 12.7em;
    font: inherit;
    line-height: 1.5;
}
</style>

<button class="Btn clear" id="clear">clear log</button> <button class="Btn run" id="run">run test</button>
<textarea class="log log-output log-output-1" width="300" height="200"></textarea>
<script>
function log1(str) {
    console.log(str)
    var logEl = document.querySelector('.log-output-1')
    logEl.value += (logEl.value ? '\n' : '') + str
}
document.querySelector("#clear").addEventListener("click", function() {
    document.querySelector('.log-output-1').value = ''
})
document.querySelector("#run").addEventListener("click", function() {
    log1('script start')
    setTimeout(function() {
        log1('setTimeout')
    }, 0)
    
    Promise.resolve().then(function() {
        log1('promise1')
    }).then(function() {
        log1('promise2')
    })
    
    log1('script end')
})
</script>

正确答案是：`script start`，`script end`，`promise1`，`promise2`，`setTimeout`，打印的结果与浏览器的支持程度有莫大的关系

Microsoft Edge, Firefox 40, iOS Safari and desktop Safari 8.0.8 会在 `promise1` 和 `promsie2` 之前打印 `setTimeout`

### 原因
要理解这一点，你需要知道 `event loop` 如何处理 `tasks` 和 `microtasks` 的

每个‘线程’都有自己的 `event loop`，因而每个 [`web worker`](https://developer.mozilla.org/zh-CN/docs/Web/API/Web_Workers_API)也都有自己的 `event loop`，同源的所有窗口共享一个 `event loop` 以便能够同步的交流。`event loop` 会持续不断地运行，执行队列中的所有任务。一个 `event loop` 会有多个任务源，可保证该源中任务的执行顺序(特别是 [IndexedDB](http://w3c.github.io/IndexedDB/#database-access-task-source) 等规范定义了自己的任务源)，但是浏览器会在 `event loop` 的每一个回合中选择从哪个源获取任务。这允许浏览器优先考虑性能敏感的任务，例如用户输入。

`Tasks` 都是被安排好的，以便浏览器可以从其内部进入 JavaScript/DOM 域并确保这些操作能够按顺序执行。在 `tasks` 之间，浏览器可能会渲染更新。从鼠标单击到事件回调都需要安排任务，解析 `HTML` 以及上述例子中的 `setTimeout` 也是如此。

`setTimeout` 等待给定的延迟，然后为其回调安排新的任务。这可以解释为什么 `setTimeout` 会在 `script end` 后面打印—— `script end` 是第一个任务的一部分，`setTimeout` 是在另一个任务中打印的。

微任务(`Miscrotasks`)通常被安排用于处理在当前正在执行的脚本之后发生的事，例如对批量的行为作出反应，或者不必创建一个全新的任务去处理异步的事务。只要没有其他 JavaScript 在执行中，并且在每个任务结束时，就会在回调后处理微任务队列。在微任务中排列的任何额外的微任务都会添加到队列的末尾并进行处理。微任务包括变动观察回调函数（mutation observer callbacks），以前上面提到的 promise 回调

一旦 promise 的状态凝滞，或者它的状态已经凝滞，便会为其回调往队列中添加一个微任务。这样能够确保即使 promise 的状态已经凝滞， promsie 回调也是异步的。所以对一个状态确定的 promise 调用 `.then(yey, nay)` 会立即往队列中添加一个微任务。`promise1` 和 `promise2` 会在 `script end` 后面打印便是如此，因为当前运行的脚本必须在处理微任务之前完成。微任务总是会在下一个任务之前发生，所以 `promise1` 和 `promise2` 会在 `setTimeout` 之前打印。






















