npm-shrinkwrap.json vs package-lock.json
=======

## package-lock.json

`package-lock.json` 是整个包依赖树的快照文件，它描述了一个精确的、可重现的 `node_modules` 树。npm 利用它实现锁定包依赖项及其子依赖项，以保证后续安装可以生成 `node_modules` 树。<br/>
*[npm-shrinkwrap.json](#npm-shrinkwrap.json) 👇的功能和行为与 `package-lock.json` 类似，所以我们称这些文件包锁或锁文件。*

**什么场景下我们想要锁定依赖？**

我们或多或少遇到过这样的场景，来自同一仓库的相同代码，自己电脑里正常，但同事拉取后在他的电脑里运行却报错；或者自己本地调试正常，在CI上进行打包发布测试环境或者部署线上后却出现了bug。类似这些情况通常都是依赖树不一致导致的打包结果差异。

而我们不会提交 `node-modules` 目录，因为它太大了。但是有时候我们希望代码提交到仓库后，代码用于部署、CI/持续集成，或者团队中的任何人拉取后本地 `npm install` 都能得到完全相同的`node_modules` 树。这样一来只要本地做好充分的测试，即可很大程度上保证上述场景后运行结果一致。

这样也会导致无法享受到[语义化版本控制规范](https://semver.org/lang/zh-CN/) 带来的好处，比如当你安装的依赖包十分稳定可靠，在依赖版本号上正确使用限定规则（如`~1.2.2`、`^3.1.3`），集成部署时如果没有锁文件的存在，将会自动安装修订版本，发布线上后一些潜藏的隐患得以自动修复。相反的，这也可能带来更多不可控性，比如更新的版本可能带来新的bug，甚至是依赖树中的某个包发布没有遵守语义化版本控制规范，导致依赖方的自动更新了不符合语义期望。

如果你认为不能锁定依赖是一个问题，你就需要生成锁文件并把它提交到代码仓库。

**为何 `package.json` 不能锁定依赖树？**

最主要的原因在于 `package.json` 中依赖项都是顶级依赖，而且通常情况下依赖包的版本号都是不是精确版本，而是匹配了一个修订版本或是可兼容版本，例如（`~5.2.4`）。这时如果自上次安装依赖包之后发布了可以被匹配的新版本，npm 会自动使用新版本，例如（`~5.2.5`）；或者某依赖 a 之一子依赖项 b 可能已发布新版本，即便你对 a 使用固定版本号（比如 `1.2.3` 而不是 `^1.2.3`），该 b 版本还是会更新，因为除非你是 a 的作者，否则你没办法让 a 固定 b 的版本号。

**生成与更新 `package-lock.json` ？**

由于 `package-lock` 是 npm5 新增的特性，且因为 `>= 5.0.0 <=5.4.2` npm 版本关于 `package-lock.json` 使用和更新策略上都存在重大缺陷，详见 [issue #16866](https://github.com/npm/npm/issues/16866)、[issue #17979](https://github.com/npm/npm/issues/17979)。故推荐使用 `>=5.4.2` 的 npm 版本。

当项目中存在 `package.json` 时，执行 `npm install` 即可生成 `package-lock.json` 文件。

> any commands that update node_modules and/or package.json's dependencies will automatically sync the existing lockfile. This includes npm install, npm rm, npm update , etc.

官方[文档](https://docs.npmjs.com/cli/v6/configuring-npm/package-locks#using-locked-packages)中称任何更新 `node_modules` 和/或 `package.json` 依赖项的命令都会自动同步现有的锁文件。包括 `npm install`、`npm rm`、`npm update` 等。

更详细同步策略是：
- `package-lock.json` 不存在时，`npm install` 根据 `package.json` 安装依赖并创建 `package-lock.json`。
- `package.json` 中依赖项或使用 `npm install <PACKAGE>` 安装与 `package-lock.json` 中版本不兼容时 ，`npm install` 根据 `package-lock.json` 安装依赖项。
- 如手动修改 `package.json` 某依赖项并导致与 `package-lock.json` 锁定依赖版本不兼容时，`package-lock.json` 将更新到兼容 `package.json` 的版本。

## npm-shrinkwrap.json

`npm-shrinkwrap.json` 通过 `npm shrinkwrap` 命令手动创建，如果根目录已存在 `package-lock.json` 文件，则会将 `package-lock` 重命名为 `npm-shrinkwrap`。

如果你使用的是npm v5之前的版本，因为还不支持 `package-lock.json`，需要使用 `npm-shrinkwrap.json` 实现锁定依赖项的功能，所以它与 `package-lock.json` 具有相同的格式，执行类似的功能。唯一区别在于 `npm-shrinkwrap.json` 允许发布到 npm 上（设置`package.json#files` 字段将其设置为发布内容），而 `package-lock.json` 永远发被 npm 自动忽略。<br/>
如果 `package-lock.json` 和 `npm-shrinkwrap.json` 同时存在于根目录中时 `npm-shrinkwrap` 具有更高的优先级，`package-lock.json` 将被忽略。

> It's strongly discouraged for library authors to publish this file, since that would prevent end users from having control over transitive dependency updates.

虽然 `npm-shrinkwrap` 允许发布，但官方建议 “强烈不鼓励库作者发布此文件，因为这会阻止最终用户控制传递依赖项更新”。这里的目的是希望把选择权交给最终的包使用者，如果用户不希望锁定依赖树，发布此文件的包的依赖项中如果有可更新的依赖也将不能自动更新。

## 参考

- [shrinkwrap.json | npm Docs](https://docs.npmjs.com/cli/v6/configuring-npm/shrinkwrap-json)
- [package-locks | npm Docs](https://docs.npmjs.com/cli/v6/configuring-npm/package-locks)