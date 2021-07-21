tsconfig.json 详解
====

## 示例

`tsc --init` 创建 `tsconfig.json`，对所有选项添加注释说明👇：

```
{
  "compilerOptions": {
    /* 基础选项 */
    "incremental": true,                          // 启用增量编译。相对于全量编译，增量编译会加快编译速度，并在编译之后会创建一系列 .tsbuildinfo 文件储存增量编译信息。
    "target": "es5",                              // 指定ECMAScript目标版本: 'ES3' (default), 'ES5', 'ES2015', 'ES2016', 'ES2017', 'ES2018', 'ES2019', 'ES2020', or 'ESNEXT'（最新的es语法）
    "module": "commonjs",                         // 指定模块系统: 'none', 'commonjs', 'amd', 'system', 'umd', 'es2015', 'es2020', or 'ESNext'（最新的es模块系统）
    "lib": [],                                    // 指定要包含在编译中的库文件（声明文件）。详见👇
    "allowJs": true,                              // 允许编译 javascript 文件（js/jsx）。
    "checkJs": true,                              // 报告 javascript 文件中的错误。通常与 allowJS 一起使用。
    "jsx": "preserve",                            // 指定 jsx 代码的生成: 'preserve', 'react-native', 'react', 'react-jsx' or 'react-jsxdev'。详见👇
    "jsxFactory": "h",                            // 指定编译 JSX 元素要使用的工厂函数，默认为 'React.createElement'。
    "declaration": true,                          // 生成声明文件。为每个 ts 或 js 文件生成相应的 '.d.ts' 文件，详见👇
    "declarationMap": true,                       // 为每个相应的 '.d.ts' 文件生成 sourcemap。
    "sourceMap": true,                            // 生成相应的 '.map' 文件。
    "outFile": "./",                              // 所有全局（非模块）文件将被合并到指定的单个输出文件中。仅限 module 是 None，System 或 AMD 时，此选项不能用来打包 CommonJS 或 ES6 模块。
    "outDir": "./",                               // 指定输出目录。指定后 .js （以及 .d.ts, .js.map 等）将会被生成到这个目录下。如果没有指定，.js 将被生成至于生成它们的 .ts 文件相同的目录中。outDir 与 rootDir 共同计算出的根目录。
    "rootDir": "./",                              // 指定输入文件的根目录。与 outDir 控制输出目录结构，详见👇
    "composite": true,                            // 开启后意味着工程可以被引用。详见👇
    "tsBuildInfoFile": "./",                      // 控制 .tsbuildinfo 文件（增量编译信息）被编译到哪个文件夹
    "removeComments": true,                       // 移除注释
    "noEmit": true,                               // 不生成输出文件。例如你想用 Babel 来编译文件，TypeScript 仅作为提供编辑器集成的工具，或用来对源码进行类型检查
    "downlevelIteration": true,                   // 开启迭代器降级。转换为旧版本的 JavaScript（ES5 或 ES3）时更准确的模拟 ES6 迭代器的行为。详见👇
    "importHelpers": true,                        // 是否从 tslib 导入辅助工具函数。对于某些降级行为（转换到旧版本的 JavaScript），例如展开数组或对象（[...arr]），TypeScript 使用一些辅助代码来进行操作。详见👇
    "isolatedModules": true,                      // 孤立模块。如果你使用其他转译器（如 Babel）编译代码，由于它们一次只能在一个文件上操作，这意味着它们不能进行在 完全理解类型系统 的基础上实现代码转译。故存在一些 TypeScript 特性（例如 const enum 和 namespace）不能被正确处理，开启此项后，当你写到这些特性的代码时，TypeScript 会提示警告。

    /* 严格类型检查选项 */
    "strict": true,                               // 启用所有严格类型检查选项。你可以根据需要关闭某些严格模式检查👇：
    "noImplicitAny": true,                        // 在表达式和声明上有隐含的any类型时报错
    "strictNullChecks": true,                     // 启用严格的空检查。开启后不再忽略 null 和 undefined 类型检查，且 null 和 undefined 属于各自不同的类型。
    "strictFunctionTypes": true,                  // 启用对函数类型的严格检查
    "strictBindCallApply": true,                  // 对函数启用严格的 bind、call和 apply方法。启用后会检查调用的入参是否符合原函数的参数调用
    "strictPropertyInitialization": true,         // 在 class 中启用属性初始化的严格检查，即所有的属性值都需要赋有初始值
    "noImplicitThis": true,                       // 对隐含 any 类型的 this 表达式将生成错误。详见👇
    "alwaysStrict": true,                         // 以ECMAScript strict模式下解析文件，并在每个文件里加入 'use strict'

    /* 附加检查 */
    "noUnusedLocals": true,                       // 存在未使用的变量时，报告错误
    "noUnusedParameters": true,                   // 存在未使用的参数时，报告错误
    "noImplicitReturns": true,                    // 当函数中并非所有代码路径都返回值时，报告错误。
    "noFallthroughCasesInSwitch": true,           // 报错 switch 语句中的 fallthrough 错误（存在某个`case` 中不包含 break 或 return）
    "noUncheckedIndexedAccess": true,             // 在索引签名的类型结果中包含 undefined。详见👇
    "noPropertyAccessFromIndexSignature": true,   // 开启后通过`.`访问索引签名(obj.key)将报错，需使用索引访问(obj["key"]) 。

    /* 模块解析选项 */
    "moduleResolution": "node",                   // 指定模块解析策略：'node' （Node.js） 或 'classic' （在 TypeScript 1.6 版本之前使用）
    "baseUrl": "./",                              // 设置解析非绝对路径模块名时的基准目录。通常用于你不想使用含有 '../' 或 './' 的路径来导入文件，或不希望在移动文件时需要更改路径。详见👇
    "paths": {},                                  // 将模块导入重新映射到相对于 baseUrl 路径的配置。通常用于自定义前缀别名来寻找模块，避免代码中出现过长的相对路径。如 { @: './src' }
    "rootDirs": [],                               // 根目录。告诉编译器有许多“虚拟”的目录都作为一个根目录，效果相当于将这些目录合并到同一目录中，这将会允许编译器在这些“虚拟”目录中解析相对应的模块导入。
    "typeRoots": [],                              // 包含类型定义的文件夹列表。这里配置的路径都是相对于 tsconfig.json。默认情况下 'node_modules/@types' 中的包都是类型定义，配置此项后将只包含文件目录。例如 "typeRoots": ["./typings", "./vendor/types"]，此时将不包括 node_modules/@types 下的包。
    "types": [],                                  // 列出全局范围内可见的 '@types' 包。默认情况下 node_modules/@types 中的任何包都被认为是可见的。此项被指定后，只有列出的包才会被包含在全局范围内，例如 "types": ["node", "jest", "express"]。
    "allowSyntheticDefaultImports": true,         // 允许合成默认导入。允许从没有设置默认导出的模块中默认导入（即import React from "react"; 而不需要 import * as React from "react"）。详见👇
    "esModuleInterop": true,                      // TypeScript 处理 CommonJS/AMD/UMD 导入 es6 模块时会出现问题，开启此项用于修复缺陷，使得导入的 CommonJS/AMD/UMD 模块可以符合 es6 模块规范。详见👇
    "preserveSymlinks": true,                     // 不要解析符号链接（软连接）的真实路径。启用后，对于模块和包的引用（例如 import 和 /// <reference type="..." /> 指令都相对于符号链接所在的位置进行解析，而不是相对于符号链接解析后的路径。此选项表现出与 Webpack 中 resolve.symlinks 选项相反的行为。
    "allowUmdGlobalAccess": true,                 // 允许在模块中访问 UMD 全局变量。当未设置这个选项时，使用 UMD 模块的导出需要首先导入声明。

    /* Source Maps 选项 */
    "sourceRoot": "",                             // 指定TypeScript源文件的路径。比如 "sourceRoot": "https://my-website.com/debug/source/" 表示 index.js 的源文件在 https://my-website.com/debug/source/index.ts。
    "mapRoot": "",                                // 指定sourcemap文件的路径。比如 "mapRoot": "https://my-website.com/debug/sourcemaps/" 表示 index.js 的 sourcemap 文件在 https://my-website.com/debug/sourcemaps/index.js.map。
    "inlineSourceMap": true,                      // 不生成单独的 .map 文件，而将 SourceMap 内容以内联嵌入到 .js 文件中（底部注释中），会导致 JS 文件变大。与 sourceMap 选项（生成 .map 文件）互斥。
    "inlineSources": true,                        // 将 .ts 文件的原始内容也包含在 SourceMap 内容中。需要开启 sourceMap 或 inlineSourceMap，通常与 inlineSourceMap 一起开启的情况下很有用（此时 .js 文件的底部注释中将包含 SourceMap 和 ts源码内容）。

    /* 实验选项 */
    "experimentalDecorators": true,               // 启用装饰器（ES7）
    "emitDecoratorMetadata": true,                // 为装饰器提供元数据的支持，即 reflect-metadata。

    /* 高级选项 */
    "skipLibCheck": true,                         // 忽略所有的声明文件（ *.d.ts）的类型检查。这可以节省编译时间，但牺牲了类型系统的准确性，TypeScript 将不会对所有文件进行全面检查，仅对您在应用程序源代码中专门引用的代码进行类型检查。有一种常见情况是 node_modules 中存在同一个库的两个不同版本时可以使用 skipLibCheck 忽略掉类型检查错误，但实际上你应该考虑使用类似 yarn's resolutions 的功能来确保依赖树中只有该依赖项的一个版本。
    "forceConsistentCasingInFileNames": true      // 禁止对同一个文件大小写不一致的引用
  },
  /* 顶层属性 */
  "extends": "",                                  // 继承配置，比如 "extends": "./tsconfig.base.json"。详见👇
  "include": [],                                  // 指定需要编译的文件或模式数组，例如 "include": ["src/**/*", "tests/**/*"]。详见👇
  "exclude": [],                                  // include 配置中需排除的文件或模式数组。由于代码中的 import 语句、'types' 中包含、/// <reference 指令或在 'files' 列表中指定，由 exclude 指定的文件仍然可以成为代码库的一部分。不能通过它阻止文件包含在代码库中的机制，它只是改变了 'include' 设置可发现的内容，也不排除 'files' 选项中的文件。
  "files": [],                                    // 指定需要编译的单个文件列表。当您只有少量文件并且不需要使用 glob 模式来指定许多文件时，这很有用。否则请使用 'include'。
  "references": {},                               // 指定依赖的引用工程，比如 [{ "path": "./common" }]。详见composite👇
  "compileOnSave": {},                            // 让IDE在保存文件的时候根据 tsconfig.json 重新生成文件。例如 "compilerOptions": { "noImplicitAny" : true }，支持这个特性需要 Visual Studio 2015、TypeScript1.8.4以上且安装 atom-typescript 插件。
}
```

