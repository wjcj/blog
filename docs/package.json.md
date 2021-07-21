package.json配置详解
=====

## 示例

``` json
{
  "name": "redux",
  "version": "4.0.4",
  "description": "Predictable state container for JavaScript apps",
  "license": "MIT",
  "homepage": "http://redux.js.org",
  "repository": "github:reduxjs/redux",
  "bugs": "https://github.com/reduxjs/redux/issues",
  "keywords": [
    "redux",
    "reducer",
    "state",
    "predictable",
    "functional",
    "immutable",
    "hot",
    "live",
    "replay",
    "flux",
    "elm"
  ],
  "authors": [
    "Dan Abramov <dan.abramov@me.com> (https://github.com/gaearon)",
    "Andrew Clark <acdlite@me.com> (https://github.com/acdlite)"
  ],
  "main": "lib/redux.js",
  "unpkg": "dist/redux.js",
  "module": "es/redux.js",
  "types": "types/index.d.ts",
  "files": [
    "dist",
    "lib",
    "es",
    "src",
    "types"
  ],
  "scripts": {
    "clean": "rimraf lib dist es coverage types",
    "format": "prettier --write \"{src,test}/**/*.{js,ts}\" \"**/*.md\"",
    "format:check": "prettier --list-different \"{src,test}/**/*.{js,ts}\" \"**/*.md\"",
    "lint": "eslint --ext js,ts src test",
    "check-types": "tsc --noEmit",
    "test": "jest",
    "test:watch": "npm test -- --watch",
    "test:cov": "npm test -- --coverage",
    "build": "rollup -c",
    "pretest": "npm run build",
    "prepublishOnly": "npm run clean && npm run check-types && npm run format:check && npm run lint && npm test",
    "examples:lint": "eslint --ext js,ts examples",
    "examples:test": "cross-env CI=true babel-node examples/testAll.js"
  },
  "dependencies": {
    "@babel/runtime": "^7.9.2"
  },
  "devDependencies": {
    "@babel/cli": "^7.8.4",
    "@babel/core": "^7.9.0",
    "@babel/node": "^7.8.7",
    // ...
  },
  "npmName": "redux",
  "npmFileMap": [
    {
      "basePath": "/dist/",
      "files": [
        "*.js"
      ]
    }
  ],
  "jest": {
    "testRegex": "(/test/.*\\.spec\\.[tj]s)$",
    "coverageProvider": "v8"
  },
  "sideEffects": false
}
```

## name

模块名称。

