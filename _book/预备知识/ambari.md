# Ambari学习笔记

## 通过Ambari安装Hadoop集群操作手册
**注：整个安装过程主要参考博客：https://blog.csdn.net/henianyou/article/details/81509401**

## 一、安装Ambari工具
### 1.1虚拟机局域网的搭建
(1)准备三台虚拟机，所装系统为Centos6.7，网络连接方式设置为NAT模式。
| 主机名 | 内存分配 | 磁盘分配 |
| ------ | -------- | -------- |
| node01 | 2g       | 20g      |
| node02 | 2g       | 20g      |
| node03 | 2g       | 20g      |

(2)修改/etc/hosts文件，添加（三台主机都需要做）
```shell
[root@node01 ~]# vim /etc/hosts
```
10.0.0.245 node01
10.0.0.246 node02
10.0.0.247 node03

(3)配置SSH免密码登陆
* 生成公钥和私钥：
```
ssh-keygen -t rsa 
```
* 将公钥id_rsa.pub拷贝到需要免密登陆的主机中
执行`ssh-copy-id -i `将公钥文件传输给远程的主机，输入远程主机对应的密码。命令如下：
`ssh-copy-id -i /root/.ssh/id_rsa.pub root@192.168.0.104`
* 用ssh登陆测试

(4)关闭selinux

```shell
vim /etc/sysconfig/selinux 
将SELINUX的值设置为disabled即可

\#SELINUX=enforcing
SELINUX=disabled
\# SELINUXTYPE= can take one of these two values:
\#     targeted - Targeted processes are protected,
\#     mls - Multi Level Security protection.
SELINUXTYPE=targeted
```

(5)关闭防火墙

```
基本语法：
service iptables status    （功能描述：查看防火墙状态）
chkconfig iptables –list    （功能描述：查看防火墙开机启动状态）（双横线）
service iptables stop    （功能描述：临时关闭防火墙）
chkconfig iptables off    （功能描述：关闭防火墙开机启动）
chkconfig iptables on    （功能描述：开启防火墙开机启动）
```
(6)安装NTP
```
yum install -y ntp
chkconfig ntpd on
service ntpd start
chkconfig --list ntpd
```
### 1.2虚拟机中JAVA环境的配置
（1）下载JDK包，在本机下载好用xftp传至虚拟机中
包名：jdk-8u221-linux-x64.rpm
（2）在虚拟机中安装
``` rpm -ivh jdk-8u221-linux-x64.rpm```
（3）配置环境变量

``` 
    vi /etc/profile
    JAVA_HOME=/usr/java/default
    source /etc/prifile
```
（4）测试java环境
```java -version```
### 1.3安装Mysql数据库（仅在服务器端安装）
参考链接：https://blog.csdn.net/henianyou/article/details/78657002

```
1.查看CentOS下是否已安装mysql
命令：yum list installed | grep mysql
2.删除已安装mysql
命令：yum -y remove mysql
3.从yum库中的安装mysql 
命令：yum -y install mysql mysql-server mysql-devel 
4.启动服务
命令：service mysqld start
5.进入mysql并修改密码
命令：mysql -u root
mysql > use mysql; 
mysql > update user set password=password(‘lv102114‘) where user=‘root‘;
GRANT ALL PRIVILEGES ON * . * TO ‘root’@’%’ IDENTIFIED BY ‘lv102114’ WITH GRANT OPTION; 
FLUSH PRIVILEGES;
6.创建ambari所需要的数据库及用户
create database ambari character set utf8 ; 
CREATE USER 'ambari'@'%'IDENTIFIED BY 'lv102114';
允许用户远程登陆
GRANT ALL PRIVILEGES ON * . * TO ‘ambari’@’%’ IDENTIFIED BY ‘lv102114’ WITH GRANT OPTION; 
FLUSH PRIVILEGES;
FLUSH PRIVILEGES;

```

### 1.4添加Ambari的yum源 （仅在服务器端安装）
```
cd /etc/yum.repos.d/

wget http://public-repo-1.hortonworks.com/ambari/centos6/2.x/updates/2.6.2.0/ambari.repo
\#查看是否添加成功，出现 epel 则成功
yum repolist
```
### 1.5在线安装Ambari（仅在服务器端安装）
参考链接：https://blog.csdn.net/henianyou/article/details/81509401
（1）安装
```yum -y install ambari-server```
（2）配置
```ambari-server setup```
**注意：若中途退出则需要先ambari-server reset再执行上述命令**
（3）启动
配置完成之后即可启动
```ambari-server start```
注:配置过程中遇到的问题已在word文档中呈现
（4）web页面进行后续配置
地址为:http://服务端ip:配置的端口号(默认8080)

### 1.5 安装httpd服务

参考链接：https://zhidao.baidu.com/question/203344355842402205.html

命令：

```
# yum install httpd
# service httpd start
# chkconfig httpd on
```



## 二、通过Ambari进行hadoop集群搭建
由于主机磁盘空间不足，未制作本地源，故采用在线安装，速度较慢。遇到的问题已在word文件中呈现。
**注:上传rsa密钥时，一定注意是上传服务器端的rsa私钥。**

## 三、通过Ambari进行集群监控与管理