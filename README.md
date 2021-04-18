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

通过 then 接收了两个回调。实现了分别调用回调的内容。但是发现，他们两个都会执行。

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

对执行时机进行调整。使其在调用 resolve 或 reject 才执行相关的回调

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

改装后，发现 resolve 和 reject 的执行时间比 then 的回调要快。导致无法执行 then 中的回调。我们需要对 resolve 和 reject 中执行回调的部分进行 **延迟执行**。可以使用 setTimeout 进行延迟

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

考虑到 微任务 和 宏任务。我们可以使用 MutationObserver 替代 setTimeout

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

## 5、实现 catch 方法

在调用 `catch` 的时候自动在回调队列中添加一个错误回调函数。

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
        cbFn && cbFn(res)
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
        cbFn && cbFn(err)
      }
    }
    const ob = new MutationObserver(run)
    ob.observe(document.body, { attributes: true })
    document.body.setAttribute('lpromise', 'layouwen')
  }
  then(onResolve, onReject) {
    return new LPromise((resolve, reject) => {
      const cbResolve = res => {
        const resolveRes = onResolve && onResolve(res)
        if (resolveRes instanceof LPromise) {
          resolveRes.then(resolve)
        } else {
          resolve(res)
        }
      }
      this.cbResolveQueue.push(cbResolve)
      const cbReject = err => {
        onReject && onReject(err)
        reject(err)
      }
      this.cbRejectQueue.push(cbReject)
    })
  }
  /* new content start */
  catch(err) {
    this.then(undefined, err)
  }
  /* new content end */
}
const p1 = new LPromise((resolve, reject) => reject('我是p1的错误信息'))
p1.then(res => console.log(res)).catch(err => console.log(err))
```

## 6、resolve 和 reject 静态方法

这两个静态方法比较简单。只需要返回一个固定状态的 **Promise** 即可。

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
        cbFn && cbFn(res)
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
        cbFn && cbFn(err)
      }
    }
    const ob = new MutationObserver(run)
    ob.observe(document.body, { attributes: true })
    document.body.setAttribute('lpromise', 'layouwen')
  }
  then(onResolve, onReject) {
    return new LPromise((resolve, reject) => {
      const cbResolve = res => {
        const resolveRes = onResolve && onResolve(res)
        if (resolveRes instanceof LPromise) {
          resolveRes.then(resolve)
        } else {
          resolve(res)
        }
      }
      this.cbResolveQueue.push(cbResolve)
      const cbReject = err => {
        onReject && onReject(err)
        reject(err)
      }
      this.cbRejectQueue.push(cbReject)
    })
  }
  /* new content start */
  static resolve(res) {
    return new LPromise(resolve => resolve(res))
  }
  static reject(err) {
    return new LPromise((undefined, reject) => reject(err))
  }
  /* new content end */
}
const p1 = LPromise.resolve('成功')
console.log(p1)
const p2 = LPromise.reject('失败')
console.log(p2)
```

## 7、实现 finally 方法

