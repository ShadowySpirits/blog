---
title: Deluge 2.0.3 修改 user-agent 和 peer-id 教程
slug: deluge-modify-user-agent-and-peer-id-tutorials
date: 2019-08-23T10:57:00+08:00
image: deluge-logo.jpg
tags:
  - BT/PT
categories:
  - 某不存在的技术
  - 杂七杂八的分享
---

本文提供一种修改 Deluge 的 user-agent 和 peer-id 的方法，用于伪装其他 BT 下载工具，绕过某些限制

PS：附部分 BT 下载工具 user-agent 和 peer-id 列表

 <!--more-->

## 修改 user-agent

打开文件 `/usr/lib/python3/dist-packages/deluge/core/core.py`

!> 如果 core.py 不在上述位置可以使用 find 命令找到 core.py 的位置：sudo find / -path '\*deluge/core/core.py'

修改 123 行左右：

```diff
# Start the libtorrent session.
- user_agent = 'Deluge/{} libtorrent/{}'.format(DELUGE_VER, LT_VERSION)
+ user_agent =  'Transmission/2.11'
```

## 修改 peer-id

修改 291 行左右：

```diff
peer_id = substitute_chr(peer_id, 6, release_chr)

- return peer_id
+ return '-TR2110-'
```

i> 虽然这是 Deluge 2.0.3 的教程，但 Deluge 其他版本修改方式大同小异，搜索字符串 user_agent 和 peer_id 也能找到关键代码位置

## 部分 BT 下载工具 user-agent 和 peer-id 列表

| name              | user-agent              | peer-id  |
| ----------------- | ----------------------- | -------- |
| utorrentMac 1.6.4 | uTorrentMac/1640(27255) | -UM1640- |
| utorrent 2.2.1    | uTorrent/2210(25110)    | -UT2210- |
| Transmission 2.11 | Transmission/2.11       | -TR2110- |
| Deluge 1.3.5      | Deluge/1350             | -DE1350- |
