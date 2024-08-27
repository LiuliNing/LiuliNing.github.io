# Ubuntu1804研发环境搭建


<!--more-->


```
Author:JDZT.LLN
CreateDate:2020-12-05
UpdateDate:2021-03-21
```

# 一：服务器环境安装

```
系统环境:ubuntu-18.04.5-live-server-amd64
安装过程:参考链接：https://blog.csdn.net/github_38336924/article/details/82427252
Xshell连接设置:参考链接：https://blog.csdn.net/sunset108/article/details/40818815
```

## JDK安装

```
卸载服务器自带openJDK,安装当前项目开发使用版本，当前为	JDK1.8
通过xftp将下载的jdk文件(注意jdk版本要和系统位数一致，64位系统使用64位的包)，放入服务器中指定文件夹
如果上传失败，检查文件夹权限，并修改权限
上传成功后，进行文件解压缩
配置环境变量文件
进入配置文件:vim /etc/profile
文件尾部添加:
export JAVA_HOME=/java/jdk8  (jdk安装路径)
export CLASSPATH=.:$CLASSPATH:$JAVA_HOME/lib/dt.jar:$JAVA_HOME/lib/tools.jar
export PATH=$PATH:$JAVA_HOME/bin
export JRE_HOME=$JAVA_HOME/jre
刷新文件:
	source /etc/profile
测试是否安装完成
	java -version
```

## 安装mysql

```
血泪教训：ubuntu默认安装 mysql5 !!!  一定要注意版本
```

```
安装参考链接：
https://blog.csdn.net/wm609972715/article/details/83759266
ubuntu安装-卸载mysql8
https://www.cnblogs.com/zhangxuel1ang/p/13456116.html	https://www.cnblogs.com/yxym2016/p/12669532.html
修改可远程访问,允许mysql远程连接：参考链接：
https://blog.csdn.net/winterking3/article/details/86080434
https://www.cnblogs.com/devjiajun/articles/9635464.html
https://blog.csdn.net/weixx3/article/details/80782479
https://www.cnblogs.com/wzwyc/p/10121409.html
```

```
安装后操作
1.注释bind-address = 127.0.0.1。
代码如下:
sudo vim /etc/mysql/mysql.conf.d/mysqld.cnf
将bind-address = 127.0.0.1注释掉（即在行首加#）
除了注视掉这句话之外，还可以把后面的IP地址修改成允许连接的IP地址。
但是，如果只是开发用的数据库，为了方便起见，还是推荐直接注释掉。
从上面的注释中，可以看出，旧版本的MySQL（从一些资料上显示是5.0及其以前的版本）上使用的是skip-networking。所以，善意提醒一下，使用旧版本的小伙伴请注意一下。
2.登录进数据库：
mysql -u root -p
切换到数据库mysql。SQL如下：
use mysql;
3.增加允许远程访问的用户或者允许现有用户的远程访问。
删除匿名用户后，给root授予在任意主机（%）访问任意数据库的所有权限。SQL语句如下：
grant all privileges on *.* to 'root'@'%' identified by '@Lisijiang1994321ZaQ!' with grant option;
grant all privileges on *.* to 'root'@'%' identified by '123456' with grant option;
@Lisijiang1994321ZaQ!
update user set host=% where user=root;
flush privileges;
4.退出数据库
exit
5.重启数据库
sudo service mysql restart
```

```
create user 'root'@'%' identified by '@Lisijiang1994321ZaQ!';
```

卸载mysql

```
参考：https://www.cnblogs.com/pighui/p/10422927.html
1.卸载mysql 以下命令分开执行
sudo apt-get autoremove --purge mysql-server
sudo apt-get autoremove --purge mysql-server-*
sudo apt-get autoremove --purge mysql-client
sudo apt-get autoremove --purge mysql-client-*
sudo apt-get remove mysql-common
2.删除数据
dpkg -l |grep ^rc|awk '{print $2}' |sudo xargs dpkg -P 
注意，清楚的过程中会弹出几个窗口，内容大概是问你是否需要清除用户数据之类的，要选择yes！
3.删除目录
sudo rm -rf /etc/mysql
sudo rm -rf /var/lib/mysql
4.清除残留
sudo apt autoremove
sudo apt autoreclean
```

