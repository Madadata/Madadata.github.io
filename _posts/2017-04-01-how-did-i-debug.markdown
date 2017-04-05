---
layout: post
title:  我是如何找到 Expression 应用延迟原因的
date:   2017-04-01 18:00:00+08:00
categories: 技术笔记
tags: 马达数据 Express 后端 服务器 debug
---

---------
> ### 摘要：你永远不会知道在 debug Web 服务器时会遇到什么坑 -- 但这才是它最有趣的地方！
--------

* *作者/ 马达数据CTO 刘嘉瑜*


![封面图](/images/2017/4/1/cover.jpg)


最近我发现我的 Express 应用特别慢。


背景介绍是这样：我们在 ** Madadata ** 里建立平台，在这个平台中，我们使用了一个外部服务 Leancloud 来提供用户身份验证和注册。 但是，为了在对 API 进行测试时，不用忍受从加州（我们 CI 服务器所在地）到上海（我们的服务提供商服务器所在地）发请求的痛苦，我用 Express 和 Mongoose 写了一个简单的模拟 API 服务。


在我们最近开始进行负载测试前，我们都没有意识到这个模拟服务的延迟：超过一半的请求在1秒内没有返回，从而导致负载测试失败。作为一个简单的使用 Mongoose 的 Express 应用程序，几乎没有任何写错的机会，至少，不会延迟1秒这么多。


![第一次跑 mocha 测试结果](/images/2017/4/1/1.png)


上面，本地进行 mocha 测试时的截图显示，很明显 API 服务确实有些问题！


## 什么地方出了错？


从屏幕截图我可以看出，并不是所有API都是慢的：用户登出的那个 API，以及显示当前个人档案的 API 都速度正常。此外，从我用 morgan 打印出来的开发日志中，我发现那些速度缓慢的 API，由 Express 收集的响应时间都显示出一致的延迟水平（即，那些用红色标记的 API，你可以看到，他们的总延迟大致分别来自两个请求）。


这实际上排除了“延迟是因为连接问题”的可能性（而是因为 Express 应用本身）。所以下一步，我看了一下我的 Express 应用程序。（注意，这实际上是值得排除，我个人建议尝试一两个其它工具，而不是来关注 ``` mocha ``` ，例如尝试 ``` curl ``` 甚至 ``` nc ``` ，然后再继续，因为它们几乎总是比你写的测试代码更可靠）。


## Express 应用内部


如果提起 Node 的 Web 服务器，Express 真的是一个很好的框架，它在速度和可靠性方面已经取得了很大进展。我想，延迟可能主要是因为我用的 Express 中的插件和中间件。


为了使用 MongoDB 作为会话存储，我使用了 connect-mongo 来配合我的 expression session。我也使用了相同的 MongoDB instance 作为我的主凭证和配置文件存储（为什么不呢？毕竟，这是个 CI 测试的服务）。因此，我使用了 Mongoose 作为 ODM 。


起初，我怀疑可能是因为使用了 Mongoose 内置的 Promise library。但是，在我换成了 ES6 原生实现之后，问题并没有解决。


然后，我就觉得应该检查一下模型序列化和验证部分。 应用里只有一个模型，它相当简单直接：


```javascript
const mongoose = require('mongoose')
const Schema = mongoose.Schema
const isEmail = require('validator/lib/isEmail')
const isNumeric = require('validator/lib/isNumeric')
const passportLocalMongoose = require('passport-local-mongoose')

mongoose.Promise = Promise

const User = new Schema({
  email: {
    type: String,
    required: true,
    validate: {
      validator: isEmail
    },
    message: '{VALUE} 不是一个合法的 email 地址'
  },
  phone: {
    type: String,
    required: true,
    validate: {
      validator: isNumeric
    }
  },
  emailVerified: {
    type: Boolean,
    default: false
  },
  mobilePhoneVerified: {
    type: Boolean,
    default: false
  },
  turbineUserId: {
    type: String
  }
}, {
  timestamps: true
})

User.virtual('objectId').get(function () {
  return this._id
})

const fields = {
  objectId: 1,
  username: 1,
  email: 1,
  phone: 1,
  turbineUserId: 1
}

User.plugin(passportLocalMongoose, {
  usernameField: 'username',
  usernameUnique: true,
  usernameQueryFields: ['objectId', 'email'],
  selectFields: fields
})

module.exports = mongoose.model('User', User)
```


