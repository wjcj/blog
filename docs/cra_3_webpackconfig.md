create-react-app 实现（下）—— cra 中的 webpack 配置详解
====

上两篇中介绍了 `create-react-app` 中关于 `create-react-app` 包和 `react-scripts` 包中几个核心命令实现。最后还剩下一块重要内容在本篇中介绍 —— `create-react-app` 中的 webpack 配置。

**说明：** 为了方便阅读，大部分介绍在[预览](#预览) 部分对源码做了批注说明，如说明的内容过多则添加有 `详见👇` 标识，可在[详解](#详解)部分查看更详细的内容。

## 预览

``` js
// webpackEnv: 值为 'production'、'development'。此函数返回执行 `react-scripts start/build` 所需的 webpack 配置项。
module.exports = function (webpackEnv) {
  return {
    // 模式，不同模式下启用一系列不同的默认优化配置项。详见👇
    mode: isEnvProduction ? 'production' : isEnvDevelopment && 'development',
    // 是否发现错误就立即抛出并退出编译，通常在 production 模式下开启。开发环境中使用 `HMR` 将在终端和浏览器控制台抛出错误
    bail: isEnvProduction, 
    // 选择 sourceMap 配置，详见👇
    devtool: isEnvProduction
      ? shouldUseSourceMap
        ? 'source-map' // sourceMap 信息最完整也是打包最慢的。
        : false
      : isEnvDevelopment && 'cheap-module-source-map', // 
    // 入口配置，详见👇
    entry:
    // 输出配置。webpack 如何输出结果的相关选项。
    output: {
      // 所有输出文件的目标路径，必须绝对路径（使用 Node.js 的 path 模块），paths.appBuild 指向 `build` 目录，webpack 默认是 'dist'
      path: isEnvProduction ? paths.appBuild : undefined,
      // 输出的 require() 中添加/* filename */ 注释，例如__webpack_require__(/*! ./a.js */ "./src/a.js")
      pathinfo: isEnvDevelopment,
      // 设置每个输出 bundle 的名称，开发环境下不会生成真正的文件。此处也不影响按需加载 chunk 和 loader 产生的输出文件。
      filename: isEnvProduction
        ? 'static/js/[name].[contenthash:8].js'
        : isEnvDevelopment && 'static/js/bundle.js',
      futureEmitAssets: true, // webpack 5 默认支持此特性，并移除此选项
      // 设置非入口(non-entry) chunk 文件的名称（按需加载 chunk 的输出文件）
      chunkFilename: isEnvProduction
        ? 'static/js/[name].[contenthash:8].chunk.js'
        : isEnvDevelopment && 'static/js/[name].chunk.js',
      // 按需加载资源或外部资源(如图片、文件等）的公开 url，尾部斜杠不能少。
      // publicUrlOrPath 则是按 process.env.PUBLIC_URL、'package.json#homepage' 、'/' 的顺序来推断。
      publicPath: paths.publicUrlOrPath,
      // 浏览器开发者工具 source map 文件名模板，这里将其指向原始磁盘位置（在 Windows 上格式为 URL）
      devtoolModuleFilenameTemplate: isEnvProduction
        ? info =>
            path
              .relative(paths.appSrc, info.absoluteResourcePath)
              .replace(/\\/g, '/')
        : isEnvDevelopment &&
          (info => path.resolve(info.absoluteResourcePath).replace(/\\/g, '/')),
      // jsonp 函数用于异步加载 chunk，在同一页面上使用多个 webpack runtime 时会造成 jsonp 命名冲突，因为默认都是 'webpackJsonp' 字符。这里拼接了 'package.json#name' 字段防止冲突。
      jsonpFunction: `webpackJsonp${appPackageJson.name}`,
      // 默认为 `window`，设置为 `this` 后让打包后的模块也可以在 web workers中使用。
      globalObject: 'this',
    },
    // 优化
    optimization: {
      // 是否压缩 bundle，production 模式下默认为 true
      minimize: isEnvProduction,
      // 提供一个或多个自定义的 TerserPlugin 实例覆盖默认压缩工具
      minimizer: [
        // 仅用于 'production' 模式
        // terser-webpack-plugin 👇
        new TerserPlugin({
          // https://github.com/terser/terser#minify-options
          terserOptions: {
            // 解析 ecma8 代码
            parse: {
              // 我们希望 terser 解析 ecma 8 代码。
              // 但是不希望它应用任何将有效的 ecma 5 代码转换为无效的 ecma 5 代码的压缩步骤。
              // 所以 `compress` 和 `output` 中只应用 ecma5
              // https://github.com/facebook/create-react-app/pull/4234
              ecma: 8,
            },
            // 压缩
            compress: {
              // ecma 设置为 5 不会将 ES6+ 代码转换为 ES5。只是禁用 es6+ 优化
              ecma: 5,
              warnings: false,
              // 对二进制节点应用某些优化，比如 !(a <= b) → a > b。
              // Uglify 破坏了有效代码，故被禁用。https://github.com/facebook/create-react-app/issues/2376
              // 有待进一步调查：https://github.com/mishoo/UglifyJS2/issues/2011
              comparisons: false,
              // Terser 破坏了有效代码，故被禁用:
              // https://github.com/facebook/create-react-app/issues/5250
              // 有待进一步调查: https://github.com/terser-js/terser/issues/120
              inline: 2, // 内联调用函数，3：带有参数和变量的内联函数
            },
            // 混淆
            mangle: {
              // safari10 bug 修复，https://bugs.webkit.org/show_bug.cgi?id=171041
              safari10: true, 
            },
            // Added for profiling in devtools
            // isEnvProductionProfile = isEnvProduction && process.argv.includes('--profile')
            keep_classnames: isEnvProductionProfile, // 不破坏 class 名称
            keep_fnames: isEnvProductionProfile, // 不破坏函数名
            output: {
              // `ecma` 设置为 5 不会将 ES6+ 转换为 ES5，只是在美化器的控制下优化输出
              ecma: 5,
              // 是否保留注释，不保留
              comments: false,
              // 不启用的话，emoji and regex 无法正常压缩
              // https://github.com/facebook/create-react-app/issues/2488
              ascii_only: true,
            },
          },
          sourceMap: shouldUseSourceMap, // 开启 sourceMap
        }),
        // 仅用于 'production' 模式
        // 优化、压缩 css 插件，optimize-css-assets-webpack-plugin
        new OptimizeCSSAssetsPlugin({
          // cssProcessor: require('cssnano'), // css 压缩器，默认 cssnano
          // 传给 cssProcessor 的选项，这里即 cssnano
          cssProcessorOptions: {
            // 解析器
            // safePostCssParser：查找并修复 CSS 语法错误
            parser: safePostCssParser, // require('postcss-safe-parser');
            map: shouldUseSourceMap
              ? {
                  // 强制生成单独的 SourceMap 文件，不内联
                  inline: false,
                  // `annotation: true` 将 sourceMappingURL 附加到css文件 末尾，帮助浏览器找到sourcemap
                  annotation: true,
                }
              : false,
          },
          // 传给 cssProcessor 插件的选项
          cssProcessorPluginOptions: {
            // 预设
            preset: ['default', { minifyFontValues: { removeQuotes: false } }], // 移除双引号
          },
        }),
      ],
      // 代码分割。默认会分割出一个 vendors（第三方） 和 commons （多入口共享的公共模块）
      // splitChunks 有一系列的默认配置，此处覆盖了两个默认配置，其他默认配置详见👇
      splitChunks: {
        // 选择哪些 chunk 进行优化（公共模块拆分）
        chunks: 'all', // = async(异步) + initial（同步），表示可以在异步和非异步 chunk 之间共享
        // 拆分 chunk 的名称，false 表示不修改名称
        name: isEnvDevelopment,
      },
      // 将 webpack runtime 代码分割出去，详见👇
      // webpack runtime 代码（编译后代码运行需要的代码）每次构建都会发生改变，默认注入到业务代码中导致文件名改变，导致浏览器缓存失效。
      runtimeChunk: {
        // 自定义 runtimeChunk 名称
        name: entrypoint => `runtime-${entrypoint.name}`,
      },
    },
    // 配置模块如何解析
    resolve: {
      // 告诉 webpack 解析模块时应该搜索的目录，优先匹配 'node_modules'
      // https://github.com/facebook/create-react-app/issues/253
      modules: ['node_modules', paths.appNodeModules].concat(
        modules.additionalModulePaths || []
      ),
      // 文件扩展名。引入模块时省略扩展名时(如 import File from '../path/to/file') 帮助 webpack 解析模块
      // paths.moduleFileExtensions 包含'web.mjs', 'mjs', 'web.js', 'js', 'web.ts', 'ts', 'web.tsx', 'tsx', 'json', 'web.jsx', 'jsx',
      extensions: paths.moduleFileExtensions
        .map(ext => `.${ext}`)
        // 未使用 TypeScript 时，过滤掉 ts 相关扩展名
        .filter(ext => useTypeScript || !ext.includes('ts')),
      // 创建 import 或 require 的别名，简化模块引入
      alias: {
        // 支持 React Native Web
        'react-native': 'react-native-web',
        // Allows for better profiling with ReactDevTools
        ...(isEnvProductionProfile && {
          'react-dom$': 'react-dom/profiling',
          'scheduler/tracing': 'scheduler/tracing-profiling',
        }),
        // 添加 `src: paths.appSrc`,
        ...(modules.webpackAliases || {}),
      },
      // 额外的解析插件列表
      plugins: [
        // 增加对 yarn pnp（Plug'n'Play）的支持。pnp 不再需要将依赖从缓存拷贝至 node_modules，而直接使用全局 module 缓存，减少大量文件 I/O，从而加快了模块安装速度。使用 `yarn --pnp` 开启 yarn pnp 功能。
        // `create-react-app` 中创建项目时增加 `--use-pnp` 选项开启 pnp，即 `create-react-app myapp --use-pnp`
        PnpWebpackPlugin, // require('pnp-webpack-plugin')
        // 阻止用户从 src/ 和 node_modules/ 之外的目录导入文件。因为之外的文件不被 babel 处理（见babel-loader include配置），除非你能保证引入的文件都是编译过的才可移除此插件。
        new ModuleScopePlugin(paths.appSrc, [
          paths.appPackageJson, // package.json 所在路径
          reactRefreshOverlayEntry, // require.resolve('react-dev-utils/refreshOverlayInterop');
        ]),
      ],
    },
    // 与 `resolve` 类似，但仅用于解析 webpack 的 loader 包（找到加载器在磁盘上的位置）
    resolveLoader: {
      plugins: [
        // 告诉 webpack 从当前包（PnpWebpackPlugin）加载它的加载器
        PnpWebpackPlugin.moduleLoader(module),
      ],
    },
    // 配置如何处理项目中的不同类型的模块
    module: {
      // 不存在的导入将报错而不是警告。如模块 animal 中没有 flower，但 `var { flower } = require('animal')`
      strictExportPresence: true,
      rules: [
        // 禁用 require.ensure，不是标准的语言特性，使用import()
        { parser: { requireEnsure: false } },
        // 处理包含 source maps 的 node_modules 包。从源文件中提取 source maps（从 sourceMappingURL 中提取）
        // 如果 source maps 数据没有正确提取和处理，注入 bundle 后浏览器有可能无法正确解析这些数据。source-map-loader 允许 webpack 跨库处理 source map 数据，因而更易于调试。
        shouldUseSourceMap && {
          // loader 类型，详见 Rules.enforce👇
          enforce: 'pre', // 前置 loader
          // 排除所有符合条件的模块
          exclude: /@babel(?:\/|\\{1,2})runtime/,
          test: /\.(js|mjs|jsx|ts|tsx|css)$/,
          use: 'source-map-loader',
        },
        {
          // oneOf 将遍历所有后续加载器，直到满足要求为止。当没有加载器匹配时，它将回退到加载器列表末尾的 file-loader。
          oneOf: [
            // TODO: Merge this config once `image/avif` is in the mime-db. https://github.com/jshttp/mime-db
            {
              test: [/\.avif$/], // 一种图片格式
              loader: require.resolve('url-loader'),
              options: {
                // 字节大小限制（默认无限制，单位 b）。小于限制则转化为 Data URLs 内联在 javaScript 中以避免网络请求，否则单独生成文件。
                limit: imageInlineSizeLimit, // imageInlineSizeLimit = process.env.IMAGE_INLINE_SIZE_LIMIT || '10000'
                //  MIME 类型
                mimetype: 'image/avif',
                // chunk 文件名称
                name: 'static/media/[name].[hash:8].[ext]',
              },
            },
            // url-loader 与 file-loader 工作方式类似，不同之处在于它将小于指定字节限制的资源文件作为数据 URL 内联在 javaScript 中以避免网络请求。
            {
              test: [/\.bmp$/, /\.gif$/, /\.jpe?g$/, /\.png$/], // 处理图片格式
              loader: require.resolve('url-loader'),
              options: {
                limit: imageInlineSizeLimit,
                name: 'static/media/[name].[hash:8].[ext]',
              },
            },
            // 使用 Babel 处理 src 目录 JS。
            // 预设包括 JSX、Flow、TypeScript 和一些 ESnext 特性。
            {
              test: /\.(js|mjs|jsx|ts|tsx)$/,
              include: paths.appSrc, // 仅处理 src 目录
              loader: require.resolve('babel-loader'),
              options: {
                // 自定义包装 babel-loader。
                // 在 `const myCustomLoader = require("babel-loader").custom(callback)` 中，`custom` 方法接受一个函数回调，允许用户为其处理的每个文件添加Babel 配置的自定义处理。
                // 如不想创建新文件去调用 `custom` 方法，则可配置 `customize` 选项，值为导出 custom 回调的模块路径。
                customize: require.resolve(
                  // 详见 babel-preset-react-app👇
                  'babel-preset-react-app/webpack-overrides'
                ),
                // 一组预设。预设是一系列插件的组合
                presets: [
                  [
                    require.resolve('babel-preset-react-app'), // 详见 babel-preset-react-app 👇
                    // 指定 'babel-preset-react-app' 预设的参数，方式：["presetA", options]
                    {
                      // runtime：预设参数，表示 jsx 转换方式。详见 babel-preset-react-app 👇
                      // 'classic' 是旧的方式，会把 JSX 转换为 `React.createElement(...)` 调用，但存在一些缺陷。
                      // 'automatic' (react17)是一种新的转化方式，可解决 'classic' 方式的缺陷，使用 'react/jsx-runtime' 包进行转化。
                      runtime: hasJsxRuntime ? 'automatic' : 'classic', // hasJsxRuntime：检查 `react/jsx-runtime` 模块是否存在
                    },
                  ],
                ],
                // @remove-on-eject-begin ~ @remove-on-eject-end、@remove-file-on-eject 与 `react-scripts eject` 命令弹出配置项有关，这里不做详细解释
                // @remove-on-eject-begin
                babelrc: false, // 禁用 .babelrc
                configFile: false, // 禁用 configFile 文件
                // 缓存唯一标识符，标识符更改时强制缓存失效
                // 默认由 @babel/core 版本号 + babel-loader 版本号 + .babelrc 文件内容 + 环境变量（BABEL_ENV || NODE_ENV）组成，这里使用 react-scripts 和 babel-preset-react-app 版本号替代，`react-scripts eject` 会删除这部分内容。
                cacheIdentifier: getCacheIdentifier(
                  // getCacheIdentifier：require('react-dev-utils/getCacheIdentifier')
                  isEnvProduction
                    ? 'production'
                    : isEnvDevelopment && 'development',
                  [
                    'babel-plugin-named-asset-import',
                    'babel-preset-react-app',
                    'react-dev-utils',
                    'react-scripts',
                  ]
                ),
                // @remove-on-eject-end
                // 一组插件。插件分为语法插件（语法解析）和转换插件（代码转化）。
                plugins: [
                  [
                    // 非 js/css 模块可显式命名导入。目前 create-react-app 中仅适用于 svg 资源
                    // 具体可查看 https://github.com/facebook/create-react-app/issues/3722
                    require.resolve('babel-plugin-named-asset-import'),
                    {
                      loaderMap: {
                        // svg 模块作为 React 组件导入：
                        // import { ReactComponent as Logo } from './logo.svg';
                        // const Header = () =>  <div><Logo /><div>;
                        svg: {
                          ReactComponent:
                            '@svgr/webpack?-svgo,+titleProp,+ref![path]',
                        },
                      },
                    },
                  ],
                  // 支持 react-refresh。一种模块热替换（HMR）方案，用于替代 react-hot-loader
                  isEnvDevelopment &&
                    shouldUseReactRefresh &&
                    require.resolve('react-refresh/babel'),
                ].filter(Boolean),
                // 这是 webpack 的 `babel-loader` 的一个特性（不是 Babel）。
                // 它在 ./node_modules/.cache/babel-loader/ 目录缓存结果，加快重新编译过程。
                cacheDirectory: true,
                // 使用 Gzip 进行缓存压缩。禁用原因 https://github.com/facebook/create-react-app/issues/6846，简言之对大多数项目并不会产生明显收益，除非需要转换上千个文件。
                cacheCompression: false, // 禁用
                // 紧凑模式，省略所有不必要的换行符和空格
                compact: isEnvProduction,
              },
            },
            // 使用 Babel 处理应用程序（src）之外的任何 JS。与应用 JS 不同，这里只编译标准的 ES 特性。
            {
              test: /\.(js|mjs)$/,
              exclude: /@babel(?:\/|\\{1,2})runtime/,
              loader: require.resolve('babel-loader'),
              options: {
                babelrc: false,
                configFile: false,
                compact: false,
                presets: [
                  [
                    // https://github.com/facebook/create-react-app/blob/main/packages/babel-preset-react-app/dependencies.js
                    require.resolve('babel-preset-react-app/dependencies'),
                    { helpers: true },
                  ],
                ],
                cacheDirectory: true,
                // 使用 Gzip 进行缓存压缩。禁用原因 https://github.com/facebook/create-react-app/issues/6846
                cacheCompression: false,
                // @remove-on-eject-begin
                cacheIdentifier: getCacheIdentifier(
                  isEnvProduction
                    ? 'production'
                    : isEnvDevelopment && 'development',
                  [
                    'babel-plugin-named-asset-import',
                    'babel-preset-react-app',
                    'react-dev-utils',
                    'react-scripts',
                  ]
                ),
                // @remove-on-eject-end

                // 调试 node_modules 代码需要 Babel sourcemaps。如果没有下面的选项，像 VSCode 这样的调试器会显示不正确的代码并在错误的行上设置断点。👇
                // sourceMaps 为 true 表示为代码生成 sourceMaps。
                sourceMaps: shouldUseSourceMap,
                // inputSourceMap 为 true 表示如果文件包含 `//# sourceMappingURL=...` 注释，将尝试加载 source map，加载或解析失败将丢弃。
                inputSourceMap: shouldUseSourceMap,
              },
            },
            // postcss-loader 将 autoprefixer（添加浏览器兼容前缀） 应用到 css。
            // css-loader 解析 css 中的路径（@import、url()）并将资产添加为依赖项。
            // style-loader 将 css 转换为注入 <style> 标签的 js 模块。
            // 生产环境使用 MiniCSSExtractPlugin 来提取 css 为独立的样式文件，但在开发环境下 css-loader 可以对 css 进行 HMR。
            // 默认情况下，我们支持扩展名为 .module.css 的 CSS Modules
            {
              test: cssRegex,
              // 排除扩展名为 .module.css 的 CSS Modules，交由👇处理
              exclude: cssModuleRegex,
              use: getStyleLoaders({
                // 表示配置在 `css-loader` 之前的 loader，有几个可以去处理 `@import` 资源（如 `@import 'a.css';`）。此配置中 1 表示 `@import` 进来的资源可以经过 `postcss-loader`
                importLoaders: 1, // 1 => postcss-loader
                // 生成 SourceMap
                sourceMap: isEnvProduction
                  ? shouldUseSourceMap
                  : isEnvDevelopment,
                // 启用 CSS Modules
                modules: {
                  // compileType：输入样式的编译级别。可选值 `module | icss`。
                  // - 'module'： 表示解析所有 CSS Modules 特性。
                  // - 'icss'： 表示只会编译低级别的可交互的 css 特性，即ICSS，相比标准的 CSS规范仅额外新增了两个伪类（ :import 和 :export）用于变量的导入和导出。
                  // webpack v4之前，css-loader 默认将 ICSS 特性应用于所有文件。
                  compileType: 'icss',
                },
              }),
              // Don't consider CSS imports dead code even if the
              // containing package claims to have no side effects.
              // Remove this when webpack adds a warning or an error for this.
              // See https://github.com/webpack/webpack/issues/6571
              // 声明模块具有副作用，避免 css 被 webpack tree shaking 去除而没进入生产包。
              sideEffects: true, // 业务代码在 loader 处声明
            },
            // 支持 CSS Modules (https://github.com/css-modules/css-modules)
            // 支持 .module.css 扩展名
            {
              test: cssModuleRegex,
              use: getStyleLoaders({
                importLoaders: 1,
                sourceMap: isEnvProduction
                  ? shouldUseSourceMap
                  : isEnvDevelopment,
                modules: {
                  compileType: 'module',
                  // 默认情况下 CSS Modules 使用内置函数来生成 classname，也可以指定自定义 getLocalIdent 函数的绝对路径。
                  // require('react-dev-utils/getCSSModuleLocalIdent')
                  getLocalIdent: getCSSModuleLocalIdent,
                },
              }),
            },
            // 支持 sass（使用 .scss 或 .sass 扩展名）。
            // 默认支持的 sass Modules 扩展名： .module.scss 或 .module.sass
            {
              test: sassRegex,
              exclude: sassModuleRegex,
              use: getStyleLoaders(
                {
                  // resolve-url-loader 的作用: 帮助 sass-loader 找到对应的 url 资源。
                  // saas 没有提供 url 重写的功能，所以相关的资源都必须是相对于输出文件（ouput），此 loader 设置于 sass-loader 之前就可以重写 url。
                  // 详见👉 https://www.webpackjs.com/loaders/sass-loader/#url-%E7%9A%84%E9%97%AE%E9%A2%98
                  importLoaders: 3, // 3 => postcss-loader, resolve-url-loader, sass-loader
                  sourceMap: isEnvProduction
                    ? shouldUseSourceMap
                    : isEnvDevelopment,
                  modules: {
                    compileType: 'icss',
                  },
                },
                'sass-loader'
              ),
              // Don't consider CSS imports dead code even if the
              // containing package claims to have no side effects.
              // Remove this when webpack adds a warning or an error for this.
              // See https://github.com/webpack/webpack/issues/6571
              // 声明模块具有副作用
              sideEffects: true,
            },
            // 支持 CSS Modules, 扩展名 .module.scss or .module.sass
            {
              test: sassModuleRegex,
              use: getStyleLoaders(
                {
                  importLoaders: 3,
                  sourceMap: isEnvProduction
                    ? shouldUseSourceMap
                    : isEnvDevelopment,
                  modules: {
                    compileType: 'module',
                    getLocalIdent: getCSSModuleLocalIdent,
                  },
                },
                'sass-loader'
              ),
            },
            // file-loader 将文件中的 import/require() 解析为 url，生产环境下它会将文件复制到输出文件夹（build）。
            // 没有test，所以会匹配所有的模块
            {
              loader: require.resolve('file-loader'),
              // 排除 js 文件以保持 css-loader 工作，因为它需要注入其运行时，否则将被 file-loader 处理。还要排除 `.html` 和 `.json` 扩展文件，以便它们能被 webpack 的内部加载器处理。
              exclude: [/\.(js|mjs|jsx|ts|tsx)$/, /\.html$/, /\.json$/],
              options: {
                name: 'static/media/[name].[hash:8].[ext]',
              },
            },
            // 如果你想添加新的 loader，确保在 file-loader 之前添加。
          ],
        },
      ].filter(Boolean),
    },
    plugins: [
      // 生成一个注入了 <script> 的 index.html 文件。
      new HtmlWebpackPlugin(
        Object.assign(
          {},
          {
            inject: true, // 注入打包后的脚本
            template: paths.appHtml, // html 模板
          },
          isEnvProduction
            ? {
                // 压缩选项
                // https://github.com/jantimon/html-webpack-plugin#options
                // 在线对比：https://kangax.github.io/html-minifier/
                minify: {
                  removeComments: true, // 移除注释
                  collapseWhitespace: true, // 折叠空白
                  removeRedundantAttributes: true, // 删除冗余属性
                  useShortDoctype: true, // 短文档类型，用简短的 (HTML5) doctype 替换 doctype
                  removeEmptyAttributes: true, // 删除空属性
                  removeStyleLinkTypeAttributes: true, // 删除 style 的类型属性（type="text/css"）
                  keepClosingSlash: true, // 保持单标签元素的尾部斜杠
                  minifyJS: true, // 压缩 js
                  minifyCSS: true, // 压缩 css
                  minifyURLs: true, // 压缩 url
                },
              }
            : undefined
        )
      ),
      // 将 webpack 运行时代码（runtime~xxx.[hash].js）内嵌到 html 中，减少js请求，并使用 `-` 替换 `~` 修复下面的bug。
      // https://github.com/facebook/create-react-app/issues/5358
      isEnvProduction &&
        shouldInlineRuntimeChunk &&
        new InlineChunkHtmlPlugin(HtmlWebpackPlugin, [/runtime-.+[.]js/]),
      // 使用环境变量填充模板 html 中的占位符，比如 <link rel="icon" href="%PUBLIC_URL%/favicon.ico">
      // 可以在 'package.json#homepage' 指定 PUBLIC_URL
      new InterpolateHtmlPlugin(HtmlWebpackPlugin, env.raw),

      // 为模块未找到错误提供了一些必要的上下文，例如请求资源
      new ModuleNotFoundPlugin(paths.appPath), // react-dev-utils/ModuleNotFoundPlugin

      // 设置一些环境变量，js 中可用，比如 if (process.env.NODE_ENV === 'production') { ... }
      // 生成构建期间需要将 NODE_ENV 设置为 'production'，否则 React 将在非常慢的开发模式下编译。
      new webpack.DefinePlugin(env.stringified),

      // 启用 HMR 必须使用的插件。使用后，其模块 api 将暴露在全局 module.hot 属性下，比如 module.hot.accept()。
      // https://webpack.js.org/api/hot-module-replacement/
      isEnvDevelopment && new webpack.HotModuleReplacementPlugin(),

      // react 热重载（hot reloading），基于 webpack HMR 的实现的功能（编辑运行中的 React 组件而不会丢失其状态）
      // https://github.com/facebook/react/tree/master/packages/react-refresh
      isEnvDevelopment &&
        shouldUseReactRefresh &&
        new ReactRefreshWebpackPlugin({
          overlay: {
            entry: webpackDevClientEntry, // 'react-dev-utils/webpackHotDevClient'
            // The expected exports are slightly different from what the overlay exports,
            // so an interop is included here to enable feedback on module-level errors.
            module: reactRefreshOverlayEntry, // 'react-dev-utils/refreshOverlayInterop'
            // 由于 create-react-app 提供自定义开发客户端（webpackDevClientEntry）和覆盖集成（reactRefreshOverlayEntry，overlay integration），因此可以去除 bundled 中 socket 处理逻辑。
            sockIntegration: false,
          },
        }),
        
      // 路径中输入错误大小写，Watcher 将无法正常工作，此插件开启大小写敏感
      // See https://github.com/facebook/create-react-app/issues/240
      isEnvDevelopment && new CaseSensitivePathsPlugin(),

      // 缺失依赖 `npm install` 后不需要重启 webpack 开发服务器
      // See https://github.com/facebook/create-react-app/issues/186
      isEnvDevelopment &&
        new WatchMissingNodeModulesPlugin(paths.appNodeModules),
      
      // 提取 css 为独立文件
      isEnvProduction &&
        new MiniCssExtractPlugin({
          // 类似于 webpackOptions.output 中同名选项，两个选项都是可选的
          filename: 'static/css/[name].[contenthash:8].css',
          chunkFilename: 'static/css/[name].[contenthash:8].chunk.css',
        }),

      // 将 webpack Manifest 数据（资产清单）提取 asset-manifest.json 文件。详见 WebpackManifestPlugin 👇
      new WebpackManifestPlugin({
        fileName: 'asset-manifest.json',
        publicPath: paths.publicUrlOrPath,
        generate: (seed, files, entrypoints) => {
          const manifestFiles = files.reduce((manifest, file) => {
            manifest[file.name] = file.path;
            return manifest;
          }, seed);
          const entrypointFiles = entrypoints.main.filter(
            fileName => !fileName.endsWith('.map')
          );

          return {
            files: manifestFiles,
            entrypoints: entrypointFiles,
          };
        },
      }),
      // 默认 webpack 会打包所有语言包。使用 IgnorePlugin 忽略 Moment.js 语言包，需要用户在代码中加载语言环境文件。
      // 如果使用 ContextReplacementPlugin 则可以选择打包指定的语言环境文件，此时不需要在代码中加载语言环境文件。例如：
      // new webpack.ContextReplacementPlugin(/moment[/\\]locale$/, /ja|it/)
      new webpack.IgnorePlugin(/^\.\/locale$/, /moment$/),

      // 生成要预缓存的资产列表，并注入到 Service Worker 文件中
      isEnvProduction &&
        fs.existsSync(swSrc) && // 检查 src/service-worker
        new WorkboxWebpackPlugin.InjectManifest({
          // 配置选项：https://developers.google.com/web/tools/workbox/reference-docs/latest/module-workbox-webpack-plugin.InjectManifest#InjectManifest
          swSrc, // src/service-worker
          dontCacheBustURLsMatching: /\.[0-9a-f]{8}\./,
          // 预缓存清单中排除资产
          exclude: [/\.map$/, /asset-manifest\.json$/, /LICENSE/],
          // 增加预缓存的默认最大大小 (2mb)，以减少延迟加载失败的可能性
          // See https://github.com/create-react-app-template/pwa/issues/13#issuecomment-722667270
          maximumFileSizeToCacheInBytes: 5 * 1024 * 1024,
        }),

      // TypeScript 类型检查
      useTypeScript &&
        new ForkTsCheckerWebpackPlugin({
          // 开启 TypeScript 检查器
          typescript: resolve.sync('typescript', {
            basedir: paths.appNodeModules,
          }),
          // 如果 true，则在 webpack 编译完成后报告问题，它不会阻止编译，仅在watch模式下使用
          async: isEnvDevelopment,
          // 检查语法错误
          checkSyntacticErrors: true,
          // 解析模块名称的模块
          resolveModuleNameModule: process.versions.pnp
            ? `${__dirname}/pnpTs.js`
            : undefined,
          // 解析 types 指令（/// <reference types="node" />）的模块？不敢肯定！
          resolveTypeReferenceDirectiveModule: process.versions.pnp
            ? `${__dirname}/pnpTs.js`
            : undefined,
          tsconfig: paths.appTsConfig,
          // 只报告匹配文件的错误
          reportFiles: [
            // This one is specifically to match during CI tests,
            // as micromatch doesn't match
            // '../create-react-app-template-typescript/template/src/App.tsx'
            // otherwise.
            '../**/src/**/*.{ts,tsx}',
            '**/src/**/*.{ts,tsx}',
            '!**/src/**/__tests__/**',
            '!**/src/**/?(*.)(spec|test).*',
            '!**/src/setupProxy.*',
            '!**/src/setupTests.*',
          ],
          // 静音模式？
          silent: true,
          // 格式化程序。开发环境下直接在 react-dev-utils/typescriptFormatter 调用 formatter
          formatter: isEnvProduction ? typescriptFormatter : undefined,
        }),

      // js 代码检查
      !disableESLintPlugin &&
        new ESLintPlugin({
          // Plugin options
          // 应检查的扩展名
          extensions: ['js', 'mjs', 'jsx', 'ts', 'tsx'],
          // 格式化程序
          formatter: require.resolve('react-dev-utils/eslintFormatter'),
          eslintPath: require.resolve('eslint'),
          // 如果有任何错误，将导致模块构建失败
          // const emitErrorsAsWarnings = process.env.ESLINT_NO_DEV_ERRORS === 'true';
          failOnError: !(isEnvDevelopment && emitErrorsAsWarnings),
          // 仅 src 检查
          context: paths.appSrc,
          // 使用缓存
          cache: true,
          // 缓存目录
          cacheLocation: path.resolve(
            paths.appNodeModules,
            '.cache/.eslintcache'
          ),
          // ESLint class options
          cwd: paths.appPath,
          resolvePluginsRelativeTo: __dirname,
          baseConfig: {
            extends: [require.resolve('eslint-config-react-app/base')],
            rules: {
              ...(!hasJsxRuntime && {
                'react/react-in-jsx-scope': 'error',
              }),
            },
          },
        }),
    ].filter(Boolean),

    // 一些库导入 Node 模块，但会在其他环境（如浏览器）中运行，告诉 webpack 为它们提供 empty 的模拟，此功能由 webpack 内部的 NodeStuffPlugin 提供。有四个选项可选：
    // - true：提供 polyfill。
    // - false：什么都不提供，代码可能会因为未定义而导致崩溃。
    // - 'mock'：提供 mock 实现预期接口，但功能很少或没有。
    // - 'empty'：提供空对象。
    node: {
      module: 'empty',
      dgram: 'empty',
      dns: 'mock',
      fs: 'empty',
      http2: 'empty',
      net: 'empty',
      tls: 'empty',
      child_process: 'empty',
    },

    // 配置性能提示，例如一个资源超过 250kb，webpack 会对此输出一个警告来通知你，create-react-app 在 `react-scripts build` 中 FileSizeReporter 做类似工作，所以关闭此选项。
    // FileSizeReporter：'react-dev-utils/FileSizeReporter'
    performance: false,
  };
};
```


## 详解

### mode

`'none' | 'development' | 'production'（默认）`。

表示启用对应环境的默认配置项，`none` 表示不使用任何默认优化选项。通过将 `DefinePlugin` 中 `process.env.NODE_ENV` 的值设置为 `development/production` 实现启动不同的默认优化配置。

> `production` 为模块和 chunk 启用确定性的混淆名称，FlagDependencyUsagePlugin，FlagIncludedChunksPlugin，ModuleConcatenationPlugin，NoEmitOnErrorsPlugin 和 TerserPlugin。

### entry

入口配置。

``` js
{
  entry: 
    isEnvDevelopment && !shouldUseReactRefresh
    ? [
        webpackDevClientEntry, // 替代默认的 WebpackDevServer 客户端
        paths.appIndexJs, // paths.appIndexJs 指向 src/index
      ]
    : paths.appIndexJs,
}
```

开发环境下且未使用 `react-refresh` 的情况下，`entry` 中除了程序入口，还包含了一个自定义 `WebpackDevServer` 客户端（即 `webpackDevClientEntry`），客户端作用是通过 socket 连接到 `WebpackDevServer` 并获得有关文件更改的通知，当你保存文件时，客户端将应用热更新或刷新页面，当你出现语法错误时也将显示错误。`webpackDevClientEntry` 替代了默认的 `WebpackDevServer` 客户端的替代品，相对体验更好。

``` js
// require.resolve：以同步的方式获取模块ID（模块完整路径）
const webpackDevClientEntry = require.resolve(
  'react-dev-utils/webpackHotDevClient'
);
```

如果使用默认的 `webpack-dev-server` 客户端：
``` js
{
  entry: [
    // 默认的 WebpackDevServer 客户端👇
    require.resolve('webpack-dev-server/client') + '?/',
    require.resolve('webpack/hot/dev-server'), // 👈这不是
    // 入口
    paths.appIndexJs, // paths.appIndexJs 指向 src/index。
  ]
}
```

启用 `react-refresh` 时，使用将由 `@pmmmwh/react-refresh-webpack-plugin` 插件负责注入开发客户端。

### bail

`boolean = false`。

是否发现错误就抛出失败结果退出编译，通常生产环境下开启，`development` 模式下启用 `HMR` 将在终端和浏览器控制台抛出错误。

### devtool

`string = 'eval'` `false`。

设置 `source map` 格式。可以使用 `SourceMapDevToolPlugin/EvalSourceMapDevToolPlugin` 插件来替换此选项，但不能与此选项同时使用。

``` js
{
  devtool: isEnvProduction
    ? shouldUseSourceMap
      ? 'source-map' // 映射到源代码，映射信息最完整但构建也最慢
      : false
    : isEnvDevelopment && 'cheap-module-source-map', // 映射到源代码，缺少列映射信息
}
```
选项说明：

- `eval`（默认）：不生成 `.map` 文件，将模块内容封装到 `eval` 函数里执行，并在末尾追加注释（`//# sourceURL=`）。优点是构建非常快，缺点是由于会映射到转换后的代码，而不是映射到原始代码，所以不能正确的显示行数。

