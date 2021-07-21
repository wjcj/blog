webpack 实现（入门篇）
======

相比全网其他介绍 webpack 实现的文章，本文刨去了所有 webpack 进阶特性，只介绍 webpack 基础的 js 和 css 打包功能实现，只为更加最简单易懂。同时本文通过 webpack 从简到繁的打包结果由浅入深推导实现，也不失为一种不错的学习方法。

## 从结果出发

我们都知道 webpack 是一个模块打包工具，根据入口解析依赖，并生成一个或多个 bundle 文件。我们就从先写一个最简单的入口文件，得到一个最简单的 bundle 入手，看看 webpack 都做了什么。<br/>*本文中使用的 webpack 版本： `webpack@^5.40.0`、 `webpack-cli@^4.7.2`*

先写一个最简单的入口文件 `index.js`，里面仅输出一个 `hello world`，不引入其他模块。执行 `npx webpack --mode development` 得到 bundle 文件（`dist/main.js`）:
``` js
// src/index.js
console.log('hello world')
```

简化变量命名、去除注释和部分功能代码（如缓存）后的 bundle 文件👇。`__modules__` 中是模块信息，`key` 是模块文件路径，`value` 是一个使用 `eval` 执行模块内容的箭头函数。最后调用入口模块 `__modules__[entry](）`。

``` js
// dist/main.js
(() => {
  var __modules__ = ({

    "./src/index.js":
      (() => {
        eval("console.log('hello world');");
      })

  });
  
  var __exports__ = {};

  __modules__["./src/index.js"]();
})();
```  

这样看起来 `webpack --mode development` 命令做的事情很简单，使用默认的入口（`src/index.js`） 读取入口的模块内容保存在 `__modules__` 对象中，最后执行入口模块的内容。

为了区分 `webpack` 命令，我们实现一个 `my-webpack` 命令。步骤如下：

1. 配置以下 `package.json#bin` 字段

指定项目中命令对应的执行文件的位置。

``` json
// package.json
{
  "bin": {
    "my-webpack": "bin/webpack.js"
  },
}
```

2. `bin/webpack.js`

目录结构：
```
webpack
├── bin
│   └── webpack.js
├── src
│   └── index.js
└── package.json
```

`#! /usr/bin/env node` 是声明此命令执行文件用 node 解析。<br/>
输出内容为 `bundleTemplate`，替换掉 `bundleTemplate` 中的 `入口模块路径`（即`entry`）和 `模块内容`（即 `script`）即可，这里使用模板引擎 `ejs`来简化操作。

