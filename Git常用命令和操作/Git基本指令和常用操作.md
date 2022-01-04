# 概述

自从工作以来，一直在使用Git来做项目的版本管理，随着使用时间越来越长，理解也越来越深刻，所以想根据自己的工作经验写一篇Git相关的文章，介绍Git的指令、常用且快捷的操作和相关的一些概念知识。网上也有很多Git的详细教程，这篇文章准备不深入讲解指令原理，只注重实践和应用，以便提高工作效率。

# Git基本指令

从0开始使用Git，准备工作就是安装Git，创建远程仓库，生成SSH公私钥，配置到远程仓库上（或者配置用户名密码）

- 初始化本地仓库


```shell
git init
```

- 添加远程仓库


```shell
git remote add origin git@github.com:AE86jag/Article.git
```

- 当前分支（master）关联远程master分支


```shell
git branch --set-upstream-to origin/master
```

- 添加所有修改文件到暂存区


```shell
git add .
```

- 提交到版本库


```shell
git commit -m "init project"
```

- 拉取远程最新代码


```shell
git pull -r
#如果有冲突，解决冲突
git add .
git rebase --continue
```

- 推送到远程仓库


```shell
git push
```

- 保存/恢复当前修改


```shell
git stash
git stash pop
```

- 分支合并


```shell
#把V2.1.0分支代码合并到当前分支
git merge V2.1.0
#如果有冲突解决完冲突
git add .
git merge --continue
git push
```



# Git常规操作

- 未添加到暂存区的代码（没有执行<kbd>git add</kbd>），移到另一个分支

```shell
#在当前分支进行保存现场
git stash
#切换到要修改的分支 V2.1.0
git checkout V2.1.0
#恢复现场
git stash pop
```

> 适用场景：
>
> 本来应该在其他分支修改的内容，没有切换分支，直接在master上修改了，但是没有添加到暂存区，这部分修改又是不上线的，不能提交的



- 已经commit的代码移到另一个分支

```shell
git cherry-pick 587f23ff2e20ab4c97c12aa24f5d67a5257c2c5d
```

> 使用场景：
>
> 在某些分支策略下，在分支A修复的BUG，需要同步在master分支上修复，分支A的有一部分代码是不上线的，不能用<kbd>git merge</kbd> ，这时候就可以使用该指令将部分commit复制到其他分支



撤销未push的commit

撤销已push的commit





