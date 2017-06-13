---
date: 2016-11-04
layout: post
title: Mysql读写分离实践 mysql-proxy(Atlas) + mha + vip
thread: 2016-11-04-mysql-proxy_mha_vip.md
categories: 经验
tags: 经验
---

<br/>
#### 一 实验材料
<br/>
1 外部环境
<br/>
* amd64主机
<br/>
* Windows x64
<br/>
* Oracle VM VirtualBox

<br/>
2 实验环境   # 文章最后有下载链接
<br/>
* VirtualBox的4个虚拟机debian x64
<br/>
* mysql-proxy(Atlas)
<br/>
* mha


<br/>
#### 二 实验步骤
<br/>
1 确保hosts和hostname配置正确

1.1 每台主机都相同(主机：debtest1, debtest2, debtest3, debtest4)

root@debtest1:~/.ssh# cat /etc/hosts
<pre>
127.0.0.1    localhost
192.168.56.13    debtest1.sam.test    debtest1
192.168.56.14    debtest2.sam.test    debtest2
192.168.56.15    debtest3.sam.test    debtest3
192.168.56.16    debtest4.sam.test    debtest4
# The following lines are desirable for IPv6 capable hosts
::1     localhost ip6-localhost ip6-loopback
ff02::1 ip6-allnodes
ff02::2 ip6-allrouters
</pre>
<br/>
1.2 按照实际情况设定主机名，重启

root@debtest1:~/.ssh# cat /etc/hostname
<pre>
debtest1
</pre>
<br/>
2 使用ssh-keygen工具生成统一公私钥对，并同步到所有机器。测试公私钥验证访问

2.1 使用ssh-keygen工具生成统一公私钥对(主机：debtest1)

ssh-keygen -t rsa

cd ~/.ssh

cat id_rsa.pub authorized_keys
<br/>
2.2 同步到所有机器

