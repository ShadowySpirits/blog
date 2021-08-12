---
title: Nextcloud15 优化心得
slug: nextcloud-15-optimization-experience
description: 最近抽空把 Nextcloud 从 14 升级到 15，这里记录一下升级流程和优化心得
date: 2019-02-17T12:07:16+08:00 
image: nextcloud.jpg
tags:
  - Nextcloud
  - NAS
categories:
  - 某不存在的技术
  - 杂七杂八的分享
---

最近抽空把 Nextcloud 从 14 升级到 15，这里记录一下升级流程和优化心得

## Nextcloud15 功能更新

这里列出几个更新的功能，给观望的朋友们做个参考

- 全文搜索
- 工作流：文档自动转 PDF，自动执行 shell 命令
- 分享功能升级：现在一个文件可以创建多个分享链接、可以分享只读文件
- 协作功能加强
- 2-3x loading 速度提升

## 升级流程

因为 Nextcloud14 无法直接升级到 15，所以我采用全新安装的方式。安装完成后用备份恢复数据。详细流程可以参照文档中的 [Migrating to a different server](https://docs.nextcloud.com/server/15/admin_manual/maintenance/migrating.html#migrating-to-a-different-server)

i> 将旧的 data 文件夹复制过去后需要用 sudo -uwww php occ files:scan --all 扫描更改

## 优化心得

### 美化 URL

这里分享一下我的 nginx 配置（去掉了 URL 中的 index.php 和 remote.php）：

```nginx
server {
    listen 443 ssl;
    ssl_certificate <your certificate>;
    ssl_certificate_key <your certificate key>;
    ssl_session_timeout 5m;
    ssl_protocols TLSv1 TLSv1.1 TLSv1.2;
    ssl_ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE+AES128:RSA+AES128:ECDHE+AES256:RSA+AES256:ECDHE+3DES:RSA+3DES;
    ssl_prefer_server_ciphers on;

    server_name <your server name>;
    root <your nextcloud path>;
    error_log /var/log/nextcloud15_error.log;
    access_log /var/log/nextcloud15_access.log;

    location = /robots.txt {
        allow all;
        log_not_found off;
        access_log off;
    }

    # Deny access to the data and config directories, .ht* files,
    # the README, and the database structure definition
    location ~ ^/(data|config|\.ht|db_structure\.xml|README) {
        deny all;
    }

    # Pretty URLs for WebDAV
    location ~* \/remote\/(?:.*)$ {
        rewrite ^ /remote.php$uri$is_args$args last;
    }

    # Pretty URLs
    location / {

        location ~* ^\/core\/(?=preview) {
            rewrite ^ /index.php$uri$is_args$args last;
        }

        location ~* ^\/core(/.*)?$ {
            break;
        }

        rewrite ^ /index.php last;
    }

    location ~ ^(.+?\.php)(/.*)?$ {
        try_files $1 = 404;
        include fastcgi_params;
        fastcgi_param SCRIPT_FILENAME $document_root$1;
        fastcgi_param PATH_INFO $2;
        fastcgi_param front_controller_active true;
        fastcgi_pass unix:/tmp/php-cgi.sock;
    }

    rewrite /.well-known/carddav /remote.php/dav permanent;
    rewrite /.well-known/caldav /remote.php/dav permanent;

    location ~ \.(?:css|js|woff|svg|gif|png|html|ttf|woff|woff2|ico|jpg|jpeg)$ {
        try_files $uri /index.php$request_uri;
        add_header Cache-Control "public, max-age=2592000";
        access_log off;
    }


}
```

### 开启 OPcache

一般的源比如 `ppa:ondrej/php` 都会默认安装 OPcache（可以用 `php -m | grep Zend\ OPcache` 来检查是否安装）但是一般默认都不会开启 OPcache，需要在 php.ini 中手动开启。官方推荐配置如下：

```ini
zend_extension=opcache.so
opcache.enable=1
opcache.enable_cli=1
opcache.interned_strings_buffer=8
opcache.max_accelerated_files=10000
opcache.memory_consumption=128
opcache.save_comments=1
opcache.revalidate_freq=1
```

### 开启内存缓存

官方文档推荐本地缓存用 APCu，分布式缓存和锁用 Redis，配置文件如下：

```php
// <your-nextcloud-path>/config/config.php

'memcache.local' => '\OC\Memcache\APCu',
'memcache.distributed' => '\OC\Memcache\Redis',
'memcache.locking' => '\OC\Memcache\Redis',
'redis' => [
     'host' => 'redis-host.example.com',
     'port' => 6379,
],
```

使用 APCu 和 Redis 都需要安装相应 PHP 拓展（Redis 还需要安装相应的服务端： `apt install redis-server`）我个人推荐自行编译安装：

1.  下载对应源码：[APCu](https://pecl.php.net/package/APCu)、[Redis](https://pecl.php.net/package/redis)
2.  解压到你的服务器上
3.  进入对应目录并编译安装：

```bash
phpize // 这个命令在 php 的安装路经下的 bin 目录里
./configure --with-php-config=<your-php-bin-dir>/php-config
make && make install
```

4.  在 php.ini 中启用对应拓展：

```ini
extension = redis.so
extension = apcu.so
apc.enabled=1
apc.shm_size=32M
apc.enable_cli=1
```

### 后台任务设置为 cron

在管理员面板基本设置里将后台任务设置成 cron，然后登陆服务器配置 cron：

```bash
sudo -uwww crontab -e // -u 参数为运行 php 的用户

// 在打开的文件中添加：
*/15  *  *  *  * php -f <your-next-cloud-path>/cron.php
```