``` js
eval("console.log(1);\n\n//# sourceURL=webpack://root/./src/index.js?");
```

- `source-map`：生成 `.map` 文件，映射到源代码，包含最完整的行列映射信息，bundle 尾部会添加一个引用注释 `//# sourceMappingURL=`（如下👇），以便开发工具能找到它。缺点构建慢，用于生产环境。

``` js
(() => {
  var __webpack_exports__ = {};
  console.log(1);
})();
//# sourceMappingURL=main.js.map
```

- `cheap-module-source-map`：与 `source-map` 相同，但缺少列映射(column mapping)，将 loader source map 简化为每行一个映射(mapping)，相比 `source-map` 构建速度稍快。

### optimization.splitChunks

> 从 webpack v4 开始，移除了 CommonsChunkPlugin，取而代之的是 optimization.splitChunks

`splitChunks` 默认配置：

``` js
splitChunks: {
  // 选择哪些 chunk 进行优化（拆分出共享模块）。有效值 'async'（默认值，异步，即按需加载 `import('moduleName')`）、'initial' （同步），'all' （同步 + 异步）
  chunks: 'async', // 异步，即只拆分按需加载模块中的共享模块
  // 生成 chunk 的最小体积（以 bytes 为单位）
  minSize: 20000,
  // 最小剩余体积（以 bytes 为单位）
  minRemainingSize: 0,
  // 拆分前必须共享模块的最小 chunks 数
  minChunks: 1,
  // 按需加载时的最大并行请求数
  maxAsyncRequests: 30,
  // 入口点的最大并行请求数
  maxInitialRequests: 30,
  enforceSizeThreshold: 50000,
  // 缓存组。可以继承或覆盖来自 `splitChunks.*` 的任何选项。test、priority、reuseExistingChunk 只能在 `cacheGroups` 中设置
  // 下面是两个默认的缓存组，即默认分割出两个chunks：vendors（第三方）和 commons （多入口共享的公共模块）👇
  cacheGroups: {
    // 默认缓存组：vendor（来自 node_modules）
    defaultVendors: {
      // 匹配模块
      test: /[\\/]node_modules[\\/]/,
      // 设置优先级。一个模块可以属于多个缓存组，优化将考虑优先级更高的缓存组。
      // 自定义缓存组的优先级默认值为 0。
      priority: -10,
      // 是否复用已被拆分的模块。true 表示复用，不再生成新的模块
      reuseExistingChunk: true,
    },
    // 默认缓存组（多入口共享的公共模块）
    default: {
      // 拆分前必须共享模块的最小 chunks 数
      minChunks: 2,
      priority: -20,
      reuseExistingChunk: true,
    },
  },
  /* 其他选项👇 */
  // 拆分 chunk 的名称。名称与 entry 中相同，entry 中将被删除
  // name: isEnvDevelopment, // `false` 表示不会更改名称（生产环境推荐）
  // 指定用于生成名称的分隔符，默认 `~`。默认情况下，webpack 将使用 chunk 的来源和名称生成名称（例如 `vendors~main.js`）
  // automaticNameDelimiter: '~',
}
```

