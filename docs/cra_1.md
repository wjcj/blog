create-react-app 实现（上）
=====

[`create-react-app`](https://github.com/facebook/create-react-app) 项目内部使用 [lerna](https://www.lernajs.cn/) 管理了多个包:

```
.
├── packages
│   ├── create-react-app  // 全局命令 create-react-app
│   ├── react-scripts   // 脚本与配置
│   ├── cra-template // 基础模板
│   ├── cra-template-typescript // TypeScript 模板，添加 `--template typescript` 选项将使用此模板
├── package.json
├── lerna.json    // lerna 配置文件
├── ...
```

将多个软件包（package）存放在一个代码仓库中便于开发和管理，但也需要解决一些问题，比如包之间存在互相依赖，但 `npm link` 操作不便；package 之间有独立依赖的第三方依赖，但也存在共用依赖，如何管理各个 package 的依赖安装。这使用 `yarn workspace` 功能就可以解决（`leran`也可以）。而每个 package 需要做独立的 git 和发布管理（如版本号、CHANGELOG），这就需要使用到 `leran`。

## 创建项目、安装依赖

### 1. 创建 lerna 项目

安装 lerna 并初始化 lerna 项目：
```
npm install lerna -g
lerna init
```

此时目录下将自动创建 `package.json`，并新增了 `packages` 目录，执行下面命令，在 `packages` 目录下创建三个包。

``` bash
lerna create create-react-app
lerna create cra-template
lerna create react-scripts
```

*`cra-template` 包只是一个模板，直接复制官方 `cra-template` 即可。*

### 2. 开启 yarn workspaces 功能

在 `package.json` 新增 `workspaces` 字段：

``` json
{
  "workspaces": [
    "packages/*"
  ],
}
```

执行 `yarn install` 安装依赖并自动创建软链（`npm link`），此时包之间就可以互相 `require()` 了。

### 3. 安装依赖

通用依赖：

```
yarn add chalk cross-spawn fs-extra --ignore-workspace-root-check
```
`--ignore-workspace-root-check` 选项表示将上述依赖安装在根目录 `node_modules` 中（默认情况下 yarn 启动 `workspaces` 后，依赖是不允许安装在根目录中）。<br/>

特定 package 需要的依赖安装：

使用 `yarn workspace <packageName> add <moduleName>` 命令，比如：

``` bash
yarn workspace create-react-app add react react-dom -D
yarn workspace create-react-app add commander
```

`chalk` `cross-spawn` `fs-extra` `commander` 四个包的主要作用分别是改变终端输出的文字样式、开启子进程（可跨平台）和更简单易用的 fs 模块（文件系统操作）、解析命令行参数。详细了解可查看[快速了解 fs-extra、chalk、commander、cross-spawn](https://github.com/wjcj/blog/issues/31)。

## packages/create-react-app 包实现

`packages/create-react-app` 这个包是实现 `create-react-app <project-directory>` 命令的代码。

### 1. `bin` 字段与入口文件。

首先在 `packages/create-react-app/package.json` 中增加可执行文件配置，使得 `create-react-app` 命令执行 `index.js` 文件。

``` json
// packages/create-react-app/package.json
{
  "name": "create-react-app",
  "bin": {
    "create-react-app": "./index.js"
  },
  // ...
}
```

``` js
// packages/create-react-app/index.js
#!/usr/bin/env node

const { init } = require('./createReactApp');
init();
```
`bin` 字段中命令指向的可执行文件头部需要添加 `#!/usr/bin/env node`，告知系统使用 node 解析。<br/>

### 2. 解析命令行参数（`init` 方法）

这里我们仅实现 `create-react-app <project-directory>`，忽略 `--template [template-name]` `--use-npm` 等命令选项的实现。

下面代码中，通过 `Command` 声明了一个程序实例 `program`，通过 `.arguments` 方法给顶级命令 `create-react-app` 配置了一个必填的命令参数 `project-dicectory`（如果使用 `[]` 包裹则表示选填）；通过 `.action` 方法给该命令注册处理函数，函数参数与命令参数一一对应；最后使用 `.parse` 方法解析命令行参数(`process.argv`) 获取用户输入的指令，假如在命令行输入 `npx create-react-app my-app`，则命令参数 `project-dicectory`、`name` 的值为 `my-app`。

``` js
// packages/create-react-app/createReactApp.js
const chalk = require('chalk');
const { Command } = require('commander');
const packageJson = require('./package.json');

let projectName;
async function init() {
  const program = new Command(packageJson.name)
    .version(packageJson.version)
    .arguments('<project-dicectory>') // 配置命令参数
    // .usage(`${chalk.green('<project-dicectory>')}`) // 修改帮助信息中的首行提示信息
    .action(name => { // name 对应 命令参数 `project-dicectory`
      projectName = name;
    })
    // 配置 `--template [template-name]`、 `--use-npm` 选项 👇
    // .option('--template <path-to-template>', 'specify a template for the created project')
    // .option('--use-npm')
    // .allowUnknownOption() // 允许输入未知选项，默认情况下在在命令行上输入未知的选项会提示异常
    .parse(process.argv);

    await createApp(projectName); // 继续看👇
}

module.exports = {
  init,
}
```

### 3. 创建项目目录、写入 `package.json` 文件（`createApp` 方法）

上步骤中获取到了用户指定的项目目录，此步骤主要做了以下几件事：

1. 创建项目目录，并获取项目目录绝对路径。
2. 在项目目录下创建 `package.json` 文件。
3. 将 Node 进程工作目录改为项目目录。后续需要在此目录下开启子进程安装 `react`、`react-dom`、`cra-template`、`react-scripts`。

``` js
async function createApp(appName) {
  const root = path.resolve(appName); // 获取用户指定项目目录的绝对路径
  fs.ensureDirSync(appName); // 确保目录存在，否则创建

  console.log(`Creating a new React app in ${chalk.green(root)}.`);

  // 2. 创建 `package.json` 
  const packageJson = {
    name: appName,
    version: '0.1.0',
    private: true,
  };
  fs.writeFileSync(
    path.join(root, 'package.json'),
    JSON.stringify(packageJson, null, 2) // JSON.stringify(value[, replacer [, space]])
  );

  // 3. 改变 Node 进程工作目录
  const originalDirectory = process.cwd(); // 保存原始工作目录（项目目录的父目录）
  process.chdir(root); // 改变当前工作目录至项目目录

  // 4. 安装 `react`、`react-dom`、`cra-template`、`react-scripts` // 继续看👇
  await run(root, appName, originalDirectory);
};
```

### 4. 安装 `cra-template`、`react-scripts` (`run` 方法)

1. 安装 `cra-template`、`react-scripts`。`react-scripts` 含有初始化项目的 `init` 脚本，`cra-template` 则是项目模板，执行 `init` 脚本将复制 `cra-template` 的内容到项目目录。

2. `react-scripts` 安装完成后，执行 `init.js` 脚本完成项目构建。

``` js
async function run(root, appName, originalDirectory) {
  const scriptName = 'react-scripts';
  const templateName = 'cra-template';
  const allDependencies = ['react', 'react-dom', scriptName, templateName];

  // 1. 安装依赖
  await install(root, allDependencies);
  
  // 执行脚本所需参数，分别为项目根目录，项目名称，是否显示详细内容，原始目录，项目模板
  const data = [root, appName, true, originalDirectory, templateName]; 
  const source = `
    var init = require('react-scripts/scripts/init.js');
    init.apply(null, JSON.parse(process.argv[1]));
  `;
  // 2. 执行 require(react-scripts/scripts/init.js)(...data);
  await executeNodeScript({ cwd: process.cwd() }, data, source);
  process.exit();
};

// 安装依赖：执行 `yarnpkg add --exact <dependencies...> --cwd <command>`
async function install(root, allDependencies) {
  return new Promise((resolve, reject) => {
    const command = 'yarnpkg';
    const args = ['add', '--exact', ...allDependencies, '--cwd', root];
    const child = spawn(command, args, { stdio: 'inherit' });
    child.on('close', resolve);
  })
};
```

说明：

- `spawn` 方法开启一个异步子进程，执行 `yarnpkg add --exact <dependencies...> --cwd <command>` 命令。

### 5. 使用模板（`executeNodeScript` 方法）

``` js
async function executeNodeScript({ cwd }, data, source) {
  return new Promise((resolve, reject) => {
    // node --eval <scriptSource> -- [arguments]
    const child = spawn(
      process.execPath, // node 可执行路径
      ['-e', source, '--', JSON.stringify(data)],
      { cwd, stdio: 'inherit' },
    );
    child.on('close', resolve);
  })
};
```

`executeNodeScript` 方法中使用 `spawn` 开启子进程，使用 `node --eval <scriptSource> -- [arguments]` 命令。`scriptSource` 是调用 `react-scripts` 包中的 `init` 函数的代码字符串：

``` js
`var init = require('react-scripts/scripts/init.js');
init.apply(null, JSON.parse(process.argv[1]));`
```

其中 `-e` 是 `--eval` 的短选项名，`node -e <scriptSource>` 类似 `eval(scriptSource)`，`arguments` 是命令参数，也是 `init` 方法所需参数，通过 `process.argv[1]` 获取到。

`init` 函数中主要做的几件事：

1. 修改了 `package.json` 文件。
2. 复制模板文件。
3. 初始化 git 仓库。
4. 安装模板所需依赖。
5. 删除模板 `cra-template`。

具体实现请查看 [create-react-app 实现（中）](https://github.com/wjcj/blog/issues/30)，文中详细介绍了 `react-scripts` 中 `init`、`build`、`start` `eject` 命令的实现。