## maven

```
下载maven最新jar.gz压缩包，解压到指定路径
分别在系统配置文件中进行环境变量的配置
vim /etc/profile  
vim ~/.bashrc
添加配置信息,MAVEN_HOME 为解压后的maven路径
MAVEN_HOME=/www/maven3.6.3  
PATH=${MAVEN_HOME}/bin:${PATH}
```



## jenkins

```
安装流程官方说明：
https://pkg.jenkins.io/debian-stable/
# 安装命令
  wget -q -O - https://pkg.jenkins.io/debian-stable/jenkins.io.key | sudo apt-key add -
  deb https://pkg.jenkins.io/debian-stable binary/
  sudo apt-get update
  sudo apt-get install jenkins
启动
service jenkins start
systemctl restart jenkins
重启
service jenkins restart
停止
service jenkins stop
安装后配置密码
sudo cat /var/lib/jenkins/secrets/initialAdminPassword
修改端口
vim /etc/sysconfig/jenkins
或vim /etc/default/jenkins
修改JENKINS_PORT="9090"
查看日志
vim /var/log/jenkins/jenkins.log

```

## Nginx：

```
ps aux | grep nginx
nginx -s stop 立即停止nginx
nginx -s quit 这种方法较stop相比就比较温和一些了，需要进程完成当前工作后再停止
killall nginx 这种方法是比较野蛮的，在上面使用没有效果时，我们用这种方法还是比较好的
systemctl stop nginx.service
systemctl restart nginx.service
nginx -s reload 重新载入配置文件
```

## Redis

```
 安装
sudo apt-get install redis-server
进入
 cd /etc/init.d/
//直接启动
./redis-server  
//后台配置文件启动
./redis-server redis.conf  
启动、停止redis
/etc/init.d/redis-server stop
/etc/init.d/redis-server start
卸载
sudo apt-get purge --auto-remove redis-server

```

## 安装禅道项目管理

```
参考官方文档
https://www.zentao.net/book/zentaopmshelp/40.html
下载官方linux一键安装包，解压至 /opt

#修改默认端口，防止和已安装服务冲突
/opt/zbox/zbox -ap 10024 -mp 3307
#启动
sh /opt/zbox/zbox start
#停止
sh /opt/zbox/zbox stop

#配置自启动
https://blog.csdn.net/lizhe_dashuju/article/details/56484733

1. 在/etc/init.d目录下创建chandao文件
内容如下：
#!/bin/bash
/opt/zbox/zbox restart

然后增加全选
chmod 755 chandao

2. 运行runlevel命令，查看现在的run level是多少，

Linux系统有7个运行级别(runlevel)
运行级别0：系统停机状态，系统默认运行级别不能设为0，否则不能正常启动
运行级别1：单用户工作状态，root权限，用于系统维护，禁止远程登陆
运行级别2：多用户状态(没有NFS)
运行级别3：完全的多用户状态(有NFS)，登陆后进入控制台命令行模式
运行级别4：系统未使用，保留
运行级别5：X11控制台，登陆后进入图形GUI模式
运行级别6：系统正常关闭并重启，默认运行级别不能设为6，否则不能正常启动

eg： 显示  N 5

3. 既然是5,就在/etc/rc5.d目录下，创建一个链接
ln -s /etc/init.d/chandao S99chandao

4. 重启服务器，检查禅到是否开启
检查浏览器地址是否能打开禅道
```



# 二：出现的异常及解决方案

## 1.系统配置

#### 1.1服务器ip冲突，centos7修改静态ip

```
修网络配置文件
	vim /etc/sysconfig/network-scripts/ifcfg-ens33
        BOOTPROTO="static" # 使用静态IP地址，默认为dhcp IPADDR="19.37.33.66" # 设置的静态IP地址
        NETMASK="255.255.255.0" # 子网掩码 
        GATEWAY="19.37.33.1" # 网关地址 
        DNS1="192.168.241.2" # DNS服务器（此设置没有用到，所以我的里面没有添加）
        ONBOOT=yes  #设置网卡启动方式为 开机启动 并且可以通过系统服务管理器 systemctl 控制网卡
修改映射文件
	NETWORKING=yes
	GATEWAY=19.37.33.1  # 网关地址 同上
重启网络服务
	service network restart

```

