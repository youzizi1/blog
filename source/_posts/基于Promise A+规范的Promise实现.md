---
title: 基于Promise A+规范的Promise简单实现
date: 2019-10-18 15:29:45
tags:
  - Promise
author: ugu
category: JavaScript
---

首先我们创建`promise.js`文件，用于初始化`Promise`构造函数。

```js
//promise.js
function Promise(task){
  const that = this

  // 初始化状态
  that.status = 'pending'

  // 存放回调
  that.successCbs = []
  that.failureCbs = []

  // 初始化传递的value值
  that.value = undefined

  function resolve(value){
    if (that.status === 'pending') {
      that.status = 'fulfilled'
      that.value = value
      that.successCbs.forEach(item => item(that.value))
    }
  }

  function rejected(value){
    if (that.status === 'pending') {
      that.status = 'rejected'
      that.value = value
      that.failureCbs.forEach(item => item(that.value))
    }
  }

  // 可能会在实例化时候抛出错误
  try {
    task(resolve, rejected)
  }catch (e) {
    rejected(e)
  }
}

Promise.prototype.then = function(successCb, failureCb){
  const that = this

  that.successCbs.push(successCb)
  that.failureCbs.push(failureCb)
}

module.exports = Promise
```

上面这个Promise实现了如果task是异步的情况，如果是task是同步任务，那么就会失效。因为此时，会先调用task任务，后调用then方法，所以导致then其中的回调函数没有被push进回调数组中。也就是说数组中没有回调函数可执行。

```js
// promise.js
function Promise(task){
  const that = this

  // 初始化状态
  that.status = 'pending'

  // 存放回调
  that.successCbs = []
  that.failureCbs = []

  // 初始化传递的value值
  that.value = undefined

  function resolve(value){
    if (that.status === 'pending') {
      that.status = 'fulfilled'
      that.value = value
      that.successCbs.forEach(item => item(that.value))
    }
  }

  function rejected(value){
    if (that.status === 'pending') {
      that.status = 'rejected'
      that.value = value
      that.failureCbs.forEach(item => item(that.value))
    }
  }

  // 可能会在实例化时候抛出错误
  try {
    task(resolve, rejected)
  }catch (e) {
    rejected(e)
  }
}

Promise.prototype.then = function(successCb, failureCb){
  const that = this

  const status = that.status
  switch (status) {
    case 'pending':
      that.successCbs.push(successCb)
      that.failureCbs.push(failureCb)
      break;
    case 'fulfilled':
      successCb(that.value)
      break;
    case 'rejected':
      failureCb(that.value)
      break;
    default:
      break;
  }
}

module.exports = Promise
```

这样就支持同步代码，下面是测试代码。

```js
//demo.js
const Promise = require('./promise')

const p = new Promise((resolve, reject) => {
    resolve(1000)
})

p.then(res => {
    console.log(res)
},err => {
    console.log(err)
})
```

但是上面还是有问题，例如如果第一次then的err回调函数中返回了信息，应该要走到下一次then的success回调函数中去执行。

```js
//demo.js
const Promise = require('./promise')

const p = new Promise((resolve, reject) => {
    reject(1000)
})

p.then(res => {
    console.log(res)
},err => {
    console.log(err)
    return 'hi,ugu'
})

p.then(res => {
    console.log(res)
    console.log('执行res回调')
},err => {
    console.log(err)
})
/*
	实际上第二次then的success回调并没有执行，所以两次then都走了err回调，输出两次1000
*/
```



