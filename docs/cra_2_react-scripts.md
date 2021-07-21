create-react-app 实现（中）——— react-scripts
====

[create-react-app 实现（上）](https://github.com/wjcj/blog/issues/29) 介绍了 `create-react-app` 仓库中管理了多个包，并介绍了 `create-react-app/packages/create-react-app` 包的实现。本文则继续实现 `create-react-app/packages/react-scripts`。

``` json
{
  "name": "my-app",
  "scripts": {
    "start": "react-scripts start",
    "build": "react-scripts build",
    "test": "react-scripts test",
    "eject": "react-scripts eject"
  },
}
```

## 准备工作

```
yarn workspace react-scripts add react react-dom -D
yarn workspace react-scripts add cross-spawn fs-extra chalk webpack webpack-dev-server babel-loader babel-preset-react-app html-webpack-plugin open
```

### 项目结构

```
packages
├── create-react-app
├── cra-template
├── react-scripts
│   ├── bin
│   │   ├── react-scripts.js
│   ├── scripts
│   │   ├── init.js
│   │   ├── start.js
│   │   ├── build.js
│   │   ├── eject.js
│   ├── config
│   │   ├── paths.js
│   │   ├── webpack.config.js
│   │   ├── webpackDevServer.config.js
│   ├── package.json
```

### 其他文件

`package.json` 文件配置 `bin` 字段：

``` json
{
  "name": "react-scripts",
  "bin": {
    "react-script": "./bin/react-scripts.js"
  },
  // ...
}
```

**`./bin/react-scripts.js`：** 获取命令参数执行 `scripts` 目录中对应脚本文件。如 `react-scripts start`，即执行 `scripts/start.js`。

``` js
#!/usr/bin/env node

const spawn = require('cross-spawn');

const args = process.argv.slice(2);
const script = args[0]; // 如 'start'、'build'

spawn.sync(
  process.execPath, // node.js 可执行文件路径
  [require.resolve('../scripts/' + script)],
  { stdio: 'inherit' }
);
```
**`config/paths.js`：** 定义了一系列路径。

``` js
const path = require('path');
const appDirectory = process.cwd(); // 当前工作目录

const resolveApp = relativePath => path.resolve(appDirectory, relativePath);

module.exports = {
  appHtml: resolveApp('public/index.html'), // html-webpack-plugin
  appIndexJs: resolveApp('src/index.js'), // 默认的入口文件
  appBuild: resolveApp('build'), // 打包后的输出目录
  appPublic: resolveApp('public'), // 静态资源目录
  appSrc: resolveApp('src'), // app 工作目录
}
```

## init.js

`init()` 方法实际接受五个参数 `appPath, appName, verbose, originalDirectory, templateName`。这里为了简化实现过程固定使用 `yarn` 与 `cra-template`，只需要 `appPath` 和 `appName` 参数。

``` js
module.exports = function init(appPath, appName) {
  // ...
}
```

`cra-template` 包结构：

```
cra-template
├── package.json
├── README.md
├── template.json // 模板 package.json 部分字段配置，将合并到项目 package.json 中
├── template // 项目模板文件目录👇
│   ├── public
│   │   ├── favicon.ico
│   │   ├── index.html
│   │   ├── ...
│   ├── src
│   │   ├── App.js
│   │   ├── index.js
│   │   ├── ...
│   ├── gitignore // 项目模板所需 .gitignore 选项，拷贝到项目后重命名为 `.gitignore`
│   ├── README.md
```

### 1. 修改 package.json

在此之前，项目目录 `package.json` 的内容如下👇。此步骤将完善 `package.json` 字段并重新创建 package.json 文件。

``` json
// package.json
{
  "name": "my-app",
  "version": "0.1.0",
  "private": true,
  "dependencies": {
    "cra-template": "1.1.2",
    "react": "^17.0.2",
    "react-dom": "^17.0.2",
    "react-scripts": "4.0.3"
  }
}
```
`cra-template/template.json` 中 `package` 字段的内容项目模板所需依赖和 packageJson 配置。 

``` json
// cra-template/template.json
{
  "package": {
    "dependencies": {
      "@testing-library/jest-dom": "^5.11.4",
      "@testing-library/react": "^11.1.0",
      "@testing-library/user-event": "^12.1.10",
      "web-vitals": "^1.0.1"
    },
    "eslintConfig": {
      "extends": ["react-app", "react-app/jest"]
    }
  }
}
```

实现：

``` js
module.exports = function init(appPath, appName) {
  const templateName = 'cra-template';
  fs.existsSync(path.join(appPath, 'yarn.lock')); // 创建 yarn.lock

  // App package.json
  const appPackage = require(path.join(appPath, 'package.json'));

  // 获取项目模板 packageJson 配置：cra-template/template.json#package
  const templatePath = path.dirname(
    require.resolve(`cra-template/package.json`, { paths: [appPath] })
  );
  const templateJsonPath = path.join(templatePath, 'template.json');
  const templatePackage = require(templateJsonPath).package;

  // 合并 package.json 选项：dependencies、scripts、eslintConfig
  appPackage.dependencies = appPackage.dependencies || {};
  appPackage.scripts = Object.assign(
    {
      start: 'react-scripts start',
      build: 'react-scripts build',
      test: 'react-scripts test',
      eject: 'react-scripts eject',
    },
    templatePackage.scripts || {},
  );
  appPackage.eslintConfig = appPackage.eslintConfig;

  // 新增 package.json 配置项：browserslist
  appPackage.browserslist = {
    production: [
      '>0.2%',
      'not dead',
      'not op_mini all'
    ],
    development: [
      'last 1 chrome version',
      'last 1 firefox version',
      'last 1 safari version'
    ]
  };

  // 重新创建 package.json 文件
  fs.writeFileSync(
    path.join(appPath, 'package.json'),
    JSON.stringify(appPackage, null, 2) + os.EOL
  );
}
```

### 2. 拷贝项目模板

将模板文件（`cra-template/template`）拷贝至项目目录。

``` js
const templateDir = path.join(templatePath, 'template');
if (fs.existsSync(templateDir)) {
  fs.copySync(templateDir, appPath);
} else {
  console.error(
    `Could not locate supplied template: ${chalk.green(templateDir)}`
  );
  return;
}
```
### 3. 创建 git 仓库

将 gitignore 重命名为 `.gitignore`，执行 `git init` 初始化一个 git 仓库。

``` js
// 将 gitignore 重命名为 .gitignore
// https://github.com/npm/npm/issues/1862
fs.moveSync(
  path.join(appPath, 'gitignore'),
  path.join(appPath, '.gitignore'),
  []
);

initializedGit = false;
if (tryGitInit()) {
  initializedGit = true;
  console.log('Initialized a git repository.');
}

function tryGitInit() {
  try {
    execSync('git init', { stdio: 'ignore' });
    return true;
  } catch (e) {
    console.warn('Git repo not initialized', e);
    return false;
  }
};
```

### 4. 安装模板所需依赖

执行 `yarnpkg add <dependencies>` 命令安装模板项目依赖。

``` js
let args = ['add'];
// 获取需安装的依赖项
const dependenciesToInstall = Object.entries({
  ...templatePackage.dependencies,
  ...templatePackage.devDependencies,
});

if (dependenciesToInstall.length) {
  args = args.concat(
    dependenciesToInstall.map(([dependency, version]) => {
      return `${dependency}@${version}`;
    })
  );
}
console.log(`Installing template dependencies using yarnpkg...`);

const proc = spawn.sync('yarnpkg', args, { stdio: 'inherit' });
if (proc.status !== 0) {
  console.error(`yarnpkg \`${args.join(' ')}\` failed`);
  return;
}
```

### 5. 删除模板

执行 `yarnpkg remove cra-template` 命令移除项目模板 npm 包（`cra-template`）。

``` js
const proc = spawn.sync('yarnpkg', ['remove', templateName], {
  stdio: 'inherit',
});
if (proc.status !== 0) {
  console.error(`yarnpkg \`${args.join(' ')}\` failed`);
  return;
}
```

### 6. git 提交

git 仓库初始化成功的话，最后尝试创建一次 commit 提交，否则移除 `.git` 目录，交由用户初始化。

``` js
if (initializedGit && tryGitCommit(appPath)) {
  console.log('Created git commit.');
}

