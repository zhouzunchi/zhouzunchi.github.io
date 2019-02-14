---
title:  "Git"
---

{% include toc icon="cog" title="Git" %}

# GIT

[中文官方文档](https://git-scm.com/book/zh/v2)

> Git 的三中状态__已提交(committed)__、__已修改(modified)__和__已暂存(staged)__，已提交表示数据已经安全的保存在本地数据库中。 已修改表示修改了文件，但还没保存到数据库中。 已暂存表示对一个已修改文件的当前版本做了标记，使之包含在下次提交的快照中。

![](.\images\Git01.png)

## Git 保证完整性

> Git 中所有数据在存储前都计算校验和，然后以校验和来引用。 这意味着不可能在 Git 不知情时更改任何文件内容或目录内容。 这个功能建构在 Git 底层，是构成 Git 哲学不可或缺的部分。 若你在传送过程中丢失信息或损坏文件，Git 就能发现。
>
> Git 用以计算校验和的机制叫做 SHA-1 散列（hash，哈希）。 这是一个由 40 个十六进制字符（0-9 和 a-f）组成字符串，基于 Git 中文件的内容或目录结构计算出来。 SHA-1 哈希看起来是这样：
>
> ```
> 24b9da6552252987aa493b52f8696cd6d3b00373
> ```
>
> Git 中使用这种哈希值的情况很多，你将经常看到这种哈希值。 实际上，Git 数据库中保存的信息都是以文件内容的哈希值来索引，而不是文件名。

## Git推送方式

>**simple**方式，默认只推送当前分支
>
>**matching**方式，会推送所有对应的远程分支的本地分支
>
>如果修改这个设置，可以采用 git config命令
>
>```git config --global push.default matching```
>
>  或者
>
> ```git config --global push.default simple```

## 命令

#### 新建代码库

```bash
git init 							# 本地新建Git代码库（当前目录）
git init PROJECT_NAME				# 新建一个项目目录 
git clone PROJECT_NAME&PROJECT_URL	# 下载一个项目和整个代码历史
```

#### 配置

使用了 `--global` 选项，那么该命令只需要运行一次，因为之后无论你在该系统上做任何事情， Git 都会使用那些信息。 当你想针对特定项目使用不同的用户名称与邮件地址时，可以在那个项目目录下运行没有 `--global` 选项的命令来配置

```bash
git config --list
git config -e [--global]
git config [--global] user.name "[name]"
git config [--global] user.email "[email address]"
```

#### 增加、删除文件

```bash
git add [file1] [file2]			# 添加指定文件到暂存区
git add [dir]					# 添加指定目录到暂存区
git add .						# 添加当前目录所有文件到暂存区
git rm [file1] [file2]			# 删除工作区文件，并且将这次删除放入暂存区
git rm --cached [file]			# 停止追踪指定文件，但该文件会保留在工作区
git mv [file1] [file-renamed]	# 改名文件，并且将这个改名放入暂存区
```

#### 代码提交

```bash
git commit -m [message]					# 提交暂存区到仓库
git commit [file1] [file2] -m [message]	# 提交暂存区指定文件到仓库
git commit -a							# 提交工作区自上次 commit 之后的变化（直接到存储区）
git commit -v							# 提交时显示所有 diff 信息
git commit --amend -m [message]			# 使用一次新的commit，替代上一次提交。如果代码没有任何新变化，则用来改写上一次commit的提交信息
```

#### 分支

```bash
git branch					# 列出所有本地分支
git branch -r				# 列出所有远程分支
git branch -a				# 列出所有本地分支和远程分支
git branch -d [branch-name]	# 删除分支

#  删除远程分支
git push origin --delete
git branch -dr

git branch [branch-name]								# 新建一个分支，但停留在当前分支
git checkout -b [branch-name]							# 新建一个分支，并切换到该分支
git branch [branch-name] [commit]						# 新建一个分支，指向指定的 commit
git branch --track [branch-name] [rmote-branch]			# 新建一个分支，与指定的远程分支建立追踪关系
git checkout [branch-name]								# 切换到指定分支，并更新工作区
git branch --set-upstream [branch-name] [remote-branch]	# 建立追踪关系，在现在分支与指定的远程分支之间
git merge [branch-name]									# 合并指定分支到当前分支
git cherry-pick [commit]								# 选择一个 commit，合并进当前分支
```

#### 标签

```bash
git tag								# 列出所有tag
git tag [tag_name]					# 新建一个tag再当前commit
git tag [tag_name] [commit]			# 新建一个tag再制定commit
git show [tag_name]					# 查看 tag 信息
git push [remove] [tag_name]		# 提交制定 tag
git push [remote] --tags			# 提交所有 tag
git checkout -b [branch] [tag_name]	# 新建一个分支，指向某个tag
```

#### 查看信息

```bash
git status				# 显示有变更的文件
git log					# 显示当前分支的历史版本
git log --stat			# 显示 commit 历史，以及每次 commit 发生变更的文件
# 显示某个文件的版本历史，包括文件改名
git log --follow [file]
git whatchanged [file]

git log -p [file]		# 显示制定文件相关的每一次 diff
git blame [file]		# 显示指定文件是什么人什么时间修改过

git diff				# 显示暂存区和工作区的差异
git diff --cached		# 显示暂存区和上一个 commit 的差异
git diff HEAD			# 显示工作区与当前分支最新 commit 之间的差异
git diff [first_barch] [second_branch]	# 显示两次提交之间的差异
git show [commit]						# 显示某次提交的元数据和内容变化
git show -name-only [commit]			# 显示某次提交发生变化的文件
git show [commit]:[filename]			# 显示某次提交，某个文件的内容
git reflog								# 显示当前分支的最近几次提交
```

#### 远程同步

```bash
git fetch [remote]					# 下载远程仓库的所有变动
git remote -v						# 显示所有远程仓库
git remote show [remote]			# 显示某个远程仓库的信息
git remote add [shortname] [url]	# 增加一个新的远程仓库，并命名
git pull [remote] [branch]			# 取回远程仓库的变化，并与本地分支合并
git push [remote] [branch]			# 上传本地制定分支到远程长裤
git push [remote] --force			# 强制上传分支到远程仓库，即时存在冲突
git push [remote] --all				# 推送所有分支到远程仓库
```

#### 撤销

```bash
git checkout [file]				# 恢复暂存区的制定文件到工作区
git checkout [commit] [file]	# 恢复某个 commit 的制定文件到工作区
git checkout					# 恢复上一个 commit 的所有文件到工作区
git reset [file]				# 重置暂存区的指定文件，与上一次 commit 保持一致，但工作区不变
git reset --hard				# 重置暂存区与工作区，与上一次 commit 保持一致
git reset [commit]				# 重置当前分支的指针为指定 commit，同事重置暂存区，但工作区不变
git reset --hard [commit]		# 重置当前分支的 head 为指定 commit，同时暂存区和工作区，与指定 commit 一致
git reset --keep [commit]		# 重置当前 HEAD 为指定 commit，但保持暂存区和工作区不变
# 新建一个commit，用来撤销指定commit。后者的所有变化都将被前者抵消，并且应用到当前分支
git revert [commit]
```



