---
layout: post
title: ubuntu12.04下安装MySQL
category: mysql
---
想在虚拟机的ubuntu中安装MySQL数据库来做测试，发现网上的文章大部分是apt-get方式安装，但是我在ubuntu上安装mysql使用`sudo apt-get install mysql-server`一直没成功，所以只能另辟蹊径，其次在在ubuntu上，有人使用Linux上源码安装，这个貌似比较麻烦，作罢。所以在看了很多文章，都很麻烦，很想找个简单点的，依照MySQL的官方的文档，安装了MySQL5.6。MySQL官网提供的ubuntu12.04版本的MySQL安装包是一个`tar`格式的压缩包，解压出来有12个文件。下面就开始安装。

##通过文件安装MySQL
1.  安装libaio1，`sudo apt-get install libaio1`，可能会失败，多试几次
如果不能上网，在这里下载安装包[http://packages.ubuntu.com/source/precise/libaio][1]

`sudo dpkg -i libaio1_0.3.109-2ubuntu1_i386.deb`

2. 安装mysql-server依赖的其他包

```sql
sudo dpkg -i mysql-common_5.6.23-1ubuntu12.04_i386.deb

sudo dpkg -i mysql-community-server_5.6.23-1ubuntu12.04_i386.deb

```

由于mysql初次安装后，默认不能远程访问，所以先安装MySQL客户端配置能够远程访问

```sql
sudo dpkg -i mysql-community-client_5.6.23-1ubuntu12.04_i386.deb

sudo dpkg -i mysql-client_5.6.23-1ubuntu12.04_i386.deb
```

参考文章[http://blog.csdn.net/mydeman/article/details/5432937][2]开启远程访问

```sql
GRANT ALL PRIVILEGES ON *.* TO username@"%" IDENTIFIED BY "password";

```
其中username和password分别为mysql的用户名和密码

然后修改`/etc/mysql/my.conf`文件，将以下这一行注释掉

```sql
bind-address          = 127.0.0.1  
```

再重启MySQL服务。


##Reference
[如何打开MySQL中root账户的远程登录][2]
[http://dev.mysql.com/doc/refman/5.6/en/linux-installation-debian.html][3]


[1]: http://packages.ubuntu.com/source/precise/libaio
[2]: http://blog.csdn.net/mydeman/article/details/5432937
