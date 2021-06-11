---
layout: mypost
title: cmd,npm,git常用命令
categories: [常用命令]
---

# cmd

- tree

查看文件夹树结构

- tree /f

查看文档树结构（包括文件夹中的文件）

- tree /f > tree.txt

把文档树结构导出到tree.txt

# npm

npm install xxx 安装模块

npm install xxx -g 将模块安装到全局环境中

npm ls 查看安装的模块及依赖

npm ls -g 查看全局安装的模块及依赖

npm uninstall xxx (-g) 卸载模块

npm cache clean 清理缓存

# git

## git add
在提交之前,git有一个暂存区(staging area),可以放入新添加的文件或者加入新的改动。 commit时提交的改动是上一次加入到staging area中的改动,而不是我们disk上的改动。

- git add .

会递归地添加当前工作目录中的所有文件。

## git commit
提交已经被add进来的改动.

- git commit -m "the commit message"

提交并说明提交信息

## git push
push your new branches and data to a remote repository.

- git push [alias] [branch]

将会把当前分支merge到alias上的[branch]分支。如果分支已经存在,将会更新,如果不存在,将会添加这个分支。

如果有多个人向同一个remote repo push代码, Git会首先在你试图push的分支上运行git log,检查它的历史中是否能看到server上的branch现在的tip,如果本地历史中不能看到server的tip,说明本地的代码不是最新的,Git会拒绝你的push,让你先fetch,merge,之后再push,这样就保证了所有人的改动都会被考虑进来。


## 获取/更新远程分支列表

- git remote update origin --prune

有时需要先更新才能拉到最新代码

## git fetch 获取远程主机的更新信息

## git pull 拉代码

## Git 删除提交记录

- Checkout
 
> git checkout --orphan latest_branch
 
- Add all the files

> git add -A

- Commit the changes

> git commit -am "commit message"

- Delete the branch

> git branch -D master

- Rename the current branch to master

> git branch -m master

- Finally, force update your repository

> git push -f origin master