#### 1.2无法挂载外部硬盘

```
mount: unknown filesystem type ‘ntfs’
这是由于CentOS release 5.5(Final)上无法识别NTFS格式的分区（就是用U盘或者移动硬盘挂载，系统无法识别）

解放方案：
	安装ntfs-3g插件
使用插件命令
此时依然报错
    Mount is denied because the NTFS volume is already exclusively opened.
    The volume may be already mounted, or another software may use it which
    could be identified for example by the help of the 'fuser' command.
	大意为sdb1挂载的盘上有正在运行的任务，所以需要先停止再进行挂载操作
fuser -m -u /dev/sdb2
杀死查询出来的进程，再次执行挂载
mount -t ntfs-3g /dev/sdb2 /opt/



```

#### 1.3新装系统无法ssh访问，无法ping通内部网络

```
解放方法：
安装openssh-client工具和openssh-server工具,并在系统ssh配置中添加允许远程访问
参考方案：
https://jingyan.baidu.com/article/60ccbceb1fae7f25cbb19767.html
//无法连接ssh
https://blog.csdn.net/qq_23730693/article/details/107732884

sudo apt-get install openssh-server
#如果安装失败，先卸载openssh-client，再安装
sudo apt-get purge openssh-client
```



## 2.MySQl

#### mysql8运行group by异常  （5升级8后 语法模式调整，需要手动改sqlmode）

```
mysql8 无法使用group by 语句
异常信息：this is incompatible with sql_mode=only_full_group_by
修改 配置文件中的 sql_mode

vim /etc/my.cnf 
[mysqld]
sql_mode=STRICT_TRANS_TABLES,NO_ZERO_IN_DATE,NO_ZERO_DATE,ERROR_FOR_DIVISION_BY_ZERO,NO_ENGINE_SUBSTITUTION

```

ubuntu启动mysql报错

```

Job for mysql.service failed because the control process exited with error code.
See "systemctl status mysql.service" and "journalctl -xe" for details.

```

mysql设置大小写忽略、设置端口

```
vim /etc/mysql/mysql.conf.d/mysqld.cnf
在[mysqld]标签下添加
#默认0  1代表忽略
lower_case_table_names=1   
port            = 11001
```

```
大小写忽略需要在数据库初始化时配置，如果在已经使用数据库后配置，将报错。
需要先备份数据库，然后清理 /var/lib/mysql/中的内容，重新设置数据库相关配置信息。
```

## 3 挂载在snap的/dev/loop占用100%

```
https://www.cnblogs.com/raina/p/12613164.html

运行 df 命令时添加选项，不显示它就好了：
df -x squashfs -h

如果嫌弃每次输选项麻烦，可以在 "~/.bashrc" 文件里起别名：

echo "alias df='df -x squashfs -x tmpfs -x devtmpfs'" >> ~/.bashrc

source ~/.bashrc

```



# 三：命令

## 查看已安装的jdk 	

```
rpm -qa|grep jdk
```

## 安装rpm

```
apt install rpm
```

## 重启jenkins

```
systemctl restart jenkins                      
```

## apt命令无效
	apt-get update
	apt-get upgrade

## 查看文件大小

```
//数字为深度
du -h --max-depth=2  
//当前路径下文件夹大小
du -s -h ./*
```

## 压缩

```
tar -cvf filename.tar dir  #将目录dir中压缩到filename.tar中，保留原文件
tar -zxvf filename.tar.gz #解压到当前目录，保留原文件
tar -zxvf filename.tar.gz -C dir #解压到dir目录，保留原文件
解压zip文件
unzip 文件名
```

## 文件操作

```
删除文件夹下某个文件之外（1.txt）的其他所有文件
ls | grep -v "1.txt" | xargs rm -rf
删除文件夹中的所有文件，不删除文件夹
ls | xargs rm -rf
```



