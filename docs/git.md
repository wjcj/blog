Git 基操、协作流程和常见状况处理
====

## 基本概念

![git](https://i.ibb.co/bH2Ynct/git.jpg)

- 工作区：电脑上能看到的目录。
- 版本库：工作区有一个隐藏目录 `.git` 就是 Git 的版本库，它不算工作区。
- 暂存区：版本库中标记为 `stage` 或 `index` 的区域就是缓存区，通常存放在 `.git` 目录下的 `index` 文件中。

上图中的 `master` 表示 master 分支，`HEAD` 是当前分支上的一个指针，表示当前版本（`.git/HEAD` 文件），Git 通过改变它的指向实现当前分支的不同版本的切换，`HEAD^` 表示当前分支的上一个版本。

它们之间的关联关系：

使用 `git add` 命令把工作区的文件修改更新到暂存区。
`git commit` 命令把暂存区的目录树写到版本库，准确来说是版本库中的对象库（`.git/objects` 目录），当前分支的 `HEAD` 指向的版本目录树也会更新。`git reset HEAD` 命令将暂存区的目录树重写为 `HEAD` 指向的目录树。

## 基本操作

下面罗列的是本文状况处理中可能用到的命令。`<>` 表示必填，`[]` 表示选填。

### 查看历史

``` bash
# 查看提交历史。只能查看到当前和落后于当前 commit_id 之前的提交历史
git log
# 查看命令操作历史，不仅仅是 git commit 命令
git reflog
```

### 回退或切换版本

``` bash
# 回退到上一个版本，十个 ^ 符号表示上十个版本
git reset --hard HEAD^
# 回退到上十个版本（使用数字表示）
git reset --hard HEAD~10
# 切换到指定 commit_id 提交的版本。不一定是回退，指定的 commit_id 可以领先于当前 commit_id
git reset --hard <commit_id>
```

### 脏工作目录（dirty working directory）

`git stash push` 命令（通常省略 `push`）的作用将暂存区的变更生成快照，并储存到脏工作目录（`.git/refs/stash`），这是一个缓存栈。此命令的好处是可以让工作区立即变干净，并且更变不会丢失。

``` bash
# 缓存变更至缓存栈。默认新建的文件和被忽略的文件（.gitignore）不能被缓存，-a 选项（--all）则均可被缓存；-u 选项（--include-untracked）则支持缓存新建的文件变更
git stash -a
# 查看缓存栈。stash@{0} 表示最近的缓存条目名称
git stash list
# 查看缓存条目，可指定要查看的条目名称
git stash show [stash]
# 弹出缓存条目应用到当前分支，可指定要弹出的条目名称
git stash pop [stash]
```

### 恢复

``` bash
# 恢复指定提交
git revert <commit_id>
# 恢复多个连续的提交
git revert <commit_id_start>..<commit_id_end>
```
`git reset` 和 `git revert` 命令都可以做到撤销某次 commit，但前者是真正的回退，撤销的 commit 不再可见，而`git revert` 是增加了一个反向修改的 `revert commit`，优点是保留了 commit 历史，所以通常情况下都建议使用 `git revert`。

针对这个新增的 commit 可以通过以下几个选项进行自定义：

- `--edit`: 会弹出编辑会话用于修改 commit 消息，编辑后提交。
- `--no-edit`: 自动创建 commit 消息，格式为 `Revert "${原 commit message}"`。
- `--no-commit`: 将撤销的 commit 改动放到暂存区，由用户自行提交。

### 分支操作

``` bash
# 选择分支，branch 表示分支名称
git checkout <branch>
# 新建并选择分支，new_branch 表示新分支名称，start_point 表示从该分支创建分支
git checkout -b <new_branch> [start_point] # 比如：git checkout -b hotfix/bug_1 master
# 删除本地分支
git branch -D <branch>
# 查看远程分支
git branch -a
# 删除远程分支。branch 表示远程分支名称
git push <remote> --delete <branch> # 比如：git push origin --delete fix/bug_1
```

### 变基

[git rebase](https://git-scm.com/book/zh/v2/Git-%E5%88%86%E6%94%AF-%E5%8F%98%E5%9F%BA) 翻译为变基。它和 `git merge` 都可用于整合来自不同分支的修改，区别在于它让分叉的提交变成一条直线。
![rebase_merge](https://i.ibb.co/R9wgXr9/rebase.png)

这里介绍的是 `git rebase` 对当前分支的 commit 进行删、改、合并操作。假设存在某分支的提交记录是：-> A -> B -> C -> D -> E -> F。

``` bash
# 选取一段提交历史，如果需要包含指定的commit，则添加 ^ 符号，表示指定commit_id的上一个版本
git rebase -i A^
# 可以使用区间的方式
git rebase -i A^..F
```

执行上面命令后会出现如下会话编辑面板。

``` bash
pick c369688 A
pick 7a51be7 B
pick fab39bf C
pick b1a5392 D
pick e5e8931 E

# Rebase 5279d51..e5e8931 onto 5279d51 (5 commands)
# Commands:
#  p, pick = 使用提交
#  r, reword = 使用提交，并且想重新定义 commit message
#  e, edit = 使用提交，但需要停下来修改提交内容
#  s, squash = 使用提交，但与上一个提交合并为一个提交
#  f, fixup = 类似于squash，但会丢弃此提交的 commit message
#  x, exec = 运行 shell 命令
#  d, drop = 删除提交
```
此时需要需要 vi 命令，它分`命令模式`、`输入模式`和`底线命令模式`。

`git rebase -i` 后进入的是命令模式，下面是常用指令：

- `gg`: 移动游标到第一行
- `G`: 移动游标到最后一行
- `dd`: 删除游标所在整行内容
- `dw`: 删除游标所在单词
- `yy`: 复制游标所在整行内容
- `yw`: 复制游标所在单词
- `u`: 复原
- `[Ctrl]+r`: 重做
- `.`: 重复上一个动作，常用于重复删除、重复粘贴
- 更多可查看 [这里](https://www.runoob.com/linux/linux-vim.html)

按 `i` 键进入输入模式，输入字符将 `pick` 下面进行更改：

``` bash
pick c369688 A
d 7a51be7 B # 删除 B 提交
s fab39bf C # C 与 A 合并为一个提交
d b1a5392 D # 删除 D 提交
pick e5e8931 E
```

按 `esc` 退出输入摸式，回退到命令模式。输入`:` 进行底线命令模式：

- `:wq`: 回车确认后保存并退出 vi
- `:wq!`: 回车确认后强制储存后退出 vi 
- `:w`: 回车后仅保存但不退出
- `:q!`: 回车后强制放弃修改并退出 vi
- 更多可查看 [这里](https://www.runoob.com/linux/linux-vim.html)

退出 vi 模式后，将自动进行删除合并等操作，如果期间发生异常或冲突，提示中会有如下几个命令选项：

- `git rebase --continue`: 解决合并冲突后，重新启动变基过程。
- `git rebase --abort`: 终止变基操作。
- `git rebase --skip`: 跳过合并失败的提交。该 commit 会被丢弃，慎用。
- 其他选项查看[这里](https://git-scm.com/docs/git-rebase#_options)

### 嫁接

``` bash
# 选择一个或多个提交到当前分支
git cherry-pick <commit_id>

# 选择多个提交
git cherry-pick commit_id1 commit_id3 commit_id5

# 选择某个区间范围内的提交，不包含 commit_id1
git cherry-pick commit_id1..commit_id3
# 包含了 commit_id1
git cherry-pick commit_id1^..commit_id3
```
以上命令均可添加 `--no-commit` 选项，表示将嫁接过来的提交放到暂存区。通常建议加上这个选项，方便查看或修改嫁接过来的修改，确认无误后再 `git commit`。

## Git Flow

### 分支类型

![git branch](https://nvie.com/img/git-model@2x.png)

- `master`: 主分支。用于发布和 tag 标记，不允许直接更改。
- `develop`: 开发分支。从 master 分支上检出，表示下一个版本，不允许直接更改，通常检出 feature 或 fix 分支进行更改。
- `release`: 发布分支，可以命名为 `release/*`。从 develop 分支上检出，用作发版前的测试分支。测试完成同时后合并至 develop 和 master 分支。
- `feature`: 功能分支，可以命名为 `feat/*`。通常从 develop 分支上检出，团队每个成员维护自己的 feature 分支并行开发功能，开发完成后合并至 develop 分支。
- `fix`: 修复分支，可以命名为 `fix/*`。从 develop 分支检出，用于修复 develop 分支上的 bug，修复完合并至 develop 分支。
- `hotfix`: 热修复分支，命名为 `hotfix/*`。从 master 分支检出，用于修复 master 分支的 bug。修复完同时合并至 master 和 develop 分支。

以上是**库**开发协作常用的分支模型。如果在**业务开发**中，因为存在很多突发性和不确定性，比如原定的下个版本内容中有几个功能需要提前上线，或是某功能突然取消或者需要延期。这些都是日常业务开发中十分常见的情况，所以可以做以下调整：

- `master`: master 分支与线上保持完全同步，如领先于线上则表示马上就要发布。
- `develop`: 分支作为发布分支，用于合并原定的功能和优化。如果遇到突发的功能发布，并且可以从 master 再检出一个 develop 分支，用于合并需要提前发布的功能或修复（`develop-*`）。
- `release`: release 分支不再必要，develop 也可以作为测试分支，测试完成后合并至 master。
- `feature`: feature 分支依然合并至 develop 分支，但必须从 master 分支检出，避免检出时携带了已合并至 develop 分支中的其他功能分支代码。合并时如发生冲突，应在本地 feature 分支上检出专门用于解决冲突的新分支，将该分支合并至 develop 分支。
- `fix` 和 `hotfix` 无需调整。

**团队规范：**这是一些团队协作的约定，所有意外状况的处理都应遵循的规则。

1. master 分支中的任何代码都应该是可部署的，且是马上就会发生部署。
2. 从 master 创建功能分支。
3. 注意 commit 粒度拆分，尽量小且独立。
4. 本地的功能分支或修复分支应定期推送到远程的同名分支，确保始终在远程存在备份。
5. 所有合并操作都需使用向目标分支发起 Pull Requests 的形式，在 Pull Requests 进行代码审查（Code Review）。
6. pull request 通过团队其他成员审查后才可以合并，使用评论功能提出错误或优化建议，全部修改后可以点赞表示通过审查。合并后会自动关闭 Pull Request。
7. Pull Request 合并到 master 后应立即部署。因为你如果不及时部署，下一次部署（可能会在几个小时内发生）会将它一起部署。期间其他成员还可能基于它创建新的功能分支。

## 常见状况处理

下面介绍日常开发中常见的状况及其解决方式。`<>` 表示必填，`[]` 表示选填。

### 场景1：文件已修改，但还未 `git add` 至缓存区，想要丢弃更改

修改但未提交到暂存区的文件为被跟踪（`tracked`）状态。

``` bash
# 丢弃所有更改
git checkout .
# 或者
git checkout -- .

# 丢弃指定文件或目录更改
git checkout -- <pathspec>
```

`git checkout` 不会丢弃新创建的目录/文件的更改，如果需要丢弃这些更改，则需要使用 `git clean` 命令：

``` bash
# 丢弃所有文件或目录更改。path 选项用于指定目录或文件路径。
git clean -df [path]

# -d 表示仅丢弃目录更改，-f 表示仅丢弃文件，-df 表示丢弃文件和目录。
git clean -d [path]
```

### 场景2：已 `git add` 至暂存区，但未 `git commit`

通常的场景是某个更改应该写到另一个分支，但不小心在当前分支提交到了暂存区。思路是先将其从暂存区退出，使其变为被跟踪态，这样可以保留变更。

``` bash
# 从暂存区退出所有文件
git reset HEAD .

# 从暂存区退出指定文件或目录文件
git reset HEAD <pathspec>
```

这时先把剩余的暂存区文件进行 `git commit` 提交，退出暂存区的文件现在都是被跟踪态，可以单独进行处理。比如需要转移到其他分支，见下面场景3：

### 场景3：未 git commit，但发现弄错了分支

先将所有更改的文件添加至暂存区，使用 `git stash` 将暂存区的文件存储至缓存栈（脏工作目录），此时工作区完全干净，可以切换正确的分支，然后弹出缓存的文件。

``` bash
# 将所有提交添加至暂存区
git add .

# 将暂存区文件生成快照条目存储至缓存栈（脏工作目录）
git stash -a

# 切换到正确的分支
git checkout <branch_name>

# 弹出条目并应用到当前分支。使用 stash 可指定条目，否则弹出最近存储的条目
git stash pop [stash]
```

### 场景4：已 `git commit` 后发现提交错了分支，还未 push 到远程

关键是需要保留住这次错误的提交，所以在当前分支检出一个新分支用于保留错误的 commit，当前分支回退即可。

``` bash
# 在当前分支检出新分支，此时并没有切换到新分支
git branch <new_branch> # 比如 git branch feat/1_copy

# 回退到上一个版本
git reset --hard HEAD~1

# 如果已经连续错误提交了多个 commit，则回退到指定的提交。通过 `git log` 查看历史 commit_id。
# git reset --hard <commit_id>

# 切换到新分支，提交错的代码将出现在新分支
git checkout <new_branch>
```

如果多次 commit 中夹杂着错误的提交，比如 `feat/1` 分支中的 C 和 E 本应该存在于 `feat/2`。

feat/1: A -> B -> C -> D -> E -> F

``` bash
# 处理错误时检出分支备份总是好的习惯
git branch feat/1_copy

# 选取变基操作的起始 commit。进入编辑会话后使用 vi 命令设置变基规则（剔除 C 和 E）
git rebase -i <start_commit_id> # git rebase -i commit_b^
# 或者选择一个区间
git rebase -i <start_commit_id>^..<end_commit_id> # 比如 git rebase -i commit_c^..commit_e
```
进行 vi 模式，按 `i` 键进入插入模式，将需要删除的 commit 对应的 `pick` 修改为 `drop`（简写`d`），完成后 `esc` 退出到插入模式，输入 `:wq` 保存并退出 vi 模式。

``` bash
pick c369688 B # 不操作 B，第一个必须是 pick
d 7a51be7 C # 删除 B
pick fab39bf D # 不操作
d b1a5392 E # 删除 D
pick e5e8931 F # 不操作 F
```

现在 `feat/1`  已完成 C 和 E 的剔除操作，对于 `feat/2` 只需要把 `feat/1_copy` 分支中的 C 和 E 嫁接过来即可。嫁接操作看接下来的场景5。

### 场景5：某一个或者多个 commit 被多个分支需要，但却提交到了某一个功能分支

如果仅两个分支（`brand_a`、`brand_b`），`brand_b` 也需要 `brand_a` 分支中的 A 和 B 两个 commit。则切换到 `brand_b`，嫁接 `brand_a` 中 A 和 B。<br/>
如果存在多个分支，则可以从 `master` 上检出一个干净分支 `brand_common`，用于嫁接 `brand_a` 中 A 和 B，其他分支 merge `brand_common` 即可。

``` bash
# 切换到 brand_a，使用 git log 查看需要嫁接的 commit_id 并记录下来
git checkout brand_a
git log # 记录 commit_a 和 commit_b，如果是区间只需要记录需嫁接的首位提交的commit_id

# 切换到 brand_b
git checkout brand_b

# 嫁接一个或多个 commit。推荐添加 --no-commit 选项，使得嫁接过来的 commit 将进入暂存区，确认无误后自行提交
git cherry-pick commit_a commit_b --no-commit
# 嫁接某范围内的所有 commit（包含了commit_a 和 commit_d）
git cherry-pick commit_a^..commit_d
```

### 场景6：某次错误的提交已 push 到远程，但还未合并

如果错误仅仅是个人的功能分支，可以直接如场景4中一样使用 `git reset --hard HEAD^` 回退到上一个版本，然后 `git push -f` 强推至远程即可。

如果该远程分支是还有其他成员在使用，他人可能已经 pull 或者从该分支检出了新分支，导致他人的分支携带了意外的 commit。此时使用 `git reset` 回退再强推将不再安全，因为它会使远程分支落后一个提交，他人分支在下次合并时仍会将错误的 commit 合并进去。而 `git revert` 命令会增加一个反向操作的 commit 提交，使得远程领先一个提交，其他成员在 push 或 merge 操作时会收到需要 pull 的提示。

``` bash
# revert最近的一次提交，--no-edit 表示不编辑 commit 信息
git revert HEAD --no-edit

# 将 revert commit push 到远程，远程同名分支上增加了一个 revert commit。
git push
# 如果是不是同名远程分支
git push origin <brand_name>

# 切换到正确的分支
git checkout <brand_name_other>

# 将该提交错误 commit 嫁接到正确的分支
git cherry-pick <commit_id>
```

### 场景7：某次错误的提交 push 远程，并且发起的 pull request 已合并

类似场景6，先使用 `git revert` 后 push 到远程，由于领先了一个提交，就可以重新发起 pull request 进行合并。

``` bash
# revert 最近一次提交
git revert HEAD --no-edit

# 将 revert commit push 到远程
git push

# 远程分支向目标分支重新发起 pull request，审查通过后进行合并
# ...
```

`git revert` 之后如想还原，查看 `revert` 生成的 revert_commit_id，再次 `git revert <revert_commit_id>`。

### 场景8: 代码意外丢失

找回的前提是进行过 `git add .` 或 `git commit` 操作。

``` bash
# 查看命令操作历史，找到并记录丢失代码对应的 commit_id
git reflog

# 切换至该 commit_id
git reset --hard <commit_id>

# 如果想嫁接commit
# 切换到某分支，嫁接 commit_id 到当前分支
git cherry-pick <commit_id>
```

## 参考

- [Git](https://git-scm.com/book/zh/v2)
- [A successful Git branching model](https://nvie.com/posts/a-successful-git-branching-model/)
- [GitHub Flow](https://githubflow.github.io/)
- [Git 误操作救命篇一： 如何将改动撤销？](https://zhuanlan.zhihu.com/p/42929114)
