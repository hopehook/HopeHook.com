---
date: 2015-12-05
layout: post
title: SSH keys无密码访问github
thread: 2015-12-05-SSH_Keys_github.md
categories: 经验
tags: github
---
</br>

操作环境：Ubuntu

#### 1 生成一对SSH key

$ `ssh-keygen -t rsa -C "your_email@example.com"`

Enter file in which to save the key (/Users/you/.ssh/id_rsa): `/Users/you/.ssh/"your_ssh_name"`

Enter passphrase (empty for no passphrase): `[Press enter]`

Enter same passphrase again: `[Press enter]`


</br>

#### 2 将生成的SSH key公钥添加到github账户设置里
- 打开并复制　/Users/you/.ssh/"your_ssh_name.pub"　里面的内容，
- github账户　——> settings ——> SSH keys  ——>  Add SSH key  ——>  粘贴复制的内容

</br>

#### 3 注意
<ul>
    <li>不同github账号不能共用一对SSH key</li>
    <ul><li>需重新生成一对，详见【拓展】</li></ul>
    <li>相同github账号可以添加多对SSH key</li>
    <li>不同终端(OS User Account)下可以使用相同的SSH key </li>
    <ul><li>在生成同名SSH key后用原来的SSH key内容覆盖即可</li></ul>
    <li>相同终端下可以添加多对SSH key</li>
</br>

#### 【拓展】一个PC终端用户配置多个github账户的SSH key
</br>
(1)按照１和２的步骤生成对应github账户的多对SSH key
</br>
(2)新增或配置`~/.ssh/config`，内容示例：
<pre>
Host github-first
HostName github.com
 User git
 IdentityFile ~/.ssh/id_rsa_first


Host github-second
 HostName github.com
 User git
 IdentityFile ~/.ssh/id_rsa_second


Host github-third
 HostName github.com
 User git
 IdentityFile ~/.ssh/id_rsa_third
</pre>

</br>
(3)配置克隆仓库目录下的`.git/config`,内容示例：
<pre>
[core]
	repositoryformatversion = 0
	filemode = true
	bare = false
	logallrefupdates = true
[remote "origin"]
	url = git@github-first:hopehook1/test.git
	fetch = +refs/heads/*:refs/remotes/origin/*
</pre>
</br>
<pre>
[core]
	repositoryformatversion = 0
	filemode = true
	bare = false
	logallrefupdates = true
[remote "origin"]
	url = git@github-second:hopehook2/test.git
	fetch = +refs/heads/*:refs/remotes/origin/*
</pre>
</br>
<pre>
[core]
	repositoryformatversion = 0
	filemode = true
	bare = false
	logallrefupdates = true
[remote "origin"]
	url = git@github-third:hopehook3/test.git
	fetch = +refs/heads/*:refs/remotes/origin/*
</pre>

(4)特别说明：
<ul>
<li>
github根据配置文件的user.email来获取github帐号显示author信息，所以对于多帐号用户一定要记得将user.email改为相应的email(如：second@mail.com)
</li>
<li>
测试`~/.ssh/config`配置情况：
`ssh -T git@github-first`
`ssh -T git@github-second`
`ssh -T git@github-third`
</li>
<li>执行git clone [URL]的时候，[URL]原样写入`.git/config`的url (HTTPS和SSH方式)
<ul>
<li>
例如：git clone git@github-second:hopehook2/test.git的时候`.git/config`的url = git@github-second:hopehook2/test.git
Host
</li>
</ul>
</li>
</ul>