## 端口

```
端口使用情况
netstat -ntap | grep 8899
端口是否开放
firewall-cmd --list-ports
开放tcp端口
firewall-cmd --permanent --zone=public --add-port=8899/tcp
停止指定端口的服务
sudo kill -9 \$(netstat -nlp | grep :6003 | awk '{print \$7}' | awk -F"/" '{ print \$1 }')
```

## 防火墙

```
关闭防火墙：
ufw disable
查看防火墙状态：
ufw status
开启防火墙
ufw enable
开启端口
ufw allow 9000

```

## 磁盘

```
parted -l
fdisk -l
查看磁盘空间大小
df -h
查看当前文件夹所有文件大小
du -sh
查看指定文件夹大小
du -h /data
查看指定文件夹下所有文件的大小
du -h /data/
查看指定文件大小
du -h data.log查看目录挂载点df /data加上-kh以g单位显示df /data -kh
```

##  Tomcat

```
linux下启动tomcat服务
 Linux下tomcat服务的启动、关闭与错误跟踪，使用PuTTy远程连接到服务器以后，通常通过以下几种方式启动关闭tomcat服务：
切换到tomcat主目录下的bin目录（cd usr/local/tomcat/bin）
1，启动tomcat服务
方式一：直接启动 ./startup.sh
方式二：作为服务启动 nohup ./startup.sh &
方式三：控制台动态输出方式启动 ./catalina.sh run 动态地显示tomcat后台的控制台输出信息,Ctrl+C后退出并关闭服务
解释：
通过方式一、方式三启动的tomcat有个弊端，当客户端连接断开的时候，tomcat服务也会立即停止，通过方式二可以作为linux服务一直运行
通过方式一、方式二方式启动的tomcat，其日志会写到相应的日志文件中，而不能动态地查看tomcat控制台的输出信息与错误情况，通过方式三可以以控制台模式启动tomcat服务，
直接看到程序运行时后台的控制台输出信息，不必每次都要很麻烦的打开catalina.out日志文件进行查看，这样便于跟踪查阅后台输出信息。tomcat控制台信息包括log4j和System.out.println()等输出的信息。
2，关闭tomcat服务
./shutdown.sh
```

## Mysql

```
查看数据库运行状态
sudo service mysql status  
启动数据库服务
sudo service mysql start  
停止数据库服务
sudo service mysql stop 
重启数据库服务
sudo service mysql restart 

数据文件夹
cd /var/lib/mysql
```

## 系统时间

```
#设置时间同步网络
cd /usr/share/zoneinfo
tzselect
#选择时区，亚洲4，中国9，北京1
sudo timedatectl set-timezone Asia/Shanghai
sudo ntpdate -u ntp.ntsc.ac.cn
#没有ntpdate命令的话, 先安装一下, 执行
sudo apt install ntpdate
date

timedatectl status查看当前时间状态
sudo date -s MM/DD/YY //修改日期
sudo date -s hh:mm:ss //修改时间
sudo hwclock --systohc //修改生效
```

## SCP移动文件

```
scp /test.txt root@192.168.1.102:~/test.txt
```

## FDisk 挂载硬盘

本操作以该场景为例：当云服务器挂载了一块新的数据盘时，使用fdisk分区工具将该数据盘设为主分区，分区形式默认设置为MBR，文件系统设为ext4格式，挂载在“/mnt/sdc”下，并设置开机启动自动挂载。

执行以下命令，查看新增数据盘。

```
fdisk -l
```

回显类似如下信息：

```
[root@ecs-test-0001 ~]# fdisk -l

Disk /dev/vda: 42.9 GB, 42949672960 bytes, 83886080 sectors
Units = sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disk label type: dos
Disk identifier: 0x000bcb4e

   Device Boot      Start         End      Blocks   Id  System
/dev/vda1   *        2048    83886079    41942016   83  Linux

Disk /dev/vdb: 107.4 GB, 107374182400 bytes, 209715200 sectors
Units = sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
```

表示当前的云服务器有两块磁盘，**“/dev/vda”**是系统盘，**“/dev/vdb”**是新增数据盘。

