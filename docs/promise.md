Promise 实现
====

``` js
const promise = new Promise((resolve, reject) => {
  setTimeout(() => {
    const n = Math.random();
    n > 0.5 ? resolve(n) : reject(new Error(n));
  }, 0);
});
promise.then(
  value => console.log(value),
  reason => console.error(reason)
);
```

## 基本概念

### promise 实例的状态与值

promise 实例有 3 个状态：
- 待定（`pending`）：初始状态，既没有被兑现，也没有被拒绝。
- 已兑现（`fulfilled`）：意味着操作成功完成。
- 已拒绝（`rejected`）：意味着操作失败。

promise 实例必须处在其中一个状态，待定状态的 promise 实例可以通过一个值变为已兑现状态，或者通过一个原因（reason）变为已拒绝的状态。已兑现和已拒绝都为终态，一旦为终态后不可以改变，值也不能被改变。如果处于已拒绝状态，这个值即为拒绝的原因（通常为一个错误对象）。

### 一些私有属性

每个 Promise 对象都有状态和值，所以最基本的需要有两个私有属性用于分别维护状态和值：

- `_state` 属性：标识来保存 promise 的状态。待定、已兑现、已拒绝分别用 `0`、`1`、`2` 表示；
- `_value` 属性：保存 promise 的值。

另外，promise 通过 `then` 方法来获取到值，并注册已兑现和已拒绝状态时的回调函数 `onFulfilled` 和 `onRejected`。
当 promise 为待定状态时，因为不确定终态到底是已兑现还是已拒绝，所以还需要通过一个 `_deferreds` 属性保存这两个回调函数。

``` js
function onFulfilled (value) { /* ... */ }
function onRejected (reason) { /* ... */ }
promise.then(onFulfilled, onRejected)
```

这里使用一个 `Handler` 构造函数来储存 `onFulfilled`、`onRejected` 和当前的 promise 实例，
`then()` 方法调用一次就 `new` 一个 `deferred` 对象添加到 `_deferreds` 属性上。

``` js
function Handler (onFulfilled, onRejected, promise) {
  this.onFulfilled = typeof onFulfilled === 'function' ? onFulfilled : null
  this.onRejected = typeof onRejected === 'function' ? onRejected : null
  this.promise = promise
}
// 创建 deferred 对象
var deferred = new Handler(onFulfilled, onRejected, promise)
```

同一个 promise 实例可以调用 `then` 方法多次，比如这样：

``` js
var promise = new Promise(function (resolve, reject) {
  setTimeout(function () {
    resolve(1)
  }, 1000)
})

function onFulfilled_1 (value) {}
function onRejected_1 (reason) {}

function onFulfilled_2 (value) {}
function onRejected_2 (reason) {}

promise.then(onFulfilled_1, onRejected_1)
promise.then(onFulfilled_2, onRejected_2)
```

如此就会产生多个 `deferred` 对象，这时私有属性 `_deferreds` 就为一个数组。当 promise 为终态时还需要遍历 `_deferreds`，按照 `then()` 注册的顺序处理 `deferred` 对象。也就是说 `_deferreds` 可能为不同类型的值（`null`、`deferred` 或 `Array<deferred>`），所以再定义一个私有属性 `_deferredState` 来表示 `_deferreds` 的状态。

`_deferredState` 可能的值：

- `0`：表示 `_deferreds` 初始为 `null`。
- `1`：表示 `_deferreds` 为一个 `deferred` 对象，即 `_deferreds = deferred`。
- `2`：表示 `_deferreds` 保存了多个 `deferred` 对象，即 `_deferreds = [deferred1, deferred2, ... ]`

*也有些实现是将 `_deferreds` 初始为一个 `[]`, 这样也就不需要 `_deferredState` 。*

## 具体实现

### Promise 构造函数

Promise 构造函数内部初始化上面提到的几个实例私有属性，`doResolve` 用于执行传入的 `fn` 函数（即 `(resolve, reject) => { ... }`）。

