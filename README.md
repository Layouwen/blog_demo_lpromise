## 1、前言

本篇文章，默认你已经知道什么是 **Promise** ，然后我会带你一步步的实现一个简易的 Promise。将会以循序渐进的方式，分步骤实现。

## 2、三种状态

Promise 它一共会有三种状态：

1. pending
2. fulfilled
3. rejected

下面我们自己实现一个类，默认为 **pending** 状态，通过调用 **resolve** 或者 **reject** 改变其状态

```js
class LPromise {
  constructor(callbackFn) {
    this['[[PromiseState]]'] = 'pending'
    this['[[PromiseResult]]'] = undefined
    callbackFn(this.#resolve.bind(this), this.#reject.bind(this))
  }
  #resolve(res) {
    this['[[PromiseState]]'] = 'fulfilled'
    this['[[PromiseResult]]'] = res
  }
  #reject(err) {
    this['[[PromiseState]]'] = 'reject'
    this['[[PromiseResult]]'] = err
  }
}
console.log(new LPromise((resolve, reject) => console.log('pending'))) // pending 状态
const l1 = new LPromise((resolve, reject) => {
  resolve('我调用了resolve')
})
console.log(l1) // fulfilled 状态
const l2 = new LPromise((resolve, reject) => {
  reject('我调用了reject')
})
console.log(l2) // rejected 状态
```

## 3、实现 then 参数回调

返回的 **Promise** ，可以通过使用 **then** 传递成功和失败的回调。

##### 通过 then 接收了两个回调。实现了分别调用回调的内容。但是发现，他们两个都会执行。

```js
class LPromise {
  constructor(callbackFn) {
    this['[[PromiseState]]'] = 'pending'
    this['[[PromiseResult]]'] = undefined
    callbackFn(this.#resolve.bind(this), this.#reject.bind(this))
  }
  #resolve(res) {
    this['[[PromiseState]]'] = 'fulfilled'
    this['[[PromiseResult]]'] = res
  }
  #reject(err) {
    this['[[PromiseState]]'] = 'reject'
    this['[[PromiseResult]]'] = err
  }
  /* new content start */
  then(onResolve, onReject) {
    onResolve()
    onReject()
  }
  /* new content end */
}
const l1 = new LPromise((resolve, reject) => resolve())
l1.then(
  res => console.log('res'),
  err => console.log('err')
)
```

##### 对执行时机进行调整。使其在调用 resolve 或 reject 才执行相关的回调

```js
class LPromise {
  constructor(callbackFn) {
    this['[[PromiseState]]'] = 'pending'
    this['[[PromiseResult]]'] = undefined
    callbackFn(this.#resolve.bind(this), this.#reject.bind(this))
  }
  #resolve(res) {
    this['[[PromiseState]]'] = 'fulfilled'
    this['[[PromiseResult]]'] = res
    /* new content start */
    this.cbResolve() // 报错
    /* new content end */
  }
  #reject(err) {
    this['[[PromiseState]]'] = 'reject'
    this['[[PromiseResult]]'] = err
    /* new content start */
    this.cbReject() // 报错
    /* new content end */
  }
  then(onResolve, onReject) {
    /* new content start */
    this.cbResolve = onResolve
    this.cbReject = onReject
    /* new content end */
  }
}
const l1 = new LPromise((resolve, reject) => resolve())
l1.then(
  res => console.log('res'),
  err => console.log('err')
)
```

##### 改装后，发现 resolve 和 reject 的执行时间比 then 的回调要快。导致无法执行 then 中的回调。我们需要对 resolve 和 reject 中执行回调的部分进行 **延迟执行**。可以使用 setTimeout 进行延迟

```js
class LPromise {
  constructor(callbackFn) {
    this['[[PromiseState]]'] = 'pending'
    this['[[PromiseResult]]'] = undefined
    callbackFn(this.#resolve.bind(this), this.#reject.bind(this))
  }
  #resolve(res) {
    this['[[PromiseState]]'] = 'fulfilled'
    this['[[PromiseResult]]'] = res
    /* new content start */
    setTimeout(() => this.cbResolve())
    /* new content end */
  }
  #reject(err) {
    this['[[PromiseState]]'] = 'reject'
    this['[[PromiseResult]]'] = err
    /* new content start */
    setTimeout(() => this.cbReject())
    /* new content end */
  }
  then(onResolve, onReject) {
    this.cbResolve = onResolve
    this.cbReject = onReject
  }
}
const l1 = new LPromise((resolve, reject) => resolve())
l1.then(
  res => console.log('res'),
  err => console.log('err')
)
```

##### 考虑到 微任务 和 宏任务。我们可以使用 MutationObserver 替代 setTimeout

```js
class LPromise {
  constructor(callbackFn) {
    this['[[PromiseState]]'] = 'pending'
    this['[[PromiseResult]]'] = undefined
    callbackFn(this.#resolve.bind(this), this.#reject.bind(this))
  }
  #resolve(res) {
    this['[[PromiseState]]'] = 'fulfilled'
    this['[[PromiseResult]]'] = res
    /* new content start */
    const run = () => this.cbResolve()
    const ob = new MutationObserver(run)
    ob.observe(document.body, { attributes: true })
    document.body.setAttribute('lpromise', 'layouwen')
    /* new content end */
  }
  #reject(err) {
    this['[[PromiseState]]'] = 'reject'
    this['[[PromiseResult]]'] = err
    /* new content start */
    const run = () => this.cbReject()
    const ob = new MutationObserver(run)
    ob.observe(document.body, { attributes: true })
    document.body.setAttribute('lpromise', 'layouwen')
    /* new content end */
  }
  then(onResolve, onReject) {
    this.cbResolve = onResolve
    this.cbReject = onReject
  }
}
const l1 = new LPromise((resolve, reject) => resolve())
l1.then(
  res => console.log('res'),
  err => console.log('err')
)
```