## Mongose Hooks


Mongoose 有一个很好的功能，你可以使用 ``` pre- ``` 和 ``` post-hooks ``` 来检查文档的验证和保存过程。


使用 ``` console.time ``` 和 ``` console.timeEnd ``` ，我们就可以实际测量在这些进程中花费的时间。


```javascript
User.pre('init', function (next) {
  console.time('init')
  next()
})
User.pre('validate', function (next) {
  console.time('validate')
  next()
})
User.pre('save', function (next) {
  console.time('save')
  next()
})
User.pre('remove', function (next) {
  console.time('remove')
  next()
})
User.post('init', function () {
  console.timeEnd('init')
})
User.post('validate', function () {
  console.timeEnd('validate')
})
User.post('save', function () {
  console.timeEnd('save')
})
User.post('remove', function () {
  console.timeEnd('remove')
})
```


然后我们得到了 mocha 运行中更详细的信息：


![第二次跑 mocha 测试结果](/images/2017/4/1/2.png)


显然，文档验证和保存根本不是造成延迟的主要原因。它也排除了这两个可能性：1）延迟来自于 Express 应用程序和 MongoDB 服务器之间的连接问题，或者 2）MongoDB 服务器本身运行缓慢。


## Passport + Mongoose


当我把焦点从 Mongoose 身上移开，我开始看到我使用的 passport 插件：passport-local-mongoose。


这个名字是有点长，但它基本上告诉了你它是干嘛的。Passport 负责会话管理、注册和登录样板，而Passport-local-mongoose 则将 Mongoose 转变为 passport 的本地策略。


这个 library 小而且简单，所以我开始直接在 ``` node_modules/ ```文件夹 中编辑我的 ``` index.js ``` 文件。由于函数 ``` #register (user, password, cb)```  调用函数 ``` #setPassword (password, cb)``` ，也就是这一行，所以我开始注意后者。在添加了更多的 ``` console.time```  和 ``` console.timeEnd ``` 之后，我确认了，原来延迟主要是由于这个函数调用：


```javascript
pbkdf2(password, salt, function(pbkdf2Err, hashRaw) {
  // omit
}
```


## PBKDF2


这个名称本身就表示它是一个加密 library 的调用。再看 README 可以发现，这个 library 使用了 ``` 25,000```  次迭代。


像 bcrypt 一样，``` pbkdf2 ```  也是一个缓慢的哈希算法，也就是说，它就是偏延迟的，这个延迟可以在迭代时进行调整，来适应不断增强的计算能力。这个概念被称为：密钥延伸 (key streching)。


如维基百科里写的，最开始提出的迭代次数是 ``` 1,000 ``` 次，而最近一次的更新达到了 ``` 100,000 ``` 次。所以其实默认的   ```25,000 ``` 次是合理的。


在将迭代减少到 ``` 1,000 ``` 后，我的 mocha 测试输出如下：


![修改迭代次数后再跑 mocha 测试的结果](/images/2017/4/1/3.png)


最后，终于，这个延迟和安全性变得可以接受了，毕竟它只是个测试应用程序！注意，我为我的测试应用程序做了这个更改，并不意味着你也要减少你的应用程序的迭代次数。另外，将迭代次数设置得太高，会使应用程序容易受到 DoS 攻击。


## 最后的想法


我想，分享一些 debug 经验还是有意义的，我很高兴这不真的是一个 bug（对，是一个伪装起来的功能）。


另外值得一提的是，对于对计算机安全或密码学不是很了解的开发人员来说，通常，最好不要自己写一些与 会话 / 密钥 / 令牌管理 相关的代码。使用好的、如 passport 这样的开源库，会更好。


不过，你永远不会知道在 debug Web 服务器时会遇到什么坑——但这才是它最有趣的地方！



> 原链接地址 [马达数据官方公众平台](http://chuansong.me/n/1726635752327)，[刘嘉瑜的博客](http://www.jiayul.me/hacking/2017/03/18/how-i-discovered-the-cause-of-slowness-in-my-express-app.html)，[掘金](https://juejin.im/post/58ddcfa4b123db00603f3073)