### optimization.runtimeChunk

将 webpack `runtime` 代码提取出来，避免 runtime 代码注入业务代码后导致实际未修改的 bundle 文件名改变，导致浏览器缓存失效。

``` js
runtimeChunk: {
  // 自定义 runtimeChunk 名称
  name: entrypoint => `runtime-${entrypoint.name}`,
}
```

**runtime 与 manifest**

webpack 打包后的代码中包含三种主要的代码类型：你或你的团队编写的源码；源码中依赖的任何第三方模块；webpack 的 runtime 和 manifest，管理所有模块的交互。

> runtime：在浏览器运行时，webpack 用来连接模块化的应用程序的所有代码。runtime 包含：在模块交互时，连接模块所需的加载和解析逻辑。包括浏览器中的已加载模块的连接，以及懒加载模块的执行逻辑。

`runtime` 即保证模块正常运行的所有代码，比如 `__webpack_require__` 、`webpackJsonp` 函数等等。

`manifest` 是一个保存了所有模块的信息的数据集合。`runtime` 代码通过 `manifest` 数据来解析和加载模块。比如 `__webpack_require__(moduleId)`，`moduleId` 是一个模块标识符（module identifier），通过使用 `manifest` 中的数据，能够根据 `moduleId` 检索出背后对应的模块。

