Babel 常用插件、预设详解
=====

# 常用插件

## `@babel/plugin-transform-runtime`

此插件可以复用 Babel 注入的辅助代码以减少代码体积；另一个作用可以避免全局注入 polyfill，从而避免污染全局。

Babel 会使用很多辅助函数来帮助实现代码转换，比如下面示例中的 `_extends`，这些函数统称为 `helpers` 。默认情况下，每个文件顶部会添加当前文件中所需要的 `helpers`，如果存在多个文件，就可能重复添加相同的 `helper`，编译输出中就会重复打包这些函数导致包体积增大。

``` jsx
// 源码：
<Component {...props} prop="a"/>;

// 转换后：
require("core-js/modules/es.object.assign.js");

// helper 👇
function _extends() { _extends = Object.assign || function (target) { for (var i = 1; i < arguments.length; i++) { var source = arguments[i]; for (var key in source) { if (Object.prototype.hasOwnProperty.call(source, key)) { target[key] = source[key]; } } } return target; }; return _extends.apply(this, arguments); }

React.createElement(Component, _extends({}, props, {
  prop: "a"
}));
```

`@babel/plugin-transform-runtime` 的作用就是将所有 `helpers` 都从 `@babel/runtime` 模块导入，从而避免在编译输出中出现重复。因此使用此插件还需要安装 `@babel/runtime` 作为生产依赖被编译到构建中。

``` bash
npm install @babel/plugin-transform-runtime --save-dev
npm install @babel/runtime --save
```

使用 `@babel/plugin-transform-runtime` 后：

``` js
var _interopRequireDefault = require("@babel/runtime/helpers/interopRequireDefault");

var _extends2 = _interopRequireDefault(require("@babel/runtime/helpers/extends"));

React.createElement(Component, (0, _extends2.default)({}, props, {
  prop: "a"
}));
```

`@babel/plugin-transform-runtime` 的另一个作用是避免 polyfill 全局污染。无论是导入 `core-js` 或 `@babel/polyfill`（Babel 7.4.0 开始已废弃），polyfill 的行为如全局添加内置对象（如 `Promise`）、内置对象新增静态属性（如 `Array.from`）或实例属性（如 `[].includes`），这些都会导致全局污染。虽然这对于应用程序或命令行工具来说可能没问题，但如果作为一个库供别人使用，或者你无法完全控制其运行的环境，这就会带来问题。

``` js
// 源码：
var a = new Promise();

// 转换后：
require("core-js/modules/es.promise.js"); // Promise polyfill 导致全局污染
var a = new Promise();
```

`@babel/plugin-transform-runtime` 后（插件参数选项 `"corejs": 3`，详见下面）：

``` jsx
var _interopRequireDefault = require("@babel/runtime-corejs3/helpers/interopRequireDefault");

var _promise = _interopRequireDefault(require("@babel/runtime-corejs3/core-js-stable/promise"));
var a = new _promise.default();
```

`@babel/runtime-corejs3` 是 `core-js3` 的运行时版本，转换器为其添加别名进行调用，从而无需使用 polyfill。

**插件参数选项：**

``` json_
{
  "plugins": [
    [
      "@babel/plugin-transform-runtime",
      {
        // 指定 core-js 的 runtime 版本。false 表示 @babel/runtime（只包含 helper），指定数字表示 core-js 的 runtime 版本（还包含 polyfill，如 @babel/runtime-corejs2）。详见下面👇
        "corejs": false, // false 表示 @babel/runtime
        // 指定 @babel/runtime 或对应 corejs 版本 的绝对路径。默认情况下 `@babel/runtime` 需要位于正在编译的文件的 node_modules 中，其他情况下需要指定绝对路径以保证运行时模块的位置的正确解析。
        "absoluteRuntime": false,
        // 是否复用 helpers。将内联的 helpers（如 extends）替换为对模块的调用，即复用 helpers。
        "helpers": true,
        // generator 函数是否转换为使用不污染全局范围的 regenerator runtime，详见下面👇 
        "regenerator": true,
        // 运行时模块的版本号，默认 @babel/runtime@7.0.0
        // 例如 { corejs: 2，version: '^7.7.4' } 表示依赖 @babel/runtime-corejs2@7.7.4
        "version": "7.0.0-beta.0",
        // 是否支持提案。此插件默认不填充提案，如果 `corejs: 3`，此选项为 true 可开启
        "proposals": false
      }
    ]
  ]
}
```