``` js
#! /usr/bin/env node

const fs = require('fs');
const ejs = require('ejs');

// 默认入口、出口文件配置
const entry = './src/index.js';
const output = './dist/bundle.js';

// 读取入口模块内容
const script = fs.readFileSync(entry, 'utf8');

// 替换输出模板中变量
let bundleTemplate = ` 
(() => {
  var __modules__ = ({

    "<%-entry%>":
      (() => {
        eval(\`<%-script%>\`);
      })

  });
  
  var __exports__ = {};
  __modules__["<%-entry%>"]();
})();
`;
const result = ejs.render(bundleTemplate, {
  entry,
  output,
  script,
});

// 把 bundle 写入出口文件
fs.writeFileSync(output, result);
console.log('编译完成');
```

3. 测试 `my-webpack` 命令

在根目录下执行 `npm link` 建立软链接后，本地就可执行 `my-webpack` 命令。
查看`my-webpack` 命令打包输出的 `dist/bundle.js` 与 `webpack` 输出的 `dist/main.js` 是否一致。

## 支持模块依赖

通常项目入口文件还引入其他模块，比如（`require('./a.js);`）:

``` js
// src/a.js
const b = require('./b.js');
module.exports = 'a' + b;

// src/b.js
module.exports = 'b';
```

```
webpack
├── bin
│   └── webpack.js
├── src
├── |── a.js
├── |── b.js
│   └── index.js
└── package.json
```

同样的，我们先执行 `npx webpack --mode development` 看一下 webpack 打包做了什么？

简化后的 bundle 文件（`dist/main.js`）：

``` js
(() => {
  var __modules__ = ({

    "./src/a.js":
      ((module, exports, require) => {
        eval("const b = require(\"./src/b.js\");\n\nmodule.exports = 'a' + b;");
      }),

    "./src/b.js":
      ((module, exports, require) => {
        eval("module.exports = 'b';");
      }),

    "./src/index.js":
      ((__unused_module, __unused_exports, require) => {
        eval("const result = require(\"./src/a.js\");\n\nconsole.log(result);");
      })

  });

  function require(moduleId) {
    // 保存模块导出的结果
    var module = {
      exports: {}
    };

    __modules__[moduleId](module, module.exports, require);

    return module.exports;
  }

  var __exports__ = require("./src/index.js");
})();
```

可以看出：

- 模块列表中（`__modules__`） 多了新增的 `a.js` `b.js` 模块，模块执行函数的参数分别是用于模块导出的 `module` 对象（`exports` 即 `module.exports`）、用于导入模块的 `require` 命令。
- `require` 函数是 `require` 命令具体实现，它根据模块路径（`moduleId`）从 `__modules__` 查找模块对应的执行函数。模块首次执行后，模块中导出的结果将保存在 `module.exports` 。

**修改 `bin/webpack.js` ：**

``` js
#! /usr/bin/env node

const fs = require('fs');
const path = require('path');
const ejs = require('ejs');

// 默认入口、出口文件配置
const entry = './src/index.js';
const output = './dist/main.js';

let script = fs.readFileSync(entry, 'utf8');

const modules = [];

function replaceRequire(script) {
  script = script.replace(/require\(['"](.+?)['"]\)/g, function() {
    const moduleName = arguments[1];
    const name = path.join('./src', moduleName);
    let content = fs.readFileSync(name, 'utf8');
    content = replaceRequire(content); // 处理被引入模块中可能存在的 require 命令

    // 保存模块内容
    modules.push({ name, content });

    return `require("${name}")`;
  });
  return script;
};

// 匹配代码中的 require 命令，根据模块路径读取模块内容并保存至 modules
script = replaceRequire(script);

// 入口文件之外的模块通过 for 循环生成
let bundleTemplate = ` 
(() => {
  var __modules__ = ({

    "<%-entry%>":
      ((__unused_module, __unused_exports, require) => {
        eval(\`<%-script%>\`);
      })

    <%for(let i = 0; i < modules.length; i++) {
      let module = modules[i];%>,
      "<%-module.name%>":
        ((module, exports, require) => {
          eval(\`<%-module.content%>\`);
        })
    <%}%>
  });

  function require(moduleId) {
    var module = {
      exports: {}
    };

    __modules__[moduleId](module, module.exports, require);

    return module.exports;
  }

  require("<%-entry%>");
})();
`

const result = ejs.render(bundleTemplate, {
  entry,
  script,
  modules,
});

fs.writeFileSync(output, result);
console.log('编译完成');
```

## loader

以 `css-loader` 为例，当 `require` 匹配到 `css` 模块时，将 `require` 命令转换成插入 <style> 的脚本代码。

``` js
function cssLoader(source) {
  return `
    const style = document.createElement('style');
    style.innerText = ${JSON.stringify(source).replace(/(\\r)?(\\n)|\\r/g, '')};
    document.head.appendChild(style);
  `
};

function replaceRequire(script) {
  script = script.replace(/require\(['"](.+?)['"]\)/g, function() {
    const moduleName = arguments[1];
    const name = path.join('./src', moduleName);
    let content = fs.readFileSync(name, 'utf8');
    content = replaceRequire(content);

    // loader
    if (/.css$/.test(name)) {
      content = cssLoader(content);
    };

    // 保存模块内容
    modules.push({ name, content });
    return `require("${name}")`;
  });
  return script;
};
```

以上，一个可打包 js 和 css 模块的简易 webpack 就基本实现了。
