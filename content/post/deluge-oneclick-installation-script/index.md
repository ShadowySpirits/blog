---
title: Deluge 一键安装脚本
slug: deluge-oneclick-installation-script
date: 2019-08-23T10:26:00+08:00
image: deluge-logo.jpg
tags:
  - BT/PT
categories:
  - tech
  - share
---

此脚本适用于 Ubuntu，自动安装 deluged 和 deluge-webui

 <!--more-->

## 为什么需要此脚本

1.  自动安装 deluge 最新稳定版
2.  创建 deluge 用户以运行 deluge 程序
3.  创建 deluged 和 deluge-web 服务，开机自启动

## 使用方法

GitHub 仓库：[ShadowySpirits/vps-setup](https://github.com/ShadowySpirits/vps-setup)

运行此行代码即可

```bash
curl https://raw.githubusercontent.com/ShadowySpirits/vps-setup/master/deluge_install.sh | sudo bash
```

使用以下命令开开启、关闭或重启服务

```bash
sudo service deluged start/stop/restart
sudo service deluge-web start/stop/restart
```

然后访问 `http://your-ip:8112` 就可以进入 Deluge WebUI

输入默认密码 deluge 后，点击 connect 就可以连接上服务器，开始使用了