scp ./* root@debtest2:/root/.ssh/

scp ./* root@debtest3:/root/.ssh/

scp ./* root@debtest4:/root/.ssh/ 
<br/>
2.3 每台主机上测试公私钥验证访问（必须要手动访问一次，在相应主机上产生无密码ssh访问记录）

ssh debtest1

ssh debtest2

ssh debtest3

ssh debtest4
<br/>
3.规划节点用户和ip配置

3.1 规划
<pre>
192.168.56.13   debtest1 --> mha manager node & mha node
192.168.56.14   debtest2 --> mysql1(master)  &  mha node & mysql-proxy(Atlas)
192.168.56.15   debtest3 --> mysql2(slave)  &  mha node
192.168.56.16   debtest4 --> mysql3(slave)  &  mha node
192.168.56.19   vip
</pre>
<br/>
3.2 定好虚拟ip后别忘了在当前的主库上添加虚拟ip    <font color="red"># 每次重启都需要操作</font>

ip addr add 192.168.56.19/24 dev eth0

\# 删除虚拟ip的命令 ip addr del 192.168.56.19/24 dev eth0
<br/>
4 数据库主从关系配置

4.1 配置my.cnf

找到[mysqld]节点，添加或修改成以下内容。
<pre>
server-id=1 #服务器ID
log-bin=mysql-bin
binlog-do-db=test  #这里设置需要在主服务器记录日志的数据库，只有在这里设置了的数据库才能被复制到从服务器
binlog-ignore-db=mysql #这里设置在主服务器上不记度日志的数据库
expire_logs_days=10
</pre>
<br/>
4.2 进入mysql客户端，执行以下命令     <font color="red"># 每次重启都需要操作</font>

主数据库(主机：debtest2)

RESET MASTER;
STOP SLAVE;
RESET MASTER;
<br/>

从数据库(主机：debtest3, debtest4)

RESET MASTER;
STOP SLAVE;

change master to
master_host='192.168.56.14',
master_user='repl',
master_password='repl',
master_log_file='mysql-bin.000001',
master_log_pos=107;

START SLAVE;
SHOW SLAVE STATUS;
<br/>
5 安装mha

5.1 在全部节点上安装mha node包和其依赖包(主机：debtest1, debtest2, debtest3, debtest4)

apt-get install libdbd-mysql-perl

dpkg -i mha4mysql-node_0.53_all.deb

<br/>
5.2 仅需要在mha manager节点上安装mha manager包及其依赖包(主机：debtest1)

apt-get install libdbd-mysql-perl

apt-get install libconfig-tiny-perl

apt-get install liblog-dispatch-perl

apt-get install libparallel-forkmanager-perl

dpkg -i mha4mysql-manager_0.53_all.deb

<br/>

6 建立配置文件目录，编辑mha必要的三个文件，一个配置文件，2个虚拟ip管理脚本(主机：debtest1)

master_ip_online_change 和 master_ip_failover 一模一样

mha_manager.cnf

<br/>
7 测试和启动mha(主机：debtest1)

7.1 可以尝试验证一下配置是否成功
<pre>
masterha_check_ssh --conf=/root/mha_base/mha_manager.cnf
masterha_check_repl --conf=/root/mha_base/mha_manager.cnf
</pre>
<br/>

7.2 在manager节点启动mha服务,然后观察日志,并尝试关闭当前主库,注意观察日志,主要看失效发现,日志检测,ip漂移和角色切换过程     <font color="red"># 每次重启都需要操作</font>

nohup /usr/bin/masterha_manager --conf=/root/mha_base/mha_manager.cnf --ignore_last_failover  < /dev/null > /root/mha_base/manager.log 
2>&1 &

<br/>
8 mysql-proxy(Atlas)

8.1 修改Atlas的配置文件test.cnf

<pre>
# 配置主数据库
proxy-backend-addresses = 192.168.56.19:3306

#配置从数据库
proxy-read-only-backend-addresses = 192.168.56.14:3306, 192.168.56.15:3306, 192.168.56.16:3306

#配置Atlas访问各个数据库使用的用户名和密码
pwds = mha:O2jBXONX098=

#Atlas工作端口
proxy-address = 0.0.0.0:4040

#Atlas管理端口
admin-address = 0.0.0.0:4041
</pre>

<br/>
8.2 启动Atlas      <font color="red"># 每次重启都需要操作</font>

./mysql-proxyd test start

<br/>
8.3 进入Atlas工作界面，通过Atlas来执行sql语句，自动分离读写操作

mysql -u192.168.56.14 -P4040 -umha -hmha

<br/>
#### 三 实验辅助手段及遇到的坑
<br/>
1 破解debian root密码

http://jingyan.baidu.com/article/fec7a1e5f0ea281190b4e7bb.html

<br/>
2 克隆虚拟机

目的：利用debtest3克隆一个debtest4，增加实验需要的节点

<br/>
3 设置虚拟机为静态IP

目的：确保每次重启虚拟机，IP地址都是固定的，便于以后的实验。


<br/>
4 Virtual Box网卡配置

网卡一: host only

网卡二: NAT

宿主机里配置Virtual Box虚拟网卡的ip,网段和虚拟机的静态ip保持一致:192.168.56.1

效果:
宿主机可以ping通虚拟机;
虚拟机可以ping通宿主机;
虚拟机内部可以互相ping通;
虚拟机可以上互联网

<br/>
5 停止master上mysql 没有自动转移 ，manager.log 出现错误

[error][/usr/local/share/perl5/MHA/ManagerUtil.pm, ln178] Got ERROR: Use of uninitialized value $msg in scalar chomp at 
/usr/local/share/perl5/MHA/ManagerConst.pm line 90.

这是一个bug，解决方法：

在文件/usr/share/perl5/vendor_perl/MHA/ManagerConst.pm第90行(chomp $msg)前加入一行：

$msg = "" unless($msg);



<br/>
#### 四 资源下载及附件
<br/>
1 [实验环境下载链接](http://pan.baidu.com/s/1i5DehMX)

说明：
内含所有虚拟机，所有配置都已经执行过，debtest1中/root/tmp/目录下有Atlas和mha的安装包

<br/>
(1) 所有主机的root密码

123

<br/>
(2) 所有数据库的root密码

123

<br/>
(3) 所有数据库的其他账户

mha mha

repl repl

<br/>
2 mha_manager.cnf

<pre>

[server default]
manager_workdir=/root/mha_base
manager_log=/root/mha_base/manager.log
remote_workdir=/root/mha_base
ssh_user=root
ssh_port=22
user=mha
password=mha
repl_user=repl
repl_password=repl
multi_tier_slave=1
ping_interval=1
ping_type=CONNECT
master_ip_failover_script=/root/mha_base/master_ip_failover
master_ip_online_change_script=/root/mha_base/master_ip_online_change
secondary_check_script=/usr/bin/masterha_secondary_check -s 192.168.56.13 -s 192.168.56.15 -s 192.168.56.16  --user=root --port=22 --master_host=debtest2 --master_ip=192.168.56.14 --master_port=3306

[server1]
candidate_master=0
ignore_fail=1
check_repl_delay = 1
hostname=debtest2
ip=192.168.56.14
port=3306
ssh_port=22
master_binlog_dir=/var/log/mysql/

[server2]
candidate_master=0
ignore_fail=1
check_repl_delay = 1
hostname=debtest3
ip=192.168.56.15
port=3306
ssh_port=22
master_binlog_dir=/var/log/mysql/

[server3]
candidate_master=0
ignore_fail=1
check_repl_delay = 1
hostname=debtest4
ip=192.168.56.16
port=3306
ssh_port=22
master_binlog_dir=/var/log/mysql/
</pre>

3 master_ip_online_change 和 master_ip_failover 两个文件一模一样
<pre>
#!/usr/bin/env perl
## Note: This is a sample script and is not complete. Modify the script based on your environment.
use strict;
use warnings FATAL => 'all';
use Getopt::Long;
use MHA::DBHelper;
my (
  $command, $ssh_user, $orig_master_host,
  $orig_master_ip, $orig_master_port, $new_master_host,
  $new_master_ip, $new_master_port
);
my $vip = '192.168.56.19/24'; #virtual ip
my $ssh_start_vip = "ip addr add $vip dev eth0";
my $ssh_stop_vip = "ip addr del $vip dev eth0";
GetOptions(
  'command=s' => \$command,
  'ssh_user=s' => \$ssh_user,
  'orig_master_host=s' => \$orig_master_host,
  'orig_master_ip=s' => \$orig_master_ip,
  'orig_master_port=i' => \$orig_master_port,
  'new_master_host=s' => \$new_master_host,
  'new_master_ip=s' => \$new_master_ip,
  'new_master_port=i' => \$new_master_port,
  );
exit &main();
sub main {
  if ( $command eq "stop" || $command eq "stopssh" ) {
    # $orig_master_host, $orig_master_ip, $orig_master_port are passed.
    # If you manage master ip address at global catalog database,
    # invalidate orig_master_ip here.
    my $exit_code = 1;
    eval {
      print "Disabling the VIP on old master: $orig_master_host \n";
          &stop_vip();
      $exit_code = 0;
          # updating global catalog, etc
    };
    if ($@) {
      warn "Got Error: $@\n";
      exit $exit_code;
    }
    exit $exit_code;
  }
  elsif ( $command eq "start" ) {
    # all arguments are passed.
    # If you manage master ip address at global catalog database,
    # activate new_master_ip here.
    # You can also grant write access (create user, set read_only=0, etc) here.
    my $exit_code = 10;
    eval {
      print "Enabling the VIP - $vip on old master: $new_master_host \n";
          &start_vip();
      $exit_code = 0;
    };
    if ($@) {
      warn $@;
      # If you want to continue failover, exit 10.
      exit $exit_code;
    }
    exit $exit_code;
  }
  elsif ( $command eq "status" ) {
    # do nothing
    exit 0;
  }
  else {
    &usage();
    exit 1;
  }
}
# Enable the VIP on the new_master
sub start_vip() {
    `ssh $ssh_user\@$new_master_host \" $ssh_start_vip \"`;
}
# Disable the VIP on the old_master
sub stop_vip() {
my $ssh_user = "root";
    `ssh $ssh_user\@$orig_master_host \" $ssh_stop_vip \"`;
}
sub usage {
  print
"Usage: master_ip_failover --command=start|stop|stopssh|status --orig_master_host=host --orig_master_ip=ip --orig_master_port=port --new_master_host=host --new_master_ip=ip --new_master_port=port\n";
}
</pre>




