---
title: "御林招新题：Git 版本管理"
subtitle: "御林招新题：Git 版本管理"
summary: "学习 Git 的基本操作"
description: "学习 Git 的基本操作"
image: "https://raw.githubusercontent.com/calendar0917/images/master/20251021193834987.png"
date: 2025-10-18
lastmod: 2025-10-18
draft: false
hiddenFromHomePage: True
toc:
 enable: true
weight: false
categories: ["CTF"]
tags: ["CTF"]
---

## **安装与配置 Git**

- 在你的 Linux 操作系统上，安装 Git。
- **全局配置**：配置你的全局用户名和邮箱，这是 Git 记录提交者信息所必需的。
- **验证**：执行 `git --version` 命令，确认 Git 已正确安装。

```shell
[root@localhost hello]# git config --global user.name "calendar"
[root@localhost hello]# git config --global user.email "1131821081@qq.com"
[root@localhost hello]# git --version
git version 1.8.3.1
```

## **创建与克隆仓库**

- **本地仓库**：在你的主目录下创建一个新的文件夹，并在其中初始化一个 Git 仓库。
- **远程克隆**：找一个公开的 Git 仓库（例如 GitHub 上的一个开源项目），使用 `git clone` 命令将其克隆到本地。

> 这里还配置了一下 clash，记一下命令：
>
> ```
> $ clashctl + ...
> Usage:
>     clash                    命令一览
>     clashon                  开启代理
>     clashoff                 关闭代理
>     clashui                  面板地址
>     clashstatus              内核状况
>     clashtun     [on|off]    Tun 模式
>     clashmixin   [-e|-r]     Mixin 配置
>     clashsecret  [secret]    Web 密钥
>     clashupdate  [auto|log]  更新订阅
> ```

```shell
[root@localhost git_repo]# git clone https://github.com/calendar0917/learning_log.git
Cloning into 'learning_log'...
remote: Enumerating objects: 72, done.
......
Unpacking objects: 100% (72/72), done.
[root@localhost git_repo]# ls
clash-for-linux-install  learning_log
```

## **文件的提交与同步**

- **文件修改**：在你克隆的本地仓库中，对某个文件进行修改。

- **暂存与提交**：

  - 使用 `git add` 命令将修改后的文件添加到暂存区。
  - 使用 `git commit` 命令提交你的变更，并附上一条有意义的提交信息。

  > 登录的时候遇到问题，发现得要用 **Personal Access Token** 来代替 Password，然后保存一下登录：
  >
  > ```shell
  > # 配置凭据缓存（默认缓存15分钟）
  > git config --global credential.helper cache
  > # 可选：设置缓存时间（单位：秒，例如设置30天）
  > git config --global credential.helper 'cache --timeout=2592000'
  > ```

- **同步操作**：

  - 使用 `git pull` 命令从远程仓库拉取最新的变更。
  - 使用 `git push` 命令将你的本地提交推送到远程仓库。

  ```shell
  [root@localhost learning_log]# git push
  ......
  Username for 'https://github.com': calendar0917
  Password for 'https://calendar0917@github.com':
  ......
  remote: To https://github.com/calendar0917/learning_log.git
     1d476d9..5a31feb  main -> main
  ```

- **验证**：在远程仓库页面上，确认你的提交历史已成功显示。

