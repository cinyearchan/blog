---
title: '[译]Promise rookie'
date: 2019-10-16 15:12:57
tags:
---


JS程序员们，是时候承认了：我们的 promise 有问题

等等，我不是说 promise 本身有问题，基于 [A+ 规范](https://promisesaplus.com/) 编写的 Promise 自然是无懈可击

我说的这个问题在过去的一年里反复出现，我见到了无数的程序员栽在了 PouchDB API 和其他深度基于 promise 的 API 上：

我们中的绝大多数对 promise 的使用只停留在一知半解的层面

不信？不妨看看我在推特上 po 出的 [题目](https://twitter.com/nolanlawson/status/578948854411878400)：

#### 以下四个 `promise` 有什么区别

```javascript
doSomething().then(function() {
	return doSomethingElse();
});

doSomething().then(function() {
	doSomethingElse();
});

doSomething().then(doSomethingElse());

doSomething().then(doSomethingElse);
```

答案在本文的末尾



### 为什么要使用 promise

如果你读过 promise 相关文章，你会发现有一个名词会被经常提到——[the pyramid of doom 末日金字塔](https://medium.com/@wavded/managing-node-js-callback-hell-1fe03ba8baf) ——一些可怕的回调代码，层层嵌套，稳定地向屏幕右侧延伸

Promise 的出现确实解决了这个问题，但它的作用不仅限于减少代码缩进！正如 [Redemption from Callback Hell](http://youtu.be/hf1T_AONQJU) 中解释的，回调带来的真正问题是它剥夺了我们使用 `return` 和 `throw` 的权利，与此同时，我们的程序的整个流程都是基于副作用：一个函数顺带调用另一个函数

实际上，回调“作的恶”不止于此：它让我们与栈隔绝（栈：在编程语言中通常用来保证程序顺利运行的东西）！脱离栈写代码就像是开着一辆没有刹车的车一样危险——你根本不知道自己的处境有多危险，知道危险来临，你需要它时才发现它不在!

Promise 的重点在于，在我们写异步代码时，让我们重拾丢失的语言规范：`return`、`throw` 以及栈。

但你必须知道如何正确使用 promise 才能充分享受它带来的好处，否则只会是驱虎吞狼、后患无穷

### 初阶错误

#### 新手错误：promise的末日金字塔

```javascript
remotedb.allDocs({
	include_docs: true,
	attachments: true
}).then(function(result) {
	var docs = result.rows;
	docs.forEach(function(element) {
		localdb.put(element.doc).then(function(response) {
			alert("Pulled doc with id " + element.doc._id + " and added to local db.");
		}).catch(function(err) {
			if (err.name == 'conflict') {
				localdb.get(element.doc._id).then(function(resp) {
					localdb.remove(resp._id, resp._rev).then(function(resp) {
						// ... 诸如此类
					})
				})
			}
		});
	})
})
```

改进版：
```javascript
remotedb.allDocs(...).then(function(resultOfAllDocs) {
	return localdb.put(...);
}).then(function(resultOfPut) {
	return localdb.get(...);
}).then(function(resultOfGet) {
	return localdb.put(...);
}).catch(function(err) {
	console.log(err);
})
```
这被称为 `composing promises` 即“合成promises”，这是 `promises` 所提供的强大魔力之一。每一个方法只会在前一个 `promise` 转变为 `resolved` 状态时被调用，并且可以获取前一个 `promise` 的结果

#### 新手错误2：`forEach()/循环` 与 `promise` 如何兼得
```javascript
// 我想通过 remove() 处理所有文档
db.allDocs({include_docs: true}).then(function(result) {
	result.rows.forEach(function(rows) {
		db.remove(row.doc);
	});
}).then(function() {
	// 我认为所有文档都被移除了 ha?
});
```
上述代码有什么问题？问题在于，第一个 function 会返回 undefined，意味着第二个 function 不会等待所有文档都执行 db.remove()。实际上，它根本什么都不会等，任意数目的文档被移除时，第二个方法都会执行。

解决的方法是，应该用 `Promise.all()` 来替代 `forEach()`/`for`/`while` 处理异步循环
```javascript
db.allDocs({include_docs: true}).then(function(resp) {
	return Promise.all(result.rows.map(function(row) {
		return db.remove(row.doc);
	}));
}).then(function(arrayOfResults) {
	// 此时，所有文档才真正被移除
});
```
这到底发生了什么？简单来说，`Promise.all()` 以一个包含多个 promise 的数组作为输入，返回另一个 promise 作为输出，只有当输入数组中的所有 promise 的状态都转变为 `resolved`，该输出的 promise 状态才会变成 `resolved`，相当于异步的for循环

`Promise.all()` 也会向随后的处理函数传递一个包含所有结果的数组，这是非常有用的，例如，试图通过 `get()` 从 PouchDB 中获取多个数据。如果输入数组中包含的 promise，有任意一个状态转变为 `rejected`，该 `all()`的 promise 都会变为 `rejected`

#### 新手错误3：忘记加上 `.catch()`

这是另一个常见的错误。简单的认为他们写的promise绝不会抛出错误，很多开发者都会忘记在 promise 链的末尾加上 `.catch()`。一旦如此，promise 中任何被抛出的错误都会被掩盖，甚至无法再 console 中展现，无疑会提高 debug 的难度

为避免上述情况的发生，我建议在每一个promise链的末尾都加上 `.catch()`：

```javascript
somePromise().then(function() {
  return anotherPromise();
}).then(function() {
  return yetAnotherPromise();
}).catch(console.log.bind(console)); // <-- this is badass
```



#### 新手错误4：使用 `deferred`

我总是能看到这类[错误](http://gonehybrid.com/how-to-use-pouchdb-sqlite-for-local-storage-in-your-ionic-app/)，以至于我现在都不愿意重复这个单词

简而言之，promise有一段漫长且曲折的历史，JS社区花费了大量的时间来纠正期间的错误。在promise发展的早期，jQuery 和 Angular 都通过 "deferred" 来实现各自的 promise，而现如今都被 ES6 的 promise 所取代，例如Q、when、RSVP、Bluebird、Lie等库都实现了符合 promise/A+ 规范的 promise

如何避免使用 deferred 呢？

首先，绝大多数 promise 库都会为你引入第三方库实现的 promise 的方法。例如，Angular 中的 `$q` 模块允许你通过 `$q.when()` 来包裹不符合 `$q` 规范的 promise。所以，Angular 使用者可以这样包裹 PouchDB promises：

```javascript
$q.when(db.put(doc)).then(/* ... */); // <-- 这就是你迫切需要的
```

另一个方法是使用[revealing constructor pattern](https://blog.domenic.me/the-revealing-constructor-pattern/)来包裹非 promise 的API。例如，封装基于回调的API，比如说 Nodejs 中的 `fs.readFile()`，可以这样做：

```javascript
new Promise(function(resolve, reject) {
  fs.readFile('myfile.txt', function(err, file) {
    if (err) {
      return reject(err);
    }
    resolve(file);
  });
}).then(/* ... */)
```



#### 新手错误5：使用副作用而不是返回

以下代码有什么问题？

```javascript
somePromise().then(function() {
  someOtherPromise();
}).then(function() {
  // 我希望 someOtherPromise() 已经是 resolved 状态了
  // 事实却告诉我，它不是
});
```

是时候来详细了解 promise 的各个要点了

正如我之前所说，promise 的魔力在于它回馈我们之前的 `return` 和 `throw`。实际中是如何体现的呢？

每一个 promise 都会为你提供一个 `then()` 方法（或者 `catch()`方法，`then(null, ...)` 的语法糖）。让我们进入 `then()` 函数的内部：

```javascript
somePromise().then(function() {
  // 现在是 then() 函数的内部
})
```

我们在 `then()` 函数内部可以做什么？有三件事是可以做的：

1. `return` 另一个 promise

2. `return` 一个同步的值（或 `undefined`）

3. `throw` 一个同步的 error

   
  

一旦你掌握其中的诀窍，你就解开了 promise 的魔术。让我们逐点分析：

1. 返回另一个 promise

   这个模式在 promise 相关文章中很常见，就像上面提到的“合成promise”：

   ```javascript
   getUserByName('nolan').then(function(user) {
     return getUserAccountById(user.id);
   }).then(function(userAccount) {
     // I got a user account!
   });
   ```

   注意此处，我返回了第二个 promise ——返回 `return` 是关键，如果没有 `return` ，`getUserAccountById()` 就只会产生副作用，第二个 then 的处理函数就只能接收到 `undefined` 而不是 `userAccount`

2. 返回一个同步的值（或 undefined）

   返回 `undefined` 通常来说是一个错误，但返回一个同步的值，却是一种将同步代码转换为 promise 风格代码的很酷的方式。例如，我们有一份用户的缓存数据，我们可以做什么呢：

   ```javascript
   getUserByName('nolan').then(function(user) {
     if (inMemoryCache[user.id]) {
       return inMemoryCache[user.id]; // returning a synchronous value!
     }
     return getUserAccountById(user.id); // returning a promise!
   }).then(function(userAccount) {
     // I got a user account!
   });
   ```

   这不是很酷么？第二个函数根本不用关心 `userAccount` 是同步获取的还是异步获取的，第一个函数也可以随意返回同步的值或者异步的值。

   不幸的是，有个麻烦的事实摆在眼前：无返回值的函数，严格来说，都会默认返回 `undefined` ，意味着当你打算返回某些东西的时候，很容易在不经意间就产生副作用

   出于这个原因，我的个人习惯是，总是在 `.then()` 函数内加上 `return` 或者 `throw` sth，我建议你也这样做

3. 抛出同步的异常

   说到 `throw` ，这是 promise 可以变得更加令人敬畏之处。比如说我们希望能够抛出一个同步的异常以防用户登出：

   ```javascript
   getUserByName('nolan').then(function(user) {
     if (user.isLoggedOut()) {
       throw new Error('user logged out!'); // throwing a synchronous error!
     }
     if (inMemoryCache[user.id]) {
       return inMemoryCache[user.id]; // returning a synchronous value!
     }
     return getUserAccountById(user.id); // returning a promise!
   }).then(function(userAccount) {
     // I got a user account!
   }).catch(function(err) {
     // Boom, I got an error!
   });
   ```

   上述代码中，如果用户登出，`catch()` 会捕获到一个同步的异常，并且如果任意一个 promise 变为 rejected 状态， `catch()` 会捕获到一个异步的异常。同样，`catch()` 函数无需关系捕获的异常是同步的还是异步的。

   这在开发过程中确定代码异常极其有效！举个例子，在 `then()` 函数内部任意一处，我们执行 `JSON.parse()` 操作，如果是 JSON 是无效的，则会抛出一个同步的异常。在回调中，该异常会被掩盖；但在 promise 中，我们可以在 `catch()` 中轻松处理捕获到的异常。



### 进阶错误

既然你已经学到了可以让 promise 变得简单的单一技巧，不妨来看看一些边角案例，当然，编程总是会遇到边角案例

之所以说这些案例是进阶的，是因为我发现那些已经相当擅长 promise 的程序员才犯这些错。如果想弄清楚文章开头提到的那个问题，我们就有必要研究清楚这些案例



#### 进阶错误1：不知道 `Promise.resolve()`

正如我在上面提到的，promise 非常擅长将同步代码封装为异步代码。然而，如果下面这段代码你敲得多了的话：

```javascript
new Promise(function(resolve, reject) {
  resolve(someSynchronousValue);
}).then(/* ... */);
```

你可以用 `Promise.resolve()` 来更简洁的表明这一点：

```javascript
Promise.resolve(someSynchronousValue).then(/* ... */);
```

这在捕获任何同步的异常上也极其有效，正因如此，我已经养成了几乎所有 promise 返回API方法的习惯，如下所示：

```javascript
function somePromiseAPI() {
  return Promise.resolve().then(function() {
    doSomethingThatMayThrow();
    return 'foo';
  }).then(/* ... */);
}
```

记住一点：任何可能同步抛出的代码，都可以在某处找到几乎不可能调试的隐藏的异常。如果你用 `Promise.solve()` 包裹一切同步代码，则你总是可以在之后通过 `catch()` 函数捕获异常

相似的，你可以用 `Promise.reject()` 来返回一个状态立刻变为 `rejected` 的 promise 实例：

```javascript
Promise.reject(new Error('some awful error'));
```



#### 进阶错误2：`then(resolveHandler).catch(rejectHandler)` 并不全等于 `then(resolveHandler, rejectHandler)`

上面提到的 `catch()` 是语法糖，因此如下两个代码片段是等价的：

```javascript
somePromise().catch(function(err) {
  // handle error
});

somePromise().then(null, function(err) {
  // handle error
});
```

然而，下面两段代码却不是等价的：

```javascript
somePromise().then(function() {
  return someOtherPromise();
}).catch(function(err) {
  // handle error
});

somePromise().then(function() {
  return someOtherPromise();
}, function(err) {
  // handle error
});
```

如果你还在疑惑为何不等价，不妨设想当第一个函数抛出异常时：

```javascript
somePromise().then(function() {
  throw new Error('oh noes');
}).catch(function(err) {
  // I caught your error! :)
});

somePromise().then(function() {
  throw new Error('oh noes');
}, function(err) {
  // I didn't catch your error! :(
});
```

事实证明，当你使用 `then(resolveHandler, rejectHandler)` 格式时，如果 `resolveHandler` 自身抛出了异常，`rejectHandler` 是自然无法捕获到的

为此，我自己的习惯是：从不使用 `then()` 函数的第二个参数，而是使用 `catch()`。例外则是，当我在写异步的[Mocha](http://mochajs.org/) 测试用例时，我会使用第二个参数来确保异常能够被抛出：

```javascript
it('should throw an error', function() {
  return doSomethingThatThrows().then(function() {
    throw new Error('I expected an error!');
  }, function(err) {
    should.exist(err);
  });
});
```

说到测试，在测试 promise API时，[Mocha](http://mochajs.org/) 和 [Chai](http://chaijs.com/) 是绝妙的组合。[pouchdb-plugin-seed](https://github.com/pouchdb/plugin-seed)项目中有一些简单的 [测试用例](https://github.com/pouchdb/plugin-seed/blob/master/test/test.js) 可以拿来练手



#### 进阶错误3：promises VS promise factories

或许你想按顺序一个一个的处理一组 promise，处理的方式近似于 `Promise.all()` ，但其中的 promise 却不是并发执行的

你可能简单的认为理想的代码是这样的：

```javascript
function executeSequentially(promises) {
  var result = Promise.resolve();
  promise.forEach(function(promise) {
    result = result.then(promise);
  });
  return result;
}
```

实际上，上述代码无法达到你预期的效果，传递给 `executeSequentially()` 的多个 promise 仍然会并发执行

发生这种情况的原因是你根本不想操作 promise 的数组，根据 promise 规范，一旦创建了 promise，它就会开始执行，你真正想要的是 promise factories 的数组

```javascript
function executeSequentially(promiseFactories) {
  var result = Promise.resolve();
  promiseFactories.forEach(function(promiseFactory) {
    result = result.then(promiseFactory);
  });
  return result;
}
```

我知道你在想什么：“这个 Java 程序员到底是谁，为什么他在谈论 factories 呢？”promise factories 很简单，只是一个会返回一个 promise 实例的函数而已:

```javascript
function myPromiseFactory() {
  return somethingThatCreatesAPromise();
}
```

这管用么？管用！promise factory 只有被调用时才会生成 promise，它运行的方式类似于 `then` 函数——实际上，它们就是一回事

观察上述的 `executeSequentially()` 函数，然后想象 `myPromiseFactory` 被替换进 `result.then(...)` 内部，之后豁然开朗，这时你才刚接触到 promise 的启蒙



#### 进阶错误4：如果我想要两个 promise 实例的结果呢

通常，一个 promise 实例将取决于另一个 promise 实例，但如果想要获取这两个 promise 实例的输出呢？比如说：

```javascript
getUserByName('nolan').then(function(user) {
  return getUserAccountById(user.id);
}).then(function(userAccount) {
  // 槽糕，我也需要获取 user 对象啊！
});
```

既要当个优秀的JS程序员，又要避免“爆炸金字塔”的出现，我们或许应该将 `user` 对象存储在更高一级作用域的变量中：

```javascript
var user;
getUserByName('nolan').then(function(result) {
  user = result;
  return getUserAccountById(user.id);
}).then(function(userAccount) {
  // 好了，user 和 userAccount 都可以获取到了
});
```

这办法能用，但只是凑合。我的建议是：放开先前的偏见，拥抱金字塔：

```javascript
getUserByName('nolan').then(function(user) {
  return getUserAccountById(user.id).then(function(userAccount) {
  // 你看，我也可以获取 user 和 userAccount 
  });
});
```

至少，暂时的“拥抱金字塔”:)

如果代码缩进变成了一个问题，那你可以做JS程序员“自古以来”一直在做的事——将函数提取到一个命名函数：

```javascript
function onGetUserAndUserAccount(user, userAccount) {
  return doSomething(user, userAccount);
}

function onGetUser(user) {
  return getUserAccountById(user.id).then(function(userAccount) {
    return onGetUserAndUserAccount(user, userAccount);
  });
}

getUserByName('onlan')
	.then(onGetUser)
	.then(function() {
  // 运行到这里时，doSomething() 已经执行完毕，我们的缩进归零了
});
```

当你的 promise 代码越来越复杂时，你会发现你会将越来越多的函数提取到命名函数中，代码也会变得更加美观：

```javascript
putYourRightFootIn()
	.then(putYourRightFootOut)
	.then(putYourRightFootIn)
	.then(shakeItAllAbout);
```

这就是 promise 出现的意义！



#### 进阶错误5：promise fall through 通过

这是我在上面介绍 promise 时提到的错误，极其深奥的案例，或许永远都不会出现在你的代码里，但它着实让我吓了一跳

看到如下代码你有什么想法么：

```javascript
Promise.resolve('foo').then(Promise.resolve('bar')).then(function(result) {
  console.log(result);
});
```

你可能觉得会打印 `bar`，但结果却是 `foo`

原因在于，当你向 `then()` 传递一个非函数（例如 promise 实例）时，它实际上会将其看作 `then(null)` ，这会导致前一个 promise 的结果通过。你不妨试试：

```javascript
Promise.resolve('foo').then(null).then(function(result) {
  console.log(result);
});
```

就算中间加上无数的 `then(null)`，打印的结果仍然是 `foo`

这实际上回到了之前关于 promise VS promise factories 的讨论。简单地说，你可以直接向 `then()` 方法中传入一个 promise 实例，但它却不会依照你希望的那样运行。`then()` 期望获取一个函数，所以大部分情况你的意思应该是：

```javascript
Promise.resolve('foo').then(function() {
  return Promise.resolve('bar');
}).then(function(result) {
  console.log(result); // 结果是 bar，正如我们期望的
});
```

所以记住一点：给 `then()` 传入的只能是函数（不要传别的什么）！





### 答疑

##### 题目一

```javascript
doSomething().then(function() {
  return doSomethingElse();
}).then(finalHandler);
```

答案：

```javascript
doSomething
|-----------------|
                  doSomethingElse(undefined)
                  |------------------|
                                     finalHandler(resultOfDoSomethingElse)
                                     |------------------|
```



##### 题目二

```javascript
doSomething().then(function() {
  doSomethingElse();
}).then(finalHandler);
```

答案：

```javascript
doSomething
|------------------|
                   doSomethingElse(undefined)
                   |------------------|
                   finalHandler(undefined)
                   |------------------|
```



##### 题目三

```javascript
doSomething().then(doSomethingElse())
	.then(finalHandler);
```

答案：

```javascript
doSomething
|------------------|
doSomethingElse(undefined)
|---------------------------------|
                   finalHandler(resultOfDoSomething)
                   |------------------|
```



##### 题目四

```javascript
doSomething().then(doSomethingElse)
	.then(finalHandler);
```

答案：

```javascript
doSomething
|-----------------|
                  doSomethingElse(resultOfDoSomething)
                  |-----------------|
                                    finalHandler(resultOfDoSomethingElse)
                                    |-----------------|
```



如果还是不太明白这些答案，我建议你重新再把这篇文章看一遍，或者自己定义 `doSomething()` 和 `doSomethingElse()` 这两个方法，放到浏览器里试试



> 说明：这些给出的例子，我假定 `doSomething()` 和 `doSomethingElse()` 都返回 promise，并且这些 promise 表示在 JavaScript 事件循环之外完成的事（例如 IndexedDB、network、setTimeout），这就是为什么在合适的情况下，它们显示成并发，[演示](http://jsbin.com/tuqukakawo/1/edit?js,console,output)



promise 的更多进阶用法—— [promise protips chear sheet](https://gist.github.com/nolanlawson/6ce81186421d2fa109a4)



#### 关于 promise 的最后几句话

Promise 是伟大的，如果你仍然在使用回调（嵌套），我强烈建议你讲代码转换为 promise 风格，你的代码将变得更加紧凑、优雅、更具可读性

不信？你看这个例子：[a refactor of PouchDB's map/reduce module](https://t.co/hRyc6ENYGC) 其中用 promise 取代回调，结果是：290个插入操作，555个删除操作

顺带提一句，那个写出恶心的回调嵌套代码的人，其实是我自己，所以这也算是我在 promise 原初魔力的第一课，同时感谢其他 PouchDB 的贡献者一路对我的指导

话虽如此，promise 并非完美，但就像是瘦死的骆驼比马大，promise 仍旧要好过回调，因此就目前而言，其中一个要比另一个更接近“完美”，但是如果你有机会选择更好的，那就要尽力避免它们俩

虽然优于回调，promise 仍然难以理解且极易出错，正因如此我才写下这篇文章讲明其中事实。新手和专家都会经常混淆这些概念，真的，这不是他们的错，问题在于 promise ——虽然近似于我们在同步代码中使用的模式，看起来是个不错的替代品，但却不是百分之百相同

实际上，你不应该学习一堆神秘的规则和新的 API 来做这些事，在同步的世界里，你可以完全使用你熟悉的模式，如 `return`、`catch`、`throw`和 `for` 循环。你的脑子里不应该始终有两套系统并行！



#### Awaiting async/await

这是我另一篇文章 [Taming the asynchronous beast with ES7](http://pouchdb.com/2015/03/05/taming-the-async-beast-with-es7.html) 中讨论的要点，其中我研究了 ES7 的关键字 `async`/`await`，以及它们如何将 promise 深层次的融合到 JS 这门语言中。相比于写伪同步代码（其中提供的 `catch()` 方法看似与 `catch` 一致，实则不然），ES7 允许我们使用真正的 `try`/`catch`/`return` 关键字，正如我们在 CS 101 里学到的那样



就 JS 这门语言来说，这是一个巨大的福音。因为最终，只要工具不在我们犯错时及时提示，这些 promise 反模式就总是会不经意出现



以 JS 的历史为例，我认为 [JSLint](http://jslint.com/) 和 [JSHint](http://jshint.com/) 对 JS 社区做出的贡献要比 [JS 语言精粹](http://amzn.com/0596517742) 要多，即使它们有效的包含相同的信息。这就像是“明确指出你的代码中何处出错了”和“通过阅读一本书来试图搞清楚别人的代码是怎么出错的”的区别



ES7 的 `async`/`await` 的美在于，在大多数情况下，代码中的错误会将自己显示为语法、编译器错误，而不是运行时的微妙的错误。不过，在那之前，熟悉并掌握 promise，以及如何在 ES5 和 ES6 中正确使用是一件好事

但我却意识到一点，就像是《JS 语言精粹》这本书一样，这篇文章所能提供的影响是有限的。但当你看到他们犯同样的错误时，你完全有能力指出其中的错误，因为我们中有太多的人需要承认一点：我还并没有百分之百掌握 promise ！



> 更新：需要指出的一点是：Bluebird 3.0 将会提供告警信息以避免我在本文中提到的许多错误。所以，在 ES7 完全推出之前，Bluebird 会是另一个非常不错的选择



[原文地址](https://pouchdb.com/2015/05/18/we-have-a-problem-with-promises.html)

