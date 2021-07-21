Git 基操与常见状况处理
====

## 基本操作

## commit 规范

Header 部分只有一行，包括三个字段：type（必需）、scope（可选）和subject（必需）。

- type: 用于说明 commit 的类型。一般有以下几种:
  - feat: 新增feature
  - fix: 修复bug
  - docs: 仅仅修改了文档，如readme.md
  - style: 仅仅是对格式进行修改，如逗号、缩进、空格等。不改变代码逻辑。
  - refactor: 代码重构，没有新增功能或修复bug
  - perf: 优化相关，如提升性能、用户体验等。
  - test: 测试用例，包括单元测试、集成测试。
  - chore: 改变构建流程、或者增加依赖库、工具等。
  - revert: 版本回滚
- scope: 用于说明 commit 影响的范围，比如: views, component, utils, test...
- subject: commit 目的的简短描述

## 常见状况处理

## 参考