执行以下命令，进入fdisk分区工具，开始对新增数据盘执行分区操作。

```
fdisk 新增数据盘
以新挂载的数据盘“/dev/vdb”为例：
fdisk /dev/vdb
```

回显类似如下信息：

```
[root@ecs-test-0001 ~]# fdisk /dev/vdb
Welcome to fdisk (util-linux 2.23.2).

Changes will remain in memory only, until you decide to write them.
Be careful before using the write command.

Device does not contain a recognized partition table
Building a new DOS disklabel with disk identifier 0x38717fc1.

Command (m for help): 
```

输入“n”，按“Enter”，开始新建分区。

回显类似如下信息：

```
Command (m for help): n
Partition type:
   p   primary (0 primary, 0 extended, 4 free)
   e   extended

表示磁盘有两种分区类型：

- “p”表示主分区。
- “e”表示扩展分区。

说明：

磁盘使用MBR分区形式，最多可以创建4个主分区，或者3个主分区加1个扩展分区，扩展分区不可以直接使用，需要划分成若干个逻辑分区才可以使用。

磁盘使用GPT分区形式时，没有主分区、扩展分区以及逻辑分区之分。
```

以创建一个主要分区为例，输入“p”，按“Enter”，开始创建一个主分区。

回显类似如下信息：

```
Select (default p): p
Partition number (1-4, default 1): 
```

**“Partition number”**表示主分区编号，可以选择1-4。

以分区编号选择“1”为例，输入主分区编号“1”，按“Enter”。

回显类似如下信息：

```
Partition number (1-4, default 1): 1
First sector (2048-209715199, default 2048):
```

**“First sector”**表示起始磁柱值，可以选择2048-209715199，默认为2048。

以选择默认起始磁柱值2048为例，按“Enter”。

系统会自动提示分区可用空间的起始磁柱值和截止磁柱值，可以在该区间内自定义，或者使用默认值。起始磁柱值必须小于分区的截止磁柱值。

回显类似如下信息：

```
First sector (2048-209715199, default 2048):
Using default value 2048
Last sector, +sectors or +size{K,M,G} (2048-209715199, default 209715199):+500G
```

**“Last sector”**表示截止磁柱值，可以选择2048-209715199，默认为209715199。

选择增加500G的磁盘分区 ，输入+500G ，按“Enter”。

系统会自动提示分区可用空间的起始磁柱值和截止磁柱值，可以在该区间内自定义，或者使用默认值。起始磁柱值必须小于分区的截止磁柱值。

回显类似如下信息：

```
Last sector, +sectors or +size{K,M,G} (2048-209715199, default 209715199):
Using default value 209715199
Partition 1 of type Linux and of size 100 GiB is set

Command (m for help):
```

表示分区完成，即为数据盘新建了1个分区。输入“p”，按“Enter”，查看新建分区的详细信息。

回显类似如下信息：

```
Command (m for help): p

Disk /dev/vdb: 107.4 GB, 107374182400 bytes, 209715200 sectors
Units = sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disk label type: dos
Disk identifier: 0x38717fc1

   Device Boot      Start         End      Blocks   Id  System
/dev/vdb1            2048   209715199   104856576   83  Linux

Command (m for help):
```

表示新建分区**“/dev/vdb1”**的详细信息。输入“w”，按“Enter”，将分区结果写入分区表中。

回显类似如下信息：

```
Command (m for help): w
The partition table has been altered!

Calling ioctl() to re-read partition table.
Syncing disks.
```

表示分区创建完成。

说明：

如果之前分区操作有误，请输入“q”，则会退出fdisk分区工具，之前的分区结果将不会被保留。

执行以下命令，将新的分区表变更同步至操作系统。

```
partprobe
```

执行以下命令，将新建分区文件系统设为系统所需格式。

**mkfs** **-t** *文件系统格式* **/dev/vdb1**

以设置文件系统为“ext4”为例：

**mkfs -t ext4 /dev/vdb1**

回显类似如下信息：