``` js
function Promise (fn) {
  this._state = 1;
  this._value = null;
  this._deferreds = null;
  this._deferredState = 1;
  // 用于执行fn
  doResolve(fn, this);
}
```

补充上必要的验证

``` js
function Promise (fn) {
  // 保证 Promises 通过 new 构造
  if (typeof this !== 'object') {
    throw new TypeError('Promises must be constructed via new');
  }
  if (typeof fn !== 'function') {
    throw new TypeError('Promise constructor\'s argument is not a function');
  }
  this._state = 1
  this._value = null
  this._deferreds = null
  this._deferredState = 1
  // 执行fn
  doResolve(fn, this)
}
```

**`doResolve`**

`doResolve` 主要做 3 件事：

1. 定义 `resolve` 和 `reject` 函数供 `fn` 函数使用；

`resolve` 和 `reject` 使用时只需要传递一个参数 `value`，即 `resolve(value)`、`reject(reason)`。
但实际 `resolve` 和 `reject` 想要实现对 promise 实例状态的改变，所以需要把当前 promise 实例也传递进去, 即 `resolve(promise, value)`。

2. 需要保证 `resolve`、`reject` 两个函数中只有其中之一执行，且仅执行一次。

声明一个 `done` 变量，当 `done` 的值为 `true` 时，就不允许再执行 `resolve` 或 `reject` 函数。

3. `fn` 执行有可能出现错误，类似下面这样，而如果 `fn` 执行出错，应抛出错误作为传递给 `reject` 的值(即 `reason`) 。

``` js
new Promise(function(resolve, reject) {
  throw new Error('...')
})
```

`doResolve` 函数实现上面 3 件事的主要代码如下：

``` js
function resolve (promise, value) {}
function reject (promise, reason) {}

function doResolve (fn, promise) {
  var done = false
  // 1. fn(resolve, reject)
  var _resolve = function (promise, value) {
    return function (value) {
      // 2. resolve/reject 仅执行一次
      if (done) return
      done = true
      resolve(promise, value)
    }
  }
  var _reject = function (promise, reason) {
    return function (reason) {
      if (done) return
      done = true
      reject(promise, reason)
    }
  }
  try {
    fn(_resolve, _reject)
  } catch (error) {
    // 3. fn 执行出错时，reject(error)
    reject(promise, error)
  }
}
```

现在重点需要完成 `resolve` 和 `reject` 方法的实现。

`resolve` 和 `reject` 主要是实现对 Promise 解析过程，函数主体内容主要就是将 `value/reason` 存在私有属性 `_value` 上，终态时执行 `_deferreds` 上的回调函数集。

其中 `resolve` 的实现比较复杂，其中有很多 promise/A+ 规范的实现细节，`reject` 的实现相对简单，这里先行实现。

### `reject(promise, reason)`

将 promise 状态变为 `rejected`, 并将 `reason` 的值传递给 `_value`，之后根据 `_deferredState` 的状态处理 `deferred` 对象（存储了 `then()` 注册的回调函数`onFulfilled` 和 `onRejected`），这就是 `handle(promise, deferred)` 函数做的事情。

``` js
function reject (self, reason) {
  self._state = 2;
  self._value = reason;

  // _deferreds = deferred 时
  if (self._deferredState === 1) {
    handle(self, self._deferreds);
    self._deferreds = null;
  }
  // _deferreds = Array<deferred> 时
  if (self._deferredState === 2) {
    for (var i = 0; i < self._deferreds.length; i++) {
      handle(self, self._deferreds[i]);
    }
    self._deferreds = null;
  }
  // _deferreds 为 null 时就什么也不做
}
```

不管是 `resolve(promise, value)` 还是 `reject(promise, reason)`，处理 `_deferreds` 都是同样的逻辑。所以可以通过 `finale` 函数把这段处理逻辑独立出来：

