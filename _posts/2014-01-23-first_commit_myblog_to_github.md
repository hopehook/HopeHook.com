---
date: 2014-01-23
layout: post
title: 第一次提交我的博客代码到github用到的命令
thread: 2014-01-23-first_commit_myblog_to_github.md
categories: 经验
tags: github
---

####第一次提交

`f:`

`cd github\Blog`

`git init`   

`git checkout --orphan gh-pages`

`git add .`

`git commit -m "first post"`

`git remote add origin https://github.com/hopehook/Blog.git`

`git push origin gh-pages`



####后续提交

`git commit -m "提交备注"`

`git push -u origin gh-pages`
