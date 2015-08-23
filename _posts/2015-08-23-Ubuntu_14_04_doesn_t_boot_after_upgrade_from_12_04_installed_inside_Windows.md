---
date: 2015-08-23
layout: post
title: Ubuntu 14.04 doesn' t boot after upgrade from 12.04 installed inside Windows 7/8.1
thread: 2015-08-23-Ubuntu_14_04_doesn_t_boot_after_upgrade_from_12_04_installed_inside_Windows.md
categories: Linux
tags: Ubuntu
---

我在Windows下通过Wubi安装了Ubuntu 12.04，用起来一直很顺畅，
直到我升级内核到Ubuntu 14.04。
当我选择Ubuntu启动菜单后，悲剧发生了，漆黑如末日的屏幕上显示着：

<pre>
mount: mounting /dev/loop0/ on /root failed : Invalid argument
mount: mounting /dev on /root/dev failed: No such file or directory
mount: mounting /sys on /root/sys failed: No such file or directory
mount: mounting /proc on /root/proc failed: No such file or directory
Target filesystem doesn' t have requested /sbin/init
No init found. Try passing init = bootarg.

BusyBox v1.21.1 (Ubuntu 1:1:21.0-1ubuntu1) built-in shell (ash)
Enter 'help' for a list of built-in commands

(initramfs) _
</pre>

顿时，想死的都有了哇，遇到这种情况，我知道只能忍耐。
接下来，百度 -> 实验  -> 百度  ->  实验。。。
就这样，一天都快过去了，还是没有解决，一点进展都没有，哭吧。
黄天不负有心人，我Google了一下试试，嘿，第一条结果TMD就搞定了。


解决办法：

1,在**Grub菜单**按下 `e`  键，开始编辑Grub启动命令
2,找到字符串   `ro quiet splash` ，修改为  `rw quiet splash`
3,按下 `F10` 或者  `CTRL+X`  执行该启动命令    

此时已经可以顺利进入Ubuntu桌面了。
然而，上面的修改只是针对当前，并没有保存下来。
接下来，要修改文件 **/etc/default/grub** ，保存我们的修改。

4,`CTRL+ALT+T` 调出**命令终端**，终端输入 `sudo gedit /etc/default/grub`
5,修改 `GRUB_CMDLINE_LINUX_DEFAULT="quiet splash"` 
  为 `GRUB_CMDLINE_LINUX_DEFAULT="rw quiet splash"`, 别忘了注意grub文件是否保存成功：）
6,终端输入 `sudo update-grub`

重新启动Ubuntu，Have fun.