``` js
function reject (self, reason) {
  self._state = 2;
  self._value = reason;
  finale(self);
}

function resolve (self, value) {
  // ...
  finale(self);
}

function finale(self) {
  // _deferreds = deferred 时
  if (self._deferredState === 1) {
    handle(self, self._deferreds);
    self._deferreds = null;
  }
  // _deferreds = Array<deferred> 时
  if (self._deferredState === 2) {
    for (var i = 0; i < self._deferreds.length; i++) {
      handle(self, self._deferreds[i]);
    }
    self._deferreds = null;
  }
  // _deferreds 为 nulls时就什么也不做
}
```

### `resolve(promise, value)`

`resolve` 主要是实现对 Promise 的解析过程。

``` js
function resolve (promise, value) {
  // ...
  finale(self)
}
```

**Promise 解析过程**

根据 promise/A+ 规范，Promise 解析过程主要包含以下内容：

1. 如果 `promise` 和 `value` 指向相同的值, 使用 TypeError 做为原因将 `promise` 拒绝。
2. 如果 `value` 是一个 `promise`, 采用其状态。
3. 如果 `value` 是一个对象或一个函数：
  1. 将 `then` 赋值给 `value.then`， 如果在取 `value.then` 值时抛出了异常，则以这个异常做为原因将 `promise` 拒绝。
  2. 如果 `then` 是一个函数，则以 `value` 为 `this` 调用 `then` 函数。
  3. 如果 `then` 不是一个函数，则以 `value` 为值兑现 `promise`。
4. 如果 `value` 不是对象也不是函数，则以 `value` 为值兑现 `promise`。

解析过程具体代码实现：

``` js
function resolve(self, value) {
  // 1：如果 promise 和 value 指向相同的值, 使用 TypeError 做为原因将 promise 拒绝
  if (value === self) {
    return reject(self, new TypeError('A promise cannot be resolved with itself.'));
  }
  // 3：如果 value 是一个对象或一个函数
  if (value && (typeof value === 'object' || typeof value === 'function')) {
    // 3.1：将 then 赋为 value.then，如果在取 value.then 值时抛出了异常，则以此异常做为原因将promise拒绝
    try {
      var then = value.then;
    } catch (error) {
      reject(self, error);
    }
    // 2：如果 value 是一个 promise, 采用其状态
    if (then === self.then && value instanceof Promise) {
      self._state = 3;
      self._value = newValue;
      finale(self);
      return;
    }
    // 3.2：如果 then 是一个函数， 以 vlaue 为 this 调用then函数
    else if (typeof then === 'function') {
      doResolve(then.bind(value), self);
      return;
    }
    // 3.3：如果 then 不是一个函数，则以 value 为值兑现 promise
  }

  // 4：如果 value 不是对象也不是函数，则以 value 为值兑现 promise
  self._state = 1;
  self._value = newValue;

  // 处理 _deferreds
  finale(self);
}
```

重点说明一下 `2` 和 `3.2` 的解析过程：

- **2：如果 `value` 也是一个 `promise` 实例, 采用其状态**

这里的做法是通过设置 `_state` 的值等于 `3` 来标识当前promise 的值（`value`）是一个 promise 的状态（此前已约定的值 `0`、`1`、`2` 分别表示待定、已兑现、已拒绝），并且把 `value` 保存在 `_value` 中。<br/>
之前提到用 `handle` 函数去处理 `deferred` 对象，具体行为是根据当前 promise 实例的状态确定到底是执行 `deferred` 对象中的`onFulfilled` 函数还是 `onRejected` 函数。
所以 `handle` 函数实现的过程中增加对 `promise._state === 3` 的判断, 并将当前 promise 实例引用指向 `_value` 即可，并获取 `value`（另一个 promise 实例）的状态，即`promise = promise._value`。`handle` 的具体实现如下：

