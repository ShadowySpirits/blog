---
title: iptables 白名单过滤
slug: iptables-whitelist-filtering
date: 2018-12-05T20:43:00+08:00
tags:
  - Linux
  - iptables
categories:
  - tech
---

## iptables 过滤参数介绍

### -A (append)

向某个规则链尾部添加一条规则，如：

```bash

iptables -A INPUT -p icmp -m icmp --icmp-type 8 -j ACCEPT

```

### -I (Insert) 在规则链的头部插入新的规则

一共有三条规则链：Forward、Input、Output

- Forward：数据包的目的地址不是本机，也就是说，这个包将被转发

- Input：数据包的目的地址是本机

- Output：数据包是由本地系统进程产生的，并通过某个本地端口发送的

### -m (module_name)

使用扩展模块来进行数据包的匹配，本文主要介绍 `-m state`

```bash

iptables -A INPUT -p tcp -m state --state NEW -m tcp --dport 2333 -j ACCEPT

```

在 iptables 上一共有四种状态：NEW、ESTABLISHED、INVALID、RELATED

1. NEW：NEW 说明这个包是我们看到的第一个包。意思就是，这是 conntrack 模块看到的某个连接的第一个包，它即将被匹配了。比如，我们看到一个 SYN 包，是我们所留意的连接的第一个包，就要匹配它。

2. ESTABLISHED： ESTABLISHED 已经注意到两个方向上的数据传输，而且会继续匹配这个连接的包。处于 ESTABLISHED 状态的连接是非常容易理解的。只要发送并接到应答，连接就是 ESTABLISHED 的了。一个连接要从 NEW 变为 ESTABLISHED，只需要接到应答包即可，不管这个包是发往防火墙的，还是要由防火墙转发的。ICMP 的错误和重定向等信息包也被看作是 ESTABLISHED，只要它们是我们所发出的信息的应答。

3. RELATED： RELATED 是个比较麻烦的状态。当一个连接和某个已处于 ESTABLISHED 状态的连接有关系时，就被认为是 RELATED 的了。换句话说，一个连接要想是 RELATED 的，首先要有一个 ESTABLISHED 的连接。这个 ESTABLISHED 连接再产生一个主连接之外的连接，这个新的连接就是 RELATED 的了，当然前提是 conntrack 模块要能理解 RELATED。ftp 是个很好的例子，FTP-data 连接就是和 FTP-control 有关联的，如果没有在 iptables 的策略中配置 RELATED 状态，FTP-data 的连接是无法正确建立的，还有其他的例子，比如，通过 IRC 的 DCC 连接。有了这个状态，ICMP 应答、FTP 传输、DCC 等才能穿过防火墙正常工作。注意，大部分还有一些 UDP 协议都依赖这个机制。这些协议是很复杂的，它们把连接信息放在数据包里，并且要求这些信息能被正确理解。

4. INVALID：INVALID 说明数据包不能被识别属于哪个连接或没有任何状态。有几个原因可以产生这种情况，比如，内存溢出，收到不知属于哪个连接的 ICMP 错误信息。一般地，我们 DROP 这个状态的任何东西，因为防火墙认为这是不安全的东西。

**更详细的解释见[此文](http://os.51cto.com/art/201108/285209.htm)**

### 白名单过滤配置方法

1. 创建白名单

```bash

iptables -N whitelist

iptables -A whitelist -s 127.0.0.1 -j ACCEPT

```

2. 指定规则

```bash

iptables -A INPUT -p tcp -m tcp --dport 3306 -j whitelist

```

3. 在所以规则最后加上

```bash

iptables -A INPUT -j REJECT --reject-with icmp-host-prohibited

```

这条规则表示拒绝未匹配到其他规则的包

其中 --reject-with 可以有如下几种值

icmp-net-unreachable

icmp-host-unreachable

icmp-port-unreachable

icmp-proto-unreachable

icmp-net-prohibited

icmp-host-prohibited or icmp-admin-prohibited

注：

1. 忽略网络问题的情况下 `DROP` 和 `REJECT` 分别对应 `ERR_CONNECTION_TIMEOUT` 和 `ERR_CONNECTION_REFUSED` ，iptables 帮助文档中的解释：This (mean REJECT) is used to send back an error packet in response to the matched packet: otherwise, it is equivalent to DROP

2. 编辑规则的原则：放行的要先放在前面, 禁止的要放在最后