命名规则：名称需 <= 214个字符，不能以点或下划线开头、不含大写字母，不含非法的URL字符，不可重名（[https://www.npmjs.com/](https://www.npmjs.com/) 可检查模块名称是否重复）。

当重名时可通过在 `name` 中设置 `scope` 来解决，命名规则为`@scope/project-name`，`scope` 可以理解为模块的命名空间，`scope` 通常是组织或团队的名称，如 `@babel/runtime`，也可以是个人的npm 用户名，如` @fuyi0115/utils`。

通过`npm init --scope=@scopeName` 可快速初始化一个作用域包。<br/>
作用域模块默认发布是私有的，这时如果要发布成公用模块，添加 `access=public` 参数：`npm publish --access=public`。<br/>
安装和使用作用域模块：

``` bash
npm install @scope-name/project-name --save
# 使用
var projectName = require('@scope-name/project-name')
```

## version

版本号。<br/>
`version` 跟 `name` 共同组成已发布包的唯一ID，所以每一次 npm 发布都需要更新版本号，因为 npm 完全基于符合语义化版本（Semantic Versioning）来管理依赖的版本，所以版本号的更新需要符合语义化版本规范，简单总结：<br/>
版本格式：`主版本号.次版本号.修订号`（即`major.minor.patch`），版本号递增规则如下：
1. 主版本号：当你做了不兼容的 API 修改；
2. 次版本号：当你做了向下兼容的功能性新增；
3. 修订号：当你做了向下兼容的问题修正。

先行版本号及版本编译信息可以加到“主版本号.次版本号.修订号”的后面作为延伸。

## description & keywords

项目描述与关键字。有助于人们发现你的包。比如使用 `npm search <keyword>`命令时就会对 `description` 和 `keywords` 的内容进行匹配。

``` json
"description": "Predictable state container for JavaScript apps",
"keywords": [
  "redux",
  "reducer",
  "state",
  "predictable",
  "functional",
  "immutable",
  "hot",
  "live",
  "replay",
  "flux",
  "elm"
],
```

## homepage

项目主页。通常设置为项目文档首页或 Github Pages 地址。

``` json
// 文档首页
"homepage": "http://redux.js.org"
// Github Pages
"homepage": "https://fuyi0115.github.io/inc-cmpts"
```

## bugs

反馈 bug 的 `issues` 页面地址和邮箱。如果提供了 `url`，执行`npm bugs`命令时将在浏览器访问此 URL。

``` bash
"bugs": "https://github.com/reduxjs/redux/issues",
# or
"bugs": {
  "url": "https://github.com/reduxjs/redux/issues",
  "email": "dan.abramov@me.com"
}
```

## license

指定开源许可证（开源软件授权协议）。<br/>
当包需要发布到 npm 注册表时，指定不同的许可证会对使用的个人或组织有不同的使用、分享、修改等权限限制。如果未设置 `License` 则默认受版权保护，所以你如果想让他人放心使用就需要选择一个合适的 License。<br/>
常见 `License` 的核心差异如下图：
![choosealicense](https://miro.medium.com/max/1600/1*HJ3fiEAi09Ny_HNh9I5aZw.png)

可以看出 `MIT` 最为自由，基本没有任何限制，如 `redux`、`antd` 就是使用的 `MIT` 协议。如果你不想提供许可证，或者明确不想授予使用权限，也可以设置为 `UNLICENSED`。
如果你不确定要使用哪个许可证，Github 官方专门提供帮助大家选择 👉[Choose an open source license](https://choosealicense.com/)，中文版 👉 [这里](https://choosealicense.rustwiki.org/)。

## author & authors & contributors
作者与贡献者。

`author`是一个人，`contributors`是一群人。配置简写的方式均为 `Name <email> (url)`，其中 `email` 和 `url`（一般为个人网站或 github 等）都是可选的。

```json
"author": {
  "name": "Dan Abramov",
  "email": "dan.abramov@me.com",
  "url": "https://github.com/gaearon"
},
"contributors": [
  {
    "name": "Dan Abramov",
    "email": "dan.abramov@me.com",
    "url": "https://github.com/gaearon"
  },
  {
    "name": "Andrew Clark",
    "email": "acdlite@me.com",
    "url": "https://github.com/acdlite"
  },
],
// 简写
"author": "Dan Abramov <dan.abramov@me.com> (https://github.com/gaearon)",
"contributors": [
  "Dan Abramov <dan.abramov@me.com> (https://github.com/gaearon)",
  "Andrew Clark <acdlite@me.com> (https://github.com/acdlite)"
],
```
当作者有多个人时，也可以指定 `authors` 字段，写法同 `contributors`。<br/>
当项目贡献者过多时可以在项目根目录创建 `AUTHORS.md` 文件，每行以`Name <email> (url)` 格式表示一个贡献者信息，此文件的信息将作为 `contributors` 的默认值。

## funding

指定一个包含 URL 的对象或字符串，多个则可以采用数组的形式，该 URL 提供有关帮助或资助此项目开发的方法或渠道信息。

用户可以使用`npm fund <projectname>` 命令打开浏览器访问该URL，当有多个时将访问第一个。

``` json
{
  "funding": {
    "type" : "individual",
    "url" : "http://example.com/donate"
  },
  "funding": {
    "type" : "patreon",
    "url" : "https://www.patreon.com/my-account"
  },
  "funding": "http://example.com/donate",
  "funding": [
    {
      "type" : "individual",
      "url" : "http://example.com/donate"
    },
    "http://example.com/donateAlso",
    {
      "type" : "patreon",
      "url" : "https://www.patreon.com/my-account"
    }
  ]
}
```

## files

`npm publish`发布需包含的文件数组。
通常只需要发布打包后的文件，如`"files": ["dist", "lib", "es"]`。如省略`"files"`配置则默认为`["*"]`，即所有文件都将被发布。<br/>
还可以在包的根目录或子目录中添加`.npmignore`文件，以防止某些文件被包含到发布内容中。<br/>

**`"files"`、`.npmignore`、`.gitignore`之间的关系：**
- 包根目录`.npmignore`不会覆盖`package.json#files`，但子目录的`.npmignore`会覆盖`package.json#files`。
- 有`.gitignore`而没有`.npmignore`，则将使用`.gitignore`的内容。
简单总结：优先级 `package.json#files` > `.npmignore` > `.gitignore`，且互相之间不是叠加，而是完全覆盖。

而有一些特殊的文件和目录始终会被包含或排除，无论是否是否存在于`files`数组中。<br/>
总被忽略的某些文件：
```
.git
CVS
.svn
.hg
.lock-wscript
.wafpickle-N
.*.swp
.DS_Store
._*
npm-debug.log
.npmrc
node_modules
config.gypi
*.orig
package-lock.json (如果希望发布则使用 [npm-shrinkwrap.json](https://docs.npmjs.com/cli/v7/configuring-npm/npm-shrinkwrap-json))
```
*`npm-shrinkwrap.json`文件由`npm shrinkwrap`生成。它与`package-lock.json`功能基本相同，但发布包时可被包括在内（需手动添加），详细可查看[npm-shrinkwrap.json vs package-lock.json](https://github.com/fuyi0115/blog/issues/19)。*

始终包含的某些文件：

```
package.json
README
CHANGES / CHANGELOG / HISTORY
LICENSE / LICENCE
NOTICE
package.json中 main 字段指定的文件
```

**通常情况下建议仅通过设置`package.json#files`管理发布到NPM内容，这也是大部分NPM包使用的方式。**

## main

指定程序的主入口文件。`require("moduleName")`会加载这个文件，此文件即使不包括在 `package.json#files` 字段里也会被发布。如这个字段未设置，则默认值是模块根目录下的`index.js`文件。

``` json
"main": "lib/redux.js",
```

## browser

指定浏览器环境入口文件。

如果模块应该在浏览器环境下使用，则应该使用此字段，而不是 `main` 字段。这有助于提示用户此模块中使用了 nodeJs 模块中不存在的对象或 API（比如 `window`）。此字段通常指向一个 `AMD` 或 `UMD` 模块格式的文件。

``` json
"browser": "dist/index.js"
```

## scripts

脚本命令对象，key 是命令别名（即 `command` ），value 是具体的命令。<br/>
使用 `npm run[-script] < command > ` 的语法即可执行`package.json#scripts`中的任意命令，比如 `npm run build` 将执行 `rollup -c` 。<br/>
`npm run` 是 `npm run-script` 的别名，当 `command` 为 `test`、`start`、`restart` 和 `stop` 时，也可以省略 `run` 直接调用，比如`npm test`。更多[npm run-script](https://docs.npmjs.com/cli/v7/commands/npm-run-script) 的用法。

``` json
"scripts": {
  "test": "jest",
  "test:watch": "npm test -- --watch",
  "test:cov": "npm test -- --coverage",
  "build": "rollup -c",
},
```

## bin

用来指定各个内部命令对应的可执行文件的位置。<br/>
以下面代码为例：

``` json
"bin": {
  "umi-test": "./bin/umi-test.js"
}
```
可执行文件 `./bin/umi-test.js` ：
``` bash
#!/usr/bin/env node

console.log('umi-test running ...');
```

`umi-test` 命令对应的可执行文件为 `.bin/umi-test.js`。人们安装此模块后，NPM会找到这个文件，并在根目录 `node_modules/.bin/` 下建立一个符号链接名为 `node_modules/.bin/umi-test`。<br/>
由于`node_modules/.bin/` 目录会在运行时加入系统的 PATH 变量，因此在运行 npm 时就可以不带路径，直接通过命令来调用这些脚本。因此可以有如下的简写：

``` json
"scripts": {  
  "test": "./node_modules/bin/umi-test.js",
  "test:coverage": "./node_modules/bin/umi-test.js --coverage",
  // 简写
  "test": "umi-test",
  "test:coverage": "umi-test --coverage"
}
```
所有 `node_modules/.bin/` 目录下的命令可以在 `package.json#scripts` 定义后通过 `npm run command` 运行。

## config

用于添加命令行的环境变量。

``` json
{
  "name" : "foo",
  "config" : { "port" : "8080" },
  "scripts" : { "start" : "node server.js" }
}
```
在 `server.js` 脚本就可以使用 `config` 字段的值：

``` js
console.log(process.env.npm_package_config_port) // 8080
```

用户可以通过执行 `npm config set foo:port 8001` 改变这个值。

## man

用来指定当前模块的 `man` 文档的位置。

``` json
"man" :[ "./doc/calc.1" ]
```

## dependencies

生产依赖。项目生产环境运行所需要的模块。对象中每个成员由模块名和对应的版本要求组成，表示依赖的模块及其版本范围。

``` json
"dependencies": {
  "lodash": "^4.17.21",
},
```

版本号格式：`主版本号.次版本号.修订号`，前缀 `v` 和 `=` 均可省略。<br/>
对应的版本可以加上各种限定规则，主要有：
- 前缀 `v` 和 `=` 均可省略。
- 比较符（`<`、`<=`、`>`、`>=`、`=`）：例如 `<2.0.0` 即可匹配小于 `2.0.0` 的所有版本。
- 空格：表示 `且` 关系。例如 `>1.0.0 <=2.2.0` 即不匹配 `>1.0.0` 且 `<=2.2.0` 的版本。
- `||`：表示 `或` 关系。例如 `<1.0.0 || >=2.0.0` 即匹配 `<=1.0.0` 或 `>=2.2.0` 的版本。
- `X`、`x`、`*`：例如 `2.x` 或 `2.*` 等价于 `>=2.0.0 <3.0.0`；`3.1.x` 则等价于 `>=3.1.0 <3.2.0`
- `~`：主版本号和次版本号固定，表示仅匹配修订版本。例如 `~1.2.2` 即等价于 `>=1.2.2 <1.3.0`。
- `^`：主版本固定，表示仅匹配可兼容版本。例如 `^3.1.3` 即等价于 `>=3.1.3 <4.0.0`。
  - 如果大版本号为 `0`，则 `^` 的行为与 `~` 相同，例如 `~0.2.1` 即等价于 `>=0.2.1 <0.3.0`。因为主版本号为 0 时表示仍处于开发阶段，即使次要版本号变动也可能带来不兼容。

`devDependencies`、`peerDependencies`、`optionalDependencies` 中与此一致。

使用 `npm install moduleName` 并添加 `--save` 参数（简写 `-S`）即可把该模块写入 `dependencies` 属性，比如 `npm install lodash --save`。

## devDependencies

开发依赖。项目开发环境运行所需要的模块。

使用 `npm install moduleName` 并添加 `--save-dev` 参数（简写 `-D`）即可把该模块写入 `devDependencies` 属性，比如 `npm install webpack --save`。

## peerDependencies

同等依赖。提示宿主环境去安装 `peerDependencies` 中所指定的依赖，以保证当前包的正常运行。
比如基于 `react` 的组件库 `antd@4.16.0`，它就要求宿主环境需要安装指定的 `react` 版本：

``` json
"name": "antd",
"version": "4.16.0",
"peerDependencies": {
  "react": ">=16.9.0",
  "react-dom": ">=16.9.0"
},
```

在 NPM3 版本之前，某模块被 `npm install` 时，它所依赖的 `peerDependencies` 会自动安装。NPM3~6 将忽略 `peerDependencies`，而是通过警告的方式来提示我们，此时就需要手动安装这些依赖。NPM7 `peerDependencies` 又变为了自动安装，由于许多包的 `peerDependencies` 都相对宽松，NPM7 将打印警告并解决包依赖树中存在的冲突，只有存在无法自动解决的依赖冲突才会阻止安装。

## optionalDependencies

可选依赖。顾名思义定义在 `optionalDependencies` 中的依赖都是非必须的，如果它们安装失败（如找不到或无法获取）npm 仍会继续运行，也不会导致失败。但是 `optionalDependencies` 会覆盖 `dependencies` 中的同名依赖包，所以不要把一个包同时写进这两个对象中。`optionalDependencies` 常运用于判断包是否存在而执行不同的执行逻辑。

使用 `npm install moduleName` 并添加 `--save-optional` 参数（简写 `-O`）即可把该模块写入 `optionalDependencies` 属性。

## bundledDependencies

捆绑依赖。是一个包含依赖包名的数组对象，在发布时会将这个对象中的包打包到最终的发布包里，执行`npm pack`时，将数组中的声明打包进目标包中。如果我们使用 `npm publish` 来发布的话这个属性不会生效，所以不常使用。

使用 `npm install moduleName` 并添加 `--save-bundle` 参数（简写 `-B`）即可把该模块写入 `bundledDependencies` 属性。

## private

如果设置`"private": true`，那么 npm 将拒绝发布此包，目的是为了私有库被意外发布。<br/>
如果要确保包只发布到特定的NPM注册表（例如内部注册表），请使用下面[package.json#publishConfig]在发布时重写注册表配置参数。

## publishConfig

NPM发布时使用的配置。<br/>
如果您想覆盖默认的标记（tag）、注册表（registry）或访问权限（access）时会特别方便。

``` json
"publishConfig": {
  "registry": "https://registry.npm.alibaba-inc.com",
  "tag": "beta",
  "access": "public"
}
```
说明：
- `registry`：NPM注册表的URL，默认https://registry.npmjs.org/
- `tag`：添加标记，默认`"latest"`，效果等同于`npm publish --tag latest`。方便NPM基于tag做版本管理，如先发布一个测试版本标记为beta版本，稳定之后设置为最新版本`npm dist-tag add moduleName@beta latest`。
- `access`：访问权限。`"public"`（公共的）`"restricted"`(受限制的)，scoped包如果需要公开可见需要设置为`"public"`，而scoped包的访问权限始终为`"public"`。

查看[config](https://docs.npmjs.com/cli/v7/using-npm/config)可覆盖的配置选项列表。

## repository

代码存放的位置，这对想要贡献代码的人有帮助。

``` json
{
  "repository": "github:reduxjs/redux",
  "repository": {
    "type": "git",
    "url": "https://github.com/npm/cli.git"
  },
  "repository": {
    "type": "git",
    "url": "https://github.com/facebook/react.git",
    "directory": "packages/react-dom"
  }
}
```
说明：
- `type`: 版本控制系统类型，默认为git，也可以是svn。
- `url`: 公开可用的，可以直接传递给VCS（版本控制系统，version control system）使用的url，并不是项目仓库浏览器页面的 url。对于GitHub，GitHub gist，Bitbucket或GitLab存储库，也可以是一些快捷方式语法。
- `directory`: 如果你的包的`package.json`不在根目录中（比如它是 monorepo 项目中的一部分），这里可以指定它所在的目录。

## unpkg

开启 `cdn` 服务。

``` json
"unpkg": "dist/redux.js"
```

npm 上的所有内容都可以使用`cdn` 服务，以为 `unpkg.com/:package@:version/:file`的URL格式便可以从任何包中加载任何文件，比如:

```
unpkg.com/react@16.7.0/umd/react.production.min.js
unpkg.com/react-dom@16.7.0/umd/react-dom.production.min.js
```

如果您省略文件路径时（如`unpkg.com/jquery`），`unpkg` 则将提供由`package.json#unpkg`字段指定的文件，如果没有找到则会回退到`package.json#main`字段指定的文件。


详细参考 [https://unpkg.com](https://unpkg.com/)。

## `module`

指定一个支持 `ES2015` 模块语法的模块入口文件。构建工具在构建项目的时候，如果发现了这个字段则会首先使用这个字段指向的文件，如果未定义则回退到 `package.json#main` 字段指向的文件。

``` json
"module": "es/redux.js"
```

## `types` 或 `typings`

TypeScript主声明文件，可为用户提供更好的 IDE 支持，比如代码补全、接口提示等。<br/>
TypeScript项目需要将声明文件发布到NPM，主要有两种方式，一种是以单独的npm包发布到[@types organization](https://www.npmjs.com/~types)，一种是与你的npm包捆绑在一起（推荐方式）。当使用第二种方式时则需要设置`package.json#types` 或 `package.json#typings`字段为主声明文件。<br/>
同样要注意的是如果主声明文件名是index.d.ts并且位置在包的根目录里（与index.js并列），你就不需要使用"types"属性指定了

``` json
"types": "types/index.d.ts",
```


## engines

指明了模块运行的平台环境，比如指定 Node、NPM 版本。

``` json
"engines": {
  "node": ">=12",
  "npm": ">=5"
},
```

## os & cpu

指定模块运行的操作系统 和 cpu。对于不允许操作系统，可以在前面加上 `!`。

``` json
{
  "os": [
    "darwin",
    "linux",
    "!win32"
  ],
  "cpu": [
    "x64",
    "ia32",
    "!arm",
    "!mips"
  ]
}
```

## workspaces

工作区。一个术语，它支持从一个顶级根包中管理本地的多个包，NPM7 开始支持。<br/>
当我们在本地开发 npm 包且还依赖的其它本地 npm 包时，不再需要手动的去执行 `npm link` 命令去建立包，而在 `npm install` 时就会自动把 `workspaces` 下面的合法包自动创建符号链接到根目录的`node_modules` 里，工作区内 npm 包之间通过 `package.json#name` 即可互相引用，比如（`const moduleA = require('package-a')`）。

多 `packages` 项目结构示例：

``` json
{
  "name": "my-workspaces-powered-project",
  "workspaces": [
    "packages/**"
  ]
}
```
项目结构：

```
.
+-- node_modules
| `-- workspace-a -> ../packages/workspace-a
| `-- workspace-b -> ../packages/workspace-b
+-- package-lock.json
+-- package.json
`-- packages
  `-- package-a
    `-- package.json
  `-- package-b
    `-- package.json
```
详细了解[package.json#workspaces](https://docs.npmjs.com/cli/v7/using-npm/workspaces)

## sideEffects

标记代码是否有副作用，以便告知 `webpack` 可以对无副作用的模块尽情的进行 `tree shaking`。<br/>

默认值为 `true`，即都有副作用；`false` 表示包中所有文件都没有副作用；如果部分代码中确实有一些副作用，可以提供一个数组值匹配出这些文件：
``` json
"sideEffects": [
  "dist/*",
  "es/**/style/*",
  "lib/**/style/*",
  "*.less"
],
```
此数组支持简单的 `glob` 模式匹配相关文件，其内部使用了 [glob-to-regexp](https://github.com/fitzgen/glob-to-regexp)（支持：`*`，`**`，`{a,b}`，`[a-z]`）。如果匹配模式为 `*.css`，且不包含 `/`，将被视为 `**/*.css`。

此选项需要与 webpack4 新增特性 [optimization.sideEffects](https://webpack.docschina.org/guides/tree-shaking/#clarifying-tree-shaking-and-sideeffects) 配合使用。
**"副作用（side effect）"的定义：**在导入时不仅仅单纯的暴露一个 `export` 或多个 `export`，还会执行对外部变量产生影响行为的代码。例如 `polyfill`，它影响全局作用域。

对于NPM包而言，只要模块在设计上是**期望**不产生副作用的，即使你不能保证发布到 npm 的包是否含有副作用代码（可能是自己的原因，或是 [babel 之类的 transformer 编译后产生](https://zhuanlan.zhihu.com/p/32831172)），只要你是模块可以确保不是期望成为一个 `polyfill` 去影响外部变量（如修改全局对象、重写原生对象方法等），或者是需要注入的样式文件（`webpack` 会认为所有 `import 'xxx'` 语句是仅引入而未使用, 如果错误的将其声明成了“无副作用”, 它们就会被 `tree-shaking` 掉），你就可以将它们标记为无副作用的模块。

详细了解 [webpack tree shaking 和 sideEffects](https://github.com/fuyi0115/blog/issues/20)。

## 参考

- [npm Docs](https://docs.npmjs.com/cli/v7/configuring-npm/package-json)
- [Webpack 中的 sideEffects 到底该怎么用？](https://zhuanlan.zhihu.com/p/40052192)
- [语义化版本 2.0.0](https://semver.org/lang/zh-CN/)