![image-20251018204758535](https://raw.githubusercontent.com/calendar0917/images/master/image-20251018204758535.png)

## **分支管理**

- **目标**：理解分支的作用，掌握创建、切换和合并分支的操作。

> 在一个分支上进行的代码修改不会影响其他分支，进行实验性的开发，能回退到原来的状态。

- **挑战**：

  - 创建一个名为 `feature-a` 的新分支。

  > `git branch name` 新建分支
  >
  > `git checkout name` 跳转分支
  >
  > `git checkout -b name` 创建并跳转分支

  - 在新分支上对文件进行修改并提交。

  ```shell
  [root@localhost learning_log]# git branch feature-a
  [root@localhost learning_log]# git checkout feature-a
  Switched to branch 'feature-a'
  [root@localhost learning_log]# vi hello.txt
  [root@localhost learning_log]# git add hello.txt
  [root@localhost learning_log]# git commit -m "来自feature-a的修改"
  [feature-a 44b4133] 来自feature-a的修改
   1 file changed, 1 insertion(+)
  ```

  - 切换回主分支（`main` 或 `master`）。
  - 使用 `git merge` 命令将 `feature-a` 分支的修改合并到主分支上。
  - 删除 `feature-a` 分支。

  ```shell
  [root@localhost learning_log]# git branch
  * feature-a
    main
  [root@localhost learning_log]# git checkout main
  Switched to branch 'main'
  [root@localhost learning_log]# git merge feature-a
  Updating 5a31feb..44b4133
  Fast-forward
   hello.txt | 1 +
   1 file changed, 1 insertion(+)
  [root@localhost learning_log]# git branch -d feature-a
  Deleted branch feature-a (was 44b4133).
  ```

## **历史记录与回溯**

- **目标**：学会查看提交历史，并在需要时回溯到特定版本。

- **挑战**：

  - 使用 `git log` 命令查看你的提交历史。

  ```shell
  [root@localhost learning_log]# git log
  commit 44b413353e383dcbc0063558f1ce87e544c7730b
  Author: calendar <1131821081@qq.com>
  Date:   Sat Oct 18 05:54:50 2025 -0700
  
      来自feature-a的修改
  
  commit 5a31feb5ae61d40b7d6d7eaff991b5c3eeb82892
  Author: calendar <1131821081@qq.com>
  Date:   Sat Oct 18 05:38:50 2025 -0700
  
      测试暂存与提交
  
  commit 1d476d982a3144004612cc9162b83b6483e42faa
  Author: calendar0917 <1131821081@qq.com>
  Date:   Thu Feb 13 11:10:55 2025 +0800
  
      Initial commit
  ```

  

  - 找到一个较早的提交 ID。
  - 使用 `git reset` 或 `git revert` 命令将代码库回溯到该提交版本，并解释你所用命令的区别。

- 知识：

> - 工作区和暂存区：
>   - 工作区：可见的实际存放项目文件的目录
>   - 暂存区：一个临时保存文件修改的区域，它是位于 .git 目录中的一个文件（在.git/index） ，并不是一个实际可视化的文件夹。Git 利用暂存区来管理文件的状态，决定哪些文件的修改会被包含在下一次提交中。
>   - 使用 `git add` 命令后，工作区中已修改的文件会被添加到暂存区，文件进入已暂存状态。暂存区只记录那些即将被提交到版本库的文件修改信息 。
> - git revert:
>   - 原理是创建一个**新的提交**，用来抵消目标提交所做的修改。（对后面的修改不影响，且不修改历史记录）
> - git reset：
>   - 用于将当前分支的 HEAD 指针重置到指定的提交版本，同时可以选择如何处理工作区和暂存区的文件
>   - `--hard` 模式：将 HEAD 指针、暂存区和工作区都重置到指定提交版本
>   - `--soft` 模式：仅将 HEAD 指针重置到指定提交版本，暂存区和工作区的内容保持不变
>     - 假设你有一串零散的提交（比如修复同一个功能的多次小修改），想把它们合并成一个清晰的大提交，`--soft` 模式非常合适
>     - 刚执行了 `git commit`，但发现提交信息写错了，或者想补充修改后再提交
>   - `--mixed` 模式：将 HEAD 指针和暂存区重置到指定提交版本，工作区内容不变。

- 实操：

```shell
# 报错了，用 git status 看原因
[root@localhost learning_log]# git revert 5a31feb5ae61d40b7d6d7eaff991b5c3eeb82892
error: could not revert 5a31feb... 测试暂存与提交
hint: after resolving the conflicts, mark the corrected paths
hint: with 'git add <paths>' or 'git rm <paths>'
hint: and commit the result with 'git commit'

[root@localhost learning_log]# git status
# On branch main --> 所在分支、状态
# Your branch is ahead of 'origin/main' by 1 commit.
#   (use "git push" to publish your local commits)
#
# You are currently reverting commit 5a31feb.  --> 正在进行的操作
#   (fix conflicts and run "git revert --continue")
#   (use "git revert --abort" to cancel the revert operation)
#
# Unmerged paths:  --> 报错的地方，5a31feb 中创建了 hello.txt,这里需要手动删除才可以
#   (use "git reset HEAD <file>..." to unstage)
#   (use "git add/rm <file>..." as appropriate to mark resolution)
#
#       deleted by them:    hello.txt
# 手动删除
[root@localhost learning_log]# git rm hello.txt
hello.txt: needs merge
[root@localhost learning_log]# git revert --continue
[main 49e231f] Revert "测试暂存与提交"
 1 file changed, 2 deletions(-)
 delete mode 100644 hello.txt
 [root@localhost learning_log]# git push
 ......
```

## **远程仓库进阶**

- **目标**：管理多个远程仓库，并实现不同仓库间的同步。
- **挑战**：
  - 为你的本地仓库添加第二个远程仓库（例如，一个 `GitHub` 仓库和一个 `Gitee` 仓库）。
  - 将你的本地分支推送到这两个不同的远程仓库。

```shell
# 查看并添加远程仓库
[root@localhost learning_log]# git remote -v
origin  https://github.com/calendar0917/learning_log.git (fetch)
origin  https://github.com/calendar0917/learning_log.git (push)
[root@localhost learning_log]# git remote add gitee https://gitee.com/calendar917/learning_log.git
[root@localhost learning_log]# git remote -v
gitee   https://gitee.com/calendar917/learning_log.git (fetch)
gitee   https://gitee.com/calendar917/learning_log.git (push)
origin  https://github.com/calendar0917/learning_log.git (fetch)
origin  https://github.com/calendar0917/learning_log.git (push)


# push 到gitee上
[root@localhost learning_log]# git push gitee main
Username for 'https://gitee.com': calendar917
Password for 'https://calendar917@gitee.com':
Counting objects: 10, done.
......
To https://gitee.com/calendar917/learning_log.git
 * [new branch]      main -> main
```

## 标签使用

> `git restore`（Git 2.23+ 新增）
>
> **核心作用**：恢复工作区或暂存区（index）的文件内容，不影响提交历史。可以理解为 “撤销对文件的修改”，直接操作文件内容，而非提交记录。

**保存一步**： 
`git add .` 
`git commit -m "描述"`  

**恢复任意状态**： 
`git log --oneline` 找编号 
`git restore --source=编号 .` 
`git add .`  

**打标签**： 
`git tag 标签名 提交号`  

 **恢复标签**： 
`git restore --source=标签名 .`  

 **删标签**： 
`git tag -d 标签名`