console.log(chalk.cyan('  cd'), appName);
console.log(`  ${chalk.cyan(`yarn start`)}`);

function tryGitCommit(appPath) {
  try {
    execSync('git add -A', { stdio: 'ignore' });
    execSync('git commit -m "Initialize project using Create React App"', {
      stdio: 'ignore',
    });
    return true;
  } catch (e) {
    console.warn('Git commit not created', e);
    console.warn('Removing .git directory...');
    try {
      fs.removeSync(path.join(appPath, '.git'));
    } catch (removeErr) {
      // Ignore.
    }
    return false;
  }
};
```

提示 `cd my-app` `yarn start` 运行项目。

``` js
console.log(chalk.cyan('  cd'), appName);
console.log(`  ${chalk.cyan(`yarn start`)}`);
```

## build.js

```
yarn workspace react-scripts add html-webpack-plugin @babel/preset-react
```
### 1. 获取 webpack 配置

`react-scripts build` 使用 webapck 打包，并从 `config/webpack.config.js` 获取 webpack 配置，该模块导出了一个根据环境类型返回 webpack 配置的函数。

由于其中的配置项复杂 且不在本文中讨论范围内，所以这里先只导出一个简单的配置👇，想详细可以去查看官方[源码](https://github.com/facebook/create-react-app/blob/master/packages/react-scripts/config/webpack.config.js)，也可以阅读 [create-react-app 实现（下）](https://github.com/wjcj/blog/issues/33)

``` js
const HtmlWebpackPlugin = require('html-webpack-plugin');
const paths = require('./paths');