## 顶级属性

### extends
继承配置。

比如：
```
.
├── tests
│   ├── index.ts
│   └── tsconfig.json
├── package.json
├── tsconfig.base.json
└── yarn.lock
```
`./tsconfig.json` 文件内容：
``` json
  "extends": "../../tsconfig.base.json",
  // ...
```

继承规则：
- 基础文件（`tsconfig.json`）中的配置首先加载，然后被继承配置文件（`tsconfig.base.json`）中的配置覆盖。
- 继承配置文件（`tsconfig.base.json`）中的 `files`、`include` 和 `exclude` 覆盖了基础配置文件（`tsconfig.json`）中的 `files`、`include` 和 `exclude`。
- 不允许配置文件之间的循环引用。
- `references` 属性不可被继承。

### include

指定需要编译的文件或模式数组。这些文件名是相对于包含 `tsconfig.json` 文件的目录解析的。

``` json
{
  "include": ["src/**/*", "tests/**/*"]
}
```
将包括：
```
.
├── scripts                ⨯
│   ├── lint.ts            ⨯
│   ├── update_deps.ts     ⨯
│   └── utils.ts           ⨯
├── src                    ✓
│   ├── client             ✓
│   │    ├── index.ts      ✓
│   │    └── utils.ts      ✓
│   ├── server             ✓
│   │    └── index.ts      ✓
├── tests                  ✓
│   ├── app.test.ts        ✓
│   ├── utils.ts           ✓
│   └── tests.d.ts         ✓
├── package.json
├── tsconfig.json
└── yarn.lock
```

