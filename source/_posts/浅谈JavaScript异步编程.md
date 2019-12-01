---
title: 浅谈JavaScript异步编程
date: 2019-9-14 23:22:18
tags:
  - Promise
  - Async/Await
  - Generator
  - JavaScript
author: ugu
category: JavaScript
---

## 异步编程

有异步I/O，必有异步编程。

解决异步常见的方式有：

* 回调函数
* 发布订阅
* Promise
* Generator
* Async/Await

## 回调函数

```js
function fn(cb) {
  setTimeout(() => {
    console.log('fn is called firstly')
    cb()
  }, 2000)
}

fn(() => console.log('cb is called secondly'))
```

缺点：

* 异步代码`try/catch失效`，因为`try/catch`只能捕获当前事件循环内的异常。所以Node中往往将err作为回调的第一个参数，便于捕获错误
* 回调地狱
* 结构混乱，强耦合
* 流程难以追踪

> Node设计哲学之一：error-first callbacks。

## 发布订阅

```js
const fs = require('fs')
const EventEmitter = require('events')
const e = new EventEmitter()

e.on('publish', (key, value) => {
  console.log(key, value)		// 执行两次
})

function read() {
  fs.readFile('../files/a.txt', 'utf8', (err, content) => {
    if (!err) {
      e.emit('publish', 'a', content)
    }
  })

  fs.readFile('../files/b.txt', 'utf8', (err, content) => {
    if (!err) {
      e.emit('publish', 'b', content)
    }
  })
}

read()
```

## Promise/Deferred

Promise/Deferred模式在09年被Kris Zyp抽象为一个提议草案，发布在CommonJS规范中。目前草案已经抽象出了Promise/A、Promise/B、Promise/D这样典型的Promise/deferred模型。

```js
const p = new Promise((resolve, reject) => {
  setTimeout(() => {
    resolve('success')
  }, 2000)
})

p.then(data => {
  console.log(data)			// 'success'
}).catch(err => {
  console.log(err)
})
```

另外，`Promise`还可以用来模拟`sleep`功能。

```js
const sleep = delay => {
    return new Promise((resolve, reject) => {
        setTimeout(() => {
            try {
                resolve(1)
            }catch(e) {
                reject(0)
            }
        }, delay)
    })
}
```

## Generator

很多时候希望在函数执行时，能够控制内部执行的流程。

```js
// 生成器函数
function* gen(x) {
  yield x + 1
  yield x + 2
  return x + 3
}

const it = gen(1)			// 返回迭代器

console.log(it.next())
console.log(it.next())
console.log(it.next())
console.log(it.next())

/*
	{ value: 2, done: false }
	{ value: 3, done: false }
	{ value: 4, done: true }
	{ value: undefined, done: true }
*/
```

一般我们都是通过`Generator`配合`co`库来编写异步代码。

```js
const fs = require('fs')
const co = require('co')

function readFilePromisify(filename) {
  return new Promise((resolve, reject) => {
    fs.readFile(filename, (err, content) => {
      if (err) {
        reject(err)
      } else {
        resolve(content)
      }
    })
  })
}

function* read() {
  const aContent = yield readFilePromisify('../files/a.txt')
  const bContent = yield readFilePromisify('../files/b.txt')

  return aContent + '' + bContent
}

co(read).then(data => {
  console.log(data)
}).catch(err => {
  console.log(err)
})
```

`co`库的原理如下：

```js
function co(generator) {
  return new Promise((resolve, reject) => {
    const it = generator()

    ;(function next(lastVal) {
      const {value, done} = it.next(lastVal)

      if(done) {
        resolve(value)
      }else{
        value.then(next, reason => reject(reason))
      }
    })();
  })
}
```

## async/await

`async/await`本质上是`co`和`generator`的语法糖，具有如下特点：

* `await`后面必须跟上一个`Promise`
* `async`函数执行返回的也是一个`Promise`
* `await`必须嵌套在`async`函数内

```js
const fs = require('fs')
const util = require('util')
// util模块还提供promisify的能力
const readFilePromisify = util.promisify(fs.readFile)

async function read() {
  const aContent = await readFilePromisify('../files/a.txt')
  const bContent = await readFilePromisify('../files/b.txt')
  return aContent + '' + bContent
}

const result = read()

result.then(data => console.log(data))
```

实际上，我们需要通过`uril.promisify`将`Node.js`中的异步`API`转换为`Promise`形式，这个工具方法的原理如下：

```js
// promisify.js
module.exports = {
    promisify(fn){
        return function (...args) {
            return new Promise(function (resolve, reject) {
                fn.apply(null,[...args,function(err,data){
                    err?reject(err):resolve(data)
                }])
            })
        }
    },
    promisifyAll(obj){
        for(var attr in obj){
            if(obj.hasOwnProperty(attr) && typeof obj[attr] =='function'){
                obj[attr+'Async'] = this.promisify(obj[attr]);
            }
        }
        return obj;
    }
}

// app.js
const promise = require('./promisify')
const readFilePromisify = promise.promisify(fs.readFile)
```































































































































  







