### `corejs`

指定 `core-js` 的 runtime 版本。Babel v7 中已经把 helper 与它在运行时的 polyfilling 行为分开了。
`false` 表示 `@babel/runtime`（仅包含 helper），指定数字（`2` 或 `3`）表示 `core-js` 的 runtime 版本（包含 helper + polyfill，如 `@babel/runtime-corejs2`）。

``` js
// false => @babel/runtime
var _extends2 = _interopRequireDefault(require("@babel/runtime/helpers/extends"));
require("core-js/modules/es.promise.js");

// 2 => @babel/runtime-corejs2
var _extends2 = _interopRequireDefault(require("@babel/runtime-corejs2/helpers/extends"));
var _promise = _interopRequireDefault(require("@babel/runtime-corejs2/core-js/promise"));

// 3 => @babel/runtime-corejs3
var _extends2 = _interopRequireDefault(require("@babel/runtime-corejs3/helpers/extends"));
var _promise = _interopRequireDefault(require("@babel/runtime-corejs3/core-js-stable/promise"));
```

此选项需要更改用于提供 runtime helpers 的依赖项：
- `false`：默认，即 `@babel/runtime`，仅包含 helpers。
- `2`：即 `@babel/runtime-corejs2`，包含 helpers + polyfill（对应 `core-js v2`）。仅支持全局变量（如 `Promise`）和静态属性（如 `Array.from`）。
- `3`：即 `@babel/runtime-corejs3`，包含 helpers + polyfill（对应 `core-js v3`）。支持全局变量和静态属性，还支持实例属性（如 `[].includes`）。

更改此选项需要安装对应的生产依赖（`@babel/runtime | @babel/runtime-corejs2 |  @babel/runtime-corejs3`），例如 `npm install --save @babel/runtime-corejs3`。
而 `@babel/plugin-transform-runtime` 作为开发环境依赖始终都是需要安装的。

### `regenerator`

`generator` 函数是否转换为使用不污染全局范围的 `regenerator runtime`。

``` js
// 源码：
function* foo() {}

// generator=false 时：
var _marked = /*#__PURE__*/regeneratorRuntime.mark(foo);
// 依赖了 regeneratorRuntime，会污染全局
function foo() {
  return regeneratorRuntime.wrap(function foo$(_context) {
    while (1) {
      switch (_context.prev = _context.next) {
        case 0:
        case "end":
          return _context.stop();
      }
    }
  }, _marked);
}

// generator=true 时：
var _regenerator = _interopRequireDefault(require("@babel/runtime/regenerator"));

function _interopRequireDefault(obj) { return obj && obj.__esModule ? obj : { default: obj }; }

var _marked = /*#__PURE__*/_regenerator.default.mark(foo);

function foo() {
  return _regenerator.default.wrap(function foo$(_context) {
    while (1) {
      switch (_context.prev = _context.next) {
        case 0:
        case "end":
          return _context.stop();
      }
    }
  }, _marked);
}
```

# 常用预设（`presets`）

## `@babel/preset-env`

`@babel/preset-env` 是一个智能预设，指定目标环境后即可使用最新的 JavaScript 特性，它会去管理需要哪些插件（以及 polyfill）。

