esm、amd、cjs、umd
====

作为模块提供者，我们希望产出多种模块格式（`esm`，`amd`，`cjs` 和 `umd`）的文件，这样用户可以根据使用场景去选择文件，或者通过配置 `package.json` 模块入口相关的字段让 `webpack` 这样的工具可以根据目标环境（即 [target](https://webpack.docschina.org/configuration/target/#target)）选择模块文件。本文梳理一下这几种模块规范的区别，归纳 `package.json` 配置方式。

## 先说结论

![模块格式差异](https://miro.medium.com/max/1400/1*ohOcheaTdGZpG4nnK97swg.png)

### 核心差异总结

| 模块规范 | 浏览器 | nodeJs | Tree shaking | 常见场景 |
| -- | ---- | ---- | ---- | ---- |
| `cjs`（`CommonJS`） | 否 | 是 | [不完全支持](https://webpack.docschina.org/blog/2020-10-10-webpack-5-release/#commonjs-tree-shaking) | - nodeJS 模块 <br> - 有 node 场景需要，如跨平台库，`ssr`，jest 测试 |
| `amd` | 是 | 否 | 否 | 推荐使用 `umd`  |
| `umd` | 是 | 是 | 否 | 兼容了 `cjs`、`amd` <br> - 开启 `cdn` 服务，用于浏览器 `<script>` <br> - 可作为跨前后端的备用模块 |
| `esm`（`ES2015`） | 是（高级浏览器） | 是（>=nodeJs14） | 是（因为支持静态分析） | - 支持打包工具 `tree shaking`的模块，能大大减小包体积 <br> - 以 `.mjs`文件通过浏览器 `<script type="module">` 加载 |

### `package.json` 字段配置

`package.json` 中 `main` `module` `browser` `unpkg` 字段用于指定不同场景的入口文件。<br/>
`webpack` 通过目标环境设定（即 [target](https://webpack.docschina.org/configuration/target/#target)）结合 `package.json` 去解析要使用的文件（按优先级查找），例如浏览器环境（即`target: 'web'`），默认的优先级为：`['browser', 'module', 'main']`。而服务端渲染中常用的 `target: 'node'` 查找优先级为 `['module', 'main']`。也可以通过 [resolve.mainFields](https://webpack.docschina.org/configuration/resolve/#resolvemainfiles) 来指定查找优先级。

因此 `package.json` 配置务必正确：
- `main`: 指向一个 `cjs` 模块入口文件。
- `module`: 指向一个 `esm` 模块入口文件。构建工具（如 webpack）在构建项目时，存在此字段则会优先使用此字段指向的文件（可以提升 `tree shaking` 效果），如果未定义则回退到 `main` 字段指向的文件。
- `browser`: 指向一个支持 `umd` 或 `amd` 的文件，**模块仅在浏览器环境下使用时才可指定此字段**。表示模块仅在浏览器环境下使用，这有助于提示用户此模块中使用了 nodeJs 模块中不存在的对象或 API（比如 `window`）。
- `unpkg`: 指向一个 `umd` 格式文件，模块发布到 npm 后将自动开启 [cdn](https://unpkg.com/) 服务，浏览器 `<script>` 标签通过 `cdn` URL格式引入模块失败将回退使用 `main` 属性指定的文件。

示例：
``` json
{
  "name": "packageName",
  "main": "lib/index.js",
  "module": "es/index.esm.js",
  "unpkg": "dist/inc-utils.min.js",
  "files": [
    "dist",
    "lib",
    "es"
  ],
}
```
## 慢慢道来

### `cjs`

即 `CommonJS`，因为 javaScript 在服务端无模块化方案而诞生，众所周知 NodeJS 采用的就是此规范。

- 模块是同步加载的，因为服务端模块文件通常都已经放在服务器自身硬盘，加载速度快；
- 通过 `exports` 或 `modules.exports` 导出模块，`require` 引入模块；
- 模块首次执行后就会缓存，后续加载都返回缓存结果，如果想要再次执行，可清除缓存；
- 模块输出是值的拷贝，输出模块内部的变化也不会影响这个值。

### `amd` 和不重要的 `cmd`

`Asynchronous Module Definition`，即异步模块定义。因为浏览器模块都放在服务端，如果使用 `cjs` 同步模块机制，资源请求过程存在等待时间而造成页面假死，因此支持模块异步加载的 `amd` 诞生。
`cmd` 与 `amd`很类似，不同点在于 `amd` 推崇依赖前置、提前执行，`cmd` 则推崇依赖就近、延迟执行。


### `umd`

`Universal Module Definition`，通用模块定义。因为以浏览器端优先的 `cmd` 与 `amd` 都与服务端的 `cjs` 规范不兼容，这就缺少了一个可跨前后端的解决方案。所以兼容了 `amd` 和 `cjs` 的 `umd` 诞生。

### `esm`

`ES Modules` / `ES6 Modules` / `ES2015 modules`，即ES6中引入的标准语法格式。在 ES6 之前，`amd` 和 `cjs` 都只是社区制定了一些模块加载方案，而 `esm` 则是在语言标准的层面上自带的模块系统，是浏览器和服务器通用的模块解决方案。目前部分浏览器还不支持ES6语法，需要使用 `babel` 转换为 ES5。

- 支持同异步加载模块；
- 通过 `export` / `import` 导出/导入模块
-  `esm` 是静态模块，编译时加载，因此编译时就能确定模块的依赖关系、可做静态分析，完全支持 webpack rollup 的 `Tree shaking` 功能。而 `cjs` 只能在运行时加载，无法在编译时阶段做静态分析；
- `esm` 输出的是值的引用，输出模块的内部改变会影响引用值的改变。

### 输出结果差异

``` js
export function foo (x, y) {
	return x + y;
}
```

将👆代码编译为不同的模块格式的结果：（*也可以前往 [rollupjs.org/repl](https://rollupjs.org/repl) 对比查看*）

AMD：
``` js
define(['exports'], function (exports) { 'use strict';
	function foo (x, y) {
		return x + y;
	}
	exports.foo = foo;
	Object.defineProperty(exports, '__esModule', { value: true });
});
```

CommonJS：
``` js
'use strict';
Object.defineProperty(exports, '__esModule', { value: true });
function foo (x, y) {
	return x + y;
}
exports.foo = foo;
```

UMD：
``` js
(function (global, factory) {
	typeof exports === 'object' && typeof module !== 'undefined' ? factory(exports) :
	typeof define === 'function' && define.amd ? define(['exports'], factory) :
	(global = typeof globalThis !== 'undefined' ? globalThis : global || self, factory(global.myBundle = {}));
}(this, (function (exports) { 'use strict';
	function foo (x, y) {
		return x + y;
	}
	exports.foo = foo;
	Object.defineProperty(exports, '__esModule', { value: true });
})));
```

实现原理：先判断是否支持 `cjs`，否则再判断是否支持 `amd`，前两个都不存在，则将模块暴露到全局对象（`window` 或 `global`）。

ES Modules：
``` js
export function foo (x, y) {
	return x + y;
}
```

## 参考

- [Packaging a UI Library for Distribution](https://blog.bitsrc.io/packaging-a-ui-library-for-distribution-d153219def28)
- [为npm包提供多入口](https://xiaoiver.github.io/coding/2017/07/17/%E4%B8%BAnpm%E5%8C%85%E6%8F%90%E4%BE%9B%E5%A4%9A%E5%85%A5%E5%8F%A3.html)