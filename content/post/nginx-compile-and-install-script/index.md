---
title: Nginx 编译安装脚本
slug: nginx-compile-and-install-script
date: 2019-08-23T11:20:00+08:00
image: nginx-logo.jpg
tags:
  - Nginx
categories:
  - 某不存在的技术
---

此脚本适用于 Ubuntu，自动编译安装 nginx

<!-- More -->

## 为什么需要此脚本

1.  安装 Nginx 最新 Mainline version
2.  对 nginx 和 OpenSSL 打补丁

    - 为 nginx 添加 SPDY、FULL HPACK、Dynamic TLS Record 支持
    - 为 nginx 提供选择 TLS 1.3 密码套件的功能
    - 为 openssl 添加等价密码组、chacha 草案支持

3.  使用 jemalloc 优化内存管理
4.  使用 Cloudflare 优化的 zlib，提升性能
5.  添加 brotli 压缩支持
6.  添加 Strict-SNI 支持
7.  添加 Lua 支持
8.  可选项：nginx 流媒体支持（Rmtp、HLS、Dash）、简易的 web 应用防火墙（lua_waf）

## 使用方法

GitHub 仓库：[ShadowySpirits/vps-setup](https://github.com/ShadowySpirits/vps-setup)

运行此行代码即可

```bash
curl https://raw.githubusercontent.com/ShadowySpirits/vps-setup/master/nginx_install.sh | bash
```

运行脚本时添加以下参数开启可选功能：

- --with-vod &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;: 添加 nginx-vod-module（nginx 流媒体支持）
- --enable-mkv &nbsp;&nbsp;: 开启 nginx-vod-module 的 mkv 格式支持 (实验性功能)
- --with-waf &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;: 添加 lua_waf （简易的 web 应用防火墙）

使用以下命令开启、关闭或重启服务

```bash
sudo service nginx start/stop/restart
```

!> 此脚本依赖的代码仓库多位于国外服务器，如安装失败请检查是否是网络问题