如果你使用了 content hash 作为文件名中的组成，这样在内容或文件修改时，打包后的文件名将改变，从而使浏览器缓存无效。但有时候发现代码没有修改，但打包后的文件名却变了，这是因为 `runtime` 和 `manifest` 代码的在每次构建都会发生变化，这些代码的注入了导致了文件名变化。所以为了更好的利用浏览器缓存优化项目性能，可以 `runtime` 和 `manifest` 的内容单独提取出来。这里提取 `runtime` 为 `runtimeChunk`， `manifest` 可通过 [`WebpackManifestPlugin`👇](#WebpackManifestPlugin) 提取生成单独的文件。由于 `manifest` 的体积很小，为减少网络请求也可以将 `manifest` 写入到 html 中。

### Rules.enforce

指定 loader 类型。可选值 `"pre" | "post"`，没有值表示是普通 loader。

``` js
{
  enforce: 'pre',
  exclude: /@babel(?:\/|\\{1,2})runtime/,
  test: /\.(js|mjs|jsx|ts|tsx|css)$/,
  use: 'source-map-loader',
}
```

loader 总共有四种：后置（`post`）、行内（`inline`，不推荐使用）、普通（`normal`，默认）、前置（`pre`）。<br/>
其中直接在 `import/require` 语句中调用的 loader 即为 `行内 loader`，比如:

``` js
// 使用 `!` 将资源中的 loader 分开
import Styles from 'style-loader!css-loader?modules!./styles.css';
```
*`css-loader` 后面的 `modules`为查询参数 ，表示开启 CSS Modules 功能*

**loader 调用顺序：**

模块解析时将一个接一个地进入的 loader，过程分为两个阶段：

- `Pitching` 阶段：调用 loader 上的 `pitch` 方法，按照 后置（`post`）、行内（`inline`）、普通（`normal`）、前置（`pre`） 的顺序调用.
- `Normal` 阶段: loader 上的常规方法，按照 前置（`pre`）、普通（`normal`）、行内（`inline`）、后置（`post`）的顺序调用。模块源码的转换， 发生在这个阶段。

以下面配置为例：
``` js
 rules: [
  {
    //...
    use: ['a-loader', 'b-loader', 'c-loader'],
  },
],
```
将会发生这些步骤：
```
|- a-loader `pitch`
  |- b-loader `pitch`
    |- c-loader `pitch`
      |- requested module is picked up as a dependency
    |- c-loader normal execution
  |- b-loader normal execution
|- a-loader normal execution
```
### WebpackManifestPlugin

生成资产清单文件。webpack 打包后的代码中除了第三方模块和应用程序代码，还包含 runtime 和 manifest 代码。`manifest` 是一个保存了所有模块的信息的数据集合。它的体积很小，但十分重要，`runtime` 代码通过 `manifest` 数据来解析和加载模块，`HtmlWebpackPlugin` 也会利用 `manifest` 数据来重建 html。

由于只要有模块修改都会导致 manifest 内容变化，而 manifest 注入到 bundle 中会导致为实际无修改的 bundle 的 content hash 更改，但从而使得浏览器缓存失效，所以通常需要把 `manifest` 的内容提取出来。为了与[Web app manifests](https://developer.mozilla.org/en-US/docs/Web/Manifest) 或 [WebExtension manifests](https://developer.mozilla.org/en-US/docs/Mozilla/Add-ons/WebExtensions/manifest.json) 区分开来，输出文件名默认为 `asset-manifest.json`。

资产清单文件包含以下内容：
- `files`：将所有资产文件名到其对应的输出文件的映射，以便解析和加载模块。
- `entrypoints`：包含在 `index.html` 中的文件数组，可用于重建 HTML。

示例：

``` json
// asset-manifest.json
{
  "files": {
    "main.css": "/static/css/main.6dea0f05.chunk.css",
    "main.js": "/static/js/main.42e4b4ff.chunk.js",
    "main.js.map": "/static/js/main.42e4b4ff.chunk.js.map",
    "runtime-main.js": "/static/js/runtime-main.3a54720d.js",
    "runtime-main.js.map": "/static/js/runtime-main.3a54720d.js.map",
    "static/js/2.edbe677d.chunk.js": "/static/js/2.edbe677d.chunk.js",
    "static/js/2.edbe677d.chunk.js.map": "/static/js/2.edbe677d.chunk.js.map",
    "static/js/3.e512cae8.chunk.js": "/static/js/3.e512cae8.chunk.js",
    "static/js/3.e512cae8.chunk.js.map": "/static/js/3.e512cae8.chunk.js.map",
    "index.html": "/index.html",
    "static/css/main.6dea0f05.chunk.css.map": "/static/css/main.6dea0f05.chunk.css.map",
    "static/js/2.edbe677d.chunk.js.LICENSE.txt": "/static/js/2.edbe677d.chunk.js.LICENSE.txt"
  },
  "entrypoints": [
    "static/js/runtime-main.3a54720d.js",
    "static/js/2.edbe677d.chunk.js",
    "static/css/main.6dea0f05.chunk.css",
    "static/js/main.42e4b4ff.chunk.js"
  ]
}
```

### `getStyleLoaders`

``` js
const getStyleLoaders = (cssOptions, preProcessor) => {
  const loaders = [
    // 开发环境，通过 <style> 将样式插入到 <head>
    isEnvDevelopment && require.resolve('style-loader'),
    isEnvProduction && {
      // 生产环境下，将 css 提取出来生成单独的 css 文件
      loader: MiniCssExtractPlugin.loader,
      // css 位于 `static/css` 中，使用 '../../' 来定位 index.html 文件夹。
      // 在生产中 `paths.publicUrlOrPath` 可以是相对路径
      options: paths.publicUrlOrPath.startsWith('.')
        ? { publicPath: '../../' }
        : {},
    },
    {
      loader: require.resolve('css-loader'),
      options: cssOptions,
    },
    {
      // PostCSS 的选项。
      // 可根据 `package.json#browserslist` 中指定的浏览器支持添加兼容前缀。
      // "browserslist": ["> 0.5%", "last 2 versions", "Firefox ESR", "not dead", "IE 11", "not IE 10"],
      loader: require.resolve('postcss-loader'),
      options: {
        // PostCSS 的选项
        postcssOptions: {
          plugins: [
            // 修复 flexbox 相关的一些bug
            require('postcss-flexbugs-fixes'),
            [
              // 支持未来的 CSS 特性，postcss-preset-env 内部包含了 autoprefixer（添加兼容前缀）和 polyfills。
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
            postcssNormalize(),
          ],
        },
        sourceMap: isEnvProduction && shouldUseSourceMap,
      },
    },
  ].filter(Boolean);
  if (preProcessor) {
    loaders.push(
      {
        // 配合 sass-loader 使用，帮助其找到对应的 url 资源
        loader: require.resolve('resolve-url-loader'),
        options: {
          sourceMap: isEnvProduction ? shouldUseSourceMap : isEnvDevelopment,
          root: paths.appSrc,
        },
      },
      {
        // preProcessor 即 sass-loader
        loader: require.resolve(preProcessor),
        options: {
          sourceMap: true,
        },
      }
    );
  }
  return loaders;
};
```
### babel-preset-react-app

`create-react-app` 中使用的 babel 预设，使得 babel 支持 JSX、Flow、TypeScript、一些 ESnext 特性等。预设即为一组插件的集合。

**预设参数**（指定预设参数的方式：`["presetA", options]`）：

``` js
presets: [
  [
    require.resolve('babel-preset-react-app'),
    
    {
      runtime: hasJsxRuntime ? 'automatic' : 'classic',
    },
  ],
],
```

- `runtime`：表示 jsx 转换规则（使用哪个运行时）。

`classic` 是旧的转换方式，它将 jsx 转换为 `React.createElement` 调用，因此也存在一些缺陷：

>- 如果使用 JSX，则需在 React 的环境下，因为 JSX 将被编译成 `React.createElement`。
>
>- `React.createElement` 有一些无法做到的[性能优化和简化](https://github.com/reactjs/rfcs/blob/createlement-rfc/text/0000-create-element-changes.md#motivation)。

以转换 `<div></div>` 为例对比两种转换方式：

`classic`：
``` js
React.createElement('div', null)
```

`automatic`:
``` js
// 由编译器引入（禁止自己引入！）
import { jsx as _jsx } from "react/jsx-runtime";

