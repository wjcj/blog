Event Loop
=====

## 基础概念

![event loop 1.0](https://mdn.mozillademos.org/files/17124/The_Javascript_Runtime_Environment_Example.svg)

### 栈（`stack`）

或称调用栈（`Call Stack`），它是采用后进先出 (LIFO) 原则来临时存储和管理函数调用的数据结构。
它由函数调用生成，当一个函数被调用时，该函数的参数和局部变量形成一个栈帧被压入调用栈，所以也可以说调用栈是由若干栈帧组成的帧栈。

**帧（`frame`）与 堆（`heap`）**

- 帧（`frame`）：又称栈帧，它是栈中的一个内存位置，包含了函数调用参数和局部变量。
- 堆（`heap`）：存储对象（object）的内存区域，引用类型的变量就存储在这里。

**后进先出 (LIFO) 原则**

管理函数调用的本质是对栈帧进行操作，后进先出原则即最后一个入栈的函数最先执行，执行完毕然后返回时其栈帧最先弹出，同时其占用的内存也会被清除。然后弹出下一个栈帧，此过程一直进行到调用栈中栈帧清空为止。

以下面代码为例（原实例演示查看[这里](https://flaviocopes.com/javascript-event-loop/#a-simple-event-loop-explanation)）：
``` js
const bar = () => console.log('bar');

const baz = () => console.log('baz');

const foo = () => {
  console.log('foo');
  bar();
  baz();
}

foo();
```
其调用栈执行过程如下：
![LIFO example](https://flaviocopes.com/javascript-event-loop/call-stack-first-example.png)

### 队列（`queue`）

一个 JavaScript 运行时包含一个待处理消息的队列，每一个消息中都关联着一个用以处理这个消息的回调函数，所以很多地方也称之为消息队列（`Message queue`）或回调队列（`Callback Queue`）。

在事件循环期间的某个时刻，运行时会按照先进先出（FIFO）原则从最先进入队列的消息开始处理，被处理的消息会被移出队列，并作为输入参数来调用与之关联的函数。如上所说，调用一个函数总是会为其创造一个新的栈帧被压入调用栈中，函数内部可能调用了其他函数而在栈中压入多个栈帧。

*很多文章中可以看到类似事件队列（`Event Queue`）、事件循环队列（`Event Loop Queue`）的术语，均指此描述的队列。*

## 事件循环（`Event Loop`）

事件循环之所以称之为循环，因为它通常按照以下伪代码的方式实现：
``` js
while (queue.waitForMessage()) {
  queue.processNextMessage();
}
```
`queue.waitForMessage()` 会同步地等待消息队列中的消息到达(如果当前没有任何消息等待被处理)，如果存在多个消息时将从最先进入队列的消息开始处理，直到该消息在调用栈完整地执行后（执行期间持续检查调用栈直到消息关联函数压入栈中所有栈帧都弹出），消息队列中的其它消息才会被执行...无限循环。

此机制模型的优点在于函数的执行不会被抢占，只有它执行完毕之后才会执行其他的代码，才能修改这个函数操作的数据，这一方面让开发变得简单，但同时缺点在于如果一个消息执行的时间很长，Web应用程序就无法处理与用户的交互，例如点击或滚动，因为引擎执行任务（`Tasks`👇）时永远不会进行渲染。如果过程过长浏览器可能还会弹出警报建议终止整个页面。<br/>
所以一个良好的习惯是缩短单个消息处理时间，并在可能的情况下将一个消息分解成多个消息，推荐查看[这篇文章](https://javascript.info/event-loop).

## Tasks、Microtask

`Tasks`（任务） 和 `Microtask`（微任务） 都是来自 HTML 规范中的概念。<br/>

**`Tasks`（任务）：**
又称 `Macrotasks`（宏任务），这是相对微任务而言的。指任何按标准机制调度进行执行的 JavaScript 代码，比如直接执行新的 JavaScript 程序或子程序（例如从控制台，或通过在 `<script>` 元素中运行代码），执行一个事件触发的回调函数、创建的定时器到达执行其回调，这些任务都会被添加到**任务队列**（`Task Queues`）上。上面👆提及的消息队列及此处的任务队列，每个“`消息`” 的关联函数即此处的宏任务。

**`Microtask`（微任务）：** 微任务和宏任务都是由JavaScript 代码组成，但表现在任务执行时机和事件循环处理方式的差异。
- 每次事件循环只能处理单个宏任务，而在每个宏任务之后，事件循环会运行微任务队列（`Microtask Queue`）中的所有微任务，然后再执行渲染或其他宏任务。
- 如果运行微任务期间加入了新的微任务（如一个微任务中通过调用 [queueMicrotask()](https://developer.mozilla.org/zh-CN/docs/Web/API/WindowOrWorkerGlobalScope/queueMicrotask) 添加微任务），新加入的微任务会早于下一个宏任务执行，事件循环会持续调用微任务直至微任务队列为空。因此递归增加微任务可能导致事件循环无尽的处理微任务的风险。

> `queueMicrotask(() => { /* 微任务中将运行的代码 */ });`*

### Jobs、Job Queues

`Jobs`（作业）、`Job Queues`（作业队列）是来自 [ECMAScript 规范](https://262.ecma-international.org/6.0/#sec-jobs-and-job-queues)中的概念，规范中要求 Promise 的回调执行（`Job`）加入到名为 `PromiseJobs` 的 `Job Queues` 中。

考虑到 ECMAScript 是能在更多环境下运行的语言（不仅仅浏览器 HTML），规范制定者选择不在规范中使用与 HTML 规范中相同的术语。但我们可以把 `Job` 和 `Microtask`、`Job Queues` 和 `Microtask Queue` 的概念等同起来，所以`Promise`、`Async/await`（在 `Promise` 基础上实现）创建的任务称为微任务，因此表现出比 `setTimeout(fn, 0)` 更高的执行优先级。可以通过 `Promises`、`MutationObserver` 等方式添加微任务。

``` js
setTimeout(() => console.log('1'));

new Promise((resolve, reject) => {
  console.log('2')
  return resolve()
})
.then(() => console.log('3'))
.then(() => console.log('4'))

console.log('5');
```

输出顺序：
1. `2`：立即执行的同步代码。
2. `5`：常规的同步代码。
3. `3`：通过 `.then` 添加一个微任务，当前事件循环执行。
4. `4`：通过第二个 `.then` 添加到微任务队列，当前事件循环需要持续调用微任务直至清空微任务队列。
5. `1`：宏任务，下一个事件循环执行。

## 参考
- [event-loop](https://flaviocopes.com/javascript-event-loop/)
- [并发模型与事件循环](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/EventLoop)
- [Event loop: microtasks and macrotasks](https://javascript.info/event-loop)
- [In depth: Microtasks and the JavaScript runtime environment](https://developer.mozilla.org/en-US/docs/Web/API/HTML_DOM_API/Microtask_guide/In_depth)
