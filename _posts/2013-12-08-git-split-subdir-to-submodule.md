---
layout: post
title: 如何把 GIT 仓库的子目录独立为子模块
date: 2013-12-08 12:27
tags: [git, submodule]
---

最近在给考拉山后台添砖加瓦的过程中，发现了两个问题：

* 对于当前的同一套逻辑，我已经有4个view层工作在上面了，马上还要再加上一个view，专门用来显示分享出去的页面；
* 接下来还要再加上同步evernote这种异步任务；
* 而现在这些代码都还在同一个GIT仓库中；

这种混乱的场景已经快让我不能忍了，于是乎决定把这个大仓库中的所有`business`和`model`的代码独立到一个`submodule`中，这样接下来就可以进一步的拆分各个view让它们都引用这个`submodule`。

还好之前的代码结构还算比较好，所有`business`和`model`的代码都在一个名为`coloshine`的目录中，所以接下来只要解决如何把一个子目录独立成一个`submodule`并且保存分支和提交历史这个问题就好了。

对GIT还没精通到这个程度，只能上网Google，现贴出方案；

## 准备

当前的目录结构如下：

    coloshine-server
    ├── ....
    ├── coloshine  -> 想要变成子模块的目录
    ├── coloshine_account
    ├── coloshine_admin
    ├── coloshine_api
    ├── coloshine_pics
    ├── ...

Clone一个新的仓库到目录`coloshine`

    :::bash
    git clone coloshine-server coloshine

这时`coloshine`仓库的目录结构应该是和`coloshine-server`完全一样的。


## 选择要保存的分支

通常刚clone出来的`coloshine`仓库本地只会有一个分支（比如master），如果我们希望在马上要做的子模块中保存其他的分支，那就首先把它们创建出来：

    :::bash
    git branch -r br1 origin/br1
    git branch -r br2 origin/br2
    
最后`origin`这个remote是不需要的，把它删除了
    
    :::bash
    git remote rm origin
    
## 转化成子模块

这一步是最重要的步骤，命令如下：

    :::bash
    git filter-branch --tag-name-filter cat --prune-empty --subdirectory-filter coloshine -- --all

该命令过滤所有历史提交，保留对coloshine子目录有影响的提交，并且把子目录设为该仓库的根目录。下面解释下各参数意思：

* --tag-name-filter cat 该参数控制我们要如何保存旧的tag，参数值为bash命令，cat表示原样输出。所以，如果你不关心tag，就不需要这个参数了；
* --prune-empty 删除空的（对子目录没有影响的）的提交
* --subdirectory-filter coloshine 指定子模块路径
* -- --all 该参数必须跟在`--`后面，表示对所有分支做操作，即对上一步创建的所有本地分支做操作。所以，如果你只想保存当前分支，就不需要这个参数了

该命令执行完毕后，查看当前目录结构就会发现里面已经是子目录的内容了。`git log`查看提交历史已经正常保存了。

至此，主要工作已经完成。但是当前的仓库中还保存这一下不需要的`object`，如果想清理这些来减小当前仓库的体积，再看下一步。

## 清理

    :::bash
    git reset --hard
    git for-each-ref --format="%(refname)" refs/original/ | xargs -n 1 git update-ref -d
    git reflog expire --expire=now --all
    git gc --aggressive --prune=now

这些命令到底是怎么work的我也没仔细研究，就不解释了。总是，我的实践证明它们是正常work的，仓库体积减小了不少。

最后，我们就可以把这个新的仓库提交到服务器上，然后把旧仓库中的`coloshine`子目录删除并以`submodule`的方式添加`coloshine`仓库就好了。

**参考链接:**

* [Detach subdirectory into separate Git repository](http://stackoverflow.com/questions/359424/detach-subdirectory-into-separate-git-repository)
* [Splitting a subpath out into a new repository](https://help.github.com/articles/splitting-a-subpath-out-into-a-new-repository)