这个与 catch 类似的实现，只需要保证不管成功还是失败都执行里面的回调。

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
        cbFn && cbFn(res)
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
        cbFn && cbFn(err)
      }
    }
    const ob = new MutationObserver(run)
    ob.observe(document.body, { attributes: true })
    document.body.setAttribute('lpromise', 'layouwen')
  }
  then(onResolve, onReject) {
    return new LPromise((resolve, reject) => {
      const cbResolve = res => {
        const resolveRes = onResolve && onResolve(res)
        if (resolveRes instanceof LPromise) {
          resolveRes.then(resolve)
        } else {
          resolve(res)
        }
      }
      this.cbResolveQueue.push(cbResolve)
      const cbReject = err => {
        onReject && onReject(err)
        reject(err)
      }
      this.cbRejectQueue.push(cbReject)
    })
  }
  catch(err) {
    this.then(undefined, err)
  }
  /* new content start */
  finally(callback) {
    this.then(callback, callback)
  }
  /* new content end */
  static resolve(res) {
    return new LPromise(resolve => resolve(res))
  }
  static reject(err) {
    return new LPromise((undefined, reject) => reject(err))
  }
}
const p1 = new LPromise((resolve, reject) => reject('我是p1的错误信息'))
p1.then(
  res => console.log(res),
  err => console.log(err)
).finally(() => console.log('finally'))
```

## 8、实现 race 方法

**race** 就是返回最先执行成功的结果。不管是成功还是失败。这样我们只需要遍历该 **Promise** ，正常返回数据。谁先执行完成，谁先返回即可。注意要控制状态，防止返回多个结果。 **race** 只需要返回最快的一个结果。

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
        cbFn && cbFn(res)
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
        cbFn && cbFn(err)
      }
    }
    const ob = new MutationObserver(run)
    ob.observe(document.body, { attributes: true })
    document.body.setAttribute('lpromise', 'layouwen')
  }
  then(onResolve, onReject) {
    return new LPromise((resolve, reject) => {
      const cbResolve = res => {
        const resolveRes = onResolve && onResolve(res)
        if (resolveRes instanceof LPromise) {
          resolveRes.then(resolve)
        } else {
          resolve(res)
        }
      }
      this.cbResolveQueue.push(cbResolve)
      const cbReject = err => {
        onReject && onReject(err)
        reject(err)
      }
      this.cbRejectQueue.push(cbReject)
    })
  }
  catch(err) {
    this.then(undefined, err)
  }
  finally(callback) {
    this.then(callback, callback)
  }
  static resolve(res) {
    return new LPromise(resolve => resolve(res))
  }
  static reject(err) {
    return new LPromise((undefined, reject) => reject(err))
  }
  /* new content start */
  static race(promiseArr) {
    return new LPromise((resolve, reject) => {
      let isContinue = true
      promiseArr.forEach(promise => {
        promise.then(
          res => {
            if (isContinue) {
              isContinue = false
              resolve(res)
            }
          },
          err => {
            if (isContinue) {
              isContinue = false
              reject(err)
            }
          }
        )
      })
    })
  }
  /* new content end */
}
const p1 = new LPromise((resolve, reject) => setTimeout(() => resolve(1), 200))
const p2 = new LPromise((resolve, reject) => setTimeout(() => reject(2), 1000))
const p3 = new LPromise((resolve, reject) => setTimeout(() => resolve(3), 3000))
LPromise.race([p1, p2, p3]).then(
  res => console.log('res', res),
  err => console.log('err', err)
)
```

## 9、实现 all 方法

**all** 方法当所有 **Promise** 都成功时返回所有结果的数组，否则返回第一个失败的结果。我们只需要遍历该 **Promise** 数组。定义个变量存放当前 **res** 的长度。如果长度等于数组的长度，我们就 **resolve** 出去。否则发现第一个失败的时候，直接 **reject**。

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
        cbFn && cbFn(res)
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
        cbFn && cbFn(err)
      }
    }
    const ob = new MutationObserver(run)
    ob.observe(document.body, { attributes: true })
    document.body.setAttribute('lpromise', 'layouwen')
  }
  then(onResolve, onReject) {
    return new LPromise((resolve, reject) => {
      const cbResolve = res => {
        const resolveRes = onResolve && onResolve(res)
        if (resolveRes instanceof LPromise) {
          resolveRes.then(resolve)
        } else {
          resolve(res)
        }
      }
      this.cbResolveQueue.push(cbResolve)
      const cbReject = err => {
        onReject && onReject(err)
        reject(err)
      }
      this.cbRejectQueue.push(cbReject)
    })
  }
  catch(err) {
    this.then(undefined, err)
  }
  finally(callback) {
    this.then(callback, callback)
  }
  static resolve(res) {
    return new LPromise(resolve => resolve(res))
  }
  static reject(err) {
    return new LPromise((undefined, reject) => reject(err))
  }
  static race(promiseArr) {
    return new LPromise((resolve, reject) => {
      let isContinue = true
      promiseArr.forEach(promise => {
        promise.then(
          res => {
            if (isContinue) {
              isContinue = false
              resolve(res)
            }
          },
          err => {
            if (isContinue) {
              isContinue = false
              reject(err)
            }
          }
        )
      })
    })
  }
  /* new content start */
  static all(promiseArr) {
    return new LPromise((resolve, reject) => {
      const resArr = []
      const length = promiseArr.length
      promiseArr.forEach(p => {
        p.then(
          res => {
            resArr.push(res)
            if (resArr.length === length) {
              resolve(resArr)
            }
          },
          err => reject(err)
        )
      })
    })
  }
  /* new content end */
}
const p1 = new LPromise((resolve, reject) => setTimeout(() => resolve(1), 200))
const p2 = new LPromise((resolve, reject) => setTimeout(() => reject(2), 1000))
const p3 = new LPromise((resolve, reject) => setTimeout(() => reject(3), 3000))
LPromise.all([p1, p2, p3]).then(
  res => console.log('res', res),
  err => console.log('err', err)
)
```
