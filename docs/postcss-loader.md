postcss-loader、PostCss 插件相关
====

本文从一段 `postcss-loader` 配置👇展开聊聊 PostCss 及其相关插件。可 `git clone` [源码](https://github.com/wjcj/postcss-demo) 查看示例效果。

``` js
{
  loader: require.resolve('postcss-loader'),
  options: {
    // 忽略
    // require.resolve('style-loader')
    // {
    //   loader: require.resolve('css-loader'),
    //   options: cssOptions,
    // },
    // PostCss 选项
    postcssOptions: {
      plugins: [
        // 修复 flexbox 相关的一些bug
        require('postcss-flexbugs-fixes'),
        [
          // 支持未来的css特性
          require('postcss-preset-env'),
          {
            // autoprefixer 选项
            autoprefixer: {
              // 为 flexbox 相关属性添加前缀，默认 true
              flexbox: 'no-2009', // 只为最终和IE10版本的规范添加前缀
            },
            // 支持阶段3特性
            stage: 3,
          },
        ],
        // 根据目标浏览器列表设置引入 normalize.css 中需要的部分
        postcssNormalize(), // require('postcss-normalize')。
      ],
    },
  },
},
```

### PostCSS 和 postcss-loader

PostCSS 是一个允许使用 JS 插件转换样式的工具，它接收一个 CSS 文件并将其解析成抽象语法树(AST树)，并通过提供了一系列 API 供插件使用以实现 CSS 规则的分析和修改，比如自动添加 CSS 兼容前缀。

使用 PostCSS 即是组合使用一系列的插件，webpack 中通过 `postcss-loader` 使用插件，而 PostCSS 本身可以使用配置文件设置插件选项（推荐 `postcss.config.js`）：

``` js
// postcss.config.js
module.exports = {
  // parser: 'sugarss', // 编译器（css字符串 => AST）
  plugins: [
    // PostCSS 插件
    ['postcss-short', { prefix: 'x' }],  // 给插件传递参数
    'postcss-preset-env',
  ],
};
```

`postcss-loader` 中可以设置 `postcssOptions.config` 为 `true` 允许 `postcss-loader` 使用配置文件中的选项，配置文件中的选项将会和 loader 选项进行合并，且 loader 选项具有更高的优先级，但不推荐同时使用配置文件和 loader 选项。

``` js
// webpack.config.js
module.exports = {
  module: {
    rules: [
      {
        test: /\.css$/i,
        loader: 'postcss-loader',
        options: {
          postcssOptions: {
            config: true,
          },
        },
      },
    ],
  },
};
```

### postcss-flexbugs-fixes

此插件修复了 [flexbugs](https://github.com/philipwalton/flexbugs) 中罗列的、有 flex 布局相关的部分 bug。

比如 [bug 8.1.a](https://github.com/philipwalton/flexbugs#flexbug-8)：IE 10-11 会忽略 `flex` 简写中使用的 `calc()` 函数（Edge 中已修复），比如 `flex: 0 0 calc(100%/3)` 在 IE 中将不起作用。<br/>
经 `postcss-flexbugs-fixes` 处理后其变为：
``` css
.foo { flex-grow: 0; flex-shrink: 0; flex-basis: calc(100%/3); }
```

### autoprefixer

自动添加浏览器兼容前缀（如 `-webkit-`(chrome，safari)、`-moz-`(firefox)、`-ms-`(IE)、`-o-`(opera)）。比如下面代码：
``` css
::placeholder {
  color: gray;
}
```
经 `postcss-flexbugs-fixes` 处理后变为：
``` css
::-moz-placeholder {
  color: gray;
}

:-ms-input-placeholder {
  color: gray;
}

::placeholder {
  color: gray;
}
```

W3C 在制定 CSS 标准过程是复杂漫长的，一个官方的 W3C 标准通常需要经过以下几个阶段：

- 工作草案（Stage 0）：作组依据提出一系列草案，公众和 W3C 会员可以提出评论和问题由工作组处理。
- 最终工作草案（Stage 1）：工作组已完成工作，并要求公众和 W3C 会员提交最后的评论与问题。同样，工作组必须处理这些反馈。如果出现情况，可能要回到工作草案阶段。
- 候选推荐标准（Stage 2）：最终工作草案阶段问题都得到解决后将进入候选推荐标准阶段，此时可以认为该规范已经稳定，可以展开试验性实施了。工作组必须将实施中得到的反馈整合到规范中。同样，如果出现情况需返回到工作草案阶段。
- 建议推荐标准（Stage 3）：做最后审阅，如无意外，规范将进入建议推荐标准阶段。在此阶段，W3C 总监将正式请求 W3C 会员审阅这份建议推荐标准。
- 推荐标准（Stage 4）：根据审阅结果，该规范可能成为 W3C 推荐标准，也可能中间发生微小改动返回工作草案阶段，或者彻底从 W3C 工作日程上移去。技术规范一旦成为推荐标准，它就是官方的 W3C 标准了。

比如一个 `::placeholder` 属性在经历上面这个过程中，各浏览器将其实现准备供开发者可以提前使用，但又担心最终的标准会发生变更，因此各浏览器会在属性前加上一个私有前缀来支持新属性。
但对于开发者来说，并不清楚哪些 CSS 属性需要写成兼容模式、需要加哪些私有前缀。而这些最新信息可以通过 [Can I Use](https://caniuse.com/) 网站获得，上面可查询各浏览器和设备对属性的支持信息👇。
![caniuse flexbox](https://i.ibb.co/GnQwy59/caniuse.jpg)

日常开发中，我们通常在 `package.json#browserslist` 字段（或 `.browserslistrc` 文件）指定我们期望的目标浏览器列表，例如 `Create React App` 中：
``` json
{
  "browserslist": {
    "production": [
      ">0.2%",
      "not dead",
      "not op_mini all"
    ],
    "development": [
      "last 1 chrome version",
      "last 1 firefox version",
      "last 1 safari version"
    ]
  }
}
```

`autoprefixer` 插件会读取 `package.json#browserslist` 配置信息，并结合 [Can I Use](https://caniuse.com/) 网站上的数据确定哪些 css属性是否需要加、需要加哪些浏览器兼容前缀。

[autoprefixer 配置选项](https://github.com/postcss/autoprefixer#options)。

### browserslist

目标浏览器列表。<br/>
推荐通过 `package.json` 中的 `browserslist` 字段或创建 `.browserslistrc` 文件进行配置。此配置可以在不同前端工具之间共享，比如本文中的 `autoprefixer`、`postcss-preset-env`、`postcss-normalize` 都可以使用 `package.json` 或 `.browserslistrc` 文件中的目标浏览器列表配置。

如果没有设置 `browserslist` 则会使用默认的浏览器配置 `defaults`：`> 0.5%, last 2 versions, Firefox ESR, not dead`，其含义：

- `> 0.5%`：全球使用人数超过 0.5%。
- `last 2 versions`：每个浏览器的最近 2 个版本。
- `Firefox ESR`：最新的 [Firefox 扩展支持版本](https://support.mozilla.org/en-US/kb/choosing-firefox-update-channel)。
- `not dead`：排除24个月内没有官方支持或更新的浏览器

更多配置选项可查看[这里](https://github.com/browserslist/browserslist#full-list)。

**`package.json#browserslist` 和 `.browserslistrc` 配置方法：**
`package.json`：

``` json
{
  "browserslist": [
    "defaults",
    "not IE 11",
    "maintained node versions"
  ]
}
```
对应的 `.browserslistrc` 文件配置：

```
defaults
not IE 11
maintained node versions
```

针对不同环境进行配置，Browserslist 会根据 `BROWSERSLIST_ENV` 或 `NODE_ENV` 变量选择查询，如果你没有声明它们（如 `process.env.NODE_ENV = 'development'`）将首先查找 `production`。

`package.json`：
```
{
  "browserslist": {
    "production": [
      ">0.2%", // 全球超过 0.2% 人使用
      "not dead", // 排除 24 个月内没有官方支持或更新的浏览器
      "not op_mini all" // 排除所有 Opera Mini 浏览器
    ],
    "development": [
      "last 1 chrome version", // chrome 最近的一个版本
      "last 1 firefox version", // firefox 最近的一个版本
      "last 1 safari version" // safari 最近的一个版本
    ]
  }
}
```

对应的 `.browserslistrc` 文件配置：

```
[production]
>0.2%
not dead
ie 10

[development]
last 1 chrome version
last 1 firefox version
last 1 safari version
```

### postcss-preset-env

`postcss-preset-env` 允许你使用未来的（还未成为推荐标准）CSS 特性，插件会根据你的目标浏览器或运行时环境添加所需要的 polyfill。

在没有任何配置选项的情况下，PostCSS Preset Env 默认启用 `Stage 2` 阶段 CSS 特性并支持所有浏览器。

**常用选项：**

- `stage`：此选项可以启用指定阶段的 CSS 特性并填充 polyfill。
- `features`：此选项可通过 [ID] 启用或禁用特定的 polyfill，可选 [id 列表](https://github.com/csstools/postcss-preset-env/blob/main/src/lib/plugins-by-id.js#L36)，任何未明确启用或禁用的 polyfill `features` 都由该 `stage` 选项确定。
- `browsers`：根据你需要支持的目标浏览器确定需要哪些 polyfill。但通常推荐在通常在 `package.json#browserslist` 字段（或 `.browserslistrc` 文件）指定。如指定了无效的 browserslist 配置，将则将使用 [默认的 browserslist](https://github.com/browserslist/browserslist#queries) 进行查询（`> 0.5%, last 2 versions, Firefox ESR, not dead`）。

``` js
// postcss-loader 中
[
  require('postcss-preset-env'),
  // 指定插件选项
  {
    stage: 3, // 阶段3 CSS特性（建议推荐标准） 
    features: {
      'nesting-rules': true, // 支持 css 规则嵌套
    },
    browsers : 'last 2 versions', // 所有浏览器最近2个版本
  },
]
```

`postcss-preset-env` 中包含 `autoprefixer` 的插件，通过 `autoprefixer` 选项指定 `autoprefixer` 插件的配置项。

``` js
[
  require('postcss-preset-env'),
  {
    stage: 3,
    // autoprefixer 插件选项
    autoprefixer: {
      flexbox: 'no-2009',
    },
  },
]
```

### postcss-normalize

`postcss-normalize` 可根据 `browserslist`（目标浏览器列表）设置引入 `normalize.css` 或 `sanitize.css` 中需要的部分。<br/>
*`normalize.css` 使浏览器更一致地呈现所有元素并符合现代标准。与重置 CSS 样式的方式不一样，它保留了一些有用的默认样式。*<br/>

``` css
@import "normalize.css";
```

`browserslist` 为 `ie >= 9` 时：
``` css
audio,
video {
  display: inline-block;
}

img {
  border-style: none;
}
```
`browserslist` 为 `ie >= 10` 时， `audio` 和 `video` 元素的样式声明都没有引入：
``` css
img {
  border-style: none;
}
```
**配置选项：**

### mini-css-extract-plugin

此插件可将 CSS 提取到单独的文件中，为每个包含 CSS 的 JS 文件创建一个 CSS 文件，并且支持 CSS 和 SourceMaps 的按需加载，需要在 webpack 4 中使用。

``` js
const MiniCssExtractPlugin = require("mini-css-extract-plugin");

module.exports = {
  module: {
    rules: [
      {
        test: /\.css$/i,
        use: [
          MiniCssExtractPlugin.loader, // 替代 style-loader
          "css-loader"
        ],
      },
    ],
  },
  plugins: [new MiniCssExtractPlugin()],
};
```

更详细的用法查看[这里](https://webpack.docschina.org/plugins/mini-css-extract-plugin/)。

## 参考

- [postcss-loader config](https://webpack.docschina.org/loaders/postcss-loader/#config)
- [postcss](https://postcss.org/api/#processoptions)
- [PostCSS Flexbugs Fixes](https://github.com/luisrudge/postcss-flexbugs-fixes)
- [PostCSS Preset Env](https://github.com/csstools/postcss-preset-env)
- [Browserslist](https://github.com/browserslist/browserslist)
- [MiniCssExtractPlugin](https://webpack.docschina.org/plugins/mini-css-extract-plugin/)
