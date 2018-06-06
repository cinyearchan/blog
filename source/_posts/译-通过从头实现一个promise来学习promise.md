---
title: '[译]通过从头实现一个promise来学习promise'
date: 2018-06-06 21:45:14
tags:
---
[原文地址：Learn JavaScript Promises by Building a Promise from Scratch
](https://levelup.gitconnected.com/understand-javascript-promises-by-building-a-promise-from-scratch-84c0fd855720)

---

通过自建一个 `Promise` 逐步了解 `Promise` 的工作原理

你以前可能见过类似下方的代码：

```javascript
fetch('/user/1')
  .then((user) => {
      /* Do something with user after the API returns */
  })
```

`.then()` 中包裹的代码块在执行之前，会一直等待，直到接收到来自服务器的响应。这就叫做 `Promise`。但千万不要被这花名和其中的异步代码吓退——一个 `Promise` 只是一个很普通的旧的 `JavaScript` 对象，它具有特殊的方法，可以让你同步执行代码(即使有延迟，它也会按顺序执行)

```javascript
typeof new Promise((resolve, reject) => {}) === 'object'
// true
```

重申一遍(原作者语：我第一次学习 `promise` 时，我很难把握 `promise` 的要点)，`Promise` 只是一个对象。为了保证会等待服务器并在服务器返回响应后执行 `.then()` 方法链中的代码，你<b>必须</b>返回一个 `Promise` 对象。这跟某些开箱即用的函数是两码事。下面的例子中，`fetch` 函数就是这么个函数

```javascript
const fetch = function(url) {
    return new Promise((resolve, reject) => {
        request((error, apiResponse) => {
            if (error) {
                reject(error)
            }
            
            resolve(apiResponse)
        })
    })
}
```
上述的 `fetch()` 函数向服务器发送一个 http 请求，但客户端并不知道服务器何时返回结果。所以，`JavaScript` 会在等待服务器返回结果期间，执行其他无关的代码，一旦客户端接收到服务器返回的响应，它就会通过调用 `resolve(apiResponse)` 开始执行 `.then()` 语句中的代码。

---

现在让我们仔细看看 `Promise` 到底是如何做到这一点的：

```javascript
class PromiseSimple {
    constructor(executionFunction) {
        this.promiseChain = [];
        this.handleError = () => {};
        
        this.onResolve = this.onResolve.bind(this)
        this.onReject = this.onReject.bind(this)
        
        executionFunction(this.onResolve, this.onReject)
    }
    
    then(onResolve) {
        this.promiseChain.push(onResolve)
    	
    	return this
    }
    
    catch(handleError) {
        this.handleError = handleError
        
        return this
    }
    
    onResolve(value) {
        let storedValue = value
        
        try {
            this.promiseChain.forEach((nextFunction) => {
                storedValue = nextFunction(storedValue)
            })
        } catch (error) {
            this.promiseChain = [];
            
            this.onReject(error)
        }
    }
    
    onReject(error) {
        this.handleError(error)
    }
}
```

> 注意：上述版本的 `Promise` 只是用了学习 `Promise` 工作原理的，省略了一些更高级的功能，只提炼了最核心的部分

### 工作原理

我将其命名为 `PromiseSimple` ，防止在 Chrome 控制台运行上述代码时，原生的 `Promise` 被覆盖。上述版本的 promise 实现有一个构造函数，两个公共方法(与原生的 `then()` 和 `catch()` 类似)，两个内置方法： `onResolve()` 和 `onReject()`


1. 当你创建一个 promise 时，你会像这样创建： `new Promise((resolve, reject) => {/* ... */})`。你会在构造函数中向 promise 传递一个回调函数，我命名为 `executionFunction`。执行函数会携带 `resolve` 和 `reject` ，映射为内置的 `onResolve` 和 `onReject` 方法。这些方法都会在 `fetch` 调用 resolve 或 reject 时被调用。


2. 构造函数也会创建一个 `promiseChain` 数组和 `handleError` 函数。当 promise 后面接了一串 `.then(() => {})` 时，它会将每一个函数添加到 `promiseChain` 中；当用户调用 `catch(() => {})` 时，它会将函数分配给内部的 `handleError`。注意，`then()` 和 `catch` 都有 `return this`，以便可以链式调用 `then()`

> 注意：对原生的 `Promise` 而言，它的 `then()` 和 `catch()` 函数都会返回一个 `new Promise`，而上述简版 `Promise` 只是返回了 `this`。另外，多个 `.catch()` 也可以链式调用，不一定要接在 `.then()` 方法链的末尾


3. 当你的异步代码调用了 `resolve(apiResponse)`，自建的 promise 对象便会开始执行 `onResolve(apiResponse)`：迭代整个 `promiseChain` ，取出队首的方法，传入 `storedValue` 中最近保存的值并执行，然后用最近执行的结果更新 `storedValue`。它会按顺序执行 `promiseChain` 中保存的函数，借此创建同步的 `promise` 链。


4. 该循环（上述的迭代）被封装在 `try/catch` 块中，以便捕获运行时的错误。如果你的异步代码调用了 `reject(error)` 或者 `try/catch` 捕获了一个错误，它将被传递给 `onReject()` 方法，该方法调用作为参数传递给 `.catch()` 的回调函数。

### promiseSimple with test example

### 具体的实现及使用

```javascript
class PromiseSimple {
    constructor(executionFunction) {
        this.promiseChain = [];
        this.handleError = () => {}
    	
    	this.onResolve = this.onResolve.bind(this)
    	this.onReject = this.onReject.bind(this)
    
    	executionFunction(this.onResolve, this.onReject)
    }
    
    then(onResolve) {
        this.promiseChain.push(onResolve)
        
        return this
    }
    
    catch(handleError) {
        this.handleError = handleError
        
        return this
    }
    
    onResolve(value) {
        let storedValue = value
        
        try {
            this.promiseChain.forEach((nextFunction) => {
                storedValue = nextFunction(storedValue)
            })
        } catch (error) {
            this.promiseChain = [];
            
            this.onReject(error)
        }
    }
    
    onReject(error) {
        this.handleError(error)
    }
}

// test example
fakeApiBackend = () => {
    const user = {
        username: 'treyhuffine',
        favoriteNumber: 42,
        profile: 'https://gitconnected.com/treyhuffine'
    }
    
    if (Math.random() > .05) {
    	return {
            data: user,
            statusCode: 200
    	}
    } else {
        const error = {
            statusCode: 404,
            message: 'Could not find user',
            error: 'Not Found'
        }
        
        return error
    }
}

const makeApiCall = () => {
    return new PromiseSimple((resolve, reject) => {
        setTimeout(() => {
            const apiResponse = fakeApiBackend()
            
            if (apiResponse.statusCode >= 400) {
                reject(apiResponse)
            } else {
                resolve(apiResponse.data)
            }
        }, 5000)
    })
}

makeApiCall()
	.then((user) => {
        console.log('In the first .then()')
        
        return user
	})
	.then((user) => {
        console.log(`User ${user.username}'s favorite number is ${user.favoriteNumber}`)
        
        return user
	})
	.then((user) => {
        console.log('The previous .then() told you the favoriteNumber')
        
        return user.profile
	})
	.then((profile) => {
        console.log(`The profile URL is ${profile}`)
	})
	.then(() => {
        console.log('This is the last then()')
	})
	.catch((error) => {
        console.log(error.message)
	})
```