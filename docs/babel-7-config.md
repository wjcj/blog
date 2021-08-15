Babel 7.x 配置与使用
=====

本文主要介绍 Babel 7 的基本使用方式与其新特性在 monorepos 类型项目中配置应用。可以从[这里](https://github.com/wjcj/babel-7-monorepo-demo) `clone` 示例源码，简单的示例也可以使用 Babel 提供的[在线编译平台](https://babeljs.io/repl/)。

## 开始

Babel 是一个 JavaScript 编译器。主要用于将 ECMAScript 2015+ 代码转换成能够运行在当前或旧版本的浏览器或其他环境中的语法。

> 语法转换；通过 Polyfill 方式在目标环境中添加缺失的特性（通过第三方 polyfill 模块，例如 core-js，实现）；源码转换 (codemods)。

官网中提到 Babel 会为你做的几件事👆，这是因为 ECMAScript 2015+ 代码中除了包含新的语法，还包含一些新的特性：内置对象（如 `Promise`）、内置对象新增静态属性（如 `Array.from`）和实例属性（如 `[].includes`）。语法可以通过插件转换为 ES5 语法实现旧版浏览器兼容，而新增特性则通过注入 polyfill 实现，polyfill 新增特性到全局环境（`global scope`）、原生对象（如 `Array`）或其原型（如 `Array.prototype`）上，从而也会导致全局环境污染。Babel 使用 `core-js` 来提供的 polyfill。

**`@babel/polyfill` 开始已废弃，Babel 7.4.0 版本开始不推荐使用**

`@babel/polyfill` 是 Babel 提供 polyfill 独立出来的一个库。其包含 `core-js` 和一个自定义的 `regenerator runtime` 来模拟完整的 ES2015+ 环境，以前这个库本身等价于：

``` js
import "core-js/shim";  // 包含 < Stage 4 的阶段提案
import "regenerator-runtime/runtime"; // 生成器函数（generator functions）
```

使用 `@babel/polyfill` 时会注入全部的 polyfill，这对于一个库/工具来说太多了，或者你无法完全控制代码的运行的环境，污染全局可能会带来一些问题。所以它仅适用于应用程序或命令行工具。现在的 `@babel/polyfill` 本身使用 `core-js v2` 中标准 ECMAScript 特性的 polyfill，即：

``` js
import "core-js/stable"; // Stage 4 的阶段提案
import "regenerator-runtime/runtime";
```

如果你需要填充处于提案阶段（< Stage 4）的特性，需要安装 `core-js v2+` 并从其中导入对应的 polyfill，比如：

``` js
// for core-js v2:
import "core-js/fn/array/flat-map";
// for core-js v3:
import "core-js/features/array/flat-map";
```
而 `core-js v3` 也已经完全模块化，直接使用 `core-js` 可以根据需要单独引入某个 polyfill，配合 `@babel/preset-env` 可以根据目标环境智能的引入需要的 polyfill。所以 `@babel/polyfill` 的价值已经不大了。

## 常用命令

```
npm install @babel/cli @babel/core --save-dev
```

- `@babel/cli`：内置的 CLI 命令行工具，可通过命令行编译文件。
- `@babel/core`：Babel 核心功能模块。包含了解析、转换、生成相关 [API](https://babeljs.io/docs/en/babel-core#transformsync)。

常用命令：
``` bash
npx babel example.js
# 指定输出文件和启动监听模式
npx babel example.js --out-file compiled.js --watch
# 简写
npx babel example.js -o compiled.js -w

# 编译目录
npx babel src --out-dir lib
# 简写
npx babel src -d lib
```
更多 `babel` 命令查看[这里](https://www.babeljs.cn/docs/babel-cli)。

*执行 npx babel 之前首先要安装 `@babel/cli` 和 `@babel/core`，否则 npx 将安装老旧的 `babel 6.x` 版本。*

如果不想每次都需要输入 `npx` 和如此长的指令，可以将命令写入 npm 运行脚本，`npm run build` 编译 `src` 目录中的所有文件：

``` json_
{
  "scripts": {
    "build": "babel src --out-dir lib --watch"
  }
}
```
区别于 webpack、rollup 的编译方式，Babel 是文件到文件的编译，不需要指定从某个入口文件开始。

## 配置文件

执行上面的 `babel` 命令，你会发生它什么都不会做，只是将代码拷贝到另一处。只有为 Babel 指定了插件（plugins）或预设（presets，一组插件），Babel 才会对代码进行转换。

通常通过创建配置文件实现 Babel 配置，Babel 7.x 中存在两种配置文件格式：

- `项目范围配置`（Project-wide configuration）：使用 `babel.config.json` 文件，支持 `.js`, `.cjs`, `.mjs` 扩展名。后文中将称呼其为**`babel.config.json`**。

- `相对文件配置`（File-relative configuration）：使用 `.babelrc.json` 文件（`.babelrc` 是 `.babelrc.json` 的别名），同样支持不同的扩展名 `.js`, `.cjs`, `.mjs`。后文中将称呼其为**`.babelrc.json`**。

这两种配置文件可以一起使用，也可以独立使用。如果你想对某文件或某目录的子集上运行某些转换插件，使用 `.babelrc.json`。
如果您的项目中有多个包（即多个 `package.json`）目录且需要独立的 Babel 配置，那么加入 `babel.config.json` 会很有用，这种情况通常在 `monorepo` 项目中（一个项目中多个子包，且子包间存在互相依赖的。后文中有专门针对此场景的 Babel 使用介绍）。如果只是常规项目（一个项目即一个包），使用 `.babelrc.json` 即可。

`babel.config.json` 对整个项目生效，包括 `node_modules`，`babel.config.json` 作为通用配置在子包共享，同时每个子包也可以做个性配置项。

## 配置选项

这里先介绍常规项目中常用的配置项，所有配置选项查看[这里](https://www.babeljs.cn/docs/options)。

``` json_
// .babelrc.json
{
  // 插件列表。详见👇
  "plugins": [], 
   // 预设列表。详见👇
  "presets": [],
}
```

**配置项详解：**

### `plugins`（插件）

插件是小型的 JavaScript 程序，用于告诉 Babel 如何对代码进行转换。

Babel 的代码转换功能通过将插件（或预设）应用到配置文件来启动。<br/>

Babel 编译代码的过程分为三个阶段：
1. 解析（parse）：将代码字符串解析成 AST（抽象语法树）；
2. 转换（transform：对 AST 进行转换，转换后依然还是 AST；
3. 生成（generate）：将转换后的 AST 生成代码字符串。

而插件分为 `语法插件`（Syntax Plugins）和 `转换插件`（Transform Plugins）两种。官方语法插件和转换插件的名称分别以 `@babel/plugin-syntax`、`@babel/plugin-transform` 开头。<br/>
语法插件在解析阶段作用于解析器（`@babel/parser`），扩展其语法解析能力，通过转换插件来实现代码转换。如果已经配置相应的转换插件，则不需要指定语法插件，因为会自动启用它，所以提到 Babel 插件通常指的是转换插件。

例如添加 `@babel/plugin-transform-arrow-functions` 插件后可解析并转换箭头函数：

``` json_
{
  "plugins": [
    "@babel/plugin-transform-arrow-functions"
  ]
}
```
源码：
``` js
var a = b => b;
```

转换后：
``` js
var a = function (b) {
  return b;
};
```

### `presets`（预设）

表示一组插件的集合。<br/>

如果希望使用 Babel 将 ES2015（ES6）代码编译成 ES5，需要配置[很多插件](https://babeljs.io/docs/en/plugins-list#es2015)，一个一个插件进行安装配置十分不便。<br/>
如果使用Babel 预设 `@babel/preset-es2015` 则特别简单，因为其包含了 ES2015 特性的所有插件。

``` bash
npm install @babel/preset-es2015 --save-dev 
```

``` json_
// .babelrc.json
{
  "plugins": [],
  "presets": [
    "@babel/preset-es2015"
  ],
  // 此选项即将被废弃。
  "env": {
    "development": {
      "presets": [["@babel/preset-react", { "development": true }]]
    }
  }
}
```

在 Babel 7 中，所有 `Stage-x` 预设（`babel-preset-es2015` 、`babel-preset-es2016` ~ `babel-preset-latest`）都已经废弃，使用 `@babel/preset-env`（详见👇） 替代。

### 插件顺序、传递参数

**插件顺序**

插件的排列顺序很重要，如果两个转换插件都将处理某一段代码将根据下面规则执行：

- 插件在预设前运行；
- 插件顺序从前往后排列；
- 预设顺序是颠倒的（从后往前）。

``` json_
{
  "plugins": ["pluginA", "pluginB"],
  "presets": ["presetA", "presetB"]
}
```
执行顺序：pluginA => pluginB => presetB => presetA

**传递参数**

插件参数和预设参数的传递方式一样，将 `插件名/预设名` 和 `参数对象` 组成一个数组在配置中设置：

``` json_
{
  "plugins": [
    "pluginA",
    ["pluginA", { "option1": value }] // 传递参数
  ],
  "presets": [
    "presetA",
    ["presetA", { "option1": value }]
  ]
}
```

## monorepos 项目

monorepos 项目中存在多个子包，`babel.config.json` 和 `.babelrc.json` 可以配合使用。下面是 monorepos 场景一些常用到的配置选项：

``` js
{
  // 继承其他配置文件中的配置，当前配置文件中的配置字段将合并到扩展文件的配置之上。
  "extends": String,
  // 指定仅适用于存储库中某些子目录的配置
  "overrides": [],
  // 匹配目录或文件。通常与 overrides 一起使用
  "test": MatchPattern | Array<MatchPattern>,
  // babel 默认情况下不会加载任何子包中的 .babelrc.json，除非 babel.config.js 中配置了 babelrcRoots 选项。
  // 例如 ['.', './packages/*'] 表示对当前根目录和所有子包开启 .babelrc.json 的加载，根目录可同时存在 babel.config.js 和 .babelrc.json。
  "babelrcRoots": [],
}
```

下面👇选项仅允许作为 Babel 程序选项使用。例如 `babel-loader` 的 `options` 选项中（下面有示例）、命令行选项中（`babel --rootMode=upward`）等：

``` json_
{
  // “根”目录初始路径，默认为 opts.cwd
  "root": string,
  // 项目最终 'root' 值的不同方式，有三个选项：
    // - 'root'：传递 root 值不变
    // - 'upward'：自动向上找 babel.config.json 做为根目录位置，没找到就抛错；
    // - upward-optional：类似 upward，没找到回退。
  "rootMode": string,
  // Babel 默认在“根”目录自动搜索 babel.config.json 作为项目范围配置，此选项可以指定 项目范围配置文件，从而覆盖默认的搜索行为。
  "configFile": string | boolean,
}
```

结合一个 monorepos 项目（如下目录结构）对上面的配置选项进行使用说明：

```
.
├── packages
│   ├── package-a 
│   │    ├── src
│   │    │    └── index.js
│   │    ├── .babelrc.json
│   │    └── package.json
│   │
│   └── package-b
│        ├── src
│        │    └── index.js
│        ├── .babelrc.json
│        └── package.json
│
├── babel.config.json
└── package.json
```

Babel 7 中引入了一个 `root`（根）的概念，默认为当前工作目录（`process.cwd()`）。Babel 将在此根目录中自动搜索 `babel.config.json` 文件，指定 `"configFile"` 选项可以指定项目范围配置文件，从而覆盖此行为。

通常可以将所有子包的 Babel 配置放在“根”目录的 `babel.config.json` 中，通过 `"overrides"` 选项可以对项目中某些子目录文件进行独立配置。
比如下面👇 `babel.config.json` 中 `"@babel/preset-env"` 预设是共享的，但对 `package-a` 中的 JSX 元素进行了 `automatic` 模式转换（引入了 `react/jsx-runtime` 模块），其他包中的文件则使用经典的 `classic` 转换（将 jsx 转换为 `React.createElement` 调用）。

``` json_
// babel.config.json
{
  "presets": [
    "@babel/preset-env",
    ["@babel/preset-react", { "runtime": "classic" }]
  ],
  "overrides": [{
    "test": "./packages/package-a",
    "presets": [
      ["@babel/preset-react", { "runtime": "automatic" }]
    ]
  }]
}
```
*`@babel/preset-env` 和 `@babel/preset-react` 的详细使用可查看[Babel 常用插件、预设详解](https://github.com/wjcj/blog/issues/39)。*

如果你想在特定子包中运行 Babel（当前工作目录与 Babel “根”目录不一致时），上面的配置将会出现问题。比如你 `cd ./packages/package-a` 目录下后执行 `webpack` 打包将报错。

在 `babel-loader` 配置选项中添加 `rootMode: 'upward'` 就可以恢复正常👇，`rootMode: 'upward'` 表示 Babel 会自动向上找 `babel.config.json` 做为“根”目录位置（即`root` 选项的值）。

``` js
module.exports = {
  mode: 'development',
  module: {
    rules: [
      {
        test: /\.(js|jsx)$/,
        loader: require.resolve('babel-loader'),
        options: {
          rootMode: 'upward',
        }
      }
    ]
  }
};
```

Babel 默认情况下不会加载任何子包中的 `.babelrc.json`。`babel.config.json` 设置 `babelrcRoots` 后才会启用。下面配置表示启用所有子包的 `.babelrc.json` 文件配置。
此选项通常用于替代 `overrides` 选项来实现子包的个性配置。

``` json_
{
  "babelrcRoots": ["./packages/*"]
}
```

Babel 配置选项非常多，查看更多选项请查看[官网文档](https://babeljs.io/docs/en/options)。

## 参考

- [Babel](https://www.babeljs.cn/docs/)
- [Babel 用户手册](https://github.com/jamiebuilds/babel-handbook/blob/master/translations/zh-Hans/user-handbook.md#toc-babel-core)