```
[root@ecs-test-0001 ~]# mkfs -t ext4 /dev/vdb1
mke2fs 1.42.9 (28-Dec-2013)
Filesystem label=
OS type: Linux
Block size=4096 (log=2)
Fragment size=4096 (log=2)
Stride=0 blocks, Stripe width=0 blocks
6553600 inodes, 26214144 blocks
1310707 blocks (5.00%) reserved for the super user
First data block=0
Maximum filesystem blocks=2174746624
800 block groups
32768 blocks per group, 32768 fragments per group
8192 inodes per group
Superblock backups stored on blocks:
        32768, 98304, 163840, 229376, 294912, 819200, 884736, 1605632, 2654208,
        4096000, 7962624, 11239424, 20480000, 23887872

Allocating group tables: done
Writing inode tables: done
Creating journal (32768 blocks): done
Writing superblocks and filesystem accounting information: done
```

格式化需要等待一段时间，请观察系统运行状态，不要退出。

须知：

不同文件系统支持的分区大小不同，请根据您的业务需求选择合适的文件系统。

执行以下命令，新建挂载目录。

**mkdir** *挂载目录*

以新建挂载目录“/mnt/sdc”为例：

**mkdir /mnt/sdc**

执行以下命令，将新建分区挂载到创建的目录下。

**mount** *磁盘分区* *挂载目录*

以挂载新建分区“/dev/vdb1”至“/mnt/sdc”为例：

**mount /dev/vdb1 /mnt/sdc**

执行以下命令，查看挂载结果。

**df -TH**

回显类似如下信息：

```
[root@ecs-test-0001 ~]# df -TH
Filesystem     Type      Size  Used Avail Use% Mounted on
/dev/vda1      ext4       43G  1.9G   39G   5% /
devtmpfs       devtmpfs  2.0G     0  2.0G   0% /dev
tmpfs          tmpfs     2.0G     0  2.0G   0% /dev/shm
tmpfs          tmpfs     2.0G  9.1M  2.0G   1% /run
tmpfs          tmpfs     2.0G     0  2.0G   0% /sys/fs/cgroup
tmpfs          tmpfs     398M     0  398M   0% /run/user/0
/dev/vdb1      ext4      106G   63M  101G   1% /mnt/sdc
```

表示新建分区**“/dev/vdb1”**已挂载至“/mnt/sdc”。

```
fdisk 命令说明
  DOS (MBR)
   a   开关 可启动 标志
   b   编辑嵌套的 BSD 磁盘标签
   c   开关 dos 兼容性标志

  常规
   d   删除分区
   F   列出未分区的空闲区
   l   列出已知分区类型
   n   添加新分区
   p   打印分区表
   t   更改分区类型
   v   检查分区表
   i   打印某个分区的相关信息

  杂项
   m   打印此菜单
   u   更改 显示/记录 单位
   x   更多功能(仅限专业人员)

  脚本
   I   从 sfdisk 脚本文件加载磁盘布局
   O   将磁盘布局转储为 sfdisk 脚本文件

  保存并退出
   w   将分区表写入磁盘并退出
   q   退出而不保存更改

  新建空磁盘标签
   g   新建一份 GPT 分区表
   G   新建一份空 GPT (IRIX) 分区表
   o   新建一份的空 DOS 分区表
   s   新建一份空 Sun 分区表

```

```
挂载ntfs硬盘
```

## 

# 四：运维方案

## 定时任务 cron

目的：定时备份数据库binlog文件以及全量文件，定时删除15天前的备份文件

```

cron是被默认安装并启动的。而 ubuntu 下启动，停止与重启cron，均是通过调用/etc/init.d/中的脚本进行
service cron start  	/*启动服务*/
service cron stop 		/*关闭服务*/ 
service cron restart 	/ *重启服务*/
service cron reload		/*重新载入配置*/ 
pgrep cron  		    /* 可以通过以下命令查看cron是否在运行（如果在运行，则会返回一个进程ID）*/
```

```
crontab 命令用于安装、删除或者列出用于驱动cron后台进程的表格。也就是说，用户把需要执行的命令序列放到crontab文件中以获得执行，每个用户都可以有自己的crontab文件。以下是这个命令的一些参数与说明：
    crontab -u /*设定某个用户的cron服务*/ 
    crontab -l /*列出某个用户cron服务的详细内容*/ 
    crontab -r /*删除某个用户的cron服务*/ 
    crontab -e /*编辑某个用户的cron服务*/ 
```

