---
title: Promise的简单实现
date: 2019-10-18 15:29:45
tags:
  - Promise
author: ugu
category: JavaScript
---

`Promise`的目的是为异步操作提供统一接口，是常见的异步解决方案。

## 状态

`Promise`对象只有三种状态：

- `pending`：等待状态
- `resolved`：成功态
- `rejected`：失败态

`Promise`状态只能从等待状态转换为成功态或者失败态，成功态和失败态无法互相转换，并且只能转换一次。一旦状态改变，就不会在发生变化了。

## 方法和属性

`Promise`实例上的属性和方法有：

* `then`：当状态发生改变时，会执行对应的回调
* `catch`：用于捕获错误

* ...

注意：实例化`Promise`的时候，需要传递一个执行器函数，这个函数是同步执行的。

```js
new Promise(function(resolve, reject){
    console.log(1)
})
console.log(2)
// 1 2
```

### Promise.all()

当所有子Promise都完成，那么该Promise就完成，返回值是全部值所组成的数组。如果其中一个子Promise失败，那么该Promise就失败，返回值是第一个失败的子Promise的结果。

```javascript
let p1 = new Promise(res => {
  setTimeout(() => {
    res("promise1")
  }, 2000)
})
let p2 = new Promise(res => {
  setTimeout(() => {
    res("promise2")
  }, 2000)
})

Promise.all([p1, p2]).then(res => {
  console.log(res)
}).catch(err => console.log(err))
/*
	['promise1', 'promise2']
*/
```

### Promise.resolve()和Promise.reject()

```javascript
Promise.reject("err data").then(null,mes=>console.log(mes))
/*
	输出："err data"
*/
```

### Promise.race()

类似Promise.all()，区别在于它有任意一个完成就算完成。

```javascript
let p1 = new Promise(res=>{
  setTimeout(()=>res("p1"),1000)
})
let p2 = new Promise(res=>{
  setTimeout(()=>res("p2"),3000)
}) 
Promise.race([p1,p2]).then(value=>console.log(value))
/*
	只输出"p1"
*/
```

## 实现Promise

下面就是最简单的`Promise`实现，

```js
function Promise(executor) {
    this.status = 'pending'
    this.value = null
    this.reason = null
    
    const resolve = value => {
		if(this.status === 'pending') {
            this.status = 'resolved'
            this.value = value
        }        
    }
    
    const reject = reason => {
        if(this.status === 'pending') {
            this.status = 'rejected'
            this.reason = reason
        }
    }
    
    executor(resolve, reject)
}

Promise.prototype.then = function(successCb, errorCb) {
    if(this.status === 'resolved') {
        successCb(this.value)
    }
    if(this.status === 'rejected') {
        errorCb(this.reason)
    }
}
```

下面对其做如下改进：

* `executor`函数中可以有异步任务

  ```js
  const p = new Promise(function(resolve, reject){
      setTimeout(function(){
          resolve()
      },1000)	
  })
  p.then(res => {
      console.log(1)
  })				// 1s输出1
  ```

```js
function Promise(executor) {
    this.status = 'pending'
    this.value = null
    this.reason = null
    
    // 存放回调数组
    this.successCbArr = []
    this.errorCbArr = []
    
    const resolve = value => {
		if(this.status === 'pending') {
            this.status = 'resolved'
            this.value = value
            
            this.successCbArr.forEach(cb => cb())
        }        
    }
    
    const reject = reason => {
        if(this.status === 'pending') {
            this.status = 'rejected'
            this.reason = reason
            
            this.errorCbArr.forEach(cb => cb())
        }
    }
    
    try {
    	executor(resolve, reject)
    }catch(e) {
        reject(e)
    }
}

Promise.prototype.then = function(successCb, errorCb) {
    if(this.status === 'resolved') {
        successCb(this.value)
    }
    if(this.status === 'rejected') {
        errorCb(this.reason)
    }
    if(this.status === 'pending') {
        this.successCbArr.push(successCb)
        this.errorCbArr.push(errorCb)
    }
}
```

实际上，`then`方法除了支持回调，还有如下特点：

* 如果`then`中返回一个普通值，那么会作为下一个`then`方法中成功回调的参数

* 如果`then`中抛出了一个错误，那么会作为下一个`then`方法中失败回调的参数

* 如果`then`中抛出一个新的`Promise`，如果是成功态`Promise`，那么会触发下一个`then`的成功回调；如果是失败态`Promise`，会触发下一个`then`的失败回调

* `then`方法是异步的

  ```js
  new Promise((resolve, reject) => {
      console.log(1)
      resolve()
  }).then(data=>{
      console.log(2)
  })
  console.log(3)
  
  // 输出：1 3 2
  ```

另外，`Promise`还有一个特点——值的穿透。

```js
const p = new Promise((resolve, reject) => resolve('hello world'))

p.then().then().then(data => console.log(data))			// 'hello world'
```

## 实现Promise.resolve

```js
Promise.resolve = value => {
    return new Promise((resolve, reject) => {
        resolve(value)
    })
}
```

同理，`Promise.reject`实现如下：

```js
Promise.reject = err => {
    return new Promise((resolve, reject) => {
        reject(err)
    })
}
```

## 实现Promise.all

```js
Promise.all = function(values) {
    return new Promise((resolve, reject) => {
        let arr = []
        let index = 0

        function processData(key, value) {
            index++
            arr[key] = value

            if(index === values.length) {
                resolve(arr)
            }
        }

        for(let i = 0; i < values.length; i++) {
            let current = values[i]

            if(current && current.then && typeof current.then === 'function') {
                current.then(y => {
                    processData(i, y)
                }, reject)
            }else {
                processData(i ,current)
            }
        }
    })
}
```

## 实现Promise.race

```js
Promise.race = function(values) {
    return new Promise((resolve, reject) => {
        for(let i = 0; i < values.length; i++) {
            let current = values[i]

            if(current && current.then && typeof current.then === 'function') {
                current.then(resolve, reject)
            }else {
                resolve(current)
            }
        }
    })
}
```

