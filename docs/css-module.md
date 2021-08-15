CSS Modules 特性与项目实践
====

CSS Modules 本身比较简单，还要写它的原因是我发现网上很多关于 CSS Modules 的教程或实践文章都已经过时，会存在某些使用方式现今已不可行的情况。<br/>
可前往 [css-modules-demo](https://github.com/wjcj/css-modules-demo) 查看本文中所有示例完整代码。

## 特性

CSS Modules 比较简单，先介绍一下 CSS Modules 常用特性。
在此之前先做一个约定，以 `.module.css` 结尾的 css 文件表示 CSS Modules，其他 `.css` 文件仅视为普通的 css 模块，这也是很多项目中的约定。

### 局部/全局作用域 

由于 CSS 的规则都是全局生效的，所以如果希望一个类名（`.class`）需要实现“局部作用域”的效果就必须保证类名是唯一的。<br/>

- 使用类选择器语法（`.className`）或 `:local` 伪类将类声明为局部作用域类，类名将被编译成一个唯一的哈希类名（哈希字符串，如`_1wtd1y1DR22bj8P0JYV7nH`）。值得注意的是，同一个局部类名被使用多次，编译后的类名都是相同的。
- 使用 `:global` 伪类将声明为一个全局类名，编译后类名不会改变。

``` css
/* 局部作用域 */
.title {
  color: red;
}
/* 等效于：
:local(.title) {
  color: red;
}
 */

/* 全局作用域 */
:global(.title) {
  color: green;
}
```

使用：
``` jsx
import React from 'react';
import styles from "./scope.module.css";

const Demo = () => {
  return (
    <div>
      <h3 className={styles.title}>局部作用域</h3>
      <h3 className="title">全局作用域</h3>
    </div>
  )
};
export default Demo;
```

### 自定义哈希类名

默认算法生成的哈希类名虽然唯一但没有可读性（如 `title` 编译为 `_3zyde4l1yATCOkgn-DBWEL`），所以我们往往希望自定义哈希类名，有两种方式：

1. 设置 `css-loader` 的 `options.modules.localIdentName` 选项，值接受一个字符串模板。比如：

``` js
module.exports = {
  module: {
    rules: [
      {
        test: /\.module\.css$/,
        use: [
          require.resolve('style-loader'),
          {
            loader: require.resolve('css-loader'),
            options: {
              // 开启 CSS Modules
              modules: {
                compileType: 'module',
                localIdentName: "[path][name]__[local]--[hash:base64:5]",
              },
            }
          }
        ]
      },
    ],
  },
};
```

2. 通过设置 `css-loader` 中 `options.modules.getLocalIdent` 选项，值传递一个自定义函数。<br/>

这里使用 `create-react-app` 中的设置作为示例👇，`getCSSModuleLocalIdent` 函数是为了创建格式为 `[filename]_[classname]__[hash]` 唯一类名。比如 `.content.module.css` 中的 `title` 类名将编译为 `content_title__2eyf6`。

``` js
const loaderUtils = require('loader-utils');
const path = require('path');

function getCSSModuleLocalIdent (context, localIdentName, localName, options) {
  // 使用文件名或文件夹名
  const fileNameOrFolder = context.resourcePath.match(/index\.module\.(css|scss|sass)$/) ? '[folder]' : '[name]';
  // 根据文件位置和类名创建哈希
  const hash = loaderUtils.getHashDigest(
    path.posix.relative(context.rootContext, context.resourcePath) + localName,
    'md5',
    'base64',
    5
  );
  // 使用 loaderUtils 查找文件或文件夹名称
  const className = loaderUtils.interpolateName(
    context,
    fileNameOrFolder + '_' + localName + '__' + hash,
    options
  );
  // 删除类名中的 `.module`，并替换所有 "." with "_"。
  return className.replace('.module_', '_').replace(/\./g, '_');
};

module.exports = {
  module: {
    rules: [
      {
        test: /\.module\.css$/,
        use: [
          require.resolve('style-loader'),
          {
            loader: require.resolve('css-loader'),
            options: {
              modules: {
                compileType: 'module',
                getLocalIdent: getCSSModuleLocalIdent
              },
            }
          }
        ]
      },
    ],
  },
};
```

### 组合

CSS Modules 中使用 `composes` 实现样式复用。
`composes` 不仅可组合本模块类名，还可以组合其他 CSS Module 中导入的类名，均仅限在局部样式（`:local`）中使用：

``` css
/* btn.module.css */
.btn {
  border: 1px solid #ccc;
}

/* 组合本模块中的类名👇 */
.btnPrimary {
  composes: btn;
  background: #1890ff;
}

/* 组合其他 CSS Module 中导入的类名👇 */
.btnCircle {
  font-size: 14px;
  composes: circle from './shape.module.css';
  /* 要从多个模块导入，则使用多个 composes */
  composes: bgColor color from "./color.module.css";
  composes: btn;
}
```

``` jsx
// Composes.js
import React from 'react';
import styles from "./btn.module.css";

const Demo = () => {
  return (
    <div>
      <button className={styles.btn}>Button</button>
      <button className={styles.btnPrimary}>Primary Button</button>
      <button className={styles.btnCircle}><Icon type="search"/></button>
    </div>
  );
}
export default Demo;
```
`className={styles.btnPrimary}` 编译后的类名为 `class="btn_btnPrimary__1RVFt btn_btn__1qiyw"`。

在 `:global` 中使用 `composes` 将报错：

``` css
/* Error: composition is only allowed when selector is single :local class name not in ".button", ".button" is weird */
:global(.button) {
  composes: btn;
  color: red;
}
```

### 变量 

`variables.module.css`：

``` css
@value primary: #BF4040;
@value secondary: #1F4F7F;
```

`text.module.css`：
``` css
@value fontSize: 16px;

/* 从其他模块文件中导入👇 */
@value primary as color-primary, secondary from "./variables.module.css";

/* 等效于👇 */
@value variables: "./variables.module.css";
@value primary as color-primary, secondary from variables;

/* 值作为选择器名称 */
@value selectorValue: secondary-color;
.selectorValue {
  color: secondary;
}

.textPrimary {
  font-size: fontSize;
  color: color-primary;
}

.textSecondary {
  font-size: fontSize;
  color: secondary;
}
```

``` jsx
import React from 'react';
import styles from "./text.module.css";

const Demo = () => {
  return (
    <div>
      变量：<br/>
      <p className={styles.textPrimary}>这是一段话...</p>
      <p className={styles.textSecondary}>这是一段话...</p>
      <p className={styles['secondary-color']}>这是一段话...</p>
    </div>
  )
}

export default Demo;
```

### 导入、导出变量

此特性是为了将变量从 CSS 传递给 JS。CSS Module 通过 Interoperable CSS (ICSS) 实现此特性，ICSS 作为 CSS Modules 的低级文件格式规范，只是在标准 CSS 中额外增加了两个的伪选择器 `:import` 和 `:export`。也因此 ICSS 下不能使用上面👆介绍的 CSS Modules 特性。

实际项目中，通常约定扩展名为 `.module.css` 为的文件需解析为 CSS Modules，而常规的 `.css` 文件解析为 ICSS。`css-loader` 中通过 `options.modules.compileType` 选项设置 CSS Modules 解析程度。
``` js
{
  test: /\.css$/, // 匹配css 模块
  exclude: /\.module\.css$/, // 排除 `.module.css` 扩展文件
  use: [
    require.resolve('style-loader'),
    {
      loader: require.resolve('css-loader'),
      options: {
        modules: {
          // compileType：控制编译程度
          compileType: 'icss', // 仅开启 :import 和 :export
        },
      }
    },
  ]
},
{
  test: /\.module\.css$/, // 匹配 CSS Modules
  use: [
    require.resolve('style-loader'),
    {
      loader: require.resolve('css-loader'),
      options: {
        modules: {
          compileType: 'module', // CSS Modules 所有特性
        },
      }
    },
  ]
},
```
*如果设置为 `false` 值会提升性能，因为避免了 CSS Modules 特性的解析。*

**`:export` 和 `:import`**

``` css
/* 导入变量 */
:import("path/to/dep.css") {
  localAlias: keyFromDep;
  /* ... */
}
/* 导出变量 */
:export {
  exportedKey: exportedValue;
	/* ... */
}
```

`:export` 相当于 `cjs` 中的 `module.exports`：

``` js
module.exports = {
	exportedKey: exportedValue;
}
```

推荐约定，但不强制：
- `:export`：只有一个 `:export`块，位于文件的顶部，但在任何 `:import` 块之后。
- `:import`：每个依赖项应该有一个导入；所有导入都应位于文件顶部；本地别名应以双下划线（`__`）为前缀。

具体示例：

`./dep.css`：
``` css
:export {
  theme-color: #1890ff;
  header-height: 60px;
  header-name: abc-header;
  color-secondary: #666;
  screen-min: 768px;
  screen-max: 1200px;
}
```

`icss.css`：
``` css
:import("./dep.css") {
  __themeColor: theme-color;
  __headerHeight: header-height;
  __headerName: header-name;
  __secondary: color-secondary;
  __screenMin: screen-min;
  __screenMax: screen-max;
}

/* 导入的变量可用于任何选择器、任何值和媒体查询参数中 */
/* 任何选择器 */
.__headerName .logo {
  color: red;
  line-height: __headerHeight;
}

/* 任何值 */
.border-theme {
  border: 1px solid __themeColor;
}

/* 媒体查询参数 */
@media (min-width: __screenMin) and (max-width: __screenMax) {
  .__headerName {
    box-shadow: 0 4px 4px __secondary;
  }
}

/* 导出供 js 模块使用 */
:export {
  headerHeight: __headerHeight;
  headerName: __headerName;
}
```
*👆 因为上面使用了导出的变量（`__headerName`、`__headerHeight`），所以 `:export` 块需放在样式规则下方，否则会导致样式规则失效。*

使用：
``` js
import React from 'react';
import styles from "./icss.css";
const { headerHeight, headerName } = styles;

const Demo = () => {
  return (
    <div className={`${headerName} border-theme`} style={{ height: headerHeight }}>
      <span className="logo">logo</span>
    </div>
  )
}

export default Demo;
```

## 项目实践

实际项目开发中都使用 webpack，使用 `css-loader` 解析使用 `css-loader` 解析 CSS Modules。本文中使用 webpack5 做示例演示，去[这里](https://github.com/wjcj/css-modules-demo)可查看完整代码，`package.json` 和 `webpack.config.js` 配置如下👇，使用 `npm run dev` 命令启动本地服务。

### webpack 配置

`package.json`：
``` json
{
  "name": "css-modules-demo",
  "scripts": {
    "dev": "webpack serve --hot",
    "build": "webpack"
  },
  "devDependencies": {
    "@babel/core": "^7.14.6",
    "@babel/preset-react": "^7.14.5",
    "babel-loader": "^8.2.2",
    "css-loader": "^5.2.6",
    "html-webpack-plugin": "^5.3.2",
    "loader-utils": "^2.0.0",
    "postcss-loader": "^6.1.1",
    "style-loader": "^3.0.0",
    "webpack": "^5.44.0",
    "webpack-cli": "^4.7.2",
    "webpack-dev-server": "^3.11.2"
  },
  "dependencies": {
    "react": "^17.0.2",
    "react-dom": "^17.0.2"
  },
  // ...
}
```

项目中通常做如下约定：
- 以 `.module.css` 结尾的扩展文件表示 CSS Modules（解析所有 CSS Modules 特性）。
- 其他 `.css` 文件仅视为常规的 css 模块（仅解析 ICSS 特性, 相比标准 CSS 规范仅额外支持 `:import` 和 `:export`）。

``` jsx
import React from 'react';
import styles from './Button.module.css'; // 导入 CSS Modules 样式文件
import './another-stylesheet.css'; // 导入常规样式

const Button = () => {
  // 作为 js 对象引用
  return <button className={styles.error}>Error Button</button>;
}
```

这样约定的目的一方面是为了规范代码书写，另一方面避免所有样式文件 webpack 都要做 CSS Modules 解析，造成性能浪费。

> 通过为所有未匹配到 `*.module.scss` 命名约定文件设置 compileType 选项，只允许使用可交互的 CSS 特性（即 ICSS 特性, 如 `:import` 和 `:export`），而不使用其他的 CSS Module 特性。此处仅供参考，因为在 v4 之前，css-loader 默认将 ICSS 特性应用于所有文件。

因此做如下 webpack 配置，其中设置 `css-loader` 的 `modules` 选项已启用 CSS Modules，`modules.compileType` 控制编译程度，有 `module` 和 `icss` 可选。
*css-loader 更多配置详解，请查看[这里](https://webpack.docschina.org/loaders/css-loader/#modules)。*

``` js
// webpack.config.js
const HtmlWebpackPlugin = require('html-webpack-plugin');
const path = require('path');

const cssModuleRegex = /\.module\.css$/;

module.exports = {
  mode: 'development',
  devServer: {
    contentBase: './dist',
    hot: true,
  },
  module: {
    rules: [
      {
        oneOf: [
          {
            test: /\.(js|mjs|jsx|ts|tsx)$/,
            loader: require.resolve('babel-loader'),
            options: {
              presets: ['@babel/preset-react']
            }
          },
          // 匹配普通 css 模块
          {
            test: /\.css$/,
            exclude: /\.module\.css$/, // 排除以 `.module.css` 扩展名的文件
            use: [
              require.resolve('style-loader'),
              {
                loader: require.resolve('css-loader'),
                options: {
                  modules: {
                    // 控制编译程度
                    compileType: 'icss', // 仅开启 :import 和 :export
                  },
                }
              },
            ]
          },
          // 匹配 CSS Modules
          {
            test: /\.module\.css$/,
            use: [
              require.resolve('style-loader'),
              {
                loader: require.resolve('css-loader'),
                options: {
                  modules: {
                    compileType: 'module', // CSS Modules 所有特性
                    getLocalIdent: getCSSModuleLocalIdent, // 自定义哈希类名，上面👆介绍过
                  },
                }
              },
            ]
          },
        ]
      }
    ]
  },
  plugins: [
    new HtmlWebpackPlugin({
      inject: true,
      template: 'index.html',
    })
  ]
};
```

### sass/scss

项目中通常都会使用像 less、sass 这些预处理器，这里就以 sass 为例。
同样约定仅扩展名为 `.module.scss` 或 `.module.sass` 解析 CSS Modules 特性，其他 `.scss` 或 `.sass` 扩展文件视为普通的 sass 模块，只做支持 ICSS 特性（`:import` 和 `:export`）的解析。

修改 `webpack.config.js`，增加两个匹配组，配置上主要区别在于 `modules.compileType` 选项：

``` js
// 匹配常规的 sass 模块，仅支持 ICSS
{
  test: /\.(scss|sass)$/,
  exclude: /\.module\.(scss|sass)$/, // 排除 .module.scss 或 .module.sass 扩展文件
  use: [
    require.resolve('style-loader'),
    {
      loader: require.resolve('css-loader'),
      options: {
        importLoaders: 3, // 3 => postcss-loader, resolve-url-loader, sass-loader
        modules: {
          compileType: 'icss',
        },
      }
    },
    {
      // 帮助 sass-loader 找到对应的 url 资源
      loader: require.resolve('resolve-url-loader'),
      options: {
        root: resolveApp('src'),
      },
    },
    {
      loader: require.resolve('sass-loader'),
      options: {
        sourceMap: true, // 这里不可少
      },
    },
  ],
},
// 支持 CSS Modules, 仅匹配 .module.scss 或 .module.sass 扩展文件
{
  test: /\.module\.(scss|sass)$/,
  use: [
    require.resolve('style-loader'),
    {
      loader: require.resolve('css-loader'),
      options: {
        importLoaders: 3,
        modules: {
          compileType: 'module',
          getLocalIdent: getCSSModuleLocalIdent,
        },
      }
    },
    {
      loader: require.resolve('resolve-url-loader'),
      options: {
        root: resolveApp('src'),
      },
    },
    {
      loader: require.resolve('sass-loader'),
      options: {
        sourceMap: true,
      },
    },
  ],
},
// 下面示例中将使用 file-loader 处理图片
{
  loader: require.resolve('file-loader'),
  exclude: [/\.(js|jsx)$/, /\.html$/, /\.json$/],
  options: {
    name: 'static/media/[name].[hash:8].[ext]',
  },
},
```
说明：

- `importLoaders`：表示配置在 `css-loader` 之前的 loader，有几个可以去处理 `@import` 资源（如 `@import 'a.css';`）。上面的配置中 `3` 表示 `@import` 进来的资源可以经过 `postcss-loader`、 `resolve-url-loader` 和 `sass-loader` 处理。
- `resolve-url-loader`：帮助 `sass-loader` 找到对应的 url 资源。Saas 没有提供 [url 重写](https://www.webpackjs.com/loaders/sass-loader/#url-%E7%9A%84%E9%97%AE%E9%A2%98)的功能，此 loader 设置于 loader 链中的 `sass-loader` 之前就可以重写 url。

**示例：**

分别创建了一个 `header.module.sass` 和 `header.sass` 文件，内容均如下：

``` scss
$shadow-color: #ccc
$header-height: 60px
$theme: #1890ff

.className
  border: 1px solid $theme
  padding: 20px
  box-shadow: 0 4px 4px $shadow-color
  a
    background-color: #fff
    display: inline-block

// 导出变量
:export
  height: $header-height
```

``` jsx
import React from 'react';
import styles1 from "./header.module.sass";
import styles2 from "./header.sass"; // ICSS
import logo from './logo.png';

console.log('styles1', styles1); // {height: "60px", className: "header_className__rUUph"}
console.log('styles2', styles2); // {height: "60px"}

const Logo = ({ height }) => (
  <img style={{ height, width: height }} src={logo} />
);

const Demo = () => {
  return (
    <div className={styles1.className}>
      <Logo height={styles1.height} />
    </div>
  );
};
export default Demo;
```

### `classnames`

匹配 `classnames` 使用更加方便：

``` js
import classNames from 'classnames';
import styles from './dialog.module.css';

const Dialog = ({ disabled }) => {
  const cx = classNames({
    [styles.confirm]: !disabled,
    [styles.disabledConfirm]: disabled
  });
  return (
    <div className={styles.root}>
      <a className={cx}>Confirm</a>
      ...
    </div>
  );
};
```

## 参考
- [css-loader#modules](https://webpack.docschina.org/loaders/css-loader/#modules)
- [CSS Modules](https://github.com/css-modules/css-modules)
- [Exporting values variables](https://github.com/css-modules/css-modules/blob/master/docs/values-variables.md)
- [CSS Modules ICSS](https://github.com/css-modules/icss#specification)
- [Create React App](https://create-react-app.dev/docs/adding-a-css-modules-stylesheet/)