## mysql自动备份

```
/usr/bin/mysqldump -u root -p123456 -B -F -R -x  easysitedb | gzip > /mysql_backup/easysitedb_$(date +%Y%m%d).tar.gz

mysql -uroot -p@Lisijiang1994321ZaQ! -e"flush logs"
chmod 777 /var/lib/mysql/binlog.*



docker exec -it /usr/bin/mysqldump -u root -p数据库密码 -B -F -R -x  plm | gzip > /opt/mysql/mysql_backup/plm$(date +%Y%m%d).tar.gz


docker exec -it mysql mysql -u root -p@Lisijiang1994321ZaQ! -e"flush logs"




0 3 * * 6 /etc/cron/mysql_backup/mysqlautoback.sh
0 23 * * * /etc/cron/mysqlbinlogautoback.sh

```

## 系统日志发送-邮件服务搭建

目的：定时扫描系统日志，将日志文件发送至邮箱

```
思路
1.配置邮件服务器
2.配置扫描服务器状态脚本，并发送邮件，添加为定时任务计划
```

```
邮件服务搭建参考博客
https://blog.csdn.net/mdx20072419/article/details/103901254
https://blog.csdn.net/weixin_39278265/article/details/98593267
```

```
搭建邮件服务
编辑源
	sudo vi  /etc/apt/sources.list
添加deb信息
	deb http://cz.archive.ubuntu.com/ubuntu xenial main universe
更新源
	sudo apt-get update
安装heirloom-mailx
	sudo apt install heirloom-mailx
配置外部SMTP
	vim /etc/s-nail.rc
添加内容：
    set from="623872553@qq.com"  //修改为自己的邮箱
    set smtp="smtps://smtp.qq.com:465" //默认smtp服务地址
    set smtp-auth-user="623872553@qq.com" 修改为自己的邮箱
    set smtp-auth-password="aaaaaaaaaa" //qq邮箱授权码,邮箱设置里需要手动开启，然后发短信确认
    set smtp-auth=login  	
测试发送
    发送文字
    echo "邮件内容" | s-nail  -s "邮件主题" 623872553@qq.com
    发送文件，会变成bin结尾的文件，接收下载之后修改文件后缀名可以恢复使用
	s-nail  -s "邮件主题" 623872553@qq.com  < /www/test.jar		
```

```
#Author:                    jdzt.lln
#Date:                      2021-04-02
#FileName:                  systeminfo.sh
#---------------------------------------
export Start_Color="\e[1;32m"
export End_Color="\e[0m"
Ipaddr=`ifconfig | grep -Eo --color=auto "(\<([1-9]|[1-9][0-9]|[1-9][0-9]{2}|2[0-4][0-9]|25[0-5])\>\.){3}\<([1-9]|[1-9][0-9]|[1-9][0-9]{2}|2[0-4][0-9]|25[0-5])\>"| head -1`
KerneVer=`uname -r`
Cpu_Info=`lscpu | grep "Model name:"|tr -s " " |cut -d: -f2`
Mem_Info=`free -mh|tr -s " "|cut -d" " -f2 |head -2| tail -1`
HD_Info=`lsblk | grep "^sd\{1,\}"| tr -s " "|cut -d" " -f1,4`
echo -e "The System Hostname is : $Start_Color `hostname` $End_Color"
echo -e "The System IP ADDER is : $Start_Color $Ipaddr $End_Color"
echo -e "The System kerneVer is : $Start_Color $KerneVer $End_Color"
echo -e "The System Cpu_Info is :$Start_Color $Cpu_Info $End_Color"
echo -e "The System Mem_Info is : $Start_Color $Mem_Info $End_Color"
echo -e "The System HD_Info  is : $Start_Color \n$HD_Info $End_Color"
disk.sh
unset Start_Color
unset End_Color
unset Ipaddr
unset KerneVer
unset Cpu_Info
unset Mem_Info
unset HD_Info


```


