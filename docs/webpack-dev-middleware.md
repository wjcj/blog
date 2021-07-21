webpack-dev-server vs webpack-dev-middleware vs webpack-hot-middleware
======

## 概念

- `Live Reload`：即实时重载。模块更改则自动刷新整个应用。
- `HMR`（Hot Module Replacement）：即[模块热替换](https://webpack.docschina.org/guides/hot-module-replacement/)。webpack 的功能之一，它允许通过 HMR API 在运行时更新所有类型的模块，而无需完全刷新，也就是常说的局部刷新。
- `Hot Reloading`：即热重载。基于 webpack HMR 功能实现的功能特性，可以直接理解为 HMR。表现为在保持应用程序运行同时替代已修改的模块，而且不会丢失应用状态。

## webpack-dev-middleware

`webpack-dev-middleware` 是一个中间件。主要作用是以监听模式启动 webpack，将编译后的文件输出到内存，为服务器提供资源服务，比如 `webpack-dev-server` 就是它与 `express` 封装的服务器。同时它也可以作为单独的包用来使用，以便进行更多的自定义设置。

**如何输出文件到内存？**

`webpack-dev-middleware` 内部依赖 `memory-fs` 模块，借助 `memory-fs` 的能力将 webpack `outputFileSystem` 替换成 `MemoryFileSystem` 对象，这是个 javaScript 对象，，从而实现文件系统输出到内存。避免了写文件时的消耗，访问内存中的资源信息也更快。

## webpack-dev-server

`webpack-dev-server` 是一个能基于 webpack 配置（`devServer` 配置项）快速启动的服务器（`express` + `webpack-dev-middleware`），其中使用 `webpack-dev-middleware` 来提供最新的 webpack 打包资源服务。另外在浏览器端生成 WebpackDevServer 客户端，使用 sockjs 在客户端和服务端之间建立一个 websocket 长连接，处理客户端中热更新 (HMR) 请求。

**客户端的主要作用？**

当文件改动时 webpack 会重新编译，客户端监听来自服务端推送的、与文件变更有关的 websocket 消息（`hash` 和 `ok` 消息），当客户端收到 `ok` 消息后会执行 `reloadApp`方法进行更新，
`reloadApp` 方法中会对 `devServer.hot` 配置进行判断是否支持热更新，如果支持则触发 `webpackHotUpdate` 事件应用热更新（HMR），如果不支持则直接刷新浏览器（live reload）。

**客户端是如何生成的？**

`webpack-dev-server` 通过修改 webpack `entry` 配置项添加客户端（`webpack-dev-server/client`）代码，使得 webpack 打包后的 bundle 中包含 `webpack-dev-server/client` 代码，从而监听 websocket 消息。

``` js
{
  entry: [
    require.resolve('webpack-dev-server/client') + '?/', // 👈 WebpackDevServer 客户端
    require.resolve('webpack/hot/dev-server'), // 👈 这不是客户端，监听执行热更新的事件
    // 入口
    paths.appIndexJs,
  ]
}
```

**`webpack/hot/dev-server` 的作用是什么？**

`webpack/hot/dev-server.js` 会监听来自 `webpack-dev-server/client` 发出的 `webpackHotUpdate` 事件（应用 HMR），执行热更新的整个过程交由 webpack 的 HMR API 执行。

``` js
// webpack/hot/dev-server.js
let hotEmitter = new Emitter()
hotEmitter.on('webpackHotUpdate', () => {
  if (!hotCurrentHash || hotCurrentHash == currentHash) {
    return hotCurrentHash = currentHash
  }
  hotCheck(); // HMR API，来自 webpack/lib/HotModuleReplacement.runtime
})
```

执行热更新（`hotCheck()`）的主要流程：
1. 先调用 ajax 向服务器请求检测是否有新的模块更新，有则返回更新列表（`webpack/lib/web/JsonpMainTemplate.runtime` 中 `hotDownloadManifest` 方法）；
2. 通过 jsonp 请求最新的模块代码（`webpack/lib/web/JsonpMainTemplate.runtime` 中 `hotDownloadUpdateChunk` 方法）；
3. 返回的模块代码内容是直接执行 `webpackHotUpdateCallback()` 方法（来自 `webpack/lib/HotModuleReplacement.runtime`） 进行模块热替换，热更新过程中如出现错误将回退到刷新浏览器。

## webpack-hot-middleware

也是一个中间件。当不使用 `webpack-dev-server` 时，通过 `webpack-hot-middleware` + `webpack-dev-middlerware` 实现服务器的 HMR 功能。
原理与 `webpack-dev-server` 类似，也需要添加客户端代码 `(webpack-hot-middleware/client)` 到 webpack `entry` 数组中，用于接受服务端的文件变更消息。不同的是此模块使用 `window.EventSource` 接收服务器发送的事件，不再是 websocket。

使用方式：

``` js
// webpack.config.js
var webpack = require('webpack');

module.exports = {
  entry: [
    'webpack-hot-middleware/client?path=/__webpack_hmr&timeout=20000',
    './index.js',
  ],
  // ...
};
```

``` js
// server.js
var express = require('express');
var webpack = require('webpack');
var webpackConfig = require('./webpack.config');

var compiler = webpack(webpackConfig);
var app = express();

app.use(require("webpack-dev-middleware")(compiler, {
    noInfo: true, publicPath: webpackConfig.output.publicPath
}));
app.use(require("webpack-hot-middleware")(compiler)); // 👈 这里
```

具体请查看 [webpack-hot-middleware example](https://github.com/webpack-contrib/webpack-hot-middleware/tree/master/example)。

## 参考

- [webpack-dev-middleware](https://github.com/webpack/webpack-dev-middleware)
- [webpack-dev-server](https://github.com/webpack/webpack-dev-server)
- [webpack-hot-middleware](https://github.com/webpack-contrib/webpack-hot-middleware)
- [Introducing Hot Reloading](https://reactnative.dev/blog/2016/03/24/introducing-hot-reloading)
- [Webpack HMR 原理解析](https://zhuanlan.zhihu.com/p/30669007)
- [搞懂 webpack 热更新原理](https://github.com/careteenL/webpack-hmr)
