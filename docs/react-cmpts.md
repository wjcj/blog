
本文章主要介绍 react 组件库从项目构建到 npm 发布、文档发布的完整过程，并详细介绍过程中需要了解的细节。

使用的工具：

- [father](https://github.com/umijs/father)：它支持基于 `rollup` 和 `babel` 支持多种打包格式，并且附带文档构建功能。
- [gh-pages](https://github.com/tschaub/gh-pages)：使用它将构建好的组件文档发布到 `github pages` 上。

## 创建项目

创建一个名为的 `inc-cmpts-demo` 的项目。

### `package.json`

使用 `npm init` 命令初始化一个 `package.json` 文件，完善一些字段后，`npm install` 安装依赖：

``` json
{
  "name": "inc-cmpts-demo",
  "version": "1.0.0",
  "description": "React components",
  "main": "index.js",
  "scripts": {
    "build": "father build",
    "doc:dev": "father doc dev",
    "doc:build": "father doc build",
    "test": "father test"
  },
  "author": {
    "name": "fuyi0115",
    "email": "fuyi0115@xxx.com"
  },
  "license": "MIT",
  "dependencies": {
    "classnames": "^2.2.6"
  },
  "devDependencies": {
    "father": "^2.30.6",
    "gh-pages": "^3.0.0",
  },
  "peerDependencies": {
    "react": "^16.8.5"
  }
}
```

`scripts` 字段中是使用 `father` 完成 `组件打包`、`文档预览`、`文档构建`、`代码测试` 功能的脚本命令。如果对 `package.json` 中某些字段的含义和配置不了解可查看[package.json 配置详解](https://github.com/fuyi0115/blog/issues/21)。还有一些与npm 包发布和文档发布的相关字段之后再添加...

### 添加组件源码

```
├── src
│   ├── index.js
│   ├── button.js
│   ├── button.mdx
│   └── button.less
├── package.json
```

`./src/button.js`：
``` jsx
import React from 'react';
import classnames from 'classnames';
import './button.less';

const Button = (props) => {
  return (
    <button
      className={
        classnames('button', { 'large': props.size === 'large' })
      }
      style={{ color: props.color }}
    >
      {props.children}
    </button>
  );
};

export default Button;
```

`./src/index.js`：

``` jsx
export { default as Button } from './button';
```

### 添加组件文档

`father` 支持 `.md` 和 `.mdx` 文件，`.mdx` 即在里面写 `jsx` 的 Markdown 文件格式，更多使用方式可以查看 [docz 文档](https://www.docz.site/)。

`./src/button.mdx`：

``` md
---
title: Button
route: /
---

import { Button } from './index';

# Button

## Normal Button
<Button>Normal Button</Button>

## Color
<Button color="red">Red Button</Button>

## Size
<Button size="large">Large Button</Button>
```

### 开启本地服务

执行 `npm run doc:dev` 可查看和调试组件和文档效果。

![开启本地服务](https://i.ibb.co/zhMv2m9/Wechat-IMG1363.png)


## 组件打包

组件打包之前我们需要先了解各种模块格式之间的差异：

各种模块格式间的一些差异：
![模块格式差异](https://miro.medium.com/max/1400/1*ohOcheaTdGZpG4nnK97swg.png)

简单总结对比：
- `AMD`: 用于浏览器环境，且支持异步加载模块，不支持 `tree shaking`（移除 JavaScript 上下文中的未引用代码）。

- `CommonJS`: 同步加载模块，用于 nodeJs 中，模块支持动态导入（`Dynamic`），但伴随的问题是对`tree shaking` 不友好。

- `UMD(Universal Module Definition)`: 通用模块定义。通用的含义就是兼容了 `AMD` 和 `CommonJS`，如果环境都不满足还支持将模块公开到全局（ `window` 或 `global`），目的是提供一个前后端跨平台的解决方案。

- `ES Modules`：支持异步加载模块，支持 nodeJs 14.* 和浏览器环境（还不通用，仅限高级浏览器），因为是静态模块虽然不支持动态导入，但完全支持 Tree shaking。

所以作为一个组件包，我们通常需要支持 `esm`、`cjs`、`umd` 三种格式的打包：

- `umd`：满足浏览器环境通过 `<script>` 标签直接引入，且发布到 npm 上后支持 `cdn` 服务，同时也可以 nodeJs 环境的备用模块。

- `cjs`（即 `CommonJS`）：用于 nodeJs 场景使用，比如用于 `ssr`、代码测试、一些工具（webpack、rollup）中。

- `esm`（即 `ES Modules`）：由于完全支持`tree shaking`，作为 npm 包在 `webpack`、`rollup` 打包后可有效减少打包文件的体积。

### 打包配置

项目根目录添加 `.fatherrc.js` 文件并添加 `father` 打包配置：

``` js
export default {
  esm: 'rollup',
  cjs: 'rollup',
  umd: {
    name: 'myBundle',
    minFile: true,
    globals: {
      react: 'React'
    }
  }
}
```

`father` 打包为 `cjs` 和 `esm` 格式时都支持 `rollup` 和 `babel` 两种打包方式，`rollup` 是跟进 `entry` 把项目依赖打包在一起输出成一个文件并存放在 `dist` 目录；`babel` 是文件到文件的编译，把 `source` 目录（`src`）编译到另一个目录下，`cjs` => `/lib` 目录，`esm` => `/es`目录，只会做语法层的转换。

`package.json` 增加如下配置：

``` json
{
  "main": "dist/index.js",
  "module": "dist/index.esm.js",
  "unpkg": "dist/index.umd.min.js",
  "files": ["dist"],
  "sideEffects": false
}
```

`npm run build` 打包后将在 `/dist` 目录生成打包文件。
```
├── dist
│   ├── index.js
│   ├── index.esm.js
│   ├── index.umd.js
│   └── index.umd.min.js
├── package.json
├── .fatherrc.js
├── .gitignore
```

- `main`: 指定一个支持`CommonJS` 的打包文件作为模块主入口。`require("moduleName")` 会加载这个文件。
- `module`： 指定一个支持 `ES Modules` 的模块入口文件。构建工具在构建项目的时候，如果发现了这个字段则会首先使用这个字段指向的文件。

- `unpkg`：指定一个支持 `umd` 格式打包文件，同时支持 `cdn` 服务，在 `<script>` 标签使用 `unpkg.com/packageName` 路径技能能访问此文件。

- `files`：指定需要发布到 npm 的文件。
  - 通常只发布打包后的文件，如果为了方便他人做 IDE 调试也可以带上源代码（`src`）；
  - 通常情况下不推荐使用 `.npmignore`，使用更高优先级的 `package#files` 字段指定需要发布的内容更加直观准确。

- `sideEffects`：`false` 标识代码是无副作用，会让 webpack 的 [tree-shaking](https://webpack.js.org/guides/tree-shaking/)（移除未使用的代码）更加高效。

**关于 `sideEffects`：**

副作用代码通常指执行时会影响外部变量的代码，比如修改全局对象、重写原生对象方法等。如果你的包不是一个 `polyfill` 功能的库，至少你的包/某些模块设计上期望它是无副作用的，不管打包后是否会产生副作用，你就可以设置为 `false`。

你需要将样式文件标识为副作用文件，否则`webpack` 会认为所有 `import 'xxx'` 语句是仅引入而未使用, 它们就会被 `tree-shaking` 掉；如果包中的确还有一些模块是期望产生副作用的......这时你可以将 `sideEffects` 的值设置为数组，标识出这些期望有副作用的模块，以 `antd` 的配置为例：
``` json
"sideEffects": [
  "dist/*",
  "es/**/style/*",
  "lib/**/style/*",
  "*.less"
],
```

### 其他

添加 `.gitignore`:

```
# dependencies
/node_modules
/npm-debug.log*
/package-lock.json

# production
/dist

#father
/.doc
/.docz
```

## 发布文档

github 上的 `Gihub Pages` 功能允许你在 Web 上实时发布网站代码，原理是通过项目仓库中一个名为 `gh-pages` 的特殊分支来管理网站代码。所以你需要做的就是在 `github` 上创建该分支，然后将网站代码发布到该分支，然后你就可以用 `username.github.io/my-repository-name` 的 URL 访问你的网站页面，网站首页对应的 `index.html` 文件。

1. 创建项目仓库后，点击 `Branch：master` 按钮，在文本输入框中输入 `gh-pages` 即创建分支 `Create branch: gh-pages`。
![创建 gh-pages 分支](https://mdn.mozillademos.org/files/12145/repo-site.png)
2. 项目仓库页面点击 `Settings` 按钮选择 `Pages` 标签对`Gihub Pages` 做源文件、主题、自定义域名等设置。

### 使用 `gh-pages`

`npm install gh-pages --save-dev`。

使用 `gh-pages` 可省略上面的步骤，在 `package.json` 中增加 `homepage` 字段指向 `Gihub Pages` URL ，然后添加在 `scripts` 中增加 `gh-pages -d .doc` 脚本（father 构建的文档打包在 `.doc` 目录），执行 `npm run doc:deploy` 即可自动创建 `gh-pages` 分支并发布文档。

``` json
{
  "scripts": {
    // ...
    "doc:build": "father doc build",
    "doc:deploy": "gh-pages -d .doc",
    "deploy": "npm run doc:build && npm run doc:deploy"
  },
  "homepage": "username.github.io/my-repository-name",
  "devDependencies": {
    // ...
    "gh-pages": "^3.0.0",
  },
}
```

**小贴士：**

- 打包后生成文档代码目录及文件通常不需要随项目源代码提交到仓库，需要在 `.gitignore` 忽略，`gh-pages -d dist` 不受 `.gitignore` 影响。
- 下一次文档更新发布只需要执行 `npm run deploy` 重新生成文档代码并发布即可。<br/>
- 执行 `gh-pages -d .doc` 时你可能会遇到这个失败消息： `fatal: A branch named 'gh-pages' already exists.`，这就需要你通过 `rm -rf node_modules/.cache/gh-pages` 删除 `gh-pages` 缓存目录，然后再部署。

## 参考
- [Packaging a UI Library for Distribution](https://blog.bitsrc.io/packaging-a-ui-library-for-distribution-d153219def28)
- [father](https://github.com/umijs/father)