它利用 [browserslist](https://github.com/browserslist/browserslist)、[compat-table](https://github.com/kangax/compat-table)（ECMAScript 兼容性表）数据，维护了目标环境中各版本与 JavaScript 语法或浏览器功能支持的映射表，以及这些语法和功能需要哪些 Babel 转换插件和 `core-js` polyfill 的映射表。所以只需要你设置目标环境，`@babel/preset-env` 就能找到所需的插件和 polyfill。

它同时替代了 `preset-es2015/es2016/es2017/latest` 这几个预设，对于开发者而言，直接关注目标环境而无需关注特性属于哪一套标准更加简单。

**预设参数选项：**

``` json_
{
  "presets": {
    // 项目支持的环境或目标环境。详见下面👇
    "targets": "> 0.25%, not dead",
    "bugfixes": false,
    // 如何优化 polyfill 引入，"usage"| "entry"| false, 详见下面👇
    "useBuiltIns": false,
    // 是否开启对处于提案中、且浏览器已经实现的内置对象/特性的支持。详见下面👇
    "shippedProposals": false,
    // 指定 core-js （polyfill）版本。@babel/preset-env 的 polyfill 来自于 core-js。详见下面👇
    "corejs": "2.0", // 其他如 "3.8"或"2.0", "3" 将解析为 "3.0"
    // 是否允许预设中插件启用松散模式转换。对比详见下面👇。
    "loose": false, // 默认 fasle，即 'normal'
    // 始终需包含的插件
    "include": [],
    // 始终排除/删除的插件
    "exclude": [],
    // 启用 ES 模块语法到另一种模块类型的转换。"amd" | "umd" | "systemjs" | "commonjs" | "cjs" | "auto" | false
    "modules": "auto"
  }
}
```

**主要选项说明：**

### `targets`

项目支持的环境或目标环境。

可以设置为 [browserslist-compatible](https://github.com/ai/browserslist) 查询，或者是一个描述支持的最低环境版本对象。

``` json_
{
  // browserslist
  "targets": "> 0.25%, not dead",
  // 或者
  "targets": { "chrome": "58", "ie": "11" }, // 环境可选：chrome, opera, edge, firefox, safari, ie, ios, android, node, electron
}
```

推荐使用 `browserslist` 配置（使用 `.browserslistrc` 文件或 `package.json#browserslist` 字段），因为这样可与其他工具共享配置，如 `Autoprefixer`、`postcss-preset-env`。<br/>
但与 [browserslist](https://github.com/ai/browserslist) 行为不同的是，当在 Babel 或 browserslist 配置中找不到目标配置时，它不使用 `defaults` 查询。而会将所有ES2015+ 代码转换为 ES5（类似 `preset-latest`），这种方式不推荐使用，因为没有使用其针对特定环境/版本的能力，为了避免可以显示的声明 `defaults`👇：

``` json_
{
  "presets": [["@babel/preset-env", { "targets": "defaults" }]]
}
```

### `corejs`

指定 [core-js](https://github.com/zloirock/core-js) 版本。

[core-js](https://github.com/zloirock/core-js) 是 JavaScript 的模块化标准库。包括 ECMAScript 到 2021 年的所有 polyfill（如 promises、symbols、ECMAScript 提案、一些跨平台的 WHATWG/W3C 特性和提案等），`@babel/preset-env` 依赖它来注入 polyfill。
此选项仅在与 `useBuiltIns: usage` 或 `useBuiltIns: entry` 一起使用时才有效。

默认情况下，只注入稳定 ECMAScript 功能的 polyfill，如果你想注入提案阶段的 polyfill，有三个不同的选项：

- 使用 `useBuiltIns: "entry"` 时，可以直接导入一个 proposal polyfill: `import "core-js/proposals/string-replace-all"`。

- 使用 `useBuiltIns: "usage"` 时，有两种选择：
  - 将 `shippingProposals` 选项设置为 `true`。这将为浏览器已支持的提案启用 polyfill 和转换。
  - 使用 `corejs: { version: "3.8", proposal: true }`。这将启用 `core-js@3.8` 支持的每个提案的 polyfill。

### `useBuiltIns`

配置 `@babel/preset-env` 如何优化注入 polyfill。

如果全量注入 polyfill （`import "core-js";`）将增加包体积，`useBuiltIns` 选项的作用就是对其进行优化。<br/>

**当 `useBuiltIns: 'entry'` 时：**

`@babel/preset-env` 将 `import "core-js";` 和 `import "regenerator-runtime/runtime"`（提供`generator`、`async`、`await` 的 polyfill) 语句替换为只引入目标环境可能用到的 polyfill。<br/>

``` js
import "core-js";
// 转换后 `import "core-js"` 替换为（因目标环境而异👇）：
import "core-js/modules/es.string.pad-start";
import "core-js/modules/es.string.pad-end";
```

如果你只需要知道只需要其中的一部分 polyfill 怎么办？

使用 `core-js@3` 时，`@babel/preset-env` 能够优化每个 `core-js` 入口点及其组合。例如你只想填充数组方法和 Math 新提案：
``` js
import "core-js/es/array";
import "core-js/proposals/math-extensions";
// 转换后：
import "core-js/modules/es.array.unscopables.flat";
import "core-js/modules/es.array.unscopables.flat-map";
import "core-js/modules/esnext.math.clamp";
import "core-js/modules/esnext.math.deg-per-rad";
import "core-js/modules/esnext.math.degrees";
import "core-js/modules/esnext.math.fscale";
import "core-js/modules/esnext.math.rad-per-deg";
import "core-js/modules/esnext.math.radians";
import "core-js/modules/esnext.math.scale";
```

**当 `useBuiltIns: 'usage'` 时：**

polyfill 会在需要时（目标环境不支持、当前文件中使用到的）自动导入。
`@babel/preset-env` 会在每个文件的开头引入需要的（目标环境不支持，当前文件中使用到）polyfill。<br/>

``` js
// a.js
var a = new Promise();
// b.js
var b = new Map();

// 转换后：(如果目标环境不支持)
// a.js
import "core-js/modules/es.promise";
var a = new Promise();
// b.js
import "core-js/modules/es.map";
var b = new Map();

// 转换后：(如果目标环境支持)
// a.js
var a = new Promise();
// b.js
var b = new Map();
```

### `shippedProposals`

是否开启对处于提案中的且浏览器已经实现的内置对象/特性的支持。

你的目标环境已经有了对某提案特性的原生支持，则会开启对应的语法插件以支持解析器完成解析，而不会执行任何代码转换操作。注意这不会开启与[@babel/preset-stage-3](https://www.babeljs.cn/docs/babel-preset-stage-3) 相同的转换，因为提案在登陆浏览器之前还会继续更改。

### `loose`

是否允许预设中包含的插件使用松散模式转换代码。

默认 `false`，即使用正常模式（`normal`）。Babel 中的很多插件都支持两种模式。松散模式产生更简单的 ES5 代码，而正常模式会尽可能遵循ES6 语义。

建议不要使用松散模式，松散模式优缺点：

- 优点：生成的代码更简洁，因为缺少了尽可能去遵循ES6 语义而添加的繁杂逻辑，运行速度可能更快，与旧引擎兼容性更好。
- 缺点：以后直接使用原生 ES6 时可能会遇到问题，这风险很少情况下值得冒。

下面是一段源码两种模式下的转换结果对比：

``` js
class Animal {
  constractor(name) {
      this.name = name;
  }
};
```

正常模式（normal mode）：
``` js
"use strict";

require("core-js/modules/es.function.name.js");

function _classCallCheck(instance, Constructor) { if (!(instance instanceof Constructor)) { throw new TypeError("Cannot call a class as a function"); } }

function _defineProperties(target, props) { for (var i = 0; i < props.length; i++) { var descriptor = props[i]; descriptor.enumerable = descriptor.enumerable || false; descriptor.configurable = true; if ("value" in descriptor) descriptor.writable = true; Object.defineProperty(target, descriptor.key, descriptor); } }

function _createClass(Constructor, protoProps, staticProps) { if (protoProps) _defineProperties(Constructor.prototype, protoProps); if (staticProps) _defineProperties(Constructor, staticProps); return Constructor; }

var Animal = /*#__PURE__*/function () {
  function Animal() {
    _classCallCheck(this, Animal);
  }

  _createClass(Animal, [{
    key: "constractor",
    value: function constractor(name) {
      this.name = name;
    }
  }]);

  return Animal;
}();
```

松散模式（loose mode）：
``` js
"use strict";

require("core-js/modules/es.function.name.js");

var Animal = /*#__PURE__*/function () {
  function Animal() {}

  var _proto = Animal.prototype;

  _proto.constractor = function constractor(name) {
    this.name = name;
  };

  return Animal;
}();
```

其他选项查看[这里](https://www.babeljs.cn/docs/babel-preset-env#options)。

## `@babel/preset-react`

解析、转换 React JSX。

此预设包含的插件：
- `@babel/plugin-syntax-jsx`：解析 JSX。
- `@babel/plugin-transform-react-jsx`：转换 JSX。
- `@babel/plugin-transform-react-display-name`：添加 `displayName`。如 `var bar = createReactClass({});` 转换为 `var bar = createReactClass({ displayName: "bar" });`

启用 `development` 选项后，如果预设选项 `runtime` 设置为 `classic` 还将添加以下两个插件：

- `@babel/plugin-transform-react-jsx-self`：在 JSX 元素添加 `__self` prop，React 将使用它来生成一些运行时警告。
``` jsx
<sometag />
// =>
<sometag __self={this} />
```
- `@babel/plugin-transform-react-jsx-source`：将源文件、行号和列号添加到 JSX 元素。
``` jsx
<sometag />
// =>
<sometag __source={ { fileName: 'this/file.js', lineNumber: 10, columnNumber: 1 } } />
```

**预设参数选项：**

``` json_
{
  "presets": [
    [
      "@babel/preset-react",
      {
        // 使用哪种运行时转换 jsx。'classic' 将 jsx 转换为 `React.createElement` 调用，'automatic' 将引入 `react/jsx-runtime`，详见下面对比👇
        "runtime": "classic", // 默认 'classic'
        // 开启特定于开发环境的某些行为。例如此选项为 true 且 runtime='classic'时，将增加两个插件用于在 jsx 元素上添加 `__source` 和 `__self` prop。
        "development": false, // 默认 false
        // 如果使用 XML 命名空间标记名称（如 `<f:image />`）是否抛出错误。虽然 JSX 规范允许，但是默认禁止，因为 React 所实现的 JSX 目前并不支持这种方式。
        "throwIfNamespace": false, // 默认 true

// 以下选项👇仅在 'automatic' runtime 生效：
        // 替换导入源。详见下面对比👇
        "importSource": "custom-jsx-library", // 默认 `react`

// 以下选项👇都仅在 'classic' runtime 生效：
        // 替换编译 JSX 表达式时使用的函数（仅 'classic' runtime 生效）。
        "pragma": "dom", // 默认 `React.createElement`
        // 替换编译 JSX 片段时使用的组件（仅 'classic' runtime 生效）
        "pragmaFrag": "DomFrag", // 默认 `React.Fragment`
        
        // todo
        // Will use the native built-in instead of trying to polyfill behavior for any plugins that require one.
        "useBuiltIns": false, // 默认 false

        // 处理 props 合并时（例如 <Component {...props} prop="a"/>），直接使用带有扩展元素的内联对象（`{ ...props, prop: "a" }`），还是使用 Babel 的 `extend` helper 或 `Object.assign`（`_extends({}, props, { prop : "a" })`）。详见下面对比👇
        "useSpread": false // 默认 false
      }
    ]
  ]
}
```

主要参数选项说明：

### `runtime`

`classic` 是旧的转换 jsx 的方式，存在一些缺陷，如必须 `React` 环境，`React.createElement` 有一些无法做到的[性能优化和简化](https://github.com/reactjs/rfcs/blob/createlement-rfc/text/0000-create-element-changes.md#motivation)。


下面是 `classic`、 `automatic` 转换 JSX 对比：

``` js
<div></div>

// classic
React.createElement('div', null);

// automatic
import { jsx as _jsx } from "react/jsx-runtime";
_jsx("div", {});
```

- `importSource`：替换默认的导入源。默认使用 `react/jsx-runtime`，若设置为 `"custom-jsx-library"`，则替换为 `custom-jsx-library/jsx-runtime`。

``` js
// react
import { jsx as _jsx } from "react/jsx-runtime";
import { Fragment as _Fragment } from "react/jsx-runtime";
import { jsxs as _jsxs } from "react/jsx-runtime";

// custom-jsx-library
import { jsx as _jsx } from "custom-jsx-library/jsx-runtime";
import { Fragment as _Fragment } from "custom-jsx-library/jsx-runtime";
import { jsxs as _jsxs } from "custom-jsx-library/jsx-runtime";

```

### `useSpread`

``` jsx
<Component {...props} prop="a"/>;

// false
function _extends() { _extends = Object.assign || function (target) { for (var i = 1; i < arguments.length; i++) { var source = arguments[i]; for (var key in source) { if (Object.prototype.hasOwnProperty.call(source, key)) { target[key] = source[key]; } } } return target; }; return _extends.apply(this, arguments); }

React.createElement(Component, _extends({}, props, {
  prop: "a"
}));

// true
React.createElement(Component, { ...props,
  prop: "a"
});
```

## 参考

- [Babel](https://www.babeljs.cn/docs/)
- [Babel loose mode](https://2ality.com/2015/12/babel6-loose-mode.html) 
