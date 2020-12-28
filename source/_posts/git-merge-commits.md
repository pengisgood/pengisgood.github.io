---
title: Git Merge Commits
date: 2020-12-09 14:25:30
tags: [Git]
categories: [Productivity]
---

![Git Branch & Commits](/images/2020-12-09/git-merge-commits.png)

平时在 Git 的使用过程中，遇到了好些场景需要把多个 Commit 合并成一个 Commit。经过在网上一通搜索，大家也是仁者见仁，智者见智。在本文中总结了几种常见场景，如果大家还遇到过其他的场景，也欢迎留言讨论。

<!-- more -->

## 场景一：把最新的改动合并本地还未提交到远端的 commit 上

先来热一下身，看一个最简单的场景。C1、C2 已经被提交到远端仓库，C3 已提交到本地仓库，现在想要把还未提交到本地的 C4 和 C3 合并。

![场景一](/images/2020-12-09/场景一.png)

步骤：
``` shell
git add .

# 按照提示修改 Commit message，如果不用修改，可以直接加上 --no-edit
git commit --amend
```

## 场景二：合并本地还未提交到远端的 commit

C1、C2 已经被提交到远端仓库，C3、C4 已提交到本地仓库，现在想要把 C3 和 C4 合并。

![场景二](/images/2020-12-09/场景二.png)

### 方法一：使用 `rebase -i`，基于 SHA 值

步骤：
``` shell
# f944f36e 为 C2 的 SHA 值
git rebase -i f944f36e
```
执行 `rebase -i` 操作后会看到类似如下的界面（**注意这里 Commit 的顺序**）：
``` shell
  1 pick 7d1cc37 C3
  2 pick 34f88df C4
  3 
  4 # Rebase f944f36..34f88df onto f944f36 (2 commands)
  5 #
  6 # Commands:
  7 # p, pick <commit> = use commit
  8 # r, reword <commit> = use commit, but edit the commit message
  9 # e, edit <commit> = use commit, but stop for amending
 10 # s, squash <commit> = use commit, but meld into previous commit
 11 # f, fixup <commit> = like "squash", but discard this commit's log message
 12 # x, exec <command> = run command (the rest of the line) using shell
 13 # b, break = stop here (continue rebase later with 'git rebase --continue')
 14 # d, drop <commit> = remove commit
 15 # l, label <label> = label current HEAD with a name
 16 # t, reset <label> = reset HEAD to a label
 17 # m, merge [-C <commit> | -c <commit>] <label> [# <oneline>]
...
```
然后根据自己的需求对每个 Commit 选择对应的操作，这里我们对 C3 选择 `pick`，对 C4 以及后面的 Commit（如果有的话）选择 `squash` 或者 `fixup` 都可以，然后保存退出 vi 模式即可，C3 后面的 Commit 都会被合并到 C3 中。这里通过灵活运用不同的操作，可以改变提交顺序，提交信息，合并提交，放弃提交等操作。

### 方法二：使用 `rebase -i`，基于 HEAD 的相对位置

该方法是基于 HEAD 的相对位置，适合合并少量的 Commit。如果想要看到2个 Commit，就写 `HEAD~2`。