_jsx("div", {});
```
可在 [https://new-jsx-transform.netlify.app/](https://new-jsx-transform.netlify.app/) 做更详细的对比。

**hasJsxRuntime** 则是检查 `react/jsx-runtime` 模块是否存在。

``` js
const hasJsxRuntime = (() => {
  if (process.env.DISABLE_NEW_JSX_TRANSFORM === 'true') {
    return false;
  }

  try {
    require.resolve('react/jsx-runtime');
    return true;
  } catch (e) {
    return false;
  }
})();
```

**`babel-preset-react-app/webpack-overrides`：**

``` js
const crypto = require('crypto');

const macroCheck = new RegExp('[./]macro');

module.exports = function () {
  return {
    // This function transforms the Babel configuration on a per-file basis
    // 这个函数在每个文件的基础上转换 Babel 配置
    config(config, { source }) {
      // Babel Macros are notoriously hard to cache, so they shouldn't be
      // https://github.com/babel/babel/issues/8497
      // We naively detect macros using their package suffix and add a random token
      // to the caller, a valid option accepted by Babel, to compose a one-time
      // cacheIdentifier for the file. We cannot tune the loader options on a per
      // file basis.
      if (macroCheck.test(source)) {
        return Object.assign({}, config.options, {
          caller: Object.assign({}, config.options.caller, {
            create-react-appInvalidationToken: crypto.randomBytes(32).toString('hex'),
          }),
        });
      }
      return config.options;
    },
  };
};
```
这里查看[源码](https://github.com/facebook/create-react-app/blob/main/packages/babel-preset-react-app/webpack-overrides.js)。

## 参考

- [split-chunks-plugin](https://webpack.docschina.org/plugins/split-chunks-plugin/)
- [Manifest & Runtime](https://www.webpackjs.com/concepts/manifest/)
- [fork-ts-checker-webpack-plugin](https://github.com/TypeStrong/fork-ts-checker-webpack-plugin)
- [create-react-app](https://create-react-app.dev/docs/getting-started)
- [babeljs](https://www.babeljs.cn/docs/presets)
- 其他文档