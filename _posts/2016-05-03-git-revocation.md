---
layout: post
title: Git中的后悔药
categories: [编程, git]
tags: [撤销]
---


> 人生是没有后悔药的，但`git`有

#### 1. 取消修改
原因：不小心修改了某个文件，现在希望回退到修改之前的状态

通过`git status`命令可看到提示`(use "git checkout -- <file>..." to discard changes in working directory)`：
```
$ git status
On branch master
Your branch is up to date with 'origin/master'.

Changes not staged for commit:
  (use "git add <file>..." to update what will be committed)
  (use "git checkout -- <file>..." to discard changes in working directory)

        modified:   _config.yml

```

按提示操作
```
$ git checkout --  _config.yml
```

> `git checkout -- <file>...`可以还原对文件的修改

#### 2. 取消add

原因：不小心`add`了一个不该提交的文件
```
$ git add *
```

通过`git status`命令可以看到提示`(use "git reset HEAD <file>..." to unstage)`：
```
$ git status
On branch master
Your branch is up to date with 'origin/master'.

Changes to be committed:
  (use "git reset HEAD <file>..." to unstage)

        modified:   _config.yml

```

按提示操作
```
$ git reset HEAD _config.yml
Unstaged changes after reset:
M       _config.yml

```

> `git reset HEAD <file>...` 撤销`add`，将文件从暂存区回退到工作区 

#### 3. 取消commit

原因：希望撤销上一次`commit`

使用`git log`查看`commit`日志
```
$ git log

commit 513a8cc4a6b65cc5c21e6af026fecbcbdd240fe1 (HEAD -> master)

commit 38fd4324bd4a28047a6555509ccb98216525bac4 (origin/master, origin/HEAD)

commit ddf858e5376ed030bf32a8a9b7544172e9f08ac6

```

> `git`中使用一串`40位的十六进制数`表示`commit`记录的`ID`

可以看到最近一次`commit`为`commit 513a8cc4a6b65cc5c21e6af026fecbcbdd240fe1 (HEAD -> master)`

撤销最近一次`commit`的操作，即回退到上一次`commit ID`
```
git reset --soft 38fd4324bd4a28047a6555509ccb98216525bac4
```

> `git reset --soft` 的意思是只撤销`commit`，不还原代码   
> `git reset --hard` 除了撤销`commit`，还会还原代码，**慎用**

#### 4. 修改commit
原因：`commit`后，发现提交信息需要修改

```
git commit --amend -m 'amendatory message'
```

> `git commit -- amend`命令可以修改上一次`commit`的内容，包括`author, message`等

#### 5. 撤销merge

误将master分支合并到了dev，幸好没有push

```
$ git checkout dev
$ git merge dev
```

使用`git reflog`命令查看操作记录

```
$ git reflog
66f5655 HEAD@{0}: merge master: Merge made by the 'recursive' strategy.
7d45b22 HEAD@{1}: checkout: moving from master to dev
f75fd61 HEAD@{2}: pull: Fast-forward
```

这里我们要恢复到上一次操作的状态，即`7d45b22`，使用`git reset`命令

```
$ git reset --hard 7d45b22
```