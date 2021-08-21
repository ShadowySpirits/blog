---
title: 原生安卓 WiFi 4G 信号去叹号去叉教程
slug: native-android-wifi-4g-signal-exclamation-defork-tutorial
date: 2018-10-30T16:13:00+08:00
tags:
  - Android
categories:
  - share
---

适用于 7.1.2+ :

此版本服务器地址判断逻辑相比 7.1.1 没有更改，但是检测的开关却变了。

检测开关：

```bash

删除变量：（删除以后默认启用）

adb shell settings delete global captive_portal_mode

关闭检测：

adb shell settings put global captive_portal_mode 0

查看当前状态：

adb shell settings get global captive_portal_mode

```

服务器地址相关（同 7.1.1）：

```bash

删除（删除默认用HTTPS）

adb shell settings delete global captive_portal_https_url

adb shell settings delete global captive_portal_http_url

分别修改两个地址

adb shell settings put global captive_portal_http_url http://captive.v2ex.co/generate_204

adb shell settings put global captive_portal_https_url https://captive.v2ex.co/generate_204

```

> 转载自 [原生安卓 WiFi 信号去叹号去叉教程 5.0-Android P](https://www.evil42.com/index.php/archives/17/)，侵删
