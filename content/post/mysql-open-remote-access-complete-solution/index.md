---
title: MySQL 开启远程访问完全解决方案
slug: mysql-open-remote-access-complete-solution
date: 2018-09-04T11:02:00+08:00
image: security-groups.jpg
tags:
  - Mysql
categories:
  - tech
---

## 适用环境

- MySQL 5.7
- Ubuntu 16.04

( 适用但不限于以上环境 ）

## 操作步骤

### 一、开启 MySQL 远程访问权限

将 mysql.host 字段的值改为 % 就表示能在任何客户端机器上登录到 MySQL 服务器

```sql
mysql> use mysql;
Database changed

mysql> grant all privileges  on *.* to username @'%' identified by "password";
Query OK, 0 rows affected (0.00 sec)

mysql> flush privileges;
Query OK, 0 rows affected (0.00 sec)

//或者新建一个用户并授权，只要设置 host 为 % 即可
CREATE USER 'username'@'host' IDENTIFIED BY 'password';
GRANT ALL privileges ON databasename.tablename TO 'username'@'host';
flush privileges;//如果授权未生效手动重启 mysql 服务即可
```

修改密码的方法：

```sql
mysql> SET PASSWORD FOR 'username'@'host' = PASSWORD('newpass');
```

### 二、 检查 MySQL 配置

查看 MySQL 服务绑定的地址：

```sql
ubuntu:~$ netstat -ano | grep 3306
tcp        0      0 0.0.0.0:3306            0.0.0.0:*               LISTEN      off (0.00/0/0)
```

如果不是如上所示（绑定在 0.0.0.0:3306），那么就需要更改 `/etc/my.cnf` 中的配置，修改下面一行

`bind-address = 0.0.0.0`

### 三、 检查防火墙配置

1.  在 `控制面板\系统和安全\Windows Defender 防火墙` 中关掉 Windows 防火墙

2.  检查服务器上的防火墙

    ```shell
    ubuntu:~$ sudo iptables -L -n
    Chain INPUT (policy ACCEPT)
    target     prot opt source               destination
    ACCEPT     all  --  0.0.0.0/0            0.0.0.0/0
    ACCEPT     all  --  0.0.0.0/0            0.0.0.0/0            state RELATED,ESTABLISHED
    ACCEPT     tcp  --  0.0.0.0/0            0.0.0.0/0            tcp dpt:22
    ACCEPT     tcp  --  0.0.0.0/0            0.0.0.0/0            tcp dpt:21
    ACCEPT     tcp  --  0.0.0.0/0            0.0.0.0/0            tcp dpt:2111
    ACCEPT     tcp  --  0.0.0.0/0            0.0.0.0/0            tcp dpts:5500:5600
    ACCEPT     tcp  --  0.0.0.0/0            0.0.0.0/0            tcp dpt:80
    ACCEPT     tcp  --  0.0.0.0/0            0.0.0.0/0            tcp dpt:443
    ACCEPT     tcp  --  0.0.0.0/0            0.0.0.0/0            tcp dpt:3306
    ACCEPT     icmp --  0.0.0.0/0            0.0.0.0/0            icmptype 8

    Chain FORWARD (policy ACCEPT)
    target     prot opt source               destination

    Chain OUTPUT (policy ACCEPT)
    target     prot opt source               destination
    ```

    如果 MySQL 监听的端口（默认 3306）被设置成 DROP 则需要改成 ACCEPT

    ```shell
    ubuntu:~$ sudo vim /etc/iptables.rules
    ubuntu:~$ sudo iptables-restore < /etc/iptables.rules
    ```

3.  检查服务商提供的安全组设置，开放对应的端口

![安全组](security-groups.jpg)