``` js
function handle (self, deferred) {
  // 当 _state = 3, 表示 promise 的值也是一个promise实例，直接获取其状态, 
  while (self._state === 3) {
    self = self._value;
  }
  // promise 状态为待定时，添加 deferred 至 _deferreds
  if (self._state === 0) {
    // _deferreds 为 null 时
    if (self._deferredState === 0) {
      self._deferredState = 1;
      self._deferreds = deferred;
      return;
    }
    // _deferreds = deferred 时，按 then 的注册顺序创建一个 Array<deferred>
    if (self._deferredState === 1) {
      self._deferredState = 2;
      self._deferreds = [self._deferreds, deferred];
      return;
    }
    // _deferreds 为 Array<deferred> 时，直接 push 添加
    self._deferreds.push(deferred);
    return;
  }
  // promise 为 终态时，直接执行 deferred
  handleResolved(self, deferred);
}

function handleResolved (self, deferred) {
  // 执行 deferred.onFulfilled 或 deferred.onRejected
}
```

如果 `promise` 实例为终态时，就不再存储 `deferred`， 而是直接执行 `deferred`, 即 `handleResolved` 函数要做的事。


- **3.2：如果 `then` 是一个函数， 以 `value` 为 `this` 调用 `then` 函数**

这里是对 `thenable` 对象的解析过程。`thenable` 就是一个包含了 `then` 方法的对象或函数，比如：

``` js
let thenable = {
  then: function(resolve, reject) {
    resolve(1)
  }
}
// or
function thenable () {}
thenable.then = function (resolve, reject) {
  resolve(1)
}
```

可以看出 `thenable` 对象的 `then` 方法相当于 `new promise(fn)` 中 `fn`。

回忆之前执行 `fn` 的过程是交给了 `doResolve(fn, promise)` 函数, 所以 `3.2` 分解为：<br/>
先保证 `then` 方法调用时 `this` 指向 `thenable`，即 `then.bind(value)`；然后类似对 `fn` 的操作: `doResolve(then.bind(value), self)`。结合 `resolve` 和 `doResolve` 的代码一起看比较清楚了：

``` js
function resolve(self, value) {
  // ...
  // 3：如果 value 是一个对象或一个函数
  if (value && (typeof value === 'object' || typeof value === 'function')) {
    if (then === self.then && value instanceof Promise) {
      // ...
      return;
    }
    // 3.2：如果 then 是一个函数， 以 value 为 this 调用 then 函数
    else if (typeof then === 'function') {
      doResolve(then.bind(value), self);
      return;
    }
  }
  // ...
}

function doResolve (fn, promise) {
  var done = false
  var _resolve = function (promise, value) {
    return function (value) {
      if (done) return
      done = true
      resolve(promise, value)
    }
  }
  var _reject = function (promise, reason) {
    return function (reason) {
      if (done) return
      done = true
      reject(promise, reason)
    }
  }
  try {
    fn(_resolve, _reject)
  } catch (error) {
    reject(promise, error)
  }
}
```

### handleResolved

上面提到如果 `promise` 实例为终态时（已兑现 或 已拒绝），`handle (self, deferred)` 不再存储 `deferred`，而是直接调用 `handleResolved` 执行 `deferred` 对象中的 `onFulfilled` 或 `onRejected`。由于 promise 标准规范中明确说明 `onFulfilled` 和 `onRejected` 是异步执行，在事件循环开始之后被调用。

> In practice, this requirement ensures that onFulfilled and onRejected execute asynchronously, after the event loop turn in which then is called, and with a fresh stack.

所以还需要把这个过程包装成异步：

``` js
function handleResolved (self, deferred) {
  asyncFn(function () {
    // ...
  })
}
```

