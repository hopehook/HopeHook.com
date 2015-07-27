---
date: 2015-07-27
layout: post
title: Git 常用技能
thread: 2015-07-27-GitCommand
categories: Git
tags: Git
---

[原文](http://wuchong.me/blog/2015/03/30/git-useful-skills/#)


**工作流**
---


Git 最核心的一个概念就是工作流。工作区(Workspace)是电脑中实际的目录；暂存区(Index)像个缓存区域，临时保存你的改动；最后是版本库(Repository)，分为本地仓库和远程仓库。下图真是一图胜千言啊，就无耻盗图了。

![工作流](/assets/images/git.jpg)


**远程仓库**
---


添加远程仓库

<pre>
git remote add origin git@server-name:path/repo-name.git  #添加一个远程库
</pre>


查看远程仓库

<pre>
git remote      #要查看远程库的信息
git remote -v   #显示更详细的信息
</pre>


推送分支

<pre>
git push origin master    #推送到远程master分支
</pre>


抓取分支

<pre>
git clone git@server-name:path/repo-name.git   #克隆远程仓库到本地(能看到master分支)
git checkout -b dev origin/dev  #创建远程origin的dev分支到本地，并命名为dev
git pull origin master          #从远程分支进行更新 
git fetch origin master         #获取远程分支上的数据
</pre>


<code>
$ git branch --set-upstream branch-name origin/branch-name
</code>，可以建立起本地分支和远程分支的关联，之后可以直接git pull从远程抓取分支。

另外，`git pull` = `git fetch` + `merge` to local


删除远程分支

<pre>
$ git push origin --delete bugfix
To https://github.com/wuchong/jacman
 - [deleted]         bugfix
</pre>


**历史管理**
---


查看历史

<pre>
git log --pretty=oneline filename #一行显示
git log -p -2      #显示最近2次提交内容的差异
git show cb926e7   #查看某次修改
</pre>


版本回退

<pre>
git reset --hard HEAD^    #回退到上一个版本
git reset --hard cb926e7  #回退到具体某个版
git reflog                #查看命令历史,常用于帮助找回丢失掉的commit
</pre>


用HEAD表示当前版本，上一个版本就是`HEAD^`，上上一个版本就是`HEAD^^`，`HEAD~100`就是上100个版本。


**管理修改**
---


<pre>
git status              #查看工作区、暂存区的状态
git checkout -- <file>  #丢弃工作区上某个文件的修改
git reset HEAD <file>   #丢弃暂存区上某个文件的修改，重新放回工作区
</pre>


查看差异

<pre>
git diff              #查看未暂存的文件更新 
git diff --cached     #查看已暂存文件的更新 
git diff HEAD -- readme.txt  #查看工作区和版本库里面最新版本的区别
git diff <source_branch> <target_branch>  #在合并改动之前，预览两个分支的差异
</pre>


使用内建的图形化git：`gitk`，可以更方便清晰地查看差异。当然 Github 客户端也不错。


删除文件

<pre>
git rm <file>           #直接删除文件
git rm --cached <file>  #删除文件暂存状态
</pre>


储藏和恢复

<pre>
git stash           #储藏当前工作
git stash list      #查看储藏的工作现场
git stash apply     #恢复工作现场，stash内容并不删除
git stash pop       #恢复工作现场，并删除stash内容
</pre>


**分支管理**
---


创建分支

<pre>
git branch develop              #只创建分支
git checkout -b master develop  #创建并切换到 develop 分支
</pre>


合并分支

<pre>
git checkout master         #切换到主分支
git merge --no-ff develop   #把 develop 合并到 master 分支，no-ff 选项的作用是保留原分支记录
git branch -d develop       #删除 develop 分支
</pre>


**标签**
---


显示标签

<pre>
git tag         #列出现有标签 
git show <tagname>  #显示标签信息
</pre>


创建标签

<pre>
git tag v0.1    #新建标签，默认位 HEAD
git tag v0.1 cb926e7  #对指定的 commit id 打标签
git tag -a v0.1 -m 'version 0.1 released'   #新建带注释标签
</pre>


操作标签


<pre>
git checkout <tagname>        #切换到标签

git push origin <tagname>     #推送分支到源上
git push origin --tags        #一次性推送全部尚未推送到远程的本地标签

git tag -d <tagname>          #删除标签
git push origin :refs/tags/<tagname>      #删除远程标签
</pre>


**Git 设置**
---


设置 commit 的用户和邮箱

<pre>
git config user.name "xx"               #设置 commit 的用户
git config user.email.com "xx@xx.com"   #设置 commit 的邮箱
git config format.pretty oneline        #显示历史记录时，每个提交的信息只显示一行
</pre>


