---
layout: post
title:  "NodeJS: 如何在 Express 中处理异步错误"
date:   2017-03-21 17:30:00+08:00
categories: 技术笔记
tags: 马达数据 商业智能 BI 技术 编程
---

---------
> ### 摘要：如何在 Express 中使用 Promise, Generators 和 Async/Await 处理异步错误
--------

*翻译&编辑/鹤爷*

*原文/Marc Harter*


![NodeJS](/images/2017/3/21/1.jpg)


### 摘要


1. 比起回调函数，使用 Promise 来处理异步错误要显得优雅许多。

2. 结合 Express 内置的错误处理机制和 Promise 极大地降低产生未捕获错误（uncaught exception）的可能性。

3. Promise 在ES6中是默认选项。如果使用 Babel 转译，它也可以与 Generators 或者 Async/Await 相结合。



本文主要阐述如何在 Express 中使用错误处理中间件（error-handling middleware）来高效处理异步错误。在 Github 上有对应[代码实例](https://github.com/strongloop-community/express-example-error-handling)可供参考。


首先，让我们一起了解 Express 提供的开箱即用的错误处理工具。然后，我们将探讨如何使用 Promise, Generators 以及 ES7 的 async/await 来简化错误处理流程。


### Express 内置的异步错误处理


在默认情况下，Express 会捕获所有在路由处理函数中的抛出的异常，然后将它传给下一个错误处理中间件：


```javascript
app.get('/', function (req, res) {
  throw new Error('oh no!')
})
app.use(function (err, req, res, next) {
  console.log(err.message) // 噢！不!
})
```


对于同步执行的代码，以上的处理已经足够简单。然而，当异步程序在执行时抛出异常的情况，Express 就无能为力。原因在于当你的程序开始执行回调函数时，它原来的栈信息已经丢失。


```javascript
app.get('/', function (req, res) {
  queryDb(function (er, data) {
    if (er) throw er
  })
})
app.use(function (err, req, res, next) {
  // 这里拿不到错误信息
})
```


对于这种情况，可以使用 **next** 函数来将错误传递给下一个错误处理中间件


```javascript
app.get('/', function (req, res, next) {
  queryDb(function (err, data) {
    if (err) return next(err)
    // 处理数据

    makeCsv(data, function (err, csv) {
      if (err) return next(err)
      // 处理 csv

    })
  })
})
app.use(function (err, req, res, next) {
  // 处理错误
})
```


使用这种方法虽然一时爽，却带来了两个问题：


1. 你需要显式地在错误处理中间件中分别处理不同的异常。

2. 一些隐式异常并没有被处理（如尝试获取一个对象并不存在的属性)


### 利用 Promise 传递异步错误


在异步执行的程序中使用 Promise 处理任何显式或隐式的异常情况，只需要在 Promise 链尾加上 **.catch(next)** 即可。


```javascript
app.get('/', function (req, res, next) {
  // do some sync stuff
  queryDb()
    .then(function (data) {
      // 处理数据
      return makeCsv(data)
    })
    .then(function (csv) {
      // 处理 csv
    })
    .catch(next)
})
app.use(function (err, req, res, next) {
  // 处理错误
})
```


现在，所有异步和同步程序都将被传递到错误处理中间件。棒棒的。


虽然 Promise 让异步错误的传递变得容易，但这样的代码仍然有一些冗长和刻板。这时候 promise generator 就派上了用场。


### 用 Generators 简化代码


如果你使用的环境原生支持 Generators，你可以手动实现以下的功能。不过这里我们将借用 Bluebird.coroutine 来说明如何使用 Promise generator 来简化刚才的代码。


> 尽管接下来的例子使用的是 **bluebird** ，其它 Promise 库（如 co）也都支持 Promise generator.


首先，我们需要使得 Express 路由函数与 Promise generator 兼容：


```javascript
var Promise = require('bluebird')
function wrap (genFn) { // 1
    var cr = Promise.coroutine(genFn) // 2
    return function (req, res, next) { // 3
        cr(req, res, next).catch(next) // 4
    }
}
```


这个函数是一个高阶函数，它做了以下几件事情：（分别与代码片段中的注释对应）


1. 以 Genrator 为唯一的输入

2. 让这个函数懂得如何 yield promise

3. 返回一个普通的 Express 路由函数

4. 当这个函数被执行时，它会使用 coroutine 来 yield promise，捕获期间发生的异常，然后将其传递给 next 函数


借助这个函数，我们就可以这样构造路由函数：


```javascript
app.get('/', wrap(function *(req, res) {
  var data = yield queryDb()
  // 处理数据
  var csv = yield makeCsv(data)
  // 处理 csv
}))
app.use(function (err, req, res, next) {
  // 处理错误
})
```


现在，Express 的异步错误处理流程的可读性已经近乎令人满意，而且你可以像写同步执行的代码一样去书写异步执行的代码，唯一不要忘了的就是 yield promises。


然而这还不是终点，ES7 的 async/await 提议可以让代码变得更简洁。


### 使用 ES7 async/await


ES7 async/await 的行为就像 Promise Generator 一样，只不过它可以被用到更多的地方（如类方法或者胖箭头函数）。


为了在 Express 中使用 async/await，同时优雅地处理异步错误，我们仍然需要一个与上文提到的 **wrap** 类似的函数：


```javascript
let wrap = fn => (...args) => fn(...args).catch(args[2])
```


这样，我们就可以按底下这种方式书写路由函数:


```javascript
app.get('/', wrap(async function (req, res) {
  let data = await queryDb()
  // 处理数据
  let csv = await makeCsv(data)
  // 处理 csv
}))
```


### 现在可以愉快地写代码了


有了对同步和异步错误的处理，你可以用新的方式来开发 Express App。但有两点需要注意：


1. 要习惯使用 **throw** ，它使得你的代码目的明确，throw 会明确地将程序引到错误处理中间件，这对同步或异步的程序都是适用的。


2. 遇到特殊情况，当你觉得有必要时，也可以自行 try/catch。


   ```javascript
   app.get('/', wrap(async (req, res) => {
     if (!req.params.id) {
       throw new BadRequestError('Missing Id')
     }
     let companyLogo
     try {
       companyLogo = await getBase64Logo(req.params.id)
     } catch (err) {
       console.error(err)
       companyLogo = genericBase64Logo
     }
   }))
   ```


3. 要习惯使用 **custom error classes**，如 BadRequestError，因为这可以让你在错误处理中间件中更方便地分类处理。


   ```javascript
   app.use(function (err, req, res, next) {
     if (err instanceof BadRequestError) {
       res.status(400)
       return res.send(err.message)
     }
     ...
   })
   ```


### 需要注意


1. 以上介绍的方法要求所有异步操作必须返回 promise。如果你的异步操作是使用回调函数的方式，你需要将其转化成 promise。（可以直接使用 Bluebird.promisifyAll 这类函数）


2. 事件发射器（如 steams）仍然会导致未捕获异常，你需要注意合理地处理这类情况：

   ```



> 原链接地址 [马达数据官方公众平台](https://mp.weixin.qq.com/s/pjyDni408qfZFR1_a4HJbQ),  [Strong Loop](https://strongloop.com/strongblog/async-error-handling-expressjs-es7-promises-generators/)