`asyncFn` 的实现可以使用 [asap](https://github.com/kriskowal/asap.git)（`asap`：as soon as possible，尽快执行的意思），为了简单可以使用 `setTimeout(fn, 0)` 代替。

``` js
var asap = require('asap/raw')
function handleResolved (self, deferred) {
  asap(function () {
    var callback = self._state === 1 ? deferred.onFulfilled : deferred.onRejected
    if (callback === null) {
      // 值穿透
      if (self._state === 1) {
        resolve(deferred.promise, self._value)
      } else {
        reject(deferred.promise, self._value)
      }
      return
    }
    try {
      var value = callback(self._value)
      resolve(deferred.promise, value)
    } catch (error) {
      reject(deferred.promise, error)
    }
  })
}
```

**`Promise.prototype.then` 注册的 `onFulfilled`、`onRejected` 函数需要异步执行的原因？**

由于 promise 内部执行的是同步操作还是异步操作是不确定的，这会导致 `onFulfilled` 和 `onRejected` 内部如果存在外部引用就会存在不确定性，比如：

``` js
var a = 1;
var promise = new myPromise(function (resolve, reject) {
  // 同步
  resolve();
  // 或者 异步
  // setTimeout(resolve, 0)
})
promise.then(function (value) {
  a = 2;
})

console.log(a);
```

假如未规定 `onFulfilled`、`onRejected` 为异步执行，则这里 `console.log(a)` 可能存在两种值：promise 内部同步操时，输出 `a` 的值为 `2`; promise 内部是异步操作，输出 `a` 的值为 `1`。为屏蔽这种不确定性，规范指出 `onFulfilled` 和 `onRejected` 方法必须异步执行。

### Promise.prototype.then(onFulfilled, onRejected)

Promise 必须提供一个 `then` 方法来获取其值或拒绝原因，Promise 的 `then` 方法接受两个参数 `promise.then(onFulfilled, onRejected)`，`then` 方法主要遵循以下规范

1. 参数均可选。`onFulfilled` 和 `onRejected` 都是可选参数，且必须被当做函数调用，如果不是函数，都将被忽略。
2. 可以被同一个 promise 调用多次。`then` 方法可以被同一个 promise 调用多次。在 promise 变为已兑现或已拒绝状态后，对应的 `onFulfilled` 或 `onRejected` 函数须按照其注册顺序依次执行，且执行次数不能超过一次。
3. 参数 `onFulfilled` 和 `onRejected` 只允许被异步执行。
`onFulfilled` 和 `onRejected` 被存储在 `deferred` 对象中，`deferred` 的执行过程是交由 `handleResolved` 函数去完成的，只允许被异步执行原因在 [handleResolved](#handleResolved) 已经说明。

4. `then` 必须返回一个新的 `promise` 实例，即 `promise2 = promise1.then(onFulfilled, onRejected)`。

**为何 `then` 链式调用必须返回一个新的 promise 实例，而不是直接返回 `this` 呢？**

``` js
var promise2 = promise1.then(function (value) {
  return Promise.reject(new Error());
})
```

上面如果 `promise1` 为已兑现状态后将会执行 `onFulfilled`，`onFulfilled` 返回一个状态为 `rejected` 的 promise。
由于 promise 状态一旦变为终态不能再更改，如果返回 `this` 则 `promise2` 无法转为 `rejected` 状态，这就产生了矛盾。

由上，我们也可以知道 `then` 执行的大致流程：

1. 实例化一个新的 promise 对象并返回（保持`then`链式调用）；
2. 构造 `deferred` 对象用于存储 `onFulfilled`、`onRejected` 和 新的 `promise` 实例；
3. 判断当前 `promise` 状态，如果为待定状态则保存 `deferred` 到 `_deferreds` 中，否则执行 `deferred` 中的 `onFulfilled` 或 `onRejected`。

``` js
function noop () {};
Promise.prototype.then = function (onFulfilled, onRejected) {
  var promise = new Promise(noop);
  handle(this, new Handler(onFulfilled, onRejected, promise))
  return promise;
}

// 结合 handle 看一下更清楚
function handle (self, deferred) {
  // resolve(value), value 为 promise 时，获取其状态
  while (self._state === 3) {
    self = self._value;
  }
  // promise 为 非终态时，添加 deferred 至 self._deferreds
  if (self._state === 0) {
    // ... 
    return;
  }
  // promise 为 终态时，直接执行 deferred
  handleResolved(self, deferred);
}
```

## es6 扩展

### Promise.prototype.catch

`promise` 内部错误都会以 `error` 作为原因 `reason` 被 `reject(reason)`：

``` js
Promise.prototype.catch = function (onRejected) {
  return this.then(null, onRejected);
}
```

### Promise.reject

`Promise.reject(reason)` 方法会返回一个新的 promise 实例，该实例的状态为已拒绝状态：

``` js
Promise.reject = function (value) {
  return new Promise(function (resolve, reject) {
    reject(value);
  })
}
```

### Promise.resolve

`Promise.resolve` 将传入的值创建为一个新的 Promise 实例：

``` js
function valuePromise (value) {
  var promise = new Promise(noop)
  promise._state = 1
  promise._value = value
  return promise
}

Promise.resolve = function (value) {
  if (value instanceof Promise) return value

  if (value === null) return valuePromise(null)
  if (value === undefined) return valuePromise(undefined)
  if (value === true) return valuePromise(true)
  if (value === false) return valuePromise(false)
  if (value === 0) return valuePromise(0)
  if (value === '') return valuePromise('')

  if (typeof value === 'object' || typeof value === 'function') {
    try {
      var then = value.then
      if (typeof then === 'function') {
        return new Promise(then.bind(value))
      }
    } catch (error) {
      return new Promise(function (resolve, reject) {
        reject(error)
      })
    }
  }
  return valuePromise(value)
}
```

### Promise.all

将多个 promise 实例包装成一个新的 promise 实例：

``` js
const p = Promise.race([p1, p2 /* , ... */])
```

主要有以下特征：

- 只有所有的 promise 实例的状态都变成已兑现，`p` 的状态才会变成已兑现，此时 `p1、p2` ... 的返回值组成一个数组，然后传递给 `p.then()` 方法注册的回调函数执行。
- 只要多个实例中有一个实例的变为已拒绝状态，`p` 的状态就变成已拒绝，此时率先被 `reject` 的实例的返回值会传递给 `p.then()` 方法注册的回调函数。
- 如果作为参数的 promise 实例本身定义了 `catch` 方法，那么它的状态一旦变为已拒绝，并不会触发 `p.catch()`。

``` js
Promise.all = function (arr) {
  var args = Array.prototype.slice.call(arr);

  return new Promise(function (resolve, reject) {
    if (args.length === 0) return resolve([]);
    var remaining = args.length;

    function res (i, val) {
      if (val && (typeof val === 'object' || typeof val === 'function')) {
        // promise 实例
        if (val instanceof Promise && val.then === Promise.prototype.then) {
          while (val._state === 3) {
            val = val._value;
          }
          // fulfilled => args[i] = val
          if (val._state === 1) return res(i, val._value);
          if (val._state === 2) reject(val._value);
          val.then(function (val) {
            res(i, val);
          }, reject);
          return;
        }
        // thenable 对象
        else {
          var then = val.then;
          if (typeof then === 'function') {
            // 如果 then 是一个函数， 以 value 为 this 调用 then 函数
            var p = new Promise(then.bind(val));
            p.then(function (val) {
              res(i, val);
            }, reject);
            return;
          }
        }
      }
      // 非 promise/thenable
      args[i] = val;
      if (--remaining === 0) {
        resolve(args);
      }
    }

    for (var i = 0; i < args.length; i ++) {
      res(i, args[i]);
    }
  })
}
```

### Promise.race

`Promise.race` 方法是将多个 promise 实例，包装成一个新的 promise 实例。

``` js
const p = Promise.race([p1, p2 /* , ... */])
```

多个 promise 实例谁率先改变状态，`p` 的状态就跟着改变。所以使用 `Promise.resolve` 包装后调用 `then` 方法，以 `resolve`、`reject` 作为 `onFulfilled` 和 `onRejected` 即可。率先执行 `onFulfilled` 或 `onRejected` 的改变 `p` 的状态。

``` js
Promise.race = function (values) {
  return new Promise(function (resolve, reject) {
    values.forEach(function (value){
      Promise.resolve(value).then(resolve, reject)
    })
  })
}
```

# 参考

- [when/promise](https://github.com/then/promise)