module.exports = function(webpackEnv) {
  const isEnvDevelopment = webpackEnv === 'development';
  const isEnvProduction = webpackEnv === 'production';

  return {
    mode: isEnvProduction ? 'production' : isEnvDevelopment && 'development',
    entry: paths.appIndexJs,
    output: {
      path: paths.appBuild,
      publicPath: '/'
    },
    module: {
      rules: [
        {
          test: /\.(js|jsx)$/,
          include: paths.appSrc,
          use: [
            {
              loader: 'babel-loader',
              options: {
                presets: ['@babel/preset-react']
              }
            }
          ]
        }
      ]
    },
    plugins: [
      new HtmlWebpackPlugin({
        inject: true,
        template: paths.appHtml,
      })
    ]
  }
};
```

**`scripts/build.js`：**

``` js
const fs = require('fs-extra');
const chalk = require('chalk');
const webpack = require('webpack');
const paths = require('../config/paths');

process.env.NODE_ENV = 'production'; // 设置环境变量为生产环境

// 2. 获取webpack配置
const configFactory = require('../config/webpack.config');
const config = configFactory('production');
```

### 2. 清空输出目录，复制资源文件

``` js
// `scripts/build.js`
const fs = require('fs-extra');
const paths = require('../config/paths');
// ... 
fs.emptyDirSync(paths.appBuild); // 清空 build 目录

copyPublicFolder(); // 拷贝 public 静态资源到 build 目录
function copyPublicFolder() {
  fs.copySync(paths.appPublic, paths.appBuild, {
    // index.html 交由webpack插件处理，不需要拷贝
    filter: src => src !== paths.appHtml, 
  })
};
```
每次打包前清空输出目录后把项目 `public` 目录下的资源文件拷贝到输出目录，其中模板 html 文件需要交给 `html-webpack-plugin` 插件自动注入 bundle。

### 3. webpack 编译

传入 `webpack` 配置项获取 `Compiler` 实例，执行 `run()` 方法完成编译，更完备的错误处理可查看[](https://www.webpackjs.com/api/node/#%E9%94%99%E8%AF%AF%E5%A4%84%E7%90%86-error-handling-)。

``` js
// `scripts/build.js`
const webpack = require('webpack');
const chalk = require('chalk');

build();

function build() {
  const compiler = webpack(config);
  compiler.run((err, stats) => {
    if (err) {
      console.error(err.stack || err);
      return;
    }
    // stats 描述对象，描述本次打包的结果
    const info = stats.toJson();
    if (stats.hasErrors()) {
      console.error(info.errors);
    }
    console.log(chalk.green('Compiled successfully!'));
  });
};
```

## start.js

`react-scripts start` 命令的核心逻辑也很简单，即创建 `compiler` 实例，通过 `new WebpackDevServer(compiler, serverConfig)` 创建本地开发服务器

`serverConfig` 就是 webpack `devServer` 配置项。`react-scripts` 将这部分配置放到独立的 [config/webpackDevServer.config.js](https://github.com/facebook/create-react-app/blob/master/packages/react-scripts/config/webpackDevServer.config.js) 文件中。

这里为了简化只开启一个热更新的功能👇，更多配置可查看[文档](https://webpack.docschina.org/configuration/dev-server/)。

``` js
// config/webpackDevServer.config.js
module.exports = function() {
  return {
    hot: true
  }
};
```

1. 设置环境变量
2. 得到一个配置工厂
3. 创建 compiler
4. 获取 webpackDevServer 配置项
5. 启动http开发服务器，监听端口

``` js
const webpack = require('webpack');

// 1. 设置环境变量
process.env.NODE_ENV = 'development';

// 2. 获取 webpack 配置
const configFactory = require('../config/webpack.config');
const config = configFactory('development');

// 3. 获取 webpackDevServer 配置项
const serverConfig = require('../config/webpackDevServer.config')();

// 4. 创建 compiler 实例
const compiler = webpack(config);

// 5. 创建开发服务器，监听端口
const webpackDevServer = require('webpack-dev-server');
const devServer = new webpackDevServer(compiler, serverConfig);

devServer.listen(3000, () => {
  console.log(chalk.cyan('Starting the development server ...'))
});
```