`include` 和 [exclude](#exclude) 都支持 glob 模式：

- `*` 匹配零个或多个字符（不包括目录分隔符）
- `?` 匹配任何一个字符（不包括目录分隔符）
- `**/` 匹配嵌套到任何级别的任何目录

如果 glob 模式不包含文件扩展名，则只包含支持扩展名的文件（例如默认情况下为 `.ts`, `.tsx` 和 `.d.ts`, 如果 `allowJs` 设置为 `true`，还将包含 `.js` 和 `.jsx`）。

## 编译选项（compilerOptions）。

### lib

TypeScript 包括一组默认的内建 JS 接口（例如 `Math`）的类型定义，以及在浏览器环境中存在的对象的类型定义（例如 `document`）。 TypeScript 还包括与你指定的 `target` 选项相匹配的较新的 JS 特性的 API。例如如果`target` 为 `ES6` 或更新的环境，那么 `Map` 的类型定义是可用的。

- `ES5`：ES3 和 ES5 的核心功能定义
- `ES2015`：ES2015 中额外提供的 API (又被称为 ES6) —— array.find， Promise，Proxy，Symbol，Map，Set，Reflect 等。
- `ES6`：ES2015 的别名。
- `ES2016`：ES2016 中额外提供的 API —— array.include 等。
- `ES7`：ES2016 的别名。
- `ES2017`：ES2017 中额外提供的 API —— Object.entries，Object.values，Atomics，SharedArrayBuffer，date.formatToParts，typed arrays 等。
- `ES2018`：ES2018 中额外提供的 API —— async iterables，promise.finally，Intl.PluralRules，rexexp.groups 等。
- `ES2019`：ES2019 中额外提供的 API —— array.flat，array.flatMap，Object.fromEntries，string.trimStart，string.trimEnd 等。
- `ES2020`：ES2020 中额外提供的 API —— string.matchAll 等。
- `ESNext`：ESNext 中额外提供的 API —— 随着 JavaScript 的发展，这些会发生变化。
- `DOM`：[DOM](https://developer.mozilla.org/docs/Glossary/DOM) 定义 —— window，document 等。
- `WebWorker`：[WebWorker](https://developer.mozilla.org/docs/Web/API/Web_Workers_API/Using_web_workers) 上下文中存在的 API。
- `ScriptHost`：[Windows Script Hosting System](https://wikipedia.org/wiki/Windows_Script_Host) 的 API。

### jsx

控制 JSX 文件输出方式。这只影响 .tsx 文件的 JS 文件输出。

- `react`: 将 `JSX` 改为 `React.createElement` 调用并生成 `.js` 文件。
- `react-jsx`: 改为 `__jsx` 调用并生成 `.js` 文件。
- `react-jsxdev`: 改为 `__jsx` 调用并生成 `.js` 文件。
- `preserve`: 不对 `JSX` 进行改变并生成 `.jsx` 文件。
- `react-native`: 不对 `JSX` 进行改变并生成 `.js` 文件。

| 模式 | 输入 | 输出 | 输出文件扩展名 |
| --- | --- | --- | --- |
| `preserve`      | `<div />` | `<div />` | `.jsx` |
| `react`         | `<div />` | `React.createElement("div")` | `.js` |
| `react-native`  | `<div />` | `<div />` | `.js` |
| `react-jsx`     | `<div />` | `_jsx("div", {}, void 0);` | `.js` |
| `react-jsxdev`  | `<div />` | `_jsxDEV("div", {}, void 0, false, {...}, this);` | `.js` |
*通过 "jsxFactory" 选项可指定编译 JSX 元素要使用的工厂函数，默认为 'React.createElement'。*

### declaration
为你工程中的每个 `TypeScript` 或 `JavaScript` 文件生成 `.d.ts` 文件。 这些 `.d.ts` 文件是描述模块外部 API 的类型定义文件。

当 `declaration` 设置为 `true` 时，用编译器执行下面的 `TypeScript` 代码：
``` js
export let helloWorld = "hi";
```

对应的 `helloWorld.d.ts`:
``` js
export declare let helloWorld: string;
```

### composite

[引用的工程](https://www.tslang.cn/docs/handbook/project-references.html)的`tsconfig.json` 中必须启用 `composite` 设置。 这个选项用于帮助 TypeScript 快速确定引用工程的输出文件位置。

> 工程引用是 TypeScript 3.0 的新特性, 它支持将 TypeScript 程序的结构分割成更小的组成部分. 这样可以改善构建时间 (打开 composite 会自动开启增量编译), 强制在逻辑上对组件进行分离, 更好地组织你的代码。

引用工程简单来说类似于一个大项目被分成多个小项目（引用工程），使得这些项目独立配置和构建。因此每个引用工程拥有独立的 `tsconfig.json` 文件来管理输出目录和其他选项，每个 `tsconfig.json` 中增加了一个新的顶层属性 `references`，它是一个对象的数组，指明该项目要使用到的其他项目（引用的工程），以下面功能为例：

我们期望将 `src` 下面的文件直接被编译到 dist 目录，但 `__test__` 除外，且 `client`、`server` 目录可以独立构建。

```
.
├── src
│   ├── client
│   │   ├── index.ts
│   │   ├── tsconfig.json // ① client工程，使用到的引用工程：②
│   ├── common
│   │   ├── index.ts
│   │   ├── tsconfig.json // ② common工程，使用到的引用工程：无
│   ├── server
│   │   ├── index.ts
│   │   ├── tsconfig.json // ③ server工程，使用到的引用工程：①
├── __test__
│   ├── client.spec.ts
│   ├── server.spec.ts
│   ├── tsconfig.json // ④ test工程，使用到的引用工程：①③
├── package.json
├── tsconfig.json // ⑤ 作为通用配置使用
```

```
// tsconfig.json⑤（作为通用配置）
{
  "compilerOptions": {
    "target": "es5",
    "module": "commonjs",
    "strict": true,
    // "outDir": "./dist"  关闭 outDir 选项, 即不在根配置中指定输入目录
    "composite": true, // 开启则意味着工程可以被引用, 并支持增量编译
    "declaration": true // 开启 composite 选项必须开启 declaration
  }
}
//  引用工程①（/src/client）tsconfig.json
{
  "extends": "../../tsconfig.json",
  "compilerOptions": {
    "outDir": "../../dist/client" // 指定独立的输入目录
  },
  "references": [{ "path": "../common" }] // 因为 client 引用了 common工程，引用的path属性指向到包含`tsconfig.json`文件所在目录，或直接指向到 `tsconfig.json`
}
// 引用工程②（/src/common）的tsconfig.json
{
  "extends": "../../tsconfig.json",
  "compilerOptions": {
    "outDir": "../../dist/common"
  },
  // "references": [] // 因为 common 没有引用其他工程
}
// 引用工程③（/src/server）tsconfig.json
{
  "extends": "../../tsconfig.json", // 首先导入 ⑤
  "compilerOptions": {
    "outDir": "../../dist/server" // 指定输入目录
  },
  "references": [{ "path": "../common" }] // server 引用了 common
}
// 引用工程④（/src/__test__）tsconfig.json
{
  "extends": "../tsconfig.json",
  "references": [{ "path": "../src/client" }, { "path": "../src/server" }]
}
```
完成以上配置后，已经实现了输出目录调整，而独立构建则可以使用 TypeScript 构建模式 `--build` （增量构建），启用`--build` 后会自动地构建依赖项：
``` bash
# 单独构建 server 工程, -b 是 --build 的简写
tsc -b src/server --verbose
```
`tsc -b`还支持其它一些选项：
- `--verbose`：打印详细的日志（可以与其它标记一起使用）
- `--dry`: 显示将要执行的操作但是并不真正进行这些操作
- `--clean`: 删除指定工程的输出（可以与 `--dry `一起使用）
- `--force`: 把所有工程当作非最新版本对待
- `--watch`: 观察模式（可以与 `--verbose` 一起使用）

若启用 `composite` 标记则会发生如下变动：
- 对于`rootDir`设置，如果没有被显式指定，默认为包含 `tsconfig` 文件的目录
- 所有的实现文件必须匹配到某个 `include` 模式或在 `files` 数组里列出。如果违反了这个限制 `tsc` 会提示你哪些文件未指定。
- 必须开启 `declaration` 选项。

总结工程引用的优点：解决了输出目录的结构问题；解决了单个工程的构建问题；通过增量编译提高了编译效率。

### downlevelIteration
开启迭代器降级。当转换为旧版本的 JavaScript（ES5 或 ES3）时可以更准确的模拟 ES6 迭代器的行为。

> 当 `downlevelIteration` 启用时，TypeScript 将会使用辅助函数来检查 `Symbol.iterator` 的实现（无论是原生实现还是 `polyfill`），如果没有实现，则将会回退到基于索引的迭代。

ECMAScript 6 增加了几个新的迭代器语法：`for / of` 循环（`for (el of arr)`），数组展开（`[a, ...b]`），参数展开（`fn(...args)`）和 `Symbol.iterator`。传统实现在不通常的场景下是符合期望的，但并不是 100% 符合 ECMAScript 迭代器协议。

以 `for / of` 循环为例，未开启 `downlevelIteration` 时，`for / of` 将被降级为传统的 `for` 循环，些字符串，例如 emoji （😜），其 `.length` 为 2（甚至更多），但在 `for-of` 循环中应该只有一次迭代。再以数组展开（`[a, ...b]`）为例，数组中存在空元素时传统降级（使用 `concat` ）带来的结果差异。

``` js
// 构建一个元素 `1` 不存在的数组
let missing = [0, , 1];
let spreaded = [...missing];
let concated = [].concat(missing);
// true
"1" in spreaded;
// false
"1" in concated;
```
*更多代码[示例](https://www.staging-typescript.org/zh/tsconfig#downlevelIteration)。*

你也可以通过 `importHelpers` 来使用 `tslib` 以减少被内联的辅助函数的数量，详见👇。

### importHelpers
对于某些降级行为（转换到旧版本的 JavaScript），TypeScript 使用一些辅助代码来实现。例如继承类、展开数组或对象，以及异步操作。 默认情况下，这些辅助代码被插入到使用它们的文件中，如果在许多不同的模块中使用相同的辅助代码，则可能会导致代码重复。

如果启用了 `importHelpers` 选项，这些辅助函数将从 [tslib](https://www.npmjs.com/package/tslib) 中被导入。

以如下 TypeScript 代码为例：
``` js
export function fn(arr: number[]) {
  const arr2 = [1, ...arr];
}
```
开启 `downlevelIteration`（迭代器降级） 并且 `importHelpers` 仍为 false：
``` js
var __read = (this && this.__read) || function (o, n) {
    var m = typeof Symbol === "function" && o[Symbol.iterator];
    if (!m) return o;
    var i = m.call(o), r, ar = [], e;
    try {
        while ((n === void 0 || n-- > 0) && !(r = i.next()).done) ar.push(r.value);
    }
    catch (error) { e = { error: error }; }
    finally {
        try {
            if (r && !r.done && (m = i["return"])) m.call(i);
        }
        finally { if (e) throw e.error; }
    }
    return ar;
};
var __spreadArray = (this && this.__spreadArray) || function (to, from) {
    for (var i = 0, il = from.length, j = to.length; i < il; i++, j++)
        to[j] = from[i];
    return to;
};
export function fn(arr) {
    var arr2 = __spreadArray([1], __read(arr));
}
```

同时开启 `downlevelIteration` 和 `importHelpers`，辅助代码从 `tslib` 导入：

``` js
import { __read, __spreadArray } from "tslib";
export function fn(arr) {
    var arr2 = __spreadArray([1], __read(arr));
}
```

### noImplicitThis

对隐含 any 类型的 this 表达式将生成错误。

``` js
class Rectangle {
  width: number;
  height: number;

  constructor(width: number, height: number) {
    this.width = width;
    this.height = height;
  }

  getAreaFunction() {
    return function () {
      return this.width * this.height;
// 'this' implicitly has type 'any' because it does not have a type annotation.
// 'this' implicitly has type 'any' because it does not have a type annotation.
    };
  }
}
```

### noUncheckedIndexedAccess

在 `索引签名` 的结果中包含 `undefined`。

TypeScript 中可通过索引签名（如`[propName: string]: string;`）来描述对象上未知键名但值类型已知的对象：

``` ts
interface EnvironmentVars {
  NAME: string;
  OS: string;
  // 此索引签名包含未知属性。
  [propName: string]: string;
}
declare const env: EnvironmentVars;
// 声明存在
const sysName = env.NAME;
const os = env.OS; // const os: string

// 未声明，但由于索引签名，则将其视为字符串
const nodeEnv = env.NODE_ENV; // const nodeEnv: string
```
开启 `noUncheckedIndexedAccess` 后，向类型中任何未声明的字段添加 `undefined`：
``` ts
declare const env: EnvironmentVars;
// 声明存在
const sysName = env.NAME;
const os = env.OS; // const os: string

// 未声明，但由于索引签名，则将其视为字符串
const nodeEnv = env.NODE_ENV; // const nodeEnv: string | undefined 👈
```

如果需要访问那个属性，你可以先检查属性是否存在或者使用非空断言运算符（!后缀字符）。

### baseUrl

设置解析非绝对路径模块名时的基准目录。通常用于你不想使用含有 `../` 或 `./` 的路径来导入文件，或不希望在移动文件时需要更改路径。
你可以定义一个根目录，以进行绝对路径文件解析，例如：
```
baseUrl
├── ex.ts
├── hello
│   └── world.ts
└── tsconfig.json
```
在这个项目中被配置为 `"baseUrl": "./"`，TypeScript 将会从首先寻找与 `tsconfig.json` 处于相同目录的文件。
```
import { helloWorld } from "hello/world";
console.log(helloWorld);
```

### rootDirs

根目录。告诉编译器有许多“虚拟”的目录作为一个根目录，这将会允许编译器在这些“虚拟”目录中解析相对应的模块导入，就像它们被合并到同一目录中一样。

例如将`"src/views"` 和 `"generated/templates/views"` 设置为虚拟根目录：
``` json
{
  "compilerOptions": {
    "rootDirs": ["src/views", "generated/templates/views"]
  }
}
```
则：

```
src
└── views
    └── view1.ts (可以 import "./template1", "./view2`)
    └── view2.ts (可以 import "./template1", "./view1`)

generated
└── templates
        └── views
            └── template1.ts (可以 import "./view1", "./view2")
```

### allowSyntheticDefaultImports

当模块没有显式指定默认导出时，你需要这样导入：`import * as React from "react";`。`allowSyntheticDefaultImports` 为 `true` 时，可以让你这样写导入：`import React from "react";`。

示例：
``` js
// ./utilFunctions
const getStringLength = (str) => str.length;
module.exports = {
  getStringLength,
};

// ./index.ts：
import utils from "./utilFunctions";
const count = utils.getStringLength("Check JS");
```

`import utils from "./utilFunctions"` 这段代码会引发一个错误，因为没有 `default` 对象可以导入。 为了使用方便，`Babel` 这样的转译器会在没有默认导出时自动为其创建，使模块看起来更像：

``` js
const getStringLength = (str) => str.length;
const allFunctions = { getStringLength };
module.exports = allFunctions;
module.exports.default = allFunctions; // 创建默认导出
```

本选项不会影响 TypeScript 生成的 JavaScript，它仅对类型检查起作用，当你使用 `Babel` 生成额外的默认导出，从而使模块的默认导出更易用时，本选项可以让 TypeScript 的行为与 Babel 一致。

### esModuleInterop

TypeScript 处理 ES6 模块中导入的 CommonJS/AMD/UMD 模块时存在两个缺陷：
- 形如 `import * as moment from "moment"` 这样的命名空间导入等价于 `const moment = require("moment")`
- 形如 `import moment from "moment"` 这样的默认导入等价于 `const moment = require("moment").default`

这种错误的行为导致了这两个问题：

- ES6 模块规范规定，命名空间导入（即 `import * as x`）只能是一个对象，TypeScript 把它处理成 `= require("x")` 的行为允许把导入当作一个可调用的函数，这样不符合规范。
- 虽然 TypeScript 准确实现了 ES6 模块规范，但是大多数使用 CommonJS/AMD/UMD 模块的库并没有像 TypeScript 那样严格遵守。

开启 `esModuleInterop` 选项将会修复 TypeScript 转译中的这两个问题。第一个问题通过改变编译器的行为来修复，第二个问题则由两个新的工具函数来解决。

具体实现请查看[tsconfig#esModuleInterop](https://www.staging-typescript.org/zh/tsconfig#esModuleInterop)

当启用 `esModuleInterop` 时，将同时启用 `allowSyntheticDefaultImports`。

## 参考
- [typescript docs](https://www.tslang.cn/docs/handbook/compiler-options.html)
- [tsconfig.json](https://aka.ms/tsconfig.json)
- [聊一聊 TypeScript 的工程引用](https://juejin.cn/post/6844904004615421966)。
- [esModuleInterop 到底做了什么？](https://zhuanlan.zhihu.com/p/148081795)