步骤：
``` shell
git rebase -i HEAD~2
```
其他的步骤和方法一相同。方法一和方法二的详细解释可以参考这个[回答](https://stackoverflow.com/a/5189600/3049524)。

### 方法三：使用 Soft Reset 和 Commit

步骤：
``` shell
git reset --soft HEAD~2
git commit -m 'your message'
```

如果想要复用之前的提交信息，可以用这个命令 `git commit --edit -m"$(git log --format=%B --reverse HEAD..HEAD@{1})"`。更详细的解释，可以参考这个[回答](https://stackoverflow.com/a/5201642/3049524)。

### 方法四：使用 Hard Reset 和 Merge Squash

> **注意：** 在采用该方式之前，先确保所有的改动都已经提交，因为 Hard Reset 会直接丢掉 Staged 和 Unstaged 的改动。

步骤：
``` shell
# Reset the current branch to the commit just before the last 2:
git reset --hard HEAD~2

# HEAD@{1} is where the branch was just before the previous command.
# This command sets the state of the index to be as it would just
# after a merge from that commit:
git merge --squash HEAD@{1}

# Commit those squashed changes.  The commit message will be helpfully
# prepopulated with the commit messages of all the squashed commits:
git commit
```

在执行 Merge 的时候可能会遇到一个错误：`fatal: refusing to merge unrelated histories`。这是因为默认情况下，Git 是不允许在不同的项目之间执行 Merge 动作的。但是默认不允许不代表不能做，可以通过加上参数 `--allow-unrelated-histories` 强制执行。具体的解释可以参考这个[回答](https://stackoverflow.com/a/37938036/3049524)。

这种方式的好处是，默认会自动带出之前的详细提交信息。如果最后需要重写提交信息，也可以直接 `git commit -m 'your message'`。更详细的解释，可以参考这个[回答](https://stackoverflow.com/a/5190323/3049524)。

## 场景三：合并本地提交和已经 Push 到远端的提交

其实该场景和场景二并没有太大的区别，唯一需要特别注意的是 Force Push（`git push -f`），如果 Force Push 不成功，可以检查一下远端仓库是否设置的有分支保护策略。有可能覆盖掉其他人的提交的潜在风险，在 Push 之前需要再三确认远端没有比本地更新的提交记录。操作步骤和场景二中的类似，这里就不再赘述。

## 场景四：将所有的提交记录合并为一个 Commit

![场景四](/images/2020-12-09/场景四.png)

先介绍一下这个场景产生的背景。有一个已经开发了一段时间的项目，从某个时间点开始，需要不定期把之前一段时间的所有提交合并成一个提交并且规整提交信息为特定的格式，然后 Push 到远端的另一个仓库中，而且这个操作在将来还需要增量进行。比如：1月1号需要把 Repo A 的 C1 和 C2 合并成 CC1 并 Push 到 Repo B 中，1月10号需要把 Repo A 的 C3 和 C4 合并成 CC2 并 Push 到 Repo B 中，依次类推。

* 第一次合并操作，即把所有的提交合并成一个提交。

``` shell
git rebase -i <第一个提交的 SHA 值>
```

因为这里必须要提供一个 SHA 值，所以最终结果一定会大于或等于2个提交记录，并**不能达到我们的目的**。

那么，聪明的程序员马上想到基于相对位置的方式，假设本地总共4个提交记录，我是不是可以通过`HEAD~4`来定位呢？

``` shell
git rebase -i HEAD~4
```

这是你发现事与愿违，并不能成功执行，而是得到一个错误：`fatal: invalid upstream 'HEAD~4'`。但是功夫不负有心人，经过一番摸索，找到一个参数 `--root` 可以帮我们完美解决问题。

``` shell
git rebase -i --root
```

* 第一次合并操作之后，后续的操作可以通过分支的方式操作更加容易，就如 Pull Request 中的 Squash and Merge 一般丝滑。

![场景四-2](/images/2020-12-09/场景四-2.png)

具体操作步骤如下：
``` shell
# 假设第一次将 C1 和 C2 合并成了 CC1，这时我们从合并完成的这个点创建出一个分支（以分支名 `repo-b` 为例）
git checkout -b repo-b <CC1 的 SHA 值>

# 假设 Repo A 的 C3 和 C4 在 master 分支上，通过 Merge Squash 的方式合并 C3 和 C4 成为 CC2
# 如果此时 master 上除了 C3 和 C4 还有其他的提交记录，那么我们可以从 C4 的地方创建出新分支（以 `repo-a` 为例），然后在下面的命令中将 `master` 替换为 `repo-a` 即可
git merge --squash master --allow-unrelated-histories

# 此时如果有冲突就解决冲突，解决完冲突后，执行 `git commit`
# 最后 Push 到 Repo B 的远端仓库即可。
```

至此，我遇到过的几种场景就列举完了，欢迎大家留言补充，或者探讨更好的方式。
