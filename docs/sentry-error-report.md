Sentry 错误采集原理
====

本文介绍前端常见错误/异常类型的捕获方式和上报方式，配合 [Sentry](https://github.com/getsentry/sentry-javascript) 源码说明其异常采集原理。

## 错误监控

错误采集的前提是捕获错误，尽可能全面的收集错误信息。

### 错误类型以及捕获方式

1. JS 语法错误

即语法上不合法的代码错误。语法错误会导致整个 JS 文件无法执行，所以 `try-catch` 不会生效，但好在语法错误在开发阶段就很容易被发现。

``` js
try {
  var = 1; // Uncaught SyntaxError: Unexpected token '='
  var a = 1；// Uncaught SyntaxError: Invalid or unexpected token
} catch (e) {
  console.log(e); // 捕获不到
}
```

如果非要捕获语法错误进行上报的话，使用 `window.onerror` 也能捕获到语法错误，但需要保证 `window.onerror` 在单独的 `<script>` 中，否则语法错误将导致 `window.onerror` 的代码无法执行。

``` js
window.onerror = function(message, source, lineno, colno, error) {
  console.log('我接收到了错误！');
  return true;
}
```

`window.onerror` 可以获取到错误信息、发生错误的脚本URL、发生错误的行号、发生错误的列号和 Error 对象，`return true` 可让浏览器不输出错误信息到控制台。

2. JS 代码运行时错误

运行时错误包含除语法错误之外的其他错误类型，比如：

- `RangeError`（范围错误）：表示你使用的值超出合法值范围。比如 `numObj.toFixed(digits)` 传入 `digits` 不在 0 ~ 20 之间（实现环境可能支持更大或更小的值，比如 Chrome 支持 0 ~ 100）。
- `ReferenceError`（引用错误）：表示你使用（引用）了未声明的变量。
- `TypeError`（类型错误）：表示你使用的操作超出预期类型的范围，比如从 `null` 读取属性。
- `URIError`（URI 错误）：表示你在 URI 函数中使用非法字符，比如将带有百分比符号的 URI 传递给 `decodeURI` 或 `decodeURIComponent` 函数。

运行时错误还可以分为同步错误与异步错误。值得注意的是异步错误无法通过 `try-catch` 捕获：

``` js
try {
  setTimeout(() => {
    a(); // Uncaught ReferenceError: a is not defined
  })
} catch(e) {
  console.log(e); // 感知不到
}
```

``` js
window.onerror = function(message, source, lineno, colno, error) {
  console.log('我接收到了错误！');
}
window.addEventListener('error', event => {
  console.log('我也接收到了错误！');
}, false);
```

但所有的运行时错误都可以通过 `window.onerror` 捕获，缺点是其很容易所覆盖，所以有些人会使用 `window.addEventListener` 来监听全局的 `error` 事件。而 `addEventListener` 的不足是获取的信息不如 `window.onerror` 丰富，所以比较推荐的方式是重写 `window.onerror` 方法。

``` js
let _oldOnErrorHandler = null;
function instrumentError() {
  _oldOnErrorHandler = global.onerror;

  global.onerror = function (msg, url, line, column, error) {
    triggerHandlers('error', { msg, url, line, column, error });

    if (_oldOnErrorHandler) {
      return _oldOnErrorHandler.apply(this, arguments);
    }

    return false;
  };
}
```

3. Promise 错误

Promise 错误指 Promise 被 `reject(reason)` ，但却没有 reject 处理器（`.catch(onRejected)` 或 `.then(onFulfilled, onRejected)`）。如果 Promise 中存在运行时错误也会将 Promise 变为 `rejected` 状态。

``` js
new Promise((resolve, reject) => {
  reject(new Error('Reason'));
});
```

由于此类错误无法被 `try-catch` 和 `onerror` 捕获，但会触发 `unhandledrejection` 事件，所以可以通过全局监听 `unhandledrejection` 事件来捕获。如果使用 `window.onunhandledrejection` ，依然可以类似上面👆的方式重写 `onunhandledrejection`。

``` js
window.addEventListener('unhandledrejection', event => { 
  console.log('unhandledrejection: ', event.reason);
});
// 或者
let _oldOnUnhandledRejectionHandler = null;
function instrumentUnhandledRejection() {
  _oldOnUnhandledRejectionHandler = global.onunhandledrejection;

  global.onunhandledrejection = function (e) {
    triggerHandlers('unhandledrejection', e);

    if (_oldOnUnhandledRejectionHandler) {
      return _oldOnUnhandledRejectionHandler.apply(this, arguments);
    }

    return true;
  };
}
```

4. 静态资源加载异常

静态资源加载异常指资源（如图片、CSS、JS）加载失败。此类错误由于不会向上冒泡到 `window` 所以不能使用 `window.onerror` 捕获，但可以通过将 `window.addEventListener('error', listener, useCapture)` 中 [useCapture](https://developer.mozilla.org/zh-CN/docs/Web/API/EventTarget/addEventListener) 设置为 `true` 在事件捕获阶段进行捕获。

<!-- todo -->
``` js
window.addEventListener('error', event => {
  // event.target 触发事件的对象 (某个DOM元素) 的引用
  console.log('event.target：', event.target); // <script src="./index1.js"></script>
  if (event.target !== window) {
    console.log('页面地址: ', window.location.href);
    console.log('加载出错的资源地址:', event.target.src);
  }
}, true);

```

5. 请求异常

请求异常包含 [XMLHttpRequest](https://developer.mozilla.org/zh-CN/docs/Web/API/XMLHttpRequest) 请求异常 和 [Fetch](https://developer.mozilla.org/zh-CN/docs/Web/API/Fetch_API) 请求异常。

**XMLHttpRequest 请求异常**

XMLHttpRequest 异常可以通过 `window.addEventListener` 捕获，但无法捕获错误码，也无法监控正常请求的耗时，所以采用重写 `XMLHttpRequest` 原型方法并监听所有的 Ajax 请求更加有效，这种方式主要是修改 XMLHttpRequest 原型上的 `open` 和 `send` 方法。

`send` 方法保存 `xhr` 实例与请求参数，并完成请求开始前的[数据体](https://developer.mozilla.org/zh-CN/docs/Web/API/XMLHttpRequest/send#%E5%8F%82%E6%95%B0)、发起时间、xhr 的数据上报；`open` 方法监听了 `readystatechange` 事件，在重写的事件处理函数 `onreadystatechangeHandler` 完成请求地址、请求方法、状态码、结束时间等数据上报。下面是简化后的代码示例：

``` js
function instrumentXHR() {
  const requestKeys = []; // 保存 xhr 实例
  const requestValues = []; // 保存请求参数
  const xhrproto = XMLHttpRequest.prototype;

  fill(xhrproto, 'send', function (originalSend) {
    return function (this, ...args) {
      requestKeys.push(this);
      requestValues.push(args);

      triggerHandlers('xhr', {
        args, // 数据体
        startTimestamp: Date.now(),
        xhr: this,
      });

      return originalSend.apply(this, args);
    };
  });

  fill(xhrproto, 'open', function (originalOpen) {
    return function (this, ...args) {
      const xhr = this;
      const url = args[1];
      xhr.__xhr__ = {
        method: args[0],
        url: args[1],
      };

      const onreadystatechangeHandler = function () {
        if (xhr.readyState === 4) {
          if (xhr.__xhr__) {
            xhr.__xhr__.status_code = xhr.status;
          }
          const requestPos = requestKeys.indexOf(xhr);
          if (requestPos !== -1) {
            requestKeys.splice(requestPos);
            const args = requestValues.splice(requestPos)[0];
            if (xhr.__xhr__ && args[0] !== undefined) {
              xhr.__xhr__.body = args[0];
            }
          }

          triggerHandlers('xhr', {
            args, // xhr.open 传入的实参，包含 method, url, async 等
            endTimestamp: Date.now(),
            startTimestamp: Date.now(),
            xhr, // xhr.__xhr__ 中保存了请求方法、地址、参数、状态码等数据
          });
        }
      };

      fill(xhr, 'onreadystatechange', function (original) {
        return function (...readyStateArgs) {
          onreadystatechangeHandler();
          return original.apply(xhr, readyStateArgs);
        };
      });

      return originalOpen.apply(xhr, args);
    };
  });
}
```

`fill` 方法：
``` js
function fill(source, name, replacementFactory) {
  const original = source[name];
  const wrapped = replacementFactory(original);

  if (typeof wrapped === 'function') {
    wrapped.prototype = wrapped.prototype || {};
    Object.defineProperties(wrapped, {
      __original__: {
        enumerable: false,
        value: original,
      },
    });
  }
  return source[name] = wrapped;
}
```

**Fetch 请求异常**

捕获 Fetch 异常的方法与上面类似，改写全局的 `fetch` 方法，内部调用原本的 Fetch 方法发起网络请求。在请求发出前收集并上报 `fetch` 方法参数、请求参数请求方法、请求地址、发起时间等数据，Promise 变为终态后上报结束时间、响应数据或错误信息。

``` js
function instrumentFetch() {
  fill(global, 'fetch', function (originalFetch) {
    return function (...args) {
      const handlerData = {
        args,
        fetchData: {
          method: getFetchMethod(args),
          url: getFetchUrl(args),
        },
        startTimestamp: Date.now(),
      };

      triggerHandlers('fetch', { ...handlerData });

      return originalFetch.apply(global, args).then(
        (response) => {
          triggerHandlers('fetch', {
            ...handlerData,
            endTimestamp: Date.now(),
            response,
          });
          return response;
        },
        (error) => {
          triggerHandlers('fetch', {
            ...handlerData,
            endTimestamp: Date.now(),
            error,
          });
          throw error;
        },
      );
    };
  });
}
```

6. 跨域 `Script error`

浏览器出于安全机制考虑，对于跨域脚本中的错误只显示 `Script error` 而不提供详细的错误信息。假如他人使用了你的线上脚本，你肯定不希望他人能获取到脚本错误信息。
但是为了性能考虑通常需要使用 CDN 托管 JS 资源，所以需要捕获 CDN 跨域脚本的详细错误信息。解决的办法是开启跨域资源共享（CORS，Cross Origin Resource Sharing）：只需要在引入跨域脚本的 `script` 标签上添加 `crossorigin` 属性，同时服务端设置 JS 资源的响应头`Access-Control-Origin` 属性为 `*` 或域名（如 `http://xxx.com`），如果为 `*` 表示该资源可以被任意站点跨站引用。大部分主流 CDN 默认添加了 `Access-Control-Allow-Origin` 属性。完成这些之后，跨域脚本中的错误信息可以被 `window.onerror` 捕获。

``` html
<script src="http://another-domain.com/index.js" crossorigin></script>
```

响应头：
``` js
Access-Control-Allow-Origin: *
// 或
Access-Control-Allow-Origin: http://xxx.com
```

## 错误上报

捕获到错误信息后就需要将捕捉到的错误信息上报给数据收集平台。上报方式通常为三种：

1. 接口请求。

可以根据当前环境判断是否支持 [Fetch](https://developer.mozilla.org/zh-CN/docs/Web/API/Fetch_API)，不支持就使用 [XMLHttpRequest](https://developer.mozilla.org/zh-CN/docs/Web/API/XMLHttpRequest)。Sentry 使用的就是这种上报方式。好处是 POST 请求能携带的大量数据，不会阻塞和影响页面渲染。不足是当前域名与收集平台的域名通常是不一致而需要处理跨域限制。

2. 图片打点。

最主流的方式，通过 `new Image()` 动态创建一个 `img` 无需插入 dom 数即可发起请求实现数据上报。在 `src` 属性设置上报的 URL，URL 参数上携带错误信息。上报时通常选用了 1x1 像素的透明 gif 图片，因为相对其他图片格式体积最小，节省了流量，合法的 gif 只需要 43 字节。而且图片打点天然支持跨域，其他资源文件（如 js、css、ttf）请求虽然也没有跨域问题，但需要插入 dom 树才能发起请求，频繁 dom 操作会带来性能问题，并且 CSS、JS 资源都会阻塞浏览器渲染。唯一的不足就是由于使用 GET 请求，浏览器会对 URL 长度做限制，从而导致可携带的数据量较小（2kb ~ 8kb）。

``` js
function report(data) {
  var img = new Image(), src = `http://xxx/report?data=${encodeURIComponent(JSON.stringify(data))}`;
  
  img.onload = function success() {
    console.log('success: ', data, src);
  };

  img.onerror = img.onabort = function failure() {
    console.log('failure: ', data, src);
  };

  img.src = img;
}
```

3. sendBeacon

[navigator.sendBeacon()](https://developer.mozilla.org/zh-CN/docs/Web/API/Navigator/sendBeacon) 是一个较新的 API，它能将少量数据（64kb左右）通过异步 POST 请求发送到服务器，非阻塞请求且无需响应。浏览器会对 Beacon 请求进行调度以保证其在浏览器空闲时执行，即使页面关闭也不会影响数据发送，不会阻塞刷新、跳转等操作，保证了页面性能数据可靠。缺点就是相对兼容性要差一些。

``` js
function report() {
  navigator.sendBeacon("/report", data);
}
```

## 参考

- [前端代码异常监控](http://rapheal.sinaapp.com/2014/11/06/javascript-error-monitor/)
- [Sentry Docs](https://docs.sentry.io/platforms/javascript/)
- [sendBeacon](https://developer.mozilla.org/zh-CN/docs/Web/API/Navigator/sendBeacon)
- [raven-js](https://github.com/wingify/raven-js)