---
date: 2015-12-05
layout: post
title: SSH keys无密码访问github
thread: 2015-12-05-SSH_Keys_github.md
categories: 经验
tags: github
---
</br>

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
    <ul><li>请按照１中操作重新生成一对</li></ul>
    <li>相同github账号可以添加多对SSH key</li>
    <li>不同终端下可以使用相同的SSH key </li>
    <ul><li>在生成同名SSH key后用原来的SSH key内容覆盖即可</li></ul>
    <li>相同终端下可以添加多对SSH key</li>
</ul>

</br>


