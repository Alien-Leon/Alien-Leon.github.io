---
title: The Usage of Git
tags: 工具 版本控制系统
key: The-Usage-Of-Git
modify_date: 2020-05-02 15:09:41 
---

# The Usage of Git
`Git`作为广泛使用的分布式版本控制系统之一，有着许多优秀的特性：
- 分支与合并：分支允许人们对仓库多线修改，并提供了快速的合并机制
- 轻量级：轻量级的提交与分支鼓励人们进行多分支的工作流
- 分布式：Git仓库可以分布在多个机器，并且每个仓库可以自由进行协作修改

本文作为[Pro Git book](https://git-scm.com/book/en/v2)的阅后总结，将会简要的介绍git中关键的实现原理，并从原理出发介绍一些常用的git命令。

<!--more-->

## 基本概念与原理
### 快照
当一个文件在修改之后并提交到git中时，git会以`Header:Content`的形式对文件进行摘要计算，生成40位的哈希校验码，并以此为名称记录文件的**完整快照**。我们可以在`.git/objects/<前2位哈希>/<后38哈希>`中找到对应的快照文件，例如：
```
$ find .git/objects -type f
.git/objects/d6/70460b4b4aece5915caf5c68d12f560a9fe3e4
```
借助于生成的哈希摘要，git可以保证文件在传输的过程中的完整性与安全性，防止在传输过程中被篡改。

尽管git会存储修改文件的完整快照，但这并不会占用过多的空间，原因主要是：
- 在计算摘要后，git会通过`zlib`对快照文件进行压缩，减少快照的存储占用，提高传输效率。
- git会定期或者在传输之前对多版本的快照文件进行合并，通常是保留最新的快照，老版本的快照会转换为差异信息来存储。

在多个文件提交到git后，git会将这些快照文件组织成快照树。如果某个文件具有多个版本的快照，那么这个文件的所有版本快照会串联在一起形成一个链表，在需要时可以通过这些版本快照对文件进行版本变更操作。


### 提交
git的每一次提交会生成一个`提交记录(commit)`，记录包括本次提交后新生成的快照树，提交者、提交信息等，与快照类似，在多次提交或分支之下，仓库中会逐渐生成一个提交树，**我们使用大多数git命令的主要功能就是对这个提交树进行增删查改**。

### 提交引用
为了便捷的对提交树进行操作，git中主要通过`分支`、`标签`来引用这颗提交树中的提交节点。

<div style="text-align: center">
<img title="Git 提交引用" src="/assets/The-Usage-Of-Git/The-Usage-Of-Git2020-04-30T21-19-45.png">
</div>

#### HEAD
`HEAD`指针对应着当前指向的提交节点，可以在提交树中自由移动。

#### 分支(branch)
分支顾名思义对应着提交树中的一个分支，具体表现为指向提交的一个指针。当在分支中提交修改之后，分支指针会跟随移动。通常`HEAD`指针会与当前分支重合，在一些操作中，也有可能会出现`HEAD`与当前分支指针分离的情况。

#### 标签(tag)
分支会跟随着提交而进行移动，标签则是固定引用某个提交节点，因此通常可以用于标记特定的版本。


### 文件状态与暂存区
git将文件分为两类状态 `tracked`, `untracked`，其中`tracked`分为`Unmodified`、`Modified`、`Staged`。
对应的文件状态周期如下图所示：

<div style="text-align: center">
<img title="Git 文件状态转移" src="/assets/The-Usage-Of-Git/The-Usage-Of-Git2020-05-01T12-04-39.png">
</div>

`Staged`状态是一个待提交的临时状态，当文件通过`git add`将修改提交到暂存区后，状态就变成了`Staged`。`git commit`提交修改时，默认是提交处于暂存区的文件修改。引入暂存区的好处是用户可以灵活指定本次提交中需要提交到git中的修改，而不是将所有的修改放到一次提交中。当然用户也可以通过`git commit -a`跳过暂存区状态，直接将所有`Modified`状态的文件均提交到git中。


### 远程仓库
在使用git的时候，一个仓库可以分布在多个机器上，用户只有在需要与远程库协作的时候才需要使用网络，因此与大多数集中式的版本控制系统相比，git的本地操作是十分快的。

当需要与远程库进行协作时，git会创建一个远程分支，让本地分支跟踪远程分支的修改，并进行同步。**实质上来说，远程协作的过程就是对两个提交树进行差异合并处理的过程**。

<div style="text-align: center">
<img title="Git 远程协作" src="/assets/The-Usage-Of-Git/The-Usage-Of-Git2020-05-01T12-40-20.png">
</div>

例如本地仓库中的`master`分支被设置为追踪远程地址`origin`中的`master`分支，因此本地地址会多出一个`只读`的远程分支`origin/master`用来标记远程仓库中的`master`分支的位置。当本地仓库中新提交了`C2`后，`master`分支前进，随后通过`git push`将本地`master`分支的更新推送到远程仓库，因此远程仓库在更新之后与本地仓库有着相同的提交，而`origin/master`也会跟随前进。

`git push`会将本地库的修改推送到远程库，而`git pull`则是从远程库中拉取修改并对修改进行合并，除了方向相反之外，二者的功能基本相同。


## 相关命令
以下命令主要是总结式地记录作者觉得比较常用的命令，因此光看是很难有比较好的学习效果的，建议大家结合相关的场景进行练习。大家可以前往挑战[Learn Git Branching](https://learngitbranching.js.org/)提供的关卡，以更好地熟悉相关的命令。


### 节点引用
在使用git的相关命令时，经常需要确定操作的提交节点`<commit>`，提交节点有两种引用方式，分别是`直接引用`和`间接引用`。

#### 直接引用
用户可以通过完整的提交哈希来引用这个提交节点，但由于哈希码长度比较长，用户也可以指定哈希码的前几位来引用对应的提交节点，例如
```
$ git log 1463
commit 1463a7d4327462ffb29816b012bde657db7d3080 (test)
```
#### 相对引用
git允许通过`^`来访问上一级节点，`~<n>`来访问上n级节点，例如

<div style="text-align: center">
<img title="节点相对引用" src="/assets/The-Usage-Of-Git/The-Usage-Of-Git2020-05-01T16-36-26.png.png">
</div>


### 状态与记录
#### status
[git status](https://git-scm.com/docs/git-status)主要用于显示当前的仓库状态，包括暂存区，原始文件的修改情况，分支合并情况等

#### log
[git log](https://git-scm.com/docs/git-log) 可以显示历史提交记录

1. 查看某个文件的修改记录
```
git log -- <file>
```

2. 查看最多n条记录
```
git log -<n>
```

#### reflog
[reflog](https://git-scm.com/docs/git-reflog)显示相关的提交节点变动情况，例如分支变更，引用指针变更等。


### 提交相关
#### commit
[git commit](https://git-scm.com/docs/git-commit)用于创建一个新的提交记录。

1. 提交暂存区中的文件
```
git commit
```
2. 提交所有`Modified`的文件，一般的git可视化工具比如`Github Desktop`默认是以-a模式提交文件。
```
git commit -a
```
3. 命令行中附加提交介绍与提交详情，未显式附加的时候git会打开默认文本编辑器来让用户填写
```
git commit -m "title" -m "description"
```
4. 当前提交与之前提交合并为一次提交，只适用于前一次分支未提交到远端的情况
```
git commit --amend
```

#### rm
[git rm](https://git-scm.com/docs/git-rm)主要用于将文件从版本库中删除。

1. 假如不小心将`.gitignore`中的文件添加到了版本库中

```
# --cached 仅删除版本库中文件而不删除实际文件
git rm -r --cached .idea  # 删除仓库中的.idea文件夹
git rm -r --cached test.py # 删除对应文件

# 删除版本库中符合.gitignore的所有文件
git ls-files -i --exclude-from=.gitignore | xargs git rm --cached
```

#### diff
[git diff](https://git-scm.com/docs/git-diff)可以帮助用户比较各版本间的差别。

1. 比对当前修改与上一次提交
```
git diff 
```

2. 比对两个提交的差异
```
git diff <commit1> <commit2> [<file>]
```



### 撤销回退相关
#### reset
[git reset](https://git-scm.com/docs/git-reset)主要用于撤销本地的提交，回退提交节点。

1. 回退到特定提交节点，默认为`--mixed`模式，在撤销更改之后会保留此前的修改处于未提交暂存区的状态。
```
git reset <commit>
```

需要注意`reset`会改写提交历史，回退之后原先的节点无法直接在提交历史中看到，需要借助`reflog`查看变更命令记录才能找到原先的节点。例如下图中回退后`C2`节点的修改会保留，但无法在历史中直接看到`C2`节点。如果想重新回到`C2`节点，需要确切知道`C2`节点的哈希码，再通过`git reset C2`恢复成原来的状态。
<div style="text-align: center">
<img title="git reset" src="/assets/The-Usage-Of-Git/The-Usage-Of-Git2020-05-01T17-24-31.png">
</div>


2. 回退到特定提交节点，丢弃所有的修改。
```
git reset --hard <commit>
```

#### revert
[git revert](https://git-scm.com/docs/git-revert)撤销修改的方式是创建新的提交记录而不像`reset`一样直接回退，并且在新提交中对文件进行回滚操作。在执行命令之前，需要确保当前仓库无修改。

1. 回滚到特定提交节点，回滚级数较多时，容易出现冲突需要手动解决
```
git revert <commit>
```

#### restore
[git restore](https://git-scm.com/docs/git-restore)是一个比较新的命令，在2020年1月13日发行的git 2.25.0版本中添加，主要用于从历史提交中恢复特定文件。

1. 如果对文件进行了修改，但未提交暂存区，可直接撤销修改至上次提交的状态
```
git restore <file|file_pattern>...
```
2. 如果对文件进行了修改，已提交暂存区，可将修改从暂存区取回
```
git restore --staged <file|file_pattern>...
```

> :memo: `git checkout -- <file|file_pattern>` 可以同时达成1、2的效果


3. 将文件直接恢复为某个提交时的状态，如果文件已经被删除也可以直接恢复
```
git restore -s <commit> <file|file_pattern>...
```


### 分支相关

#### branch
[git branch](https://git-scm.com/docs/git-branch)主要用于分支的增删查改操作。

1. 展示所有分支以及当前选择的分支
```
git branch
```
2. 创建分支
```
git branch <branch>
```
3. 删除分支
```
git branch -d <branch>...
```
4. 改变分支指向位置
```
git branch -f <branch> <commit>
```
5. 令分支追踪远程分支
```
git branch -u <remote/branch> [<branchToTrackRemote=curBranch>]
```

#### checkout
[git checkout](https://git-scm.com/docs/git-checkout)主要用于切换分支以及移动指针

1. 切换分支或移动HEAD指针
```
git checkout <branch|commit>
```
2. 根据远程分支创建一个追踪的新本地分支
```
git checkout -b <newBranchTrackRemote> <remote/branch>
```

#### tag
[git tag](https://git-scm.com/docs/git-tag)用于管理提交树中的标签

1. 在提交节点创建标签引用，添加`-f`可以强制覆盖已有标签
```
git tag <tag> <commit>
```
2. 删除标签
```
git tag -d <tag>...
```


#### merge
[git merge](https://git-scm.com/docs/git-merge)用于合并已有的分支，当合并出现冲突之后，git会给出冲突点让用户手动修改完成后再次进行合并。合并的过程主要有两种策略，分别是`FAST-FORWARD Merge`和`TRUE Merge`。


**FAST-FORWARD Merge**
如下图，基于`master`创建的`bugFix`分支和`master`分支处于同一条链中，那么`master`在合并`bugFix`时可以直接快速前向移动，bug修复完毕之后即可删除`bugFix`分支。
<div style="text-align: center">
<img title="FAST-FORWARD Merge" src="/assets/The-Usage-Of-Git/The-Usage-Of-Git2020-05-01T22-35-25.png">
</div>


**TRUE Merge**
如下图，`Feature`分支与`master`分支并不直接在同一条链中，因此合并的时候只能根据两个分支的当前点C1、C2，以及公共祖先节点三个点的内容来做合并。在这时合并可能会出现冲突，需要手动修改再将合并提交。


<div style="text-align: center">
<img title="True Merge" src="/assets/The-Usage-Of-Git/The-Usage-Of-Git2020-05-01T22-47-05.png">
</div>

1. 合并分支，将目标分支合并到当前分支
```
git merge <target_branch>
```

#### reabse
[git rebase](https://git-scm.com/docs/git-rebase)用于将某个分支放置在另一个分支之后，如下图，`Feature`在rebase之后会将当前分支与公共祖先节点差异的提交节点进行复制，并追加在`master`分支之后。需要注意的是，`rebase`操作会改变git历史，因此使用时需要小心。

<div style="text-align: center">
<img title="Git Rebase" src="/assets/The-Usage-Of-Git/The-Usage-Of-Git2020-05-01T23-30-19.png">
</div>

1. 改变当前分支的基点到目标分支
```
git rebase <target_branch>
```
2. 交互式的`rebase -i`功能十分强大，在执行之后会调用GUI或编辑器来让用户灵活地改变提交树，包括：
   - 自由调整提交顺序
   - 自由选择需要的提交
   - 合并多个提交

例如合并目标分支的前两次提交，执行后会打开编辑框并显示相关的命令提示，按照相应提示操作即可。
```
git rebase -i HEAD~2 [target_branch=cur_branch]
```

> :memo: **Do not rebase commits that exist outside your repository and people may have based work on them.** <br>
> **由于Rebase会修改历史记录，因此千万不能对远程仓库进行rebase操作，否则协作时会增加许多意外麻烦的操作，正确的操作应该是自身开发时rebase完成后再将完整的线推送至远程仓库**。
> 简单来说，**绝不要在公共的分支上使用Rebase**。

#### Rebase VS Merge
Rebase与Merge在执行合并的操作时，带来的**最终合并结果是相同的**，其**差异之处在于git log的日志信息**。Rebase能带来更干净的历史记录，而Merge仍保留有原先的分支历史记录。

#### cherry-pick
[git cherry-pick](https://git-scm.com/docs/git-cherry-pick)可以选择某些提交进行复制并追加到当前分支之上，在需要复制特定提交的时候十分方便。
```
git cherry-pick <commit>...
```



### 远程协作相关

#### clone
[git clone](https://git-scm.com/docs/git-clone)会克隆远程仓库，并为所有远程分支创建对应追踪的本地分支，最后检出master分支作为初始内容。

1. 克隆远程仓库
```
git clone <remote_url>
```

#### fetch
[git fetch](https://git-scm.com/docs/git-fetch)用于从远程仓库中下载数据，并更新本地仓库中远程分支的提交与指针位置。

1. 下载所有数据，并更新所有的远程分支
```
git fetch <remote=origin>
```

#### remote
[git remote](https://git-scm.com/docs/git-remote)用于管理与本地仓库相关联的远程仓库信息，clone后的仓库会默认将远程仓库设置为`origin`，用户也可以自行修改。

1. 查看
```
git remote # 显示当前的服务地址名
git remote -v # 具体显示所有服务地址和对应的url
```
2. 修改
```
git remote add <remote_name> <url>
git remote set-url <remote_name> <url>
git remote rename <oldname> <newname>
git remote rm <remote_name>
```
3. 展示远程库与本地库的差异
```
git remote show <remote>
```


#### pull
[git pull](https://git-scm.com/docs/git-pull)从远程仓库拉取提交并与本地仓库中相关联的分支进行合并，实际上等于`git fetch + git merge <remote_branch>`，用于简化操作

1. 拉取
```
git pull [<remote> <branch>]
```
2. 拉取后将本地分支rebase到对应的远程分支之后
```
git pull --rebase [<remote> <branch>]
```

#### push
[git push](https://git-scm.com/docs/git-push)与`git pull`是方向相反的操作，即从本地仓库推送到远程仓库，实际功能上是相同的。

如果远程仓库的branch已经提交了其他修改而本地仓库未同步，那么本地仓库便无法直接通过git push推送工作。
通常可以先使用git pull获取远程分支，默认会进行merge操作；又或者是git pull --rebase获取远程分支后执行rebase操作。

1. 推送
```
git push [<remote> <branch>]
```


## 推荐阅读
最后，对常见工作流的了解可以帮助大家选择合适的工作流以便更好地管理项目。因此推荐一些个人觉得比较好的文章：

- [git-workflow ruanyifeng](http://www.ruanyifeng.com/blog/2015/12/git-workflow.html)
- [xirong/my-git](https://github.com/xirong/my-git/blob/master/git-workflow-tutorial.md)






## Reference
- [Git - Book](https://git-scm.com/book/en/v2)
- [Learn Git Branching](https://learngitbranching.js.org/)
- [GitHub - geeeeeeeeek/git-recipes: 🥡 Git recipes in Chinese by Zhongyi Tong. 高质量的Git中文教程.](https://github.com/geeeeeeeeek/git-recipes)