## 4、链式调用

在原本的 **Promise** 中。我们是可以使用 **then** 链式调用。意味着每个 **then** 都返回一个新的 **Promise**。

因为支持链式。所以我们之前的 `cbResolve` 和 `cbReject` 就不能单单保存一个回调。要改回一个数组，将每一个 **then** 中的回调。都保存到回调队列中。等待调用 **resolve** 或者 **reject** 后才执行所有回调函数。

```js
class LPromise {
  constructor(callbackFn) {
    this['[[PromiseState]]'] = 'pending'
    this['[[PromiseResult]]'] = undefined
    /* new content start */
    this.cbResolveQueue = []
    this.cbRejectQueue = []
    /* new content end */
    callbackFn(this.#resolve.bind(this), this.#reject.bind(this))
  }
  #resolve(res) {
    this['[[PromiseState]]'] = 'fulfilled'
    this['[[PromiseResult]]'] = res
    /* new content start */
    const run = () => {
      let cbFn
      while ((cbFn = this.cbResolveQueue.shift())) {
        cbFn && cbFn()
      }
    }
    /* new content end */
    const ob = new MutationObserver(run)
    ob.observe(document.body, { attributes: true })
    document.body.setAttribute('lpromise', 'layouwen')
  }
  #reject(err) {
    this['[[PromiseState]]'] = 'reject'
    this['[[PromiseResult]]'] = err
    /* new content start */
    const run = () => {
      let cbFn
      while ((cbFn = this.cbRejectQueue.shift())) {
        cbFn && cbFn()
      }
    }
    /* new content end */
    const ob = new MutationObserver(run)
    ob.observe(document.body, { attributes: true })
    document.body.setAttribute('lpromise', 'layouwen')
  }
  then(onResolve, onReject) {
    /* new content start */
    return new LPromise((resolve, reject) => {
      const cbResolve = () => {
        onResolve && onResolve()
        resolve()
      }
      this.cbResolveQueue.push(cbResolve)
      const cbReject = () => {
        onReject && onReject()
        reject()
      }
      this.cbRejectQueue.push(cbReject)
    })
    /* new content end */
  }
}
const l1 = new LPromise((resolve, reject) => resolve())
l1.then(
  res => console.log('res'),
  err => console.log('err')
).then(
  res => console.log('res'),
  err => console.log('err')
)
```

此时我们已经完成了链式调用，但是我们会发现，此时如果返回一个新的 **Promise** ，却无法获取 **Promise** 的结果。所以我们得加一些判断条件。我们也会发现，此时此刻我们无法接收到 **res** 或 **err** 的参数。所以我们也要完善一下参数传递问题。

```js
class LPromise {
  constructor(callbackFn) {
    this['[[PromiseState]]'] = 'pending'
    this['[[PromiseResult]]'] = undefined
    this.cbResolveQueue = []
    this.cbRejectQueue = []
    callbackFn(this.#resolve.bind(this), this.#reject.bind(this))
  }
  #resolve(res) {
    this['[[PromiseState]]'] = 'fulfilled'
    this['[[PromiseResult]]'] = res
    const run = () => {
      let cbFn
      while ((cbFn = this.cbResolveQueue.shift())) {
        /* new content start */
        cbFn && cbFn(res)
        /* new content end */
      }
    }
    const ob = new MutationObserver(run)
    ob.observe(document.body, { attributes: true })
    document.body.setAttribute('lpromise', 'layouwen')
  }
  #reject(err) {
    this['[[PromiseState]]'] = 'reject'
    this['[[PromiseResult]]'] = err
    const run = () => {
      let cbFn
      while ((cbFn = this.cbRejectQueue.shift())) {
        /* new content start */
        cbFn && cbFn(err)
        /* new content end */
      }
    }
    const ob = new MutationObserver(run)
    ob.observe(document.body, { attributes: true })
    document.body.setAttribute('lpromise', 'layouwen')
  }
  then(onResolve, onReject) {
    return new LPromise((resolve, reject) => {
      /* new content start */
      const cbResolve = res => {
        const resolveRes = onResolve && onResolve(res)
        if (resolveRes instanceof LPromise) {
          resolveRes.then(resolve)
        } else {
          resolve(res)
        }
      }
      /* new content end */
      this.cbResolveQueue.push(cbResolve)
      /* new content start */
      const cbReject = err => {
        onReject && onReject(err)
        reject(err)
      }
      /* new content end */
      this.cbRejectQueue.push(cbReject)
    })
  }
}
const l1 = new LPromise((resolve, reject) => resolve('我是传入的 resolve 数据'))
l1.then(
  res => {
    console.log('第一个then的res', res)
    return new LPromise((resolve, reject) => resolve('返回的Promise'))
  },
  err => console.log('第一个then的err', err)
)
  .then(
    res => console.log('第二个then的res', res),
    err => console.log('第二个then的err', err)
  )
  .then(
    res => console.log('第三个then的res', res),
    err => console.log('第三个then的err', err)
  )
```

到现在我们已经实现了 **then** 的链式调用