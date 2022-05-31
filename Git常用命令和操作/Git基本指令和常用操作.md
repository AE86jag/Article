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
git branch --set-upstream-to origin master
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

在控制台每输入一个指令，无论正常还是错误，git都会提示下一步或者正确的指令是什么，基本上根据提示都可以顺利操作下去。下面介绍一些常用的操作：

- 未添加到暂存区的代码（没有执行<kbd>git add</kbd>），移到另一个分支

```shell
#假设当前分支是master，在当前分支进行保存现场
git stash
#切换到要修改的分支 V2.1.0
git checkout V2.1.0
#恢复现场
git stash pop
#如果切换回master执行git stash pop，也是会出现之前修改的代码的
```

> 适用场景：
>
> 本来应该在其他分支修改的内容，没有切换分支，直接在master上修改了，但是没有添加到暂存区，这部分修改又是不上线的，不能提交的；

- 已经commit的代码移到另一个分支

```shell
#这个提交在原分支还是存在的
git cherry-pick 587f23ff2e20ab4c97c12aa24f5d67a5257c2c5d
```

> 适用场景：
>
> 在某些分支策略下，在分支A修复的BUG，需要同步在master分支上修复，分支A的有一部分代码是不上线的，不能用<kbd>git merge</kbd> ，这时候就可以使用该指令将部分commit复制到其他分支

- 撤销未push的commit

```shell
#撤销上一个commit，也就是最新的commit，执行完修改是在工作区，未add的状态
git reset HEAD^
#同上，但是修改的部分也被撤销了
git reset --hard HEAD^
```

> 适用场景：
>
> 有些仓库的commit的message是有一定规则的，不符合规则不让提交，如果提交的message写错了，那么可以撤销修改，但是保留代码，重新执行<kbd>git add</kbd> <kbd>git commit</kbd> 重新提交

- 撤销已push的commit

```shell
git revert commitId
git push
```

> 已经push的不能用 git reset HEAD^，因为这样本地已经撤销，等push的时候，会提示落后远程一个提交，只能git pull 一下，这样之前撤销的commit又回来了，可以加-f强制提交，但是有些远程仓库是禁止强制提交的

- 撤销未add的修改

```shell
git checkout <file>
git restore <file>
```

> 适用场景：
>
> 书写了逻辑错误的代码，需要重头开始写，直接<kbd>git checkout .</kbd>清除所有未add的无用代码

- 撤销已add但未commit的修改

```shell
#以下三个指令都可以
git reset --mixed
git restore --staged <file>
git reset HEAD <file>
```

- 基于某个Tag修改代码


```shell
git branch <分支名> <标签名称>
```

> 适用场景：
>
> 提交上线投产申请时，是打一个Tag，后续使用这个Tag进行上线，部署，打完Tag后可以继续在该分支提交代码。假如某次上线后发现一个严重Bug，需要紧急修复，为了不引入新的代码产生风险，那么需要在上次上线的Tag基础上拉一个分支进行Bug修复，测试覆盖完后重新打一个Tag进行紧急上线。



# Git其他操作

- 分支策略

只要不混乱，适合团队就可以。比如我所在的团队就是主分支提交，主分支发布。多项目并行，上线时间不确定，则拉分支开发，根据实际情况进行分支合并。

- Pull Request

除了修改开源项目时，自己项目开发时，也可以采用PR的方式，每个人提交的代码都由技术经理进行审核，审核通过以后合并到项目中，这样可以代替代码检视，比较适合逻辑相对不复杂的项目。我所在的项目，前端就是采用PR的方式。后端因为逻辑比较复杂，还是直接提交，定期代码检视。

- Git的自动化

Git自动化可以使用Shell脚本，如果想在Java代码中使用Git，有一个类库叫JGit，可以引入依赖在Java代码中使用，其中的API和Git指令都是一一对应的。应用场景有很多，比如使用Jenkins自动化部署的时候，编写一个Shell脚本，使用Git指令从远程仓库拉取代码后进行构建部署，不用手动打包，上传、部署。JGit也有很多地方用到，比如自定义Gradle插件，修改了文件，使用JGit提交到当前项目中，

- Git的钩子

Git的钩子可以分本地的钩子和远程的钩子。

本地的钩子被存在Git目录下的hooks子目录中，有很多类型，比如提交之前，提交之后等等，可以在提交之前做一些限制，比如限制代码风格、限制commit的message格式，等等，只要你想在做的校验，自己编写脚本放在hooks目录下即可。

远程的钩子，在代码托管网站上可以设置，也有很多类型，比如Push、打Tag、Pull Request等，需要选择类型，然后输入一个URL，等事件触发以后，Git会向这个URL发送一个POST请求。比如可以设置一个Push钩子，URL写一个Jenkins的任务构建的API（http://用户名:密码@IP:端口/job/项目名/build）。这样，每次提交代码的时候就会触发构建，构建任务有自动化脚本，进行部署。

- Git子模块

Git可以将一个仓库作为另一个仓库的子目录，可以多个仓库共享某个公共的仓库，但是又保持独立的提交。比如微服务架构下，每个服务都有接口测试的Json文件，跨服务调用时，也是使用Json文件进行Mock服务间请求，这样，每个服务都依赖这些Json文件，所以把这些Json文件作为一个仓库，并作为其他微服务的子模块，就实现了复用，而且独立提交。

执行指令 <kbd>git submodule add 仓库地址</kbd>  当前项目就会多一个.gitmodules文件，和子模块的目录

