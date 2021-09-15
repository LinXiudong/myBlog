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

把文档树结构导出到 tree.txt

# npm

npm install xxx 安装模块

npm install xxx -g 将模块安装到全局环境中

npm ls 查看安装的模块及依赖

npm ls -g 查看全局安装的模块及依赖

npm uninstall xxx (-g) 卸载模块

npm cache clean 清理缓存

# git

## git add

在提交之前,git 有一个暂存区(staging area),可以放入新添加的文件或者加入新的改动。 commit 时提交的改动是上一次加入到 staging area 中的改动,而不是我们 disk 上的改动。

- git add .

会递归地添加当前工作目录中的所有文件。

## git commit

提交已经被 add 进来的改动.

- git commit -m "the commit message"

提交并说明提交信息

## git push

push your new branches and data to a remote repository.

- git push [alias] [branch]

将会把当前分支 merge 到 alias 上的[branch]分支。如果分支已经存在,将会更新,如果不存在,将会添加这个分支。

如果有多个人向同一个 remote repo push 代码, Git 会首先在你试图 push 的分支上运行 git log,检查它的历史中是否能看到 server 上的 branch 现在的 tip,如果本地历史中不能看到 server 的 tip,说明本地的代码不是最新的,Git 会拒绝你的 push,让你先 fetch,merge,之后再 push,这样就保证了所有人的改动都会被考虑进来。

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

## git 错误处理

Unable to access ‘https://github.com仓库.git/‘: OpenSSL SSL_read: Connection was reset, errno 10054

直接在 pycharm 的 Terminal 输入下面命令就成了，一般第一条就可以了，后面两条是取消代理设置，在第一条无效的时候可以试试

- git config --global http.sslVerify false
- git config --global --unset http.proxy
- git config --global --unset https.proxy
