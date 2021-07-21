Yarn PnP 原理及使用
=====

## 简介

PnP 即 Plug'n'Play，意为即插即用，是一种新的 node 依赖安装管理策略。主要作用是加快依赖安装和解析速度，并增加对忘记添加依赖项的防护。从 Yarn 2.0 开始将 PnP 作为默认设置的安装模式。

**node_modules 问题**

`node_modules` 目录通常包含大量文件，安装过程需要将依赖从缓存复制到当前目录的 `node_modules` 目录，I/O 开销非常大；当 Node 解析一个 require 依赖时，只是简单地、往上一层层地去递归 `node_modules` 目录，这种解析方式十分低效，这也是导致 Node 应用启动耗时长的部分原因；如果依赖没有找到或者发生损坏，Node 也不会有异常抛出，只有运行时才能发现。

**PnP 的方案**

PnP 的方案是抛弃了 `node_modules` 目录，由 Yarn 的解析器（resolver，位于 `.pnp.js` 中）来解析依赖，告知 Node 它需要访问的包的磁盘位置。yarn 生成一个 `.pnp.js` 文件，其中不包含已安装包的源代码，而是各种映射：一个将包名称和版本链接到它们在磁盘上的位置，另一个将包名称和版本链接到它们的依赖项列表。`.pnp.js` 还对 Node 的 `Module` 模块做了 monkey patch，修改了如 `_resolveFilename`、`_load`、`_findPath` 等方法等来实现其依赖解析器。

## 开启

运行 `npm install -g yarn` 更新全局 yarn 版本到最新 v1（本文中示例版本为 `1.22.10`），运行 `yarn set version berry` 以启用 v2（查看[安装](https://yarnpkg.com/getting-started/install)查看更多信息）。

使用 `yarn --pnp` 开启 PnP 功能。开启后 Yarn 会在 `package.json` 中增加 `installConfig` 字段，并创建一个 `.pnp.js` 文件。

``` json
// package.json
{
  "installConfig": {
    "pnp": true
  },
}
```

`.pnp.js` 文件中包含的包在磁盘中的位置信息（`locatorsByLocations`）及其依赖映射信息（`packageInformationStores`）在依赖变更后也会随之变更：

``` js
#!/usr/bin/env node

let packageInformationStores = new Map([
  [null, new Map([
    [null, {
      packageLocation: path.resolve(__dirname, "./"),
      packageDependencies: new Map([
      ]),
    }],
  ])],
]);

let locatorsByLocations = new Map([
  ["./", topLevelLocator],
]);
```

比如添加一个依赖包后（如 `yarn add qs`），两个映射表中新增了很多包的磁盘位置信息和包的依赖信息：

``` js
let packageInformationStores = new Map([
  ["qs", new Map([
    ["6.10.1", {
      packageLocation: path.resolve(__dirname, "../../../../Library/Caches/Yarn/v6/npm-qs-6.10.1-4931482fa8d647a5aab799c5271d2133b981fb6a-integrity/node_modules/qs/"),
      packageDependencies: new Map([
        ["side-channel", "1.0.4"],
        ["qs", "6.10.1"],
      ]),
    }],
  ])],
  // qs 依赖
  ["side-channel", new Map([
    ["1.0.4", {
      packageLocation: path.resolve(__dirname, "../../../../Library/Caches/Yarn/v6/npm-side-channel-1.0.4-efce5c8fdc104ee751b25c58d4290011fa5ea2cf-integrity/node_modules/side-channel/"),
      // side-channel 依赖
      packageDependencies: new Map([
        ["call-bind", "1.0.2"],
        ["get-intrinsic", "1.1.1"],
        ["object-inspect", "1.10.3"],
        ["side-channel", "1.0.4"],
      ]),
    }],
  ])],
  // ... 省略其他类似信息
  [null, new Map([
    [null, {
      packageLocation: path.resolve(__dirname, "./"),
      packageDependencies: new Map([
        ["qs", "6.10.1"],
      ]),
    }],
  ])],
]);

let locatorsByLocations = new Map([
  ["../../../../Library/Caches/Yarn/v6/npm-qs-6.10.1-4931482fa8d647a5aab799c5271d2133b981fb6a-integrity/node_modules/qs/", {"name":"qs","reference":"6.10.1"}],
  ["../../../../Library/Caches/Yarn/v6/npm-side-channel-1.0.4-efce5c8fdc104ee751b25c58d4290011fa5ea2cf-integrity/node_modules/side-channel/", {"name":"side-channel","reference":"1.0.4"}],
  // ... 省略其他类似信息
  ["./", topLevelLocator],
]);
```

## 使用

示例：
``` js
// index.js
var qs = require('qs');
var str = qs.stringify({ a: 1, b: 2 });

console.log('index.js: ', str);
```

``` json
{
  "scripts": {
    "start": "node ./index.js",
    "test": "jest"
  }
}
```

由于在开启了 PnP 的项目中不再有 `node_modules` 目录（也没有 `.bin` 目录），所有的依赖引用都必须由 `.pnp.js` 中的 resolver 处理。

``` bash
# 从命令行运行任意 Node 脚本
node -r ./.pnp.js index.js
# 即：
node --require="./.pnp.js" index.js
# 或者：
yarn node index.js
```

``` bash
# 执行 "scripts" 中脚本
yarn run jest
# 或者使用简写：
yarn jest
```

``` bash
# 从命令行运行任意 Node 脚本
yarn node index.js
```

您在一个自动执行 Node 脚本的系统上运行而您没有机会选择它们的执行方式，只需在你的 init 脚本顶部引入 PnP 文件并调用 `setup` 函数：

``` js
// index.js
require('./.pnp.cjs').setup();
```

### 调试依赖

PnP 模式下不再存在复制到 `node_modules` 的依赖可供调试，想调试依赖时使用 `yarn unplug packageName` 将指定依赖复制到项目中的 `.pnp/unplugged` 目录下。`.pnp.js ` 中的解析器会自动加载这个 `unplug` 版本。执行 `yarn unplug --clear packageName` 可将其从 `.pnp/unplugged` 目录中移除。

### webpack

Webpack 5 本身支持 PnP，如果使用 Webpack 4，则需要自行添加 [pnp-webpack-plugin](https://github.com/arcanis/pnp-webpack-plugin) 插件。

``` js
const PnpWebpackPlugin = require(`pnp-webpack-plugin`);

module.exports = {
  // 如何解析依赖
  resolve: {
    plugins: [
      PnpWebpackPlugin,
    ],
  },
  // 如何解析 loader（帮助 webpack 找到加载器在磁盘中的位置）
  resolveLoader: {
    plugins: [
      // 告诉 webpack 从 PnpWebpackPlugin 加载它的加载器
      PnpWebpackPlugin.moduleLoader(module),
    ],
  },
};
```

### `.gitignore`

`yarn install` 后会出现各种文件，你需要通过设置 `.gitignore` 忽略掉它们，这是官方[建议](https://yarnpkg.com/getting-started/qa#which-files-should-be-gitignored)。

## 参考

- [Yarn PnP](https://yarnpkg.com/features/pnp)
- [Yarn 2 Migration](https://yarnpkg.com/getting-started/migration#switching-to-plugnplay)
- [Yarn 2 Install](https://yarnpkg.com/getting-started/install)



<!-- 现在需要手动调用自定义 pre-hooks（例如 prestart